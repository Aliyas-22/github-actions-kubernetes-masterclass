# SkillPulse — EKS Production Deployment Notebook

---

## ArgoCD UI 

```
skillpulse (Application) — Synced / Healthy
  |
  |-- Wave -2 (infrastructure)
  |     |-- Namespace: skillpulse
  |     |-- StorageClass: gp3
  |     |-- GatewayClass: envoy-gateway
  |
  |-- Wave -1 (configuration)
  |     |-- PVC: mysql-pvc (Bound → real EBS volume on AWS)
  |     |-- Secret: skillpulse-db
  |
  |-- Wave 0 (database + networking)
  |     |-- StatefulSet: mysql → Pod (1)
  |     |-- Service: mysql
  |     |-- Service: backend
  |     |-- Service: frontend
  |
  |-- Wave 1 (application)
  |     |-- Deployment: backend → ReplicaSet → Pod (x2)
  |     |-- Deployment: frontend → Pod (x1)
  |     |-- Gateway: skillpulse-gateway → AWS NLB
  |     |-- HTTPRoute: skillpulse-route
  |     |-- ClusterIssuer: letsencrypt-prod
  |
  |-- Wave 2 (scaling + monitoring)
        |-- HPA: backend-hpa (min:2 max:4)
        |-- ServiceMonitor: skillpulse-backend
```

---

## Helm Repo Error First!

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo add jetstack https://charts.jetstack.io
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# run terraform
cd terraform
terraform apply
```

---

## STEP 1 — Connect kubectl to EKS

```bash
aws eks update-kubeconfig \
  --name skillpulse-eks \
  --region us-west-2

kubectl get nodes
```

---

## STEP 2 — Verify ArgoCD Running

```bash
kubectl get pods -n argocd
# all pods must be 1/1 Running

# get ArgoCD URL
export ARGOCD_URL=$(kubectl get svc argocd-server -n argocd \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "ArgoCD: http://$ARGOCD_URL"

# get password
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath='{.data.password}' | base64 -d && echo
# SAVE THIS PASSWORD
```

---

## STEP 3 — Install Gateway API CRDs

```bash
kubectl apply --server-side -f \
  https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml

kubectl get crd gateways.gateway.networking.k8s.io
# must show the CRD with a date
```

---

## STEP 4 — Install Envoy Gateway

```bash
helm install eg oci://docker.io/envoyproxy/gateway-helm \
  --version v1.2.6 \
  -n envoy-gateway-system \
  --create-namespace \
  --skip-crds \
  --wait

# install extension CRDs
helm pull oci://docker.io/envoyproxy/gateway-helm \
  --version v1.2.6 --untar -d /tmp/eg-chart

kubectl apply --server-side \
  -f /tmp/eg-chart/gateway-helm/crds/generated/

kubectl rollout restart deployment envoy-gateway \
  -n envoy-gateway-system

# verify
kubectl get gatewayclass
# envoy-gateway   gateway.envoyproxy.io/...   True

kubectl get pods -n envoy-gateway-system
# envoy-gateway-xxx   1/1 Running
```

---

## STEP 5 — Install cert-manager

```bash
helm install cert-manager jetstack/cert-manager \
  -n cert-manager \
  --create-namespace \
  --set crds.enabled=true \
  --set config.enableGatewayAPI=true \
  --wait

kubectl get pods -n cert-manager
```

---

## STEP 6 — Install Prometheus + Grafana

```bash
helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring \
  --create-namespace \
  --set grafana.adminPassword=admin123 \
  --set grafana.service.type=LoadBalancer \
  --set prometheus.prometheusSpec.retention=3d \
  --wait --timeout 600s

# get Grafana URL (no port-forward needed on EKS!)
export GRAFANA_URL=$(kubectl get svc monitoring-grafana \
  -n monitoring \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "Grafana: http://$GRAFANA_URL"
# login: admin / admin123
```

---

## STEP 7 — Deploy SkillPulse via ArgoCD

```bash
cd ~/github-actions-kubernetes-masterclass

kubectl apply -f argocd/application.yaml

# watch sync status
kubectl get application skillpulse -n argocd -w
# wait for: Synced / Healthy

# watch pods come up in wave order
kubectl get pods -n skillpulse -w
# ORDER: mysql first → backend + frontend → HPA kicks in
```

---

## STEP 8 — Verify EBS Storage

```bash
kubectl get pvc -n skillpulse
# mysql-pvc   Bound   pvc-xxx   5Gi   gp3

# see real EBS volume in AWS
aws ec2 describe-volumes \
  --filters "Name=tag:kubernetes.io/created-for/pvc/name,Values=mysql-pvc" \
  --query "Volumes[*].{ID:VolumeId,Size:Size,State:State}" \
  --output table \
  --region us-west-2
```

---

## STEP 9 — Get App URL + Enable HTTPS

```bash
# NLB 
kubectl get gateway skillpulse-gateway -n skillpulse -w

export GATEWAY_IP=$(kubectl get gateway skillpulse-gateway \
  -n skillpulse \
  -o jsonpath='{.status.addresses[0].value}')

echo "HTTP URL: http://$GATEWAY_IP"
curl http://$GATEWAY_IP   # test it works

# enable HTTPS with nip.io (free DNS, no domain needed)
echo "HTTPS: https://${GATEWAY_IP}.nip.io"

# update gateway hostname
sed -i "s|skillpulse.nip.io|${GATEWAY_IP}.nip.io|" \
  k8s/40-gateway.yaml

# ArgoCD auto-syncs in 3 minutes
# check HTTPS certificate
kubectl get certificate -n skillpulse
# skillpulse-tls   True   ← HTTPS working!

echo "Open: https://${GATEWAY_IP}.nip.io"
```

---

## STEP 10 — Grafana Dashboard

```bash
echo "Open: http://$GRAFANA_URL"
# login: admin / admin123
```

**Steps in browser:**
1. Left sidebar → **Dashboards** → **New** → **Import**
2. **Upload dashboard JSON file** → upload `grafana-dashboard.json`
3. Datasource → **Prometheus** → **Import**
4. You see: Request Rate, Error Rate, p95 Latency, Goroutines, Memory

**Pre-built dashboards:**
1. Dashboards → search `Kubernetes / Compute Resources / Namespace`
2. Namespace = `skillpulse`
3. See live CPU + memory per pod

---

## STEP 11 — Verify Everything

```bash
echo "=== NODES ===" && kubectl get nodes
echo "=== PODS ===" && kubectl get pods -n skillpulse
echo "=== PVC ===" && kubectl get pvc -n skillpulse
echo "=== HPA ===" && kubectl get hpa -n skillpulse
echo "=== GATEWAY ===" && kubectl get gateway -n skillpulse
echo "=== CERTIFICATE ===" && kubectl get certificate -n skillpulse
echo "=== ARGOCD ===" && kubectl get application -n argocd
echo "=== SERVICEMONITOR ===" && kubectl get servicemonitor -n monitoring
echo "=== HEALTH ===" && curl -s http://$GATEWAY_IP/health
echo "=== METRICS ===" && curl -s http://$GATEWAY_IP/metrics | grep skillpulse | head -5
```

---

## Troubleshooting

| Problem | Command |
|---------|---------|
| ArgoCD not syncing | `kubectl describe application skillpulse -n argocd \| tail -20` |
| Pod crashing | `kubectl logs -l app=backend -n skillpulse --tail=50` |
| PVC stuck Pending | `kubectl describe pvc mysql-pvc -n skillpulse` |
| Gateway no address | `kubectl describe gateway skillpulse-gateway -n skillpulse` |
| Prometheus not scraping | `kubectl get endpoints backend -n skillpulse` |
| Certificate not issuing | `kubectl describe certificate skillpulse-tls -n skillpulse` |
| Helm repo error | `helm repo add argo https://argoproj.github.io/argo-helm && helm repo update` |
