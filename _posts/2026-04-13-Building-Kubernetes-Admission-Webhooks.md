---
title: "Building Kubernetes Admission Webhooks in Go"
date: 2026-04-13
categories:
  - devops
tags:
  - kubernetes
  - golang
  - webhooks
  - security
---

Kubernetes is extensible by design. One of the most powerful extension points it offers is the admission webhook — a mechanism that lets you intercept any resource before it lands in etcd and either reject it or silently mutate it. Nginx Ingress uses it to configure Nginx from annotations. Linkerd uses it to inject its sidecar proxy. If you've ever wondered how those tools hook into the API server, this post walks you through building one from scratch in Go.

> Full source code: [github.com/Becram/kubernetes-webhook](https://github.com/Becram/kubernetes-webhook)

---

## How the Kubernetes API request flow works

Before writing any code it helps to understand what the API server actually does with a request.

```
kubectl apply -f pod.yaml
        │
        ▼
┌───────────────────┐
│   API Server      │
│                   │
│  1. Authn/Authz   │
│  2. Mutating      │◄──── your mutating webhook (runs first)
│     Webhooks      │
│  3. Schema        │
│     Validation    │
│  4. Validating    │◄──── your validating webhook (runs after mutation)
│     Webhooks      │
│  5. Persist       │
│     to etcd       │
└───────────────────┘
```

Two types of webhooks exist:

- **MutatingAdmissionWebhook** — runs first, can modify the object (add labels, inject sidecars, set resource limits).
- **ValidatingAdmissionWebhook** — runs after mutation, can only approve or reject.

Both are plain HTTPS servers. The API server sends them a JSON `AdmissionReview` request and expects a JSON `AdmissionReview` response.

---

## What we're building

A webhook server in Go that does three things when a Pod with the label `mutation-check: enabled` is created:

1. **Validate** that the image tag is not `latest`.
2. **Mutate** — inject default CPU/memory resource limits if none are set.
3. **Mutate** — inject environment variables into every container.

---

## Key Go modules

### k8s.io/api

This is where all the Kubernetes object structs live — `Pod`, `Deployment`, `Service`, and everything inside them. When you want to work with a Pod in Go:

```go
import (
    corev1 "k8s.io/api/core/v1"
)

pod := corev1.Pod{}
// pod.Spec.Containers, pod.Spec.Volumes, etc.
```

The full `PodSpec` struct maps 1:1 to everything you can write in a pod YAML.

### k8s.io/apimachinery

Contains the shared infrastructure types — `ObjectMeta`, `TypeMeta`, `AdmissionReview`, `Status`, JSON serialization logic, and the patch types used in webhook responses. You'll use this for building your webhook request/response objects.

---

## Setting up the webhook configuration

Before the API server can call your webhook, you register it with a `MutatingWebhookConfiguration` resource:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: kubernetes-webhook.acme.com
webhooks:
  - name: kubernetes-webhook.acme.com
    objectSelector:
      matchLabels:
        mutation-check: enabled
    rules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE"]
        resources: ["pods"]
        scope: "*"
    clientConfig:
      service:
        namespace: default
        name: kubernetes-webhook
        path: /mutate-pods
        port: 443
      caBundle: <base64-encoded-CA-cert>
    admissionReviewVersions: ["v1"]
    sideEffects: None
    timeoutSeconds: 5
```

Key things to note:

- `objectSelector` narrows which pods trigger the webhook — you don't want every pod going through it.
- `clientConfig.service` points at your webhook service inside the cluster.
- The connection **must be TLS**. The `caBundle` is the CA that signed your server cert. There's a cert generation script in the repo.

---

## The webhook server

The server is a standard Go HTTPS server with two routes — one for mutation, one for validation:

```go
func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/mutate-pods", handleMutate)
    mux.HandleFunc("/validate-pods", handleValidate)

    server := &http.Server{
        Addr:    ":8443",
        Handler: mux,
    }

    log.Fatal(server.ListenAndServeTLS("certs/tls.crt", "certs/tls.key"))
}
```

Every handler reads the `AdmissionReview` from the request body, processes it, and writes back an `AdmissionReview` response.

```go
func handleMutate(w http.ResponseWriter, r *http.Request) {
    body, err := io.ReadAll(r.Body)
    if err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    admissionReview := &admissionv1.AdmissionReview{}
    if _, _, err := deserializer.Decode(body, nil, admissionReview); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    response := mutate(admissionReview.Request)
    admissionReview.Response = response
    admissionReview.Response.UID = admissionReview.Request.UID

    jsonResponse, err := json.Marshal(admissionReview)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    w.Write(jsonResponse)
}
```

---

## Mutation: injecting resource limits

This is the most common real-world use case — clusters where developers forget to set resource requests and limits. The mutation webhook catches those pods and injects defaults before they reach the scheduler.

```go
func (m containerResources) Mutate(pod *corev1.Pod) (*corev1.Pod, error) {
    mutated := pod.DeepCopy()

    defaults := corev1.ResourceRequirements{
        Requests: corev1.ResourceList{
            corev1.ResourceCPU:    resource.MustParse("100m"),
            corev1.ResourceMemory: resource.MustParse("128Mi"),
        },
        Limits: corev1.ResourceList{
            corev1.ResourceCPU:    resource.MustParse("500m"),
            corev1.ResourceMemory: resource.MustParse("512Mi"),
        },
    }

    for i, container := range mutated.Spec.Containers {
        if container.Resources.Requests == nil && container.Resources.Limits == nil {
            mutated.Spec.Containers[i].Resources = defaults
        }
    }

    return mutated, nil
}
```

`DeepCopy()` is important — you never modify the original object. The API server expects a JSON Patch describing what changed, not a full replacement.

---

## Building the JSON Patch response

Kubernetes mutation webhooks respond with a [JSON Patch (RFC 6902)](https://datatracker.ietf.org/doc/html/rfc6902) — a list of operations describing what to change:

```go
type patchOperation struct {
    Op    string      `json:"op"`
    Path  string      `json:"path"`
    Value interface{} `json:"value,omitempty"`
}

func createPatch(original, mutated *corev1.Pod) ([]byte, error) {
    originalJSON, _ := json.Marshal(original)
    mutatedJSON, _ := json.Marshal(mutated)
    return jsonpatch.CreateMergePatch(originalJSON, mutatedJSON)
}
```

The patch is base64-encoded and returned in the `AdmissionResponse`:

```go
patchBytes, err := createPatch(pod, mutatedPod)
if err != nil {
    return errorResponse(uid, err)
}

patchType := admissionv1.PatchTypeJSONPatch
return &admissionv1.AdmissionResponse{
    UID:       uid,
    Allowed:   true,
    Patch:     patchBytes,
    PatchType: &patchType,
}
```

---

## Validation: blocking latest tags

Validation is simpler — you just return `Allowed: true` or `Allowed: false` with a message. No patch needed:

```go
func validateImageTag(pod *corev1.Pod) *admissionv1.AdmissionResponse {
    for _, container := range pod.Spec.Containers {
        image := container.Image
        if strings.HasSuffix(image, ":latest") || !strings.Contains(image, ":") {
            return &admissionv1.AdmissionResponse{
                Allowed: false,
                Result: &metav1.Status{
                    Message: fmt.Sprintf(
                        "container %q uses image %q — explicit tags required, latest is not allowed",
                        container.Name, image,
                    ),
                },
            }
        }
    }
    return &admissionv1.AdmissionResponse{Allowed: true}
}
```

When this fires, `kubectl apply` returns the message directly to the user:

```
Error from server: admission webhook "kubernetes-webhook.acme.com" denied the request:
container "app" uses image "nginx:latest" — explicit tags required, latest is not allowed
```

---

## Running locally with KinD

KinD (Kubernetes in Docker) is the easiest way to test this. The webhook service needs to be reachable from inside the cluster, so the typical flow is:

1. Build the Go binary and container image.
2. Load the image into KinD: `kind load docker-image kubernetes-webhook:latest`
3. Generate TLS certs and create the `caBundle` for the webhook configuration.
4. Apply the `Deployment`, `Service`, and `MutatingWebhookConfiguration`.
5. Create a test pod with the `mutation-check: enabled` label and verify the patch was applied.

```bash
kubectl apply -f deploy/
kubectl run test-pod \
  --image=nginx:1.25 \
  --labels="mutation-check=enabled" \
  --dry-run=server -o yaml | kubectl get -f - -o jsonpath='{.spec.containers[0].resources}'
```

---

## Things to keep in mind in production

**Failure policy matters.** The `MutatingWebhookConfiguration` has a `failurePolicy` field — `Fail` or `Ignore`. With `Fail`, if your webhook is down, no pods can be created. Use `Ignore` during rollouts, but be intentional about which you choose.

**Timeouts.** The default is 10 seconds. Set it as low as your webhook can reliably respond. The API server and users are waiting.

**Avoid webhook recursion.** If your webhook creates pods itself (e.g., for logging), add a `namespaceSelector` or `objectSelector` to exclude those pods, or you'll cause an infinite loop.

**TLS cert rotation.** Certificates expire. Automate rotation with cert-manager's `Certificate` resource and reference it in `caBundle` via a `Secret` mounted to your webhook deployment.

---

## References

1. [Kubernetes extensible admission controllers](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)
2. [Simple Kubernetes Webhook — Slack Engineering](https://slack.engineering/simple-kubernetes-webhook/)
3. [k8s.io/client-go](https://github.com/kubernetes/client-go)
4. [k8s.io/apimachinery](https://github.com/kubernetes/apimachinery)
5. [k8s.io/api](https://github.com/kubernetes/api)
