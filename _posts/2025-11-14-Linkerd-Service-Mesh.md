---
title: "Linkerd: A Practical Guide to the Lightest Service Mesh in Production"
date: 2025-11-14
categories:
  - devops
tags:
  - kubernetes
  - linkerd
  - service-mesh
  - security
  - observability
---

Running microservices on Kubernetes solves the deployment problem. It does not solve the networking problem. How does service A know which pod of service B to send a request to? How do you encrypt traffic between services without changing application code? How do you know when service C is slow and which caller is the culprit? These questions are what a service mesh answers — and Linkerd answers them with less overhead than any other option.

This post is a complete practical guide to Linkerd 2.19: what it is, how it works, how to install and configure it, and how to use its core features in production.

---

## What is Linkerd?

Linkerd is an open-source service mesh for Kubernetes, originally built at Buoyant and donated to the CNCF. Unlike Istio — which uses Envoy as its data plane proxy — Linkerd uses a purpose-built Rust micro-proxy called `linkerd2-proxy`. This proxy is extremely small (~10MB), uses very little CPU and memory, and adds single-digit millisecond latency.

The core value proposition: **Linkerd gives you mTLS, load balancing, retries, timeouts, and golden metrics for every service in your cluster — without changing a single line of application code.**

---

## Architecture

Linkerd has three components: the CLI, the control plane, and the data plane.

```
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                        │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Control Plane (linkerd namespace)       │   │
│  │                                                     │   │
│  │  ┌──────────────┐  ┌─────────────┐  ┌───────────┐  │   │
│  │  │  Destination │  │  Identity   │  │  Proxy    │  │   │
│  │  │  Service     │  │  Service    │  │  Injector │  │   │
│  │  │              │  │  (CA)       │  │           │  │   │
│  │  │ service      │  │ issues mTLS │  │ admission │  │   │
│  │  │ discovery +  │  │ certs every │  │ webhook   │  │   │
│  │  │ policy       │  │ 24 hours    │  │           │  │   │
│  │  └──────────────┘  └─────────────┘  └───────────┘  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                 Data Plane                           │   │
│  │                                                     │   │
│  │  ┌─────────────────────┐  ┌─────────────────────┐  │   │
│  │  │   Pod (service-a)   │  │   Pod (service-b)   │  │   │
│  │  │                     │  │                     │  │   │
│  │  │  ┌───────────────┐  │  │  ┌───────────────┐  │  │   │
│  │  │  │  App Container│  │  │  │  App Container│  │  │   │
│  │  │  └───────────────┘  │  │  └───────────────┘  │  │   │
│  │  │  ┌───────────────┐  │  │  ┌───────────────┐  │  │   │
│  │  │  │linkerd2-proxy │◄─┼──┼─►│linkerd2-proxy │  │  │   │
│  │  │  │ (sidecar)     │  │  │  │ (sidecar)     │  │  │   │
│  │  │  └───────────────┘  │  │  └───────────────┘  │  │   │
│  │  └─────────────────────┘  └─────────────────────┘  │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Control Plane

The control plane runs in the `linkerd` namespace and has three components:

**Destination Service** — the brains of the mesh. Every proxy queries it at startup and whenever it needs to route a request. The Destination service returns endpoint lists, load balancing weights, retry budgets, timeout policies, and authorisation rules. It is the single source of truth for all routing decisions.

**Identity Service** — a Certificate Authority built into the control plane. Every proxy receives a short-lived TLS certificate (valid 24 hours) signed by the Identity service. These certificates are used for automatic mutual TLS between all meshed pods. Certificate rotation happens continuously and transparently.

**Proxy Injector** — a Kubernetes admission webhook. When a pod is created in a namespace annotated with `linkerd.io/inject: enabled`, the Proxy Injector automatically adds the `linkerd-proxy` sidecar container and an `linkerd-init` init container that configures iptables rules to intercept all traffic.

### Data Plane

The data plane is the `linkerd2-proxy` running as a sidecar in every meshed pod. It is written in Rust, is roughly 10MB in size, and uses under 10MB of RSS memory at idle. The proxy intercepts all inbound and outbound TCP connections transparently — the application does not know it exists.

When two meshed pods communicate:
1. The outbound proxy on the calling pod queries the Destination service for endpoints
2. It selects the fastest endpoint using the EWMA algorithm (more on this below)
3. It establishes an mTLS connection to the inbound proxy on the receiving pod
4. The inbound proxy validates the certificate and enforces any authorisation policy before passing the request to the application container

---

## Installation

### Prerequisites

You need `kubectl` configured against a running Kubernetes cluster and the Linkerd CLI.

```bash
# Install the Linkerd CLI (Linux/macOS)
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh
export PATH=$PATH:~/.linkerd2/bin

# Verify the version
linkerd version
# Client version: stable-2.19.0
# Server version: unavailable
```

Before installing, run the pre-check to validate your cluster meets all requirements:

```bash
linkerd check --pre
# kubernetes-api
# ✓ can initialize the client
# ✓ can query the Kubernetes API
#
# kubernetes-version
# ✓ is running the minimum Kubernetes API version
#
# pre-kubernetes-setup
# ✓ control plane namespace does not already exist
# ✓ can create non-namespaced resources
# ✓ can create ServiceAccounts
# ✓ can create Services
# ✓ can create Deployments
# ✓ can create CronJobs
# ✓ can create ConfigMaps
# ✓ can create Secrets
# ✓ can read Secrets
# ✓ can read extension-apiserver-authentication configmap
# ✓ can list and get nodes
# Status check results are √
```

### Install via CLI

```bash
linkerd install --crds | kubectl apply -f -
linkerd install | kubectl apply -f -
linkerd check
```

### Install via Helm (recommended for production)

For production, use Helm so the installation is reproducible and version-controlled alongside the rest of your infrastructure.

```bash
helm repo add linkerd https://helm.linkerd.io/stable
helm repo update

# Generate certificates (required for Helm installs)
# Install step CLI: https://smallstep.com/docs/step-cli/installation

# Trust anchor (root CA — keep the key offline in production)
step certificate create root.linkerd.cluster.local ca.crt ca.key \
  --profile root-ca \
  --no-password \
  --insecure

# Issuer certificate (intermediate CA — rotated by Linkerd automatically)
step certificate create identity.linkerd.cluster.local issuer.crt issuer.key \
  --profile intermediate-ca \
  --not-after 8760h \
  --no-password \
  --insecure \
  --ca ca.crt \
  --ca-key ca.key

# Install CRDs
helm install linkerd-crds linkerd/linkerd-crds \
  --namespace linkerd \
  --create-namespace

# Install control plane
helm install linkerd-control-plane linkerd/linkerd-control-plane \
  --namespace linkerd \
  --set-file identityTrustAnchorsPEM=ca.crt \
  --set-file identity.issuer.tls.crtPEM=issuer.crt \
  --set-file identity.issuer.tls.keyPEM=issuer.key
```

Verify everything came up:

```bash
linkerd check
# All checks passed!

kubectl get pods -n linkerd
# NAME                                      READY   STATUS    RESTARTS
# linkerd-destination-6b9f9dc5c-xpqrd      4/4     Running   0
# linkerd-identity-5c7fccc974-ks9jl         2/2     Running   0
# linkerd-proxy-injector-58bb7fbd45-vbbnx   2/2     Running   0
```

### Install the Viz extension

The Viz extension adds the dashboard, Prometheus, and Grafana:

```bash
linkerd viz install | kubectl apply -f -
linkerd viz check

# Open the dashboard in your browser
linkerd viz dashboard &
```

---

## Meshing Your Services

The simplest way to mesh a namespace is to annotate it:

```bash
kubectl annotate namespace production linkerd.io/inject=enabled
```

Any new pod created in that namespace will automatically receive the proxy sidecar. For existing pods, restart the deployments to trigger re-injection:

```bash
kubectl rollout restart deployment -n production
```

To mesh a specific deployment without meshing the whole namespace:

```bash
kubectl get deployment my-service -n production -o yaml \
  | linkerd inject - \
  | kubectl apply -f -
```

Verify meshing:

```bash
linkerd check --proxy -n production

kubectl get pods -n production
# NAME                         READY   STATUS
# my-service-7d9f8b9c4-xkqpl   2/2     Running   ← 2/2 means app + proxy
```

The `2/2` in the READY column confirms the proxy sidecar is running alongside the application.

---

## Feature 1: Automatic mTLS

This is Linkerd's most important feature. Every connection between two meshed pods is automatically encrypted and mutually authenticated using TLS 1.3 with ML-KEM-768 key exchange — with zero configuration.

### How it works

1. On pod startup, the proxy contacts the Identity service and receives a short-lived X.509 certificate (valid 24 hours) bound to the pod's Kubernetes service account
2. When pod A connects to pod B, both proxies present their certificates
3. Each proxy verifies the other's certificate against the shared trust anchor
4. If both certificates are valid, the connection proceeds over mTLS
5. Certificates are rotated automatically before expiry — no restarts needed

### Verifying mTLS

```bash
# Check that a specific connection is using mTLS
linkerd viz tap deployment/my-service -n production

# :authority=my-service.production.svc.cluster.local:80
# src=10.0.0.5:41234 dst=10.0.0.12:8080
# tls=true   ← mTLS confirmed

# Detailed certificate info
linkerd identity -n production deployment/my-service
```

### What about non-meshed clients?

If a non-meshed pod calls a meshed pod, the connection is unencrypted. Linkerd marks these connections as `tls=no identity`. You can enforce mTLS-only using Server authorisation policies:

```yaml
# Deny all unauthenticated (non-meshed) traffic to a service
apiVersion: policy.linkerd.io/v1beta3
kind: Server
metadata:
  name: my-service-server
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: my-service
  port: 8080
  proxyProtocol: HTTP/2
---
apiVersion: policy.linkerd.io/v1beta3
kind: AuthorizationPolicy
metadata:
  name: my-service-authn
  namespace: production
spec:
  targetRef:
    group: policy.linkerd.io
    kind: Server
    name: my-service-server
  requiredAuthenticationRefs:
    - name: meshed-only
      kind: MeshTLSAuthentication
      group: policy.linkerd.io
---
apiVersion: policy.linkerd.io/v1beta3
kind: MeshTLSAuthentication
metadata:
  name: meshed-only
  namespace: production
spec:
  identities:
    - "*.production.serviceaccount.identity.linkerd.cluster.local"
```

---

## Feature 2: Load Balancing with EWMA

Standard Kubernetes load balancing is round-robin at the kube-proxy level, which distributes connections equally but ignores the actual performance of each endpoint. Linkerd replaces this with **EWMA** (Exponentially Weighted Moving Average) load balancing — also known as peak EWMA or "least latency" balancing.

### How EWMA works

Each proxy tracks the response latency of every endpoint it talks to. The EWMA algorithm weights recent measurements more heavily than old ones, so it quickly responds to a pod that becomes slow. When choosing an endpoint for a new request, the proxy picks the one with the lowest EWMA latency — effectively always routing to the fastest available pod.

This is critically important for **gRPC**. gRPC multiplexes many requests over a single persistent HTTP/2 connection. Standard round-robin load balancing at the connection level means that once a connection is established to a pod, all subsequent requests go there regardless of that pod's load. Linkerd's proxy performs request-level load balancing over gRPC, spreading individual RPCs across all available endpoints.

### Kubernetes Services vs. DNS endpoints

Linkerd behaves differently depending on how a service is addressed:

| Addressing | Load balancing |
|-----------|----------------|
| Kubernetes `Service` (ClusterIP) | Full EWMA per endpoint — Linkerd resolves all pods behind the service |
| DNS name (external, headless) | Best-effort — Linkerd balances across the IPs returned by DNS |
| Headless service | Not supported — Linkerd cannot perform endpoint-aware balancing |

---

## Feature 3: Retries and Timeouts

Retries and timeouts in Linkerd are configured using `HTTPRoute` resources (the Kubernetes Gateway API) rather than proprietary annotations.

### Timeouts

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-service-timeouts
  namespace: production
spec:
  parentRefs:
    - name: my-service
      kind: Service
      group: core
      port: 8080
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api
      timeouts:
        request: 5s          # total time from request to response
        backendRequest: 3s   # time allowed per individual backend attempt
      backendRefs:
        - name: my-service
          port: 8080
```

The `request` timeout covers the full round trip including retries. The `backendRequest` timeout applies to each individual attempt. If `backendRequest` is shorter than `request`, Linkerd can make multiple attempts within the overall budget.

### Retries

Retries are safe only for idempotent operations. Linkerd requires you to explicitly opt in per route:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-service-retries
  namespace: production
spec:
  parentRefs:
    - name: my-service
      kind: Service
      group: core
      port: 8080
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api/read   # safe to retry — GET endpoints only
      retry:
        codes:
          - 502
          - 503
          - 504
        attempts: 3
        perTryTimeout: 2s
      backendRefs:
        - name: my-service
          port: 8080
```

**Important**: Retries are processed on the **outbound** side (the calling pod's proxy). They are not compatible with headless services. Never configure retries for non-idempotent operations (POST, PUT, DELETE) unless your service explicitly handles duplicate requests.

---

## Feature 4: Observability — the Golden Metrics

Linkerd automatically collects three metrics for every meshed service, without any instrumentation in application code:

| Metric | What it measures |
|--------|-----------------|
| **Success rate** | Percentage of requests that returned a non-5xx response |
| **Request rate** | Requests per second |
| **Latency (p50/p95/p99)** | Response time percentiles |

These are called the "golden signals" (a subset of Google's four golden signals) and they tell you almost everything you need to know about whether a service is healthy.

### CLI

```bash
# Top-level view of all services in a namespace
linkerd viz stat deployments -n production

# NAME          MESHED   SUCCESS   RPS    LATENCY_P50   LATENCY_P95   LATENCY_P99
# api           3/3      99.8%     42.3   4ms           18ms          95ms
# worker        2/2      100%      8.1    12ms          45ms          120ms
# db-proxy      1/1      99.1%     50.4   2ms           8ms           31ms

# Traffic between specific services
linkerd viz stat trafficsplits -n production

# Live request stream (like tcpdump for HTTP)
linkerd viz tap deployment/api -n production
# req id=0:1 proxy=out src=10.0.0.5:54321 dst=10.0.0.9:8080 ...
# rsp id=0:1 proxy=out src=10.0.0.5:54321 dst=10.0.0.9:8080 :status=200 latency=3418µs

# Per-route metrics (requires HTTPRoute resources)
linkerd viz routes deployment/api -n production
```

### Dashboard

```bash
linkerd viz dashboard
```

The dashboard shows real-time topology of which services are calling which, success rates on every edge of the call graph, and per-pod metrics. It is the fastest way to find which service in a chain is responsible for elevated error rates.

### Prometheus and Grafana

The Viz extension installs Prometheus configured to scrape all proxy metrics. The built-in Prometheus instance has a 6-hour retention window — suitable for real-time visibility but not for historical analysis. For production, export metrics to your existing Prometheus with remote_write, or configure the Viz extension to use an external Prometheus:

```bash
helm install linkerd-viz linkerd/linkerd-viz \
  --namespace linkerd-viz \
  --set prometheus.enabled=false \
  --set prometheusUrl=http://prometheus.monitoring.svc.cluster.local:9090
```

Linkerd ships with pre-built Grafana dashboards covering cluster health, namespace health, deployment health, and individual pod metrics.

---

## Feature 5: Traffic Splitting (Canary Deployments)

Linkerd supports weighted traffic splitting between service versions, which is the foundation of canary deployments. Combined with Linkerd's automatic success rate metrics, you can do fully automated progressive delivery with [Flagger](https://flagger.app).

### Manual traffic split

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: api-canary
  namespace: production
spec:
  parentRefs:
    - name: api
      kind: Service
      group: core
      port: 8080
  rules:
    - backendRefs:
        - name: api-stable   # current production version
          port: 8080
          weight: 90
        - name: api-canary   # new version under test
          port: 8080
          weight: 10
```

Start at 10% canary traffic, watch the success rate and latency metrics for `api-canary` in the Linkerd dashboard, and gradually shift weight as confidence grows.

### Automated canary with Flagger

Flagger integrates natively with Linkerd to automate the analysis and promotion steps:

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: api
  namespace: production
spec:
  provider: linkerd
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  progressDeadlineSeconds: 60
  service:
    port: 8080
  analysis:
    interval: 30s
    threshold: 5        # fail after 5 consecutive failed checks
    maxWeight: 50       # max canary traffic percentage
    stepWeight: 5       # increment per successful check
    metrics:
      - name: request-success-rate
        thresholdRange:
          min: 99       # abort if canary success rate drops below 99%
        interval: 1m
      - name: request-duration
        thresholdRange:
          max: 500      # abort if p99 latency exceeds 500ms
        interval: 1m
```

With this configuration, Flagger automatically increments canary traffic by 5% every 30 seconds as long as success rate stays above 99% and p99 latency stays below 500ms. If either threshold is breached, it immediately rolls back.

---

## Feature 6: Multi-Cluster Communication

Linkerd extends its mTLS guarantees across cluster boundaries using a gateway component and a service mirroring controller.

```bash
# Install multicluster extension on both clusters
linkerd multicluster install | kubectl apply -f -
linkerd multicluster check

# Link cluster-west to cluster-east
# (run from the west cluster context, using east cluster credentials)
linkerd multicluster link \
  --context=east \
  --cluster-name=east \
  | kubectl apply -f -

# Export a service from the east cluster
kubectl label service my-service \
  mirror.linkerd.io/exported=true \
  -n production \
  --context=east
```

Once exported, a `my-service-east` service appears automatically in the west cluster. Traffic routed to it traverses the gateway with full mTLS encryption — the same trust anchor is shared across both clusters. From the calling application's perspective, `my-service-east` looks like any other local service.

---

## Certificate Management in Production

The trust anchor is the root of Linkerd's mTLS PKI. It has a default validity of 365 days. **Do not let it expire** — if it does, all mTLS connections in the cluster will fail simultaneously.

### Monitoring certificate expiry

```bash
# Check certificate expiry dates
linkerd check

# The check will warn 60 days before expiry:
# ‼ issuer cert is valid for at least 60 days
#   issuer certificate will expire on 2025-12-31T00:00:00Z
```

### Rotating the trust anchor

Use cert-manager to automate certificate rotation in production:

```yaml
# cert-manager issues and rotates Linkerd's certificates automatically
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: linkerd-trust-anchor
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: linkerd-trust-anchor
  namespace: linkerd
spec:
  isCA: true
  commonName: root.linkerd.cluster.local
  secretName: linkerd-trust-anchor
  duration: 8760h   # 1 year
  renewBefore: 720h # renew 30 days before expiry
  privateKey:
    algorithm: ECDSA
  issuerRef:
    name: linkerd-trust-anchor
    kind: ClusterIssuer
    group: cert-manager.io
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: linkerd-identity-issuer
  namespace: linkerd
spec:
  isCA: true
  commonName: identity.linkerd.cluster.local
  secretName: linkerd-identity-issuer
  duration: 2160h   # 90 days
  renewBefore: 720h # renew 30 days before expiry
  privateKey:
    algorithm: ECDSA
  issuerRef:
    name: linkerd-trust-anchor
    kind: ClusterIssuer
    group: cert-manager.io
```

---

## Upgrading Linkerd

Linkerd upgrades follow a four-step process. The order matters.

```bash
# Step 1: Upgrade the CLI
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh

# Step 2: Upgrade the control plane
# (Helm)
helm repo update
helm upgrade linkerd-crds linkerd/linkerd-crds -n linkerd
helm upgrade linkerd-control-plane linkerd/linkerd-control-plane \
  -n linkerd \
  --reuse-values

# Step 3: Upgrade extensions
helm upgrade linkerd-viz linkerd/linkerd-viz -n linkerd-viz --reuse-values
helm upgrade linkerd-multicluster linkerd/linkerd-multicluster \
  -n linkerd-multicluster --reuse-values

# Step 4: Upgrade the data plane (rolling restart to re-inject proxies)
kubectl rollout restart deployment -n production
kubectl rollout restart deployment -n staging

# Verify
linkerd check
```

Linkerd supports one minor version of skew between the control plane and data plane proxies, so you can upgrade the control plane first and then gradually roll the data plane without downtime.

---

## Production Checklist

- [ ] Install via Helm with explicit certificates (not auto-generated)
- [ ] Use cert-manager to automate trust anchor and issuer rotation
- [ ] Set calendar alerts for trust anchor expiry (check with `linkerd check`)
- [ ] Configure external Prometheus for metrics retention beyond 6 hours
- [ ] Enable namespace-level injection for all application namespaces
- [ ] Verify mTLS with `linkerd viz tap` after meshing services
- [ ] Add `HTTPRoute` timeout policies to all external-facing services
- [ ] Only add retry policies to idempotent read endpoints
- [ ] Use Flagger for canary deployments rather than manual traffic splits
- [ ] Test `linkerd check` as part of your CI/CD health checks post-deploy
- [ ] Monitor proxy resource usage — each proxy uses ~10MB RSS at idle

---

## Linkerd vs. Istio

|  | Linkerd | Istio |
|--|---------|-------|
| Proxy | Rust (`linkerd2-proxy`) | C++ (Envoy) |
| Proxy memory | ~10 MB | ~50–150 MB |
| Proxy CPU overhead | < 1ms latency | 2–5ms latency |
| mTLS | Automatic, zero config | Requires PeerAuthentication CRDs |
| Config complexity | Low | High |
| L7 features | HTTP, gRPC, TCP | HTTP, gRPC, TCP + WebAssembly filters |
| Multi-cluster | Built-in | Istio East-West gateway |
| CNCF status | Graduated | Graduated |

Linkerd wins on simplicity and resource efficiency. Istio wins if you need advanced L7 policies, WebAssembly extensions, or multi-cluster federation with more complex topologies.

---

## Conclusion

Linkerd is the lowest-friction path to a secure, observable service mesh. Install it in an afternoon, annotate your namespaces, restart your deployments — and you immediately have mTLS for every inter-service call, EWMA load balancing that routes away from slow pods, golden metrics in a real-time dashboard, and the infrastructure to do canary deployments without a feature flag system.

The Rust proxy keeps the operational overhead honest: a few megabytes of memory per pod and under a millisecond of added latency. That is a trade-off worth making for the security and observability you get in return.

Getting started: [linkerd.io/2.19/getting-started](https://linkerd.io/2.19/getting-started/)
