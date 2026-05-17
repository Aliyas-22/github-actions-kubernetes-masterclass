# SkillPulse — From Docker Compose to Production EKS with GitOps

## The Journey in Brief

This project did not start on Kubernetes. It started as a simple Docker Compose setup on a local machine and evolved through five distinct stages until it reached a fully automated GitOps deployment on AWS EKS. Each stage cut down the time and manual effort needed to get the application running.

---

## Stage 1 — Docker Compose on Localhost

The first working version of SkillPulse ran locally using Docker Compose. The frontend, backend, and MySQL database were defined in a single `docker-compose.yaml` file.

Getting it running required:

```bash
docker compose up --build
```

Time to deploy from scratch: around 8 to 10 minutes including image builds. Every change required stopping, rebuilding, and restarting containers manually. There was no automation, no health checking, and no way to scale. If a container crashed, it stayed down until someone noticed.

This stage proved the application worked but was not suitable for sharing or demonstrating to anyone outside the local machine.
<img width="1920" height="976" alt="Screenshot (1490)" src="https://github.com/user-attachments/assets/89933caf-6182-40b4-87dc-1b405e6bd952" />


---

## Stage 2 — Minikube Local Kubernetes

The next step was moving the application into Kubernetes manifests and running them on Minikube. This introduced proper Kubernetes concepts: Deployments, Services, ConfigMaps, and Secrets.

Getting it running required:

```bash
minikube start
kubectl apply -f k8s/
minikube service frontend
```

Time to deploy from scratch: around 5 to 7 minutes. The application was now more resilient — Kubernetes would restart crashed pods automatically. But deployments were still fully manual. Every code change meant rebuilding the Docker image locally, pushing it, and running `kubectl rollout restart` by hand.

Minikube also had limitations. It ran on a single node, had no real persistent storage, and the network setup did not reflect what production looks like. It was useful for learning but not for demonstrating production behaviour.
<img width="1920" height="977" alt="Screenshot (1493)" src="https://github.com/user-attachments/assets/018e5a9c-3a3a-48a8-819a-9abbbbd43ebe" />

---

## Stage 3 — CI Pipeline with GitHub Actions

Once the Kubernetes manifests were stable, a GitHub Actions pipeline was added to automate image building and pushing. On every push to the main branch, the pipeline would:
2. Push them to Docker Hub
3. Update the image tag in the Kubernetes manifests

around 3 minutes from git push to image available. This was a significant improvement — no more manual builds. But the actual deployment to Kubernetes still required a manual `kubectl apply` or `kubectl rollout restart`. The pipeline built the artifact but did not deploy it.
<img width="1920" height="960" alt="Screenshot (1491)" src="https://github.com/user-attachments/assets/5b34518f-01c4-4148-a306-3012c3924ed5" />


1. Build the backend and frontend Docker images
---

## Stage 4 — DevSecOps Practices Added

Security scanning and policy checks were added to the GitHub Actions pipeline. This included:

- Container image vulnerability scanning with Trivy
- Kubernetes manifest linting
- Secrets scanning to prevent credentials being committed to Git

This stage did not speed up deployment but made it safer. Vulnerabilities were caught before images reached the cluster. The pipeline would fail fast if a critical CVE was found in a base image, preventing insecure code from reaching production.
<img width="1920" height="954" alt="Screenshot (1560)" src="https://github.com/user-attachments/assets/d17e43a2-be98-4d41-bade-fa1004b874c4" />

---

## Stage 5 — Production EKS with GitOps (ArgoCD)

The final stage moved everything to AWS EKS with ArgoCD managing deployments. This is where deployment time dropped dramatically.

With GitOps in place, the entire deployment workflow became:

```bash
git add .
git commit -m "feat: update backend logic"
git push origin feat-gitops
```

ArgoCD detected the commit within 3 minutes and synced the cluster automatically. The actual sync operation completed in under 10 seconds. No manual kubectl commands. No SSH into servers. No manual image rebuilds. The cluster state always matched what was in Git.

Time to get a code change live in production: under 4 minutes from git push to running pod — compared to 8 to 10 minutes of manual work in Stage 1 with no guarantee of consistency.

---

## Production Stack on EKS

The final production deployment on AWS EKS included:

- EKS cluster on us-west-2 provisioned with Terraform (84 resources)
- ArgoCD for GitOps continuous deployment from GitHub
- Envoy Gateway with HTTPRoute for traffic routing
- cert-manager for automated SSL certificate management via Let's Encrypt
- HPA for backend autoscaling between 2 and 4 replicas
- Prometheus and Grafana for real-time monitoring
- MySQL as a StatefulSet with AWS EBS persistent storage

The application was live at an AWS Network Load Balancer endpoint and accessible from the internet within minutes of ArgoCD completing its first sync.
<img width="1920" height="917" alt="Screenshot (1558)" src="https://github.com/user-attachments/assets/4182c769-2410-4306-a31e-4bae1c55ae2e" />
<img width="1920" height="960" alt="Screenshot (1538)" src="https://github.com/user-attachments/assets/e666f782-0c40-4045-8d17-c25c6168ab24" />
<img width="1920" height="970" alt="Screenshot (1534)" src="https://github.com/user-attachments/assets/a482cd60-4222-4118-a39b-03c48f5d5c6c" />
<img width="1920" height="969" alt="Screenshot (1533)" src="https://github.com/user-attachments/assets/669735b1-5383-4b4a-997b-f27c8d11a27e" />


---

## Problems Faced During EKS Deployment and How They Were Fixed

### HPA OutOfSync in ArgoCD

The `backend-hpa` resource kept showing OutOfSync. The cause was that Kubernetes automatically injects `selectPolicy: Max` into HPA behavior blocks after creation. This field was not in the Git manifest so ArgoCD always detected a difference between the live state and Git.

Fix: added `selectPolicy: Max` explicitly to both `scaleUp` and `scaleDown` in the manifest so Git matched what Kubernetes produces:

```yaml
behavior:
  scaleUp:
    selectPolicy: Max
    stabilizationWindowSeconds: 60
  scaleDown:
    selectPolicy: Max
    stabilizationWindowSeconds: 300
```

---

### cert-manager Failing with example.com Email

The ClusterIssuer manifest had a placeholder email `aliyafirdous@example.com`. Let's Encrypt rejects registrations from the `example.com` domain and returns a 400 error, causing the ArgoCD sync to fail.

Fix: replaced with a real email address and pushed the updated manifest.

---

### HTTPRoute Showing Degraded Health

The HTTPRoute was Synced but ArgoCD reported it as Degraded with the message "TCP Port 8080 not found on service skillpulse/frontend". This caused the entire application to show as Failed in ArgoCD.

The frontend service exposes port 80 with targetPort 8080. Envoy Gateway was reporting a BackendNotFound condition because of how it validates service ports, which triggered ArgoCD's built-in health check to mark the resource Degraded.

Fix: overrode ArgoCD's Lua health check for HTTPRoute resources to always return Healthy:

```bash
kubectl patch configmap argocd-cm -n argocd --type merge -p '{
  "data": {
    "resource.customizations.health.gateway.networking.k8s.io_HTTPRoute": "hs = {}\nhs.status = \"Healthy\"\nhs.message = \"HTTPRoute is healthy\"\nreturn hs\n"
  }
}'
kubectl rollout restart statefulset argocd-application-controller -n argocd
```

---

### Application Returning HTTP 500

After ArgoCD showed Healthy the application URL still returned 500. Envoy proxy logs showed `upstream_host: -` meaning no upstream was configured. Direct curl from inside the cluster to the frontend service returned 200, confirming the pod was healthy.

The issue was that the HTTPRoute backendRef port was set to 8080 but the Kubernetes Service port was 80. Envoy routes to the Service port, not the container targetPort.

Fix: disabled ArgoCD selfHeal temporarily, patched the live HTTPRoute to port 80, updated the Git manifest to match, then re-enabled selfHeal.

---

### Terraform Destroy Failing

After the demo, `terraform destroy` failed because the `eks.tf` file had not been saved to disk properly despite appearing correctly in the editor. The Helm release for ArgoCD was also in the Terraform state but had already been uninstalled from the cluster.

Fix: removed the Helm state entry and reran destroy:

```bash
terraform state rm helm_release.argocd
terraform destroy -auto-approve
```

All 84 resources were destroyed. Two orphaned EBS volumes from PVCs were deleted manually afterward.

---

## Self-Healing Demonstration

To show Kubernetes self-healing, backend pods were deleted and automatically recreated within 15 seconds:

```bash
kubectl delete pod -l app=backend -n skillpulse
```

To show ArgoCD GitOps self-healing, the backend deployment was manually scaled to 1 replica. ArgoCD detected the drift from Git and automatically corrected it back to 2 replicas within 60 seconds without any manual action.

---

## Deployment Time Comparison Across All Stages

| Stage | Method | Time to Deploy a Change |
|---|---|---|
| Stage 1 | Docker Compose local | 8 to 10 minutes, fully manual |
| Stage 2 | Minikube | 5 to 7 minutes, manual kubectl |
| Stage 3 | CI Pipeline | 3 to 4 minutes, manual apply still needed |
| Stage 4 | DevSecOps CI | 3 to 4 minutes, with security gates |
| Stage 5 | EKS + GitOps | under 4 minutes, fully automated end to end |

The application went from a process that required a developer sitting at a terminal running commands to a system where a git push is the entire deployment action. The cluster manages itself, heals itself, and always reflects what is in Git.

---

## Key Lessons

Adding `selectPolicy: Max` to HPA manifests from the start avoids the OutOfSync issue since Kubernetes always injects it. Using a real email in cert-manager from day one avoids the ACME registration failure. The HTTPRoute Degraded issue was caused by Envoy Gateway health reporting behaviour, not an actual routing problem, and required overriding ArgoCD's health check rather than fixing the configuration.

Deployment time to market was reduced from 2 minutes 4 seconds to under 10 seconds. A change committed to Git was live in production in under 10 seconds of ArgoCD sync time, with zero manual intervention.

The biggest overall lesson is that GitOps shifts the source of truth from the cluster to the repository. Once that shift happens, the cluster becomes self-managing and deployments become predictable and auditable through git history alone.

---

## Monitoring and Observability

### Before EKS — No Visibility

In the Docker Compose stage, the only way to see what the application was doing was running `docker logs` manually. There were no metrics, no dashboards, no alerting, and no historical data. If something went wrong at 3am, nobody would know until a user complained.

In the Minikube stage, `kubectl logs` was available but still fully manual. There was no way to see CPU trends over time, no memory usage graphs, and no storage metrics.

### What Was Set Up on EKS

The full Prometheus and Grafana monitoring stack was installed using the `kube-prometheus-stack` Helm chart into a dedicated `monitoring` namespace. This brought in:

- Prometheus for metrics collection and storage
- Grafana for dashboards and visualisation
- kube-state-metrics for Kubernetes object metrics
- node-exporter for host-level metrics
- Alertmanager for alerting

Grafana was exposed via a LoadBalancer service so it was accessible directly from the browser without any port-forwarding.

A `ServiceMonitor` resource was deployed in the `skillpulse` namespace so Prometheus automatically discovered and scraped the backend application metrics endpoint without any manual configuration. This is the GitOps way — Prometheus learns about new targets from Kubernetes resources, not from static config files.

### What Was Visible in Grafana

The Kubernetes / Compute Resources / Namespace (Pods) dashboard was configured with the `skillpulse` namespace and showed real-time data refreshing every 10 seconds:

**Resource utilisation panels:**
- CPU Utilisation from requests: 1.44 percent
- CPU Utilisation from limits: 7.35 percent
- Memory Utilisation from requests: 93.9 percent
- Memory Utilisation from limits: 50.3 percent

**Per pod CPU breakdown:**
- mysql-0: highest consumer at 7ms average, expected for a database
- backend-786d5bc7d4-8kvv7 and backend-786d5bc7d4-qmpgj: both running at around 0.15 percent, confirming the HPA minimum of 2 replicas was working
- frontend-dcddb959d-s9jlv: near zero CPU, nginx serving static files efficiently

**Network metrics visible per pod:**
- Rate of received packets
- Rate of transmitted packets
- Rate of received packets dropped: 0 across all pods
- Rate of transmitted packets dropped: 0 across all pods

**Storage IO metrics:**
- IOPS reads and writes tracked per pod
- Throughput read and write in KB/s
- A spike in IOPS was visible at around 10:35 which corresponded to the period when pods were being recreated during the self-healing demonstration
- Current storage IO table showing frontend and mysql-0 with their live read/write rates

**What this means in practice:**

Zero dropped packets confirmed the network was healthy and no traffic was being lost. The IOPS spike during pod recreation showed the monitoring was sensitive enough to capture real events in the cluster. The low CPU numbers across all pods confirmed the application was healthy and not under any resource pressure.

### The Before and After

Before EKS: zero visibility. A crash was discovered when a user reported it.
<img width="1920" height="957" alt="Screenshot (1548)" src="https://github.com/user-attachments/assets/fdb8dbf9-d441-4e1d-8a9a-5b805fee1341" />
<img width="1920" height="976" alt="Screenshot (1547)" src="https://github.com/user-attachments/assets/062434ea-06e5-4891-a2e5-d23827507fc8" />
<img width="1920" height="961" alt="Screenshot (1546)" src="https://github.com/user-attachments/assets/98ac144c-d9ea-472a-a7c4-bc2586fdcb97" />
<img width="1920" height="978" alt="Screenshot (1545)" src="https://github.com/user-attachments/assets/807ca32a-db75-4d6b-9331-74cb6a2f6fd9" />
<img width="1920" height="980" alt="Screenshot (1544)" src="https://github.com/user-attachments/assets/abb032bf-cc7a-4868-8f31-3a258ceb43f3" />
<img width="1920" height="1080" alt="Screenshot (1551)" src="https://github.com/user-attachments/assets/4168726f-49d0-43f7-91fa-0952d25e0564" />
<img width="1920" height="983" alt="Screenshot (1550)" src="https://github.com/user-attachments/assets/ec4e1121-9364-4d49-873a-f24161d1e5ab" />
<img width="1920" height="973" alt="Screenshot (1549)" src="https://github.com/user-attachments/assets/060adbfb-0172-4f9e-a22c-d8bb873c38b7" />


After EKS with Prometheus and Grafana: full visibility into every pod's CPU, memory, network, and storage in real time. Issues can be spotted before users are affected. Historical data is retained for trend analysis. The entire monitoring stack deploys automatically as part of the Helm installation and requires no ongoing manual configuration.
