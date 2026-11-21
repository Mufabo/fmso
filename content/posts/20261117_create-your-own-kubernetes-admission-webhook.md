---
title: "Create Your Own Kubernetes Admission Webhook"
date: 2026-11-17
draft: true
description: "A hands-on guide to writing, deploying and operating a validating and mutating admission webhook in Go: the AdmissionReview protocol, JSON Patch, TLS with cert-manager, webhook configuration, testing on kind, and the mistakes that take clusters down."
summary: "Admission webhooks let you run your own code on every write to the Kubernetes API, to reject objects or change them before they are stored. Here is how to build one from scratch in Go, deploy it safely, and decide when you shouldn't write one at all."
tags: ["kubernetes", "admission control", "webhooks", "go", "security", "cert-manager"]
categories: ["Programming"]
---

Every write to the Kubernetes API passes through a chain of **admission controllers** before it reaches etcd. Most of them are compiled into the API server: `NamespaceLifecycle`, `LimitRanger`, `PodSecurity`, `NodeRestriction` and so on[^7]. Two of them are hooks for your own code: `MutatingAdmissionWebhook` and `ValidatingAdmissionWebhook`. With these, the API server sends the object to an HTTPS service you run, and your service decides whether to allow it, reject it, or change it[^1].

This is how tools like Kyverno, OPA Gatekeeper, Istio's sidecar injector and cert-manager plug into the cluster. In this post we'll build a small one ourselves, called `pod-guard`, that:

- **rejects** Pods without a `team` label, and Pods whose images use `:latest` or no tag at all;
- **warns** about containers without a memory limit;
- **mutates** Pods to default a `RuntimeDefault` seccomp profile and `allowPrivilegeEscalation: false`.

Everything here is based on Kubernetes v1.37, the current release at the time of writing.

## Where webhooks sit in the request path

```text
 kubectl apply
      │
      ▼
 ┌────────────────┐   ┌───────────────┐   ┌──────────────────┐   ┌────────┐   ┌─────────────────────┐   ┌──────┐
 │ Authentication │──▶│ Authorization │──▶│ Mutating         │──▶│ Schema │──▶│ Validating          │──▶│ etcd │
 │                │   │ (RBAC, Node)  │   │ admission        │   │ checks │   │ admission           │   │      │
 └────────────────┘   └───────────────┘   │ • built-in       │   └────────┘   │ • built-in          │   └──────┘
                                          │ • MutatingAdm.   │                │ • ValidatingAdm.    │
                                          │   Policies       │                │   Policies          │
                                          │ • mutating       │                │ • validating        │
                                          │   webhooks ◀─────┼── your code    │   webhooks ◀────────┼── your code
                                          └──────────────────┘                └─────────────────────┘
```

Two things follow from this order:

1. **Mutation happens before validation.** A validating webhook sees the final object, after every mutating webhook has run. That's why policy checks belong in validating webhooks: a mutating webhook can't be sure nobody changes the object after it.
2. **Mutating webhooks can run more than once.** If one mutating webhook changes the object, others that already ran may be called again (with `reinvocationPolicy: IfNeeded`). Your mutations must be **idempotent**: running them twice must give the same result as running them once[^1].

## The protocol: AdmissionReview in, AdmissionReview out

The API server POSTs an `AdmissionReview` object (API group `admission.k8s.io/v1`) to your webhook. The interesting part is `request`:

```json
{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "request": {
    "uid": "705ab4f5-6393-11e8-b7cc-42010a800002",
    "kind": { "group": "", "version": "v1", "kind": "Pod" },
    "resource": { "group": "", "version": "v1", "resource": "pods" },
    "namespace": "payments",
    "operation": "CREATE",
    "userInfo": { "username": "alice", "groups": ["devs", "system:authenticated"] },
    "object": { "apiVersion": "v1", "kind": "Pod", "metadata": { "...": "..." }, "spec": { "...": "..." } },
    "oldObject": null,
    "dryRun": false
  }
}
```

Your webhook answers with the same `apiVersion` and `kind`, and a `response` that echoes the request's `uid`:

```json
{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "response": {
    "uid": "705ab4f5-6393-11e8-b7cc-42010a800002",
    "allowed": true,
    "warnings": ["container \"app\" has no memory limit"],
    "patchType": "JSONPatch",
    "patch": "W3sib3AiOiJhZGQiLCJwYXRoIjoiL3NwZWMvc2VjdXJpdHlDb250ZXh0Ii..."
  }
}
```

- `allowed: false` rejects the request. Put a human-readable reason in `status.message`, because that's what the user sees.
- `warnings` are shown by `kubectl` as `Warning: ...` lines, even when the request is allowed.
- `patch` (mutating webhooks only) is a base64-encoded **JSON Patch** (RFC 6902)[^6]: a list of `add`, `replace` and `remove` operations against paths like `/spec/containers/0/securityContext`.

That's the whole contract. Anything that can serve HTTPS and speak this JSON can be a webhook.

## Writing the webhook in Go

We'll use only the standard library plus the Kubernetes API types. No controller-runtime, no framework, so every moving part is visible.

```bash
mkdir pod-guard && cd pod-guard
go mod init example.com/pod-guard
go get k8s.io/api@latest k8s.io/apimachinery@latest
```

### The HTTP handler

Both endpoints share the same plumbing: decode the review, call an admit function, copy the UID, encode the response.

```go
// handler.go
package main

import (
	"encoding/json"
	"io"
	"log/slog"
	"net/http"

	admissionv1 "k8s.io/api/admission/v1"
)

type admitFunc func(*admissionv1.AdmissionRequest) *admissionv1.AdmissionResponse

func serve(admit admitFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		if r.Method != http.MethodPost {
			http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
			return
		}
		if r.Header.Get("Content-Type") != "application/json" {
			http.Error(w, "expected application/json", http.StatusUnsupportedMediaType)
			return
		}

		body, err := io.ReadAll(io.LimitReader(r.Body, 3<<20)) // the API server limits objects to ~3 MiB
		if err != nil {
			http.Error(w, "reading body: "+err.Error(), http.StatusBadRequest)
			return
		}

		var review admissionv1.AdmissionReview
		if err := json.Unmarshal(body, &review); err != nil || review.Request == nil {
			http.Error(w, "invalid AdmissionReview", http.StatusBadRequest)
			return
		}

		req := review.Request
		resp := admit(req)
		resp.UID = req.UID

		slog.Info("admission",
			"path", r.URL.Path, "op", req.Operation, "ns", req.Namespace,
			"name", req.Name, "user", req.UserInfo.Username, "allowed", resp.Allowed)

		// Reuse apiVersion and kind from the request, drop the request body.
		review.Request = nil
		review.Response = resp

		w.Header().Set("Content-Type", "application/json")
		if err := json.NewEncoder(w).Encode(review); err != nil {
			slog.Error("writing response", "err", err)
		}
	}
}
```

Note that a webhook that *can't decode* the review returns an HTTP error, while a webhook that decodes it and *doesn't like the object* returns HTTP 200 with `allowed: false`. The API server treats HTTP errors as webhook failures and applies the `failurePolicy`; it treats `allowed: false` as a deliberate rejection.

### The validating logic

```go
// validate.go
package main

import (
	"encoding/json"
	"fmt"
	"net/http"
	"strings"

	admissionv1 "k8s.io/api/admission/v1"
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

func validatePod(req *admissionv1.AdmissionRequest) *admissionv1.AdmissionResponse {
	var pod corev1.Pod
	if err := json.Unmarshal(req.Object.Raw, &pod); err != nil {
		return deny(http.StatusBadRequest, "cannot decode Pod: "+err.Error())
	}

	var problems, warnings []string

	if pod.Labels["team"] == "" {
		problems = append(problems, `label "team" is required`)
	}

	for _, c := range allContainers(&pod) {
		if !pinnedImage(c.Image) {
			problems = append(problems, fmt.Sprintf(
				"container %q: image %q must use an explicit tag other than latest, or a digest", c.Name, c.Image))
		}
		if _, ok := c.Resources.Limits[corev1.ResourceMemory]; !ok {
			warnings = append(warnings, fmt.Sprintf("container %q has no memory limit", c.Name))
		}
	}

	if len(problems) > 0 {
		resp := deny(http.StatusForbidden, strings.Join(problems, "; "))
		resp.Warnings = warnings
		return resp
	}
	return &admissionv1.AdmissionResponse{Allowed: true, Warnings: warnings}
}

func allContainers(pod *corev1.Pod) []corev1.Container {
	return append(append([]corev1.Container{}, pod.Spec.InitContainers...), pod.Spec.Containers...)
}

// pinnedImage reports whether an image reference has a digest, or a tag other than "latest".
// "registry:5000/app" has no tag: the colon belongs to the registry host, not the last path segment.
func pinnedImage(image string) bool {
	if strings.Contains(image, "@sha256:") {
		return true
	}
	last := image[strings.LastIndex(image, "/")+1:]
	i := strings.LastIndex(last, ":")
	if i < 0 {
		return false
	}
	tag := last[i+1:]
	return tag != "" && tag != "latest"
}

func deny(code int32, msg string) *admissionv1.AdmissionResponse {
	reason := metav1.StatusReasonForbidden
	if code == http.StatusBadRequest {
		reason = metav1.StatusReasonBadRequest
	}
	return &admissionv1.AdmissionResponse{
		Allowed: false,
		Result:  &metav1.Status{Code: code, Reason: reason, Message: msg},
	}
}
```

The handler collects *all* problems before answering. Rejecting one problem at a time, so the user fixes it, re-applies and hits the next one, is the most common way to make a webhook unpopular.

### The mutating logic

The mutating webhook builds a JSON Patch. Each operation names a path in the object; `/` inside a key (like in an annotation name) is escaped as `~1`[^6].

```go
// mutate.go
package main

import (
	"encoding/json"
	"fmt"
	"net/http"

	admissionv1 "k8s.io/api/admission/v1"
	corev1 "k8s.io/api/core/v1"
)

type patchOp struct {
	Op    string `json:"op"`
	Path  string `json:"path"`
	Value any    `json:"value"` // no omitempty: false is a value we need to send
}

const markerAnnotation = "pod-guard.example.com/defaults" // escaped as pod-guard.example.com~1defaults in paths

func mutatePod(req *admissionv1.AdmissionRequest) *admissionv1.AdmissionResponse {
	var pod corev1.Pod
	if err := json.Unmarshal(req.Object.Raw, &pod); err != nil {
		return deny(http.StatusBadRequest, "cannot decode Pod: "+err.Error())
	}

	var patch []patchOp
	runtimeDefault := map[string]string{"type": "RuntimeDefault"}

	// Pod-level seccomp profile.
	switch {
	case pod.Spec.SecurityContext == nil:
		patch = append(patch, patchOp{"add", "/spec/securityContext",
			map[string]any{"seccompProfile": runtimeDefault}})
	case pod.Spec.SecurityContext.SeccompProfile == nil:
		patch = append(patch, patchOp{"add", "/spec/securityContext/seccompProfile", runtimeDefault})
	}

	// allowPrivilegeEscalation: false, unless the container is privileged or adds SYS_ADMIN,
	// because the API server rejects that combination as invalid.
	addAPE := func(field string, i int, c corev1.Container) {
		base := fmt.Sprintf("/spec/%s/%d/securityContext", field, i)
		sc := c.SecurityContext
		switch {
		case sc == nil:
			patch = append(patch, patchOp{"add", base, map[string]any{"allowPrivilegeEscalation": false}})
		case sc.AllowPrivilegeEscalation == nil && !needsEscalation(sc):
			patch = append(patch, patchOp{"add", base + "/allowPrivilegeEscalation", false})
		}
	}
	for i, c := range pod.Spec.InitContainers {
		addAPE("initContainers", i, c)
	}
	for i, c := range pod.Spec.Containers {
		addAPE("containers", i, c)
	}

	if len(patch) == 0 {
		return &admissionv1.AdmissionResponse{Allowed: true} // nothing to do: this is what makes it idempotent
	}

	// Record what we did, so it's visible on the object.
	if pod.Annotations == nil {
		patch = append(patch, patchOp{"add", "/metadata/annotations",
			map[string]string{markerAnnotation: "applied"}})
	} else {
		patch = append(patch, patchOp{"add", "/metadata/annotations/pod-guard.example.com~1defaults", "applied"})
	}

	raw, err := json.Marshal(patch)
	if err != nil {
		return deny(http.StatusInternalServerError, "building patch: "+err.Error())
	}
	pt := admissionv1.PatchTypeJSONPatch
	return &admissionv1.AdmissionResponse{Allowed: true, PatchType: &pt, Patch: raw}
}

func needsEscalation(sc *corev1.SecurityContext) bool {
	if sc.Privileged != nil && *sc.Privileged {
		return true
	}
	if sc.Capabilities != nil {
		for _, c := range sc.Capabilities.Add {
			if c == "SYS_ADMIN" || c == "CAP_SYS_ADMIN" {
				return true
			}
		}
	}
	return false
}
```

Two details worth pointing out:

- **Only fill in what's missing.** The webhook never overwrites a value the user set. If someone explicitly sets `allowPrivilegeEscalation: true`, that's for the validating side (or Pod Security Admission) to judge, not for the mutating side to silently change.
- **`add` on an existing key replaces it**, while `add` on a path whose *parent* doesn't exist fails. That's why the code checks for a nil `securityContext` or nil `annotations` map and adds the whole object in that case.

The JSON Patch is in the response, not the Go struct: the API server applies it to its copy of the object. You can't modify `pod` in Go and hope it gets stored.

### main: TLS and certificate reloading

The API server only talks to webhooks over HTTPS, and it verifies the certificate against a CA bundle you give it. Certificates expire and get rotated, so the server reloads the key pair from disk when the file changes instead of reading it once at startup.

```go
// main.go
package main

import (
	"context"
	"crypto/tls"
	"errors"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"sync"
	"syscall"
	"time"
)

type certReloader struct {
	certFile, keyFile string
	mu                sync.Mutex
	cert              *tls.Certificate
	modTime           time.Time
}

func (c *certReloader) get(*tls.ClientHelloInfo) (*tls.Certificate, error) {
	info, err := os.Stat(c.certFile)
	if err != nil {
		return nil, err
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.cert == nil || info.ModTime().After(c.modTime) {
		cert, err := tls.LoadX509KeyPair(c.certFile, c.keyFile)
		if err != nil {
			if c.cert != nil {
				return c.cert, nil // keep serving the old one while the new one is half-written
			}
			return nil, err
		}
		c.cert, c.modTime = &cert, info.ModTime()
		slog.Info("loaded TLS certificate", "file", c.certFile)
	}
	return c.cert, nil
}

func main() {
	certs := &certReloader{certFile: "/tls/tls.crt", keyFile: "/tls/tls.key"}

	mux := http.NewServeMux()
	mux.HandleFunc("/validate", serve(validatePod))
	mux.HandleFunc("/mutate", serve(mutatePod))
	mux.HandleFunc("/healthz", func(w http.ResponseWriter, _ *http.Request) { w.WriteHeader(http.StatusOK) })

	srv := &http.Server{
		Addr:              ":8443",
		Handler:           mux,
		ReadHeaderTimeout: 5 * time.Second,
		TLSConfig:         &tls.Config{MinVersion: tls.VersionTLS12, GetCertificate: certs.get},
	}

	ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGTERM, syscall.SIGINT)
	defer stop()

	go func() {
		slog.Info("listening", "addr", srv.Addr)
		if err := srv.ListenAndServeTLS("", ""); err != nil && !errors.Is(err, http.ErrServerClosed) {
			slog.Error("server failed", "err", err)
			os.Exit(1)
		}
	}()

	<-ctx.Done()
	shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	_ = srv.Shutdown(shutdownCtx) // finish in-flight reviews before exiting
}
```

The graceful shutdown matters more than it looks. During a rolling update, an API server request that lands on a Pod that's exiting would otherwise fail, and with `failurePolicy: Fail` a failed call means a rejected request.

### A unit test

The admit functions are plain functions, so they're easy to test without a cluster:

```go
// validate_test.go
package main

import (
	"encoding/json"
	"testing"

	admissionv1 "k8s.io/api/admission/v1"
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
)

func review(t *testing.T, pod corev1.Pod) *admissionv1.AdmissionRequest {
	t.Helper()
	raw, err := json.Marshal(pod)
	if err != nil {
		t.Fatal(err)
	}
	return &admissionv1.AdmissionRequest{Object: runtime.RawExtension{Raw: raw}}
}

func TestValidatePod(t *testing.T) {
	tests := []struct {
		name    string
		labels  map[string]string
		image   string
		allowed bool
	}{
		{"pinned tag with team", map[string]string{"team": "payments"}, "nginx:1.29", true},
		{"digest", map[string]string{"team": "payments"}, "nginx@sha256:abc123", true},
		{"latest", map[string]string{"team": "payments"}, "nginx:latest", false},
		{"no tag", map[string]string{"team": "payments"}, "nginx", false},
		{"registry port, no tag", map[string]string{"team": "payments"}, "registry:5000/app", false},
		{"missing team", nil, "nginx:1.29", false},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			pod := corev1.Pod{
				ObjectMeta: metav1.ObjectMeta{Labels: tt.labels},
				Spec:       corev1.PodSpec{Containers: []corev1.Container{{Name: "app", Image: tt.image}}},
			}
			if got := validatePod(review(t, pod)).Allowed; got != tt.allowed {
				t.Errorf("allowed = %v, want %v", got, tt.allowed)
			}
		})
	}
}

func TestMutateIsIdempotent(t *testing.T) {
	no := false
	pod := corev1.Pod{
		ObjectMeta: metav1.ObjectMeta{Annotations: map[string]string{markerAnnotation: "applied"}},
		Spec: corev1.PodSpec{
			SecurityContext: &corev1.PodSecurityContext{
				SeccompProfile: &corev1.SeccompProfile{Type: corev1.SeccompProfileTypeRuntimeDefault},
			},
			Containers: []corev1.Container{{
				Name: "app", Image: "nginx:1.29",
				SecurityContext: &corev1.SecurityContext{AllowPrivilegeEscalation: &no},
			}},
		},
	}
	if resp := mutatePod(review(t, pod)); resp.Patch != nil {
		t.Errorf("expected no patch for an already-defaulted Pod, got %s", resp.Patch)
	}
}
```

### The container image

```dockerfile
# Dockerfile
FROM golang:1.25 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY *.go ./
RUN CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o /pod-guard .

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /pod-guard /pod-guard
USER 65532:65532
ENTRYPOINT ["/pod-guard"]
```

## Deploying it

### TLS with cert-manager

The API server needs two things: a serving certificate for the webhook whose name matches the Service DNS name, and the CA that signed it in the webhook configuration's `caBundle` field. You can generate these with `openssl` and paste base64 into YAML, but then you have to rotate them by hand. **cert-manager** does both: it issues the certificate into a Secret, and its **CA injector** copies the CA into any webhook configuration annotated with `cert-manager.io/inject-ca-from`[^5].

My [post on TLS](/posts/20261105_tls-explained-etcd-https/) explains what's happening underneath if the terms are unfamiliar.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: pod-guard
---
# A small private CA: self-signed root, then a CA issuer that signs the serving cert.
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: selfsigned
  namespace: pod-guard
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: pod-guard-ca
  namespace: pod-guard
spec:
  isCA: true
  commonName: pod-guard-ca
  secretName: pod-guard-ca
  duration: 87600h # 10 years
  privateKey:
    algorithm: ECDSA
    size: 256
  issuerRef:
    name: selfsigned
    kind: Issuer
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: pod-guard-ca
  namespace: pod-guard
spec:
  ca:
    secretName: pod-guard-ca
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: pod-guard-tls
  namespace: pod-guard
spec:
  secretName: pod-guard-tls
  dnsNames:
    - pod-guard.pod-guard.svc
    - pod-guard.pod-guard.svc.cluster.local
  duration: 2160h    # 90 days
  renewBefore: 360h  # renew 15 days before expiry
  privateKey:
    algorithm: ECDSA
    size: 256
  issuerRef:
    name: pod-guard-ca
    kind: Issuer
```

Why the two-step CA instead of one self-signed certificate? With a stable CA, the `caBundle` stays the same when the serving certificate rotates every 90 days. With a self-signed leaf, the bundle changes on every rotation, and for a few seconds the API server may trust the old certificate while the Pods serve the new one.

### Deployment, Service and disruption budget

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pod-guard
  namespace: pod-guard
spec:
  replicas: 2
  selector:
    matchLabels: { app: pod-guard }
  template:
    metadata:
      labels: { app: pod-guard }
    spec:
      priorityClassName: system-cluster-critical
      automountServiceAccountToken: false   # the webhook never calls the API
      securityContext:
        runAsNonRoot: true
        seccompProfile: { type: RuntimeDefault }
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels: { app: pod-guard }
      containers:
        - name: pod-guard
          image: registry.example.com/pod-guard:0.1.0
          ports:
            - containerPort: 8443
          readinessProbe:
            httpGet: { path: /healthz, port: 8443, scheme: HTTPS }
          livenessProbe:
            httpGet: { path: /healthz, port: 8443, scheme: HTTPS }
          resources:
            requests: { cpu: 50m, memory: 32Mi }
            limits: { memory: 64Mi }
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities: { drop: ["ALL"] }
          volumeMounts:
            - name: tls
              mountPath: /tls
              readOnly: true
      volumes:
        - name: tls
          secret:
            secretName: pod-guard-tls
---
apiVersion: v1
kind: Service
metadata:
  name: pod-guard
  namespace: pod-guard
spec:
  selector: { app: pod-guard }
  ports:
    - port: 443
      targetPort: 8443
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: pod-guard
  namespace: pod-guard
spec:
  minAvailable: 1
  selector:
    matchLabels: { app: pod-guard }
```

Two replicas spread across nodes, a PodDisruptionBudget so node drains don't take both down at once, and a high priority class so the webhook isn't the first thing evicted under pressure. A webhook with `failurePolicy: Fail` is part of your control plane now; treat it that way.

### The webhook configurations

This is where you tell the API server *which requests* to send to *which endpoint*, and what to do when the webhook doesn't answer.

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: pod-guard
  annotations:
    cert-manager.io/inject-ca-from: pod-guard/pod-guard-tls
webhooks:
  - name: mutate.pod-guard.example.com
    admissionReviewVersions: ["v1"]
    sideEffects: None
    timeoutSeconds: 5
    failurePolicy: Fail
    reinvocationPolicy: IfNeeded
    matchPolicy: Equivalent
    clientConfig:
      service:
        name: pod-guard
        namespace: pod-guard
        path: /mutate
        port: 443
    rules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE"]
        resources: ["pods"]
        scope: Namespaced
    namespaceSelector:
      matchLabels:
        pod-guard.example.com/enabled: "true"
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: pod-guard
  annotations:
    cert-manager.io/inject-ca-from: pod-guard/pod-guard-tls
webhooks:
  - name: validate.pod-guard.example.com
    admissionReviewVersions: ["v1"]
    sideEffects: None
    timeoutSeconds: 5
    failurePolicy: Fail
    matchPolicy: Equivalent
    clientConfig:
      service:
        name: pod-guard
        namespace: pod-guard
        path: /validate
        port: 443
    rules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["pods"]
        scope: Namespaced
    namespaceSelector:
      matchLabels:
        pod-guard.example.com/enabled: "true"
    matchConditions:
      - name: not-break-glass
        expression: "!('system:masters' in request.userInfo.groups)"
```

What each field is for[^1]:

| Field | What it does | What we chose |
| --- | --- | --- |
| `rules` | Which API groups, versions, resources and operations are sent. | Pods only. Mutation on `CREATE`; validation also on `UPDATE`, because a Pod's image can be changed. |
| `namespaceSelector` | Only send requests for objects in matching namespaces. | **Opt-in** by label. Roll out namespace by namespace; `kube-system` and `pod-guard` itself are never sent. |
| `objectSelector` | Same, but on the object's own labels. | Not used here. |
| `matchConditions` | CEL expressions over the request; all must be true for the webhook to be called. Stable since v1.30. | Skip break-glass admins. Filtering here is cheaper than calling the webhook and having it say "allow". |
| `failurePolicy` | What happens when the webhook times out, errors, or isn't reachable: `Fail` rejects the request, `Ignore` lets it through. | `Fail`. A policy that silently stops applying when the webhook is down isn't much of a policy. |
| `timeoutSeconds` | How long the API server waits (1 to 30, default 10). | 5. Every webhook adds its latency to every matching request. |
| `sideEffects` | Whether the webhook changes anything outside the object. `None` is required for `kubectl --dry-run=server` to call it. | `None`. |
| `reinvocationPolicy` | Whether a mutating webhook is called again if a later webhook changes the object. | `IfNeeded`, which is safe because the mutations are idempotent. |
| `matchPolicy` | `Equivalent` also sends requests made through other API versions of the same resource. | `Equivalent`, the default, so nobody bypasses the webhook by using an older API version. |
| `admissionReviewVersions` | Which `AdmissionReview` versions the webhook understands. | `v1` only. |

On a private GKE cluster (or any cluster where the control plane lives in a separate network), there's one more step: the control plane must be allowed to reach the webhook Pods on their **target port**, here 8443. GKE only opens 443 and 10250 from the control plane by default, so add a firewall rule for 8443, or the API server gets timeouts and `failurePolicy: Fail` rejects every Pod in your opted-in namespaces[^9].

## Trying it on kind

A local kind cluster is the fastest way to see the whole thing work[^8]:

```bash
kind create cluster --name webhook-lab

# cert-manager (check the cert-manager docs for the current release)
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
kubectl -n cert-manager wait --for=condition=Available deploy --all --timeout=180s

# Build the image and load it into the kind nodes instead of pushing it to a registry
docker build -t registry.example.com/pod-guard:0.1.0 .
kind load docker-image registry.example.com/pod-guard:0.1.0 --name webhook-lab

kubectl apply -f deploy/
kubectl -n pod-guard rollout status deploy/pod-guard

# Opt a namespace in
kubectl create namespace demo
kubectl label namespace demo pod-guard.example.com/enabled=true
```

With `imagePullPolicy` left at its default, a non-`latest` tag uses `IfNotPresent`, so kind uses the loaded image.

Now try a Pod that breaks the rules. `--dry-run=server` sends it through admission without creating anything:

```console
$ kubectl -n demo run web --image=nginx --dry-run=server
Warning: container "web" has no memory limit
Error from server (Forbidden): admission webhook "validate.pod-guard.example.com" denied the request: label "team" is required; container "web": image "nginx" must use an explicit tag other than latest, or a digest
```

Both problems reported at once, plus the warning. Now a compliant one:

```console
$ kubectl -n demo run web --image=nginx:1.29 --labels=team=platform
Warning: container "web" has no memory limit
pod/web created

$ kubectl -n demo get pod web -o jsonpath='{.spec.securityContext}{"\n"}{.spec.containers[0].securityContext}{"\n"}{.metadata.annotations}{"\n"}'
{"seccompProfile":{"type":"RuntimeDefault"}}
{"allowPrivilegeEscalation":false}
{"pod-guard.example.com/defaults":"applied"}
```

The mutating webhook filled in the defaults; the validating webhook let the result through.

If something doesn't work, look in three places: the webhook's own logs (`kubectl -n pod-guard logs deploy/pod-guard`), the events and error message on the request (TLS errors show up as `x509: certificate signed by unknown authority` or `failed calling webhook`), and whether the CA injector filled in the bundle (`kubectl get validatingwebhookconfiguration pod-guard -o jsonpath='{.webhooks[0].clientConfig.caBundle}' | head -c 20`).

## Operating it without taking down the cluster

Admission webhooks are one of the few pieces of user code that sit in the critical path of *every* matching API request. The Kubernetes project has a whole good-practices page about it[^2]. The points that matter most:

**Never gate what you depend on.** If the webhook matches Pods in its own namespace with `failurePolicy: Fail`, and all its Pods die, the replacement Pods are rejected because the webhook isn't there to approve them. The same deadlock happens with the CNI, DNS, cert-manager, or anything else the webhook needs to start. Exclude those namespaces, which is why opt-in by label is a good default.

**Exclude `kube-system` and other platform namespaces** unless you really need to cover them, and if you do, test node restarts and cluster upgrades with the webhook in place.

**Keep it fast.** The API server calls webhooks in series for mutation and in parallel for validation, and their latency adds up. Don't call external services in the request path if you can avoid it; if you must, cache, and keep your own timeouts well below `timeoutSeconds`.

**Be idempotent and deterministic.** The same input must give the same answer, on every replica, on every reinvocation.

**Don't trust that you see everything.** Webhooks don't see requests for webhook configurations or admission policies themselves, so you can't use a webhook to protect itself from deletion. Use RBAC for that, and remember from my [post on privilege escalation](/posts/20261116_preventing-privilege-escalation-in-kubernetes/) that permission to edit webhook configurations is effectively cluster-admin: a malicious mutating webhook can inject anything into every new Pod.

**Watch the API server's metrics.** `apiserver_admission_webhook_admission_duration_seconds` shows latency per webhook, `apiserver_admission_webhook_rejection_count` shows rejections (including those caused by failures and timeouts, labelled by `error_type`), and `apiserver_admission_webhook_fail_open_count` shows how often `failurePolicy: Ignore` let something through[^2]. Alert on all three.

**Roll out in stages.** Start with warnings only (return `allowed: true` with `warnings`), watch the logs for what would have been rejected, then switch to enforcement, namespace by namespace.

**Have a break-glass plan.** If the webhook is down and blocking everything, deleting the `ValidatingWebhookConfiguration` and `MutatingWebhookConfiguration` stops the API server from calling it immediately. Make sure the people on call know that, and have the permissions to do it.

## When not to write a webhook

Writing a webhook is a good way to understand admission control. Running one in production means running a highly available HTTPS service with certificate rotation, in your control plane's critical path. For a lot of policies, you no longer need to:

- **ValidatingAdmissionPolicy** (stable since v1.30) and **MutatingAdmissionPolicy** (stable since v1.36) run CEL expressions *inside* the API server: no network hop, no certificates, no Deployment to keep alive[^3][^4]. Both of `pod-guard`'s checks fit easily into a ValidatingAdmissionPolicy, and both of its defaults into a MutatingAdmissionPolicy.
- **Pod Security Admission** already enforces the standard Pod hardening rules per namespace.
- **Kyverno** and **OPA Gatekeeper** are webhooks someone else maintains, with policy languages, reporting and audit of existing resources built in.

For comparison, here is the image check as a ValidatingAdmissionPolicy:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: pinned-images
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["pods"]
  validations:
    - expression: >
        object.spec.containers.all(c,
          c.image.contains('@sha256:') ||
          (c.image.split('/')[size(c.image.split('/')) - 1].contains(':') &&
           !c.image.endsWith(':latest')))
      message: "Images must use an explicit tag other than latest, or a digest."
```

(Bind it with a ValidatingAdmissionPolicyBinding, as shown in the privilege escalation post.)

Write your own webhook when CEL isn't enough:

- the decision needs **external data**: an image signature check against a registry, a lookup in a CMDB, a quota service;
- the logic is **too complex** for readable CEL, or needs a library you already have in Go;
- you're building an **operator** and need defaulting and validation for your own CRDs beyond what OpenAPI schemas and CEL validation rules give you (controller-runtime and Kubebuilder generate most of the webhook plumbing for that case);
- you need **mutations that depend on other objects** in the cluster, like injecting a sidecar configured from a ConfigMap.

## Bottom line

- An admission webhook is an HTTPS endpoint that receives an `AdmissionReview` and answers **allowed or denied**, optionally with **warnings** and a **JSON Patch**.
- **Mutating webhooks run first** and may run more than once, so make mutations idempotent; **validating webhooks see the final object**, so put policy there.
- Use **cert-manager** with a small private CA to issue the serving certificate and inject the `caBundle`, and **reload certificates** in the server.
- Scope webhooks tightly with **`rules`, an opt-in `namespaceSelector` and `matchConditions`**, and never let a webhook gate its own namespace or its dependencies.
- With `failurePolicy: Fail`, the webhook is part of your control plane: **run it highly available, keep it fast, watch its metrics** and have a break-glass procedure.
- For simple field checks and defaults, prefer **ValidatingAdmissionPolicy and MutatingAdmissionPolicy**, which run inside the API server. Write a webhook when you need external data or real code.

## References

[^1]: [Dynamic Admission Control](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/), Kubernetes documentation: the `AdmissionReview` request and response, webhook configuration fields, `matchConditions`, reinvocation and side effects.
[^2]: [Admission Webhook Good Practices](https://kubernetes.io/docs/concepts/cluster-administration/admission-webhooks-good-practices/), Kubernetes documentation: availability, latency, idempotence, excluding namespaces and monitoring.
[^3]: [Validating Admission Policy](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/), Kubernetes documentation.
[^4]: [Mutating Admission Policy](https://kubernetes.io/docs/reference/access-authn-authz/mutating-admission-policy/), Kubernetes documentation, and [KEP-3962: Mutating Admission Policies](https://www.kubernetes.dev/resources/keps/3962/): alpha in v1.32, beta in v1.34, stable in v1.36.
[^5]: [CA Injector](https://cert-manager.io/docs/concepts/ca-injector/), cert-manager documentation: injecting CA bundles into webhook configurations with `cert-manager.io/inject-ca-from`.
[^6]: [RFC 6902: JavaScript Object Notation (JSON) Patch](https://datatracker.ietf.org/doc/html/rfc6902), IETF, and [RFC 6901: JSON Pointer](https://datatracker.ietf.org/doc/html/rfc6901) for path syntax and the `~1` escape.
[^7]: [Admission Control in Kubernetes](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/), Kubernetes documentation: the built-in admission plugins and their order.
[^8]: [kind Quick Start: Loading an Image Into Your Cluster](https://kind.sigs.k8s.io/docs/user/quick-start/#loading-an-image-into-your-cluster), kind documentation.
[^9]: [Creating a private cluster: Adding firewall rules for specific use cases](https://cloud.google.com/kubernetes-engine/docs/how-to/private-clusters#add_firewall_rules), Google Cloud documentation.
