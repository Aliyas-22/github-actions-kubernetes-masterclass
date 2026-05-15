# =========================
# Stage 1: Dependencies
# =========================
FROM golang:1.26-alpine AS deps

WORKDIR /app

RUN apk add --no-cache git

COPY go.mod go.sum ./

RUN go mod download

# =========================
# Stage 2: Builder
# =========================
FROM golang:1.26-alpine AS builder

WORKDIR /app

COPY --from=deps /go/pkg /go/pkg
COPY . .

RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags="-s -w" \
    -trimpath \
    -o skillpulse .

# =========================
# Stage 3: Runtime
# =========================
FROM gcr.io/distroless/static-debian12:nonroot

WORKDIR /app

COPY --from=builder /app/skillpulse .

EXPOSE 8080

USER nonroot:nonroot

ENTRYPOINT ["/app/skillpulse"]