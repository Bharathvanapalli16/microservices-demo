# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Online Boutique is a cloud-native microservices demo e-commerce application. It consists of 11 microservices written in Go, Python, Node.js, Java, and C#, all communicating via gRPC. The repo has been customized to use **AWS ECR** for container images instead of Google's default GCR.

## Deployment Architecture

This repo uses a **GitOps workflow via ArgoCD**:
- `argocd-app.yaml` defines an ArgoCD `Application` that watches the `main` branch
- ArgoCD syncs manifests from `kubernetes-manifests/` into the `online-boutique` namespace
- Auto-pruning and self-healing are enabled — manual `kubectl` changes will be reverted

```
kubernetes-manifests/   ← ArgoCD syncs from here (source of truth for deployed state)
kustomize/              ← Alternative Kustomize-based deployment with optional components
helm-chart/             ← Experimental Helm chart
```

**Critical**: The manifests in `kubernetes-manifests/` contain hardcoded ECR image references and are directly deployable (unlike the upstream project's Skaffold-dependent manifests). Use `kubectl apply -k kubernetes-manifests/` or let ArgoCD handle it.

## Service Communication Map

All inter-service calls use **gRPC** (defined in `protos/demo.proto`). HTTP is only at the frontend edge.

```
Browser → Frontend (HTTP:8080)
  ├─ ProductCatalogService (gRPC:3550)
  ├─ CurrencyService (gRPC:7000)
  ├─ CartService (gRPC:7070) → Redis (6379)
  ├─ RecommendationService (gRPC:8080)
  ├─ AdService (gRPC:9555)
  └─ CheckoutService (gRPC:5050)
       ├─ CartService, ProductCatalogService, CurrencyService
       ├─ ShippingService (gRPC:50051)
       ├─ PaymentService (gRPC:50051)
       └─ EmailService (gRPC:5000)
```

## Services Quick Reference

| Service | Language | Dir |
|---|---|---|
| frontend | Go (Gorilla Mux) | `src/frontend/` |
| checkoutservice | Go | `src/checkoutservice/` |
| productcatalogservice | Go | `src/productcatalogservice/` |
| shippingservice | Go | `src/shippingservice/` |
| cartservice | C# | `src/cartservice/` |
| currencyservice | Node.js | `src/currencyservice/` |
| paymentservice | Node.js | `src/paymentservice/` |
| emailservice | Python | `src/emailservice/` |
| recommendationservice | Python | `src/recommendationservice/` |
| adservice | Java (Gradle) | `src/adservice/` |
| loadgenerator | Python (Locust) | `src/loadgenerator/` |

## Build Commands by Language

**Go services** (frontend, checkoutservice, productcatalogservice, shippingservice):
```bash
go mod download
go build -o server .
go test ./...
```

**C# cartservice**:
```bash
dotnet build
dotnet test
```

**Node.js services** (currencyservice, paymentservice):
```bash
npm install --only=production
node server.js
```

**Python services** (emailservice, recommendationservice, loadgenerator):
```bash
pip install -r requirements.txt
python server.py
```

**Java adservice**:
```bash
./gradlew installDist
./gradlew test
```

**Full cluster deployment via Skaffold** (requires a configured registry):
```bash
skaffold run --default-repo=<your-ecr-registry>
skaffold debug   # with debuggers attached
```

**Apply manifests directly** (when images are already built and tagged in manifests):
```bash
kubectl apply -k kubernetes-manifests/
```

## Kustomize Components

The `kustomize/` tree provides optional add-on components that can be layered onto the base:

| Component | Purpose |
|---|---|
| `google-cloud-operations` | Cloud Monitoring/Tracing/Profiler |
| `memorystore` | Replace Redis with Cloud Memorystore |
| `spanner` / `alloydb` | Replace Redis cart with managed DB |
| `network-policies` | Fine-grained Kubernetes NetworkPolicies |
| `service-mesh-istio` | Istio sidecar injection |
| `shopping-assistant` | Gemini-powered AI assistant service |
| `without-loadgenerator` | Exclude load generator from deployment |
| `non-public-frontend` | Remove LoadBalancer from frontend |

Apply a combination:
```bash
kubectl apply -k kustomize/base/
# or with components listed in kustomize/base/kustomization.yaml
```

## Proto / gRPC Changes

If you modify `protos/demo.proto`, regenerate stubs for each language before updating service code. Each service's `Dockerfile` handles codegen during image build, but for local development run the appropriate `protoc` command for the target language.

## ECR Image References

All manifests reference images in the format:
```
<account-id>.dkr.ecr.<region>.amazonaws.com/<service-name>:<tag>
```

When updating images, change the `image:` field in the relevant `kubernetes-manifests/<service>.yaml`. ArgoCD will detect the diff on `main` and roll it out automatically.
