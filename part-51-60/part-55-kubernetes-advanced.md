# Part 55: Advanced Kubernetes

## บทนำ

Kubernetes มีความสามารถขั้นสูงมากมายที่ช่วยให้ทีม DevOps บริหารจัดการ workloads ได้อย่างมีประสิทธิภาพ บทนี้จะเจาะลึกเรื่อง Operators, Custom Resource Definitions, Admission Controllers, Policy Engines, และ RBAC

บทนี้จะครอบคลุม:
- Kubernetes Operators
- Custom Resource Definitions (CRDs)
- Admission Controllers
- OPA/Gatekeeper
- Kyverno
- RBAC Deep Dive
- Cluster Management Best Practices

---

## 1. Kubernetes Operators

### 1.1 Operator Pattern คืออะไร?

Operator คือ software ที่ encode operational knowledge ของ application ลงใน Kubernetes controller เปรียบเสมือน "robot admin" ที่รู้จักวิธีบริหาร application นั้น ๆ

```
┌─────────────────────────────────────────────────────────────┐
│  Kubernetes Control Loop                                    │
│                                                             │
│  ┌─────────────┐    Watch      ┌─────────────────────────┐ │
│  │  Custom     │ ──────────→   │  Operator Controller    │ │
│  │  Resource   │               │                         │ │
│  │  (CR)       │ ←──────────   │  1. Observe state       │ │
│  └─────────────┘    Update     │  2. Compare with desired│ │
│                                │  3. Take action         │ │
│  ┌─────────────┐               └─────────────────────────┘ │
│  │  Kubernetes │ ←──────────── Create/Update/Delete        │ │
│  │  Objects    │               Pods, Services, etc.        │ │
│  └─────────────┘                                           │ │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 สร้าง Operator ด้วย Operator SDK

```bash
# ติดตั้ง Operator SDK
curl -sLO https://github.com/operator-framework/operator-sdk/releases/download/v1.33.0/operator-sdk_linux_amd64
chmod +x operator-sdk_linux_amd64
sudo mv operator-sdk_linux_amd64 /usr/local/bin/operator-sdk

# สร้าง Go Operator project
mkdir webapp-operator && cd webapp-operator
operator-sdk init \
    --domain example.com \
    --repo github.com/myorg/webapp-operator

# สร้าง API (CRD) และ Controller
operator-sdk create api \
    --group apps \
    --version v1alpha1 \
    --kind WebApp \
    --resource \
    --controller
```

### 1.3 กำหนด CRD (Custom Resource Definition)

```go
// api/v1alpha1/webapp_types.go

package v1alpha1

import (
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// WebAppSpec กำหนด desired state ของ WebApp
type WebAppSpec struct {
	// +kubebuilder:validation:Minimum=1
	// +kubebuilder:validation:Maximum=10
	Replicas int32 `json:"replicas"`
	
	// Container image
	Image string `json:"image"`
	
	// Resources requirements
	// +optional
	Resources corev1.ResourceRequirements `json:"resources,omitempty"`
	
	// Environment variables
	// +optional
	Env []corev1.EnvVar `json:"env,omitempty"`
	
	// Service configuration
	// +optional
	Service *ServiceSpec `json:"service,omitempty"`
	
	// Autoscaling configuration
	// +optional
	Autoscaling *AutoscalingSpec `json:"autoscaling,omitempty"`
}

type ServiceSpec struct {
	// +kubebuilder:validation:Enum=ClusterIP;NodePort;LoadBalancer
	Type string `json:"type"`
	Port int32  `json:"port"`
}

type AutoscalingSpec struct {
	Enabled     bool  `json:"enabled"`
	MinReplicas int32 `json:"minReplicas"`
	MaxReplicas int32 `json:"maxReplicas"`
	// CPU utilization percentage
	// +kubebuilder:validation:Minimum=1
	// +kubebuilder:validation:Maximum=100
	TargetCPUUtilization int32 `json:"targetCPUUtilization"`
}

// WebAppStatus กำหนด observed state ของ WebApp
type WebAppStatus struct {
	// +optional
	AvailableReplicas int32 `json:"availableReplicas,omitempty"`
	
	// +optional
	Conditions []metav1.Condition `json:"conditions,omitempty"`
	
	// URL ที่ access ได้
	// +optional
	URL string `json:"url,omitempty"`
}

//+kubebuilder:object:root=true
//+kubebuilder:subresource:status
//+kubebuilder:printcolumn:name="Replicas",type="integer",JSONPath=".spec.replicas"
//+kubebuilder:printcolumn:name="Available",type="integer",JSONPath=".status.availableReplicas"
//+kubebuilder:printcolumn:name="Age",type="date",JSONPath=".metadata.creationTimestamp"

// WebApp คือ Schema สำหรับ WebApp resource
type WebApp struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   WebAppSpec   `json:"spec,omitempty"`
	Status WebAppStatus `json:"status,omitempty"`
}
```

### 1.4 Implement Controller

```go
// controllers/webapp_controller.go

package controllers

import (
	"context"
	"fmt"
	
	appsv1 "k8s.io/api/apps/v1"
	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/errors"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/log"
	
	appsv1alpha1 "github.com/myorg/webapp-operator/api/v1alpha1"
)

// WebAppReconciler reconciles WebApp objects
type WebAppReconciler struct {
	client.Client
	Scheme *runtime.Scheme
}

//+kubebuilder:rbac:groups=apps.example.com,resources=webapps,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=apps.example.com,resources=webapps/status,verbs=get;update;patch
//+kubebuilder:rbac:groups=apps,resources=deployments,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=core,resources=services,verbs=get;list;watch;create;update;patch;delete

func (r *WebAppReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	logger := log.FromContext(ctx)
	
	// ดึง WebApp object
	webapp := &appsv1alpha1.WebApp{}
	if err := r.Get(ctx, req.NamespacedName, webapp); err != nil {
		if errors.IsNotFound(err) {
			return ctrl.Result{}, nil
		}
		return ctrl.Result{}, err
	}
	
	logger.Info("Reconciling WebApp", "name", webapp.Name, "namespace", webapp.Namespace)
	
	// Reconcile Deployment
	if err := r.reconcileDeployment(ctx, webapp); err != nil {
		return ctrl.Result{}, fmt.Errorf("failed to reconcile deployment: %w", err)
	}
	
	// Reconcile Service
	if webapp.Spec.Service != nil {
		if err := r.reconcileService(ctx, webapp); err != nil {
			return ctrl.Result{}, fmt.Errorf("failed to reconcile service: %w", err)
		}
	}
	
	// Update status
	if err := r.updateStatus(ctx, webapp); err != nil {
		return ctrl.Result{}, fmt.Errorf("failed to update status: %w", err)
	}
	
	return ctrl.Result{}, nil
}

func (r *WebAppReconciler) reconcileDeployment(ctx context.Context, webapp *appsv1alpha1.WebApp) error {
	deployment := &appsv1.Deployment{}
	err := r.Get(ctx, client.ObjectKey{
		Name:      webapp.Name,
		Namespace: webapp.Namespace,
	}, deployment)
	
	if errors.IsNotFound(err) {
		// สร้าง Deployment ใหม่
		deployment = r.buildDeployment(webapp)
		return r.Create(ctx, deployment)
	}
	
	if err != nil {
		return err
	}
	
	// Update existing deployment
	deployment.Spec.Replicas = &webapp.Spec.Replicas
	deployment.Spec.Template.Spec.Containers[0].Image = webapp.Spec.Image
	return r.Update(ctx, deployment)
}

func (r *WebAppReconciler) buildDeployment(webapp *appsv1alpha1.WebApp) *appsv1.Deployment {
	labels := map[string]string{
		"app":         webapp.Name,
		"managed-by":  "webapp-operator",
	}
	
	return &appsv1.Deployment{
		ObjectMeta: metav1.ObjectMeta{
			Name:      webapp.Name,
			Namespace: webapp.Namespace,
			Labels:    labels,
			// Set owner reference สำหรับ garbage collection
			OwnerReferences: []metav1.OwnerReference{
				*metav1.NewControllerRef(webapp, appsv1alpha1.GroupVersion.WithKind("WebApp")),
			},
		},
		Spec: appsv1.DeploymentSpec{
			Replicas: &webapp.Spec.Replicas,
			Selector: &metav1.LabelSelector{
				MatchLabels: labels,
			},
			Template: corev1.PodTemplateSpec{
				ObjectMeta: metav1.ObjectMeta{
					Labels: labels,
				},
				Spec: corev1.PodSpec{
					Containers: []corev1.Container{
						{
							Name:      webapp.Name,
							Image:     webapp.Spec.Image,
							Env:       webapp.Spec.Env,
							Resources: webapp.Spec.Resources,
						},
					},
				},
			},
		},
	}
}

func (r *WebAppReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&appsv1alpha1.WebApp{}).
		Owns(&appsv1.Deployment{}).
		Owns(&corev1.Service{}).
		Complete(r)
}
```

### 1.5 Deploy Operator

```bash
# Build และ push operator image
make docker-build docker-push IMG=ghcr.io/myorg/webapp-operator:latest

# Deploy CRDs
make install

# Deploy operator
make deploy IMG=ghcr.io/myorg/webapp-operator:latest

# ตรวจสอบ
kubectl get crds | grep webapp
kubectl get pods -n webapp-operator-system

# ใช้งาน
cat << EOF | kubectl apply -f -
apiVersion: apps.example.com/v1alpha1
kind: WebApp
metadata:
  name: my-webapp
  namespace: production
spec:
  replicas: 3
  image: ghcr.io/myorg/myapp:latest
  resources:
    requests:
      cpu: "100m"
      memory: "128Mi"
    limits:
      cpu: "500m"
      memory: "512Mi"
  service:
    type: LoadBalancer
    port: 80
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 10
    targetCPUUtilization: 70
EOF

# ดูสถานะ
kubectl get webapp my-webapp -n production
```

---

## 2. Admission Controllers

### 2.1 Admission Controller Webhook

Admission Controllers คือ plugins ที่ intercept requests ก่อนที่ Kubernetes API server จะบันทึกข้อมูล

```
API Request → Authentication → Authorization → Admission Control → Persist
                                                 /              \
                                         Mutating          Validating
                                         Webhooks          Webhooks
```

### 2.2 สร้าง Mutating Admission Webhook

```go
// webhook/webapp_webhook.go

package webhook

import (
	"context"
	"encoding/json"
	"fmt"
	"net/http"
	
	corev1 "k8s.io/api/core/v1"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/webhook/admission"
)

// PodDefaulter เพิ่ม default values ให้ Pods
type PodDefaulter struct {
	Client  client.Client
	Decoder *admission.Decoder
}

func (d *PodDefaulter) Handle(ctx context.Context, req admission.Request) admission.Response {
	pod := &corev1.Pod{}
	if err := d.Decoder.Decode(req, pod); err != nil {
		return admission.Errored(http.StatusBadRequest, err)
	}
	
	// เพิ่ม default labels
	if pod.Labels == nil {
		pod.Labels = make(map[string]string)
	}
	pod.Labels["injected-by"] = "admission-webhook"
	
	// เพิ่ม security context ถ้าไม่มี
	for i := range pod.Spec.Containers {
		if pod.Spec.Containers[i].SecurityContext == nil {
			pod.Spec.Containers[i].SecurityContext = &corev1.SecurityContext{
				RunAsNonRoot:             boolPtr(true),
				AllowPrivilegeEscalation: boolPtr(false),
				ReadOnlyRootFilesystem:   boolPtr(true),
				Capabilities: &corev1.Capabilities{
					Drop: []corev1.Capability{"ALL"},
				},
			}
		}
	}
	
	// เพิ่ม resource limits ถ้าไม่มี
	for i := range pod.Spec.Containers {
		if pod.Spec.Containers[i].Resources.Limits == nil {
			pod.Spec.Containers[i].Resources.Limits = corev1.ResourceList{
				corev1.ResourceCPU:    defaultCPULimit,
				corev1.ResourceMemory: defaultMemoryLimit,
			}
		}
	}
	
	marshaledPod, err := json.Marshal(pod)
	if err != nil {
		return admission.Errored(http.StatusInternalServerError, err)
	}
	
	return admission.PatchResponseFromRaw(req.Object.Raw, marshaledPod)
}

func boolPtr(b bool) *bool { return &b }
```

### 2.3 ValidatingAdmissionWebhook

```go
// webhook/pod_validator.go

type PodValidator struct {
	Client  client.Client
	Decoder *admission.Decoder
}

func (v *PodValidator) Handle(ctx context.Context, req admission.Request) admission.Response {
	pod := &corev1.Pod{}
	if err := v.Decoder.Decode(req, pod); err != nil {
		return admission.Errored(http.StatusBadRequest, err)
	}
	
	var warnings []string
	var errors []string
	
	for _, container := range pod.Spec.Containers {
		// ตรวจสอบว่าไม่ใช้ latest tag
		if strings.HasSuffix(container.Image, ":latest") || !strings.Contains(container.Image, ":") {
			errors = append(errors, fmt.Sprintf(
				"container %q ใช้ latest tag — ต้องระบุ version ที่ชัดเจน",
				container.Name,
			))
		}
		
		// ตรวจสอบ resource limits
		if container.Resources.Limits == nil {
			warnings = append(warnings, fmt.Sprintf(
				"container %q ไม่มี resource limits",
				container.Name,
			))
		}
		
		// ตรวจสอบว่าไม่ run as root
		if container.SecurityContext != nil {
			if container.SecurityContext.RunAsUser != nil && *container.SecurityContext.RunAsUser == 0 {
				errors = append(errors, fmt.Sprintf(
					"container %q ไม่อนุญาตให้ run as root (uid=0)",
					container.Name,
				))
			}
		}
		
		// ตรวจสอบว่า image มาจาก registry ที่อนุญาต
		allowedRegistries := []string{"ghcr.io/myorg/", "registry.example.com/"}
		allowed := false
		for _, registry := range allowedRegistries {
			if strings.HasPrefix(container.Image, registry) {
				allowed = true
				break
			}
		}
		if !allowed {
			errors = append(errors, fmt.Sprintf(
				"container %q ใช้ image จาก registry ที่ไม่อนุญาต: %s",
				container.Name, container.Image,
			))
		}
	}
	
	if len(errors) > 0 {
		return admission.Denied(strings.Join(errors, "; "))
	}
	
	response := admission.Allowed("")
	if len(warnings) > 0 {
		response = response.WithWarnings(warnings...)
	}
	
	return response
}
```

---

## 3. OPA/Gatekeeper

### 3.1 ติดตั้ง OPA Gatekeeper

```bash
# ติดตั้ง Gatekeeper
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/v3.14.0/deploy/gatekeeper.yaml

# ตรวจสอบ
kubectl get pods -n gatekeeper-system
```

### 3.2 สร้าง Constraint Templates

```yaml
# gatekeeper/templates/require-labels.yaml
# ConstraintTemplate: กำหนด policy ว่า resources ต้องมี labels ที่ระบุ

apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
  annotations:
    description: Requires resources to contain specified labels.
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        # Schema สำหรับ parameters ของ Constraint
        openAPIV3Schema:
          type: object
          properties:
            message:
              type: string
            labels:
              type: array
              items:
                type: object
                properties:
                  key:
                    type: string
                  allowedRegex:
                    type: string

  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels

        # Helper function ตรวจสอบ label
        get_message(parameters, _default) := _default {
          not parameters.message
        }

        get_message(parameters, _default) := parameters.message {
          parameters.message
        }

        # Main violation rule
        violation[{"msg": msg, "details": {"missing_labels": missing}}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_].key}
          missing := required - provided
          count(missing) > 0
          
          def_msg := sprintf("Missing required labels: %v", [missing])
          msg := get_message(input.parameters, def_msg)
        }

        # ตรวจสอบ regex pattern ของ label values
        violation[{"msg": msg}] {
          label := input.parameters.labels[_]
          has_field(label, "allowedRegex")
          
          value := input.review.object.metadata.labels[label.key]
          not re_match(label.allowedRegex, value)
          
          msg := sprintf(
            "Label '%v' มีค่า '%v' ซึ่งไม่ตรงกับ pattern '%v'",
            [label.key, value, label.allowedRegex]
          )
        }

        has_field(obj, field) {
          _ = obj[field]
        }

---
# gatekeeper/templates/disallow-privileged.yaml
# ConstraintTemplate: ห้าม privileged containers

apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sdisallowprivilegedcontainers
spec:
  crd:
    spec:
      names:
        kind: K8sDisallowPrivilegedContainers
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sdisallowprivilegedcontainers

        violation[{"msg": msg, "details": {}}] {
          c := input_containers[_]
          c.securityContext.privileged
          msg := sprintf(
            "Privileged container ไม่อนุญาต: %v",
            [c.name]
          )
        }

        input_containers[c] {
          c := input.review.object.spec.containers[_]
        }

        input_containers[c] {
          c := input.review.object.spec.initContainers[_]
        }

        input_containers[c] {
          c := input.review.object.spec.ephemeralContainers[_]
        }
```

### 3.3 สร้าง Constraints

```yaml
# gatekeeper/constraints/production-labels.yaml
# Apply policy ให้กับ production namespace

apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: pod-must-have-team-label
spec:
  # บังคับใช้กับ Pods ใน production namespace
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces: ["production", "staging"]
    # ยกเว้น system labels
    excludedNamespaces: ["kube-system", "gatekeeper-system"]
  
  parameters:
    message: "Pod ต้องมี label 'team' และ 'environment'"
    labels:
      - key: team
        allowedRegex: "^(backend|frontend|data|platform)$"
      - key: environment
        allowedRegex: "^(production|staging|development)$"
      - key: version

---
# gatekeeper/constraints/no-privileged.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sDisallowPrivilegedContainers
metadata:
  name: no-privileged-containers
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces: ["kube-system", "monitoring"]
  
  # Enforcement action: deny หรือ warn (dryrun สำหรับ testing)
  enforcementAction: deny

---
# gatekeeper/constraints/allowed-registries.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: allowed-container-registries
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces: ["kube-system"]
  parameters:
    # อนุญาตเฉพาะ registries เหล่านี้
    repos:
      - "ghcr.io/myorg/"
      - "registry.example.com/"
      - "k8s.gcr.io/"
      - "gcr.io/google_containers/"
```

---

## 4. Kyverno

### 4.1 ติดตั้ง Kyverno

```bash
# ติดตั้งด้วย Helm
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update

helm install kyverno kyverno/kyverno \
    --namespace kyverno \
    --create-namespace \
    --set replicaCount=3 \
    --set admissionController.replicas=3

# ตรวจสอบ
kubectl get pods -n kyverno
```

### 4.2 Kyverno Policies

```yaml
# kyverno/policies/require-resource-limits.yaml
# Policy: ต้องกำหนด resource requests และ limits

apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-limits
  annotations:
    policies.kyverno.io/title: Require Resource Limits
    policies.kyverno.io/category: Best Practices
    policies.kyverno.io/severity: medium
    policies.kyverno.io/description: >-
      ทุก container ต้องกำหนด resource requests และ limits
      เพื่อป้องกัน resource starvation และ OOM issues
spec:
  validationFailureAction: Enforce
  background: true
  
  rules:
    - name: validate-resources
      match:
        any:
        - resources:
            kinds:
              - Pod
      
      exclude:
        any:
        - resources:
            namespaces:
              - kube-system
              - kyverno
      
      validate:
        message: "ทุก container ต้องกำหนด CPU และ Memory requests/limits"
        
        pattern:
          spec:
            containers:
              - name: "*"
                resources:
                  requests:
                    memory: "?*"
                    cpu: "?*"
                  limits:
                    memory: "?*"
                    cpu: "?*"

---
# kyverno/policies/auto-add-labels.yaml
# Mutating Policy: เพิ่ม labels อัตโนมัติ

apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-default-labels
  annotations:
    policies.kyverno.io/title: Add Default Labels
    policies.kyverno.io/category: Best Practices
    policies.kyverno.io/description: >-
      เพิ่ม default labels ให้กับทุก Pod โดยอัตโนมัติ
spec:
  rules:
    - name: add-labels
      match:
        any:
        - resources:
            kinds:
              - Pod
      
      mutate:
        patchStrategicMerge:
          metadata:
            labels:
              +(managed-by): kyverno
              +(injected-at): "{{ request.object.metadata.creationTimestamp | time_since('', '', '') | split('@')[0] }}"
          
          # เพิ่ม annotations
          annotations:
            +(last-applied-by): "{{ serviceAccountName }}"

---
# kyverno/policies/require-pod-probes.yaml
# Policy: ต้องมี liveness และ readiness probes

apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-pod-probes
  annotations:
    policies.kyverno.io/title: Require Pod Probes
    policies.kyverno.io/category: Best Practices
    policies.kyverno.io/severity: medium
spec:
  validationFailureAction: Audit  # Audit mode ก่อน แล้วค่อยเปลี่ยนเป็น Enforce
  background: true
  
  rules:
    - name: require-probes
      match:
        any:
        - resources:
            kinds:
              - Deployment
              - StatefulSet
              - DaemonSet
      
      exclude:
        any:
        - resources:
            namespaces:
              - kube-system
      
      validate:
        message: "Container ต้องมี livenessProbe และ readinessProbe"
        
        foreach:
          - list: "request.object.spec.template.spec.containers"
            deny:
              conditions:
                any:
                  - key: "{{ element.livenessProbe }}"
                    operator: Equals
                    value: null
                  - key: "{{ element.readinessProbe }}"
                    operator: Equals
                    value: null

---
# kyverno/policies/verify-image-signature.yaml
# Policy: ตรวจสอบ image signature

apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signature
  annotations:
    policies.kyverno.io/title: Verify Image Signatures
    policies.kyverno.io/category: Software Supply Chain Security
    policies.kyverno.io/severity: high
spec:
  validationFailureAction: Enforce
  background: false
  
  rules:
    - name: verify-signature
      match:
        any:
        - resources:
            kinds:
              - Pod
            namespaces:
              - production
      
      verifyImages:
        - imageReferences:
            - "ghcr.io/myorg/*:*"
          
          attestors:
            - count: 1
              entries:
              - keyless:
                  subject: "https://github.com/myorg/myapp/.github/workflows/build.yml@refs/heads/main"
                  issuer: "https://token.actions.githubusercontent.com"
                  rekor:
                    url: https://rekor.sigstore.dev
          
          # ต้องมี SBOM attestation
          attestations:
            - predicateType: https://cyclonedx.org/bom/v1.4
              attestors:
                - entries:
                  - keyless:
                      subject: "https://github.com/myorg/myapp/.github/workflows/build.yml@refs/heads/main"
                      issuer: "https://token.actions.githubusercontent.com"
              
              # ตรวจสอบ conditions ของ SBOM
              conditions:
                - all:
                  - key: "{{ components | length(@) }}"
                    operator: GreaterThan
                    value: 0
```

---

## 5. RBAC Deep Dive

### 5.1 RBAC Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  Kubernetes RBAC                                            │
│                                                             │
│  Subjects          Roles/ClusterRoles    Resources          │
│  ┌──────────┐     ┌──────────────────┐  ┌──────────────┐  │
│  │ User     │─┐   │ Role             │  │ pods         │  │
│  │ Group    │ ├──→│ ClusterRole      │─→│ deployments  │  │
│  │ Service  │─┘   │                  │  │ services     │  │
│  │ Account  │     └──────────────────┘  │ configmaps   │  │
│  └──────────┘            │              └──────────────┘  │
│                          │                                  │
│              RoleBinding/ClusterRoleBinding                 │
│              (ผูก Subject กับ Role)                         │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 สร้าง RBAC สำหรับ CI/CD

```yaml
# k8s/rbac/cicd-rbac.yaml

# ServiceAccount สำหรับ CI/CD pipeline
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cicd-deployer
  namespace: production
  annotations:
    # AWS IRSA
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/CICDDeployerRole

---
# ClusterRole สำหรับ deployment
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cicd-deployer-role
rules:
  # อ่าน namespaces
  - apiGroups: [""]
    resources: ["namespaces"]
    verbs: ["get", "list"]
  
  # จัดการ deployments
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  
  # จัดการ services
  - apiGroups: [""]
    resources: ["services"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  
  # จัดการ configmaps (สำหรับ config updates)
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  
  # อ่าน pods สำหรับ status checks
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
  
  # จัดการ ingress
  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  
  # Rollout status
  - apiGroups: ["apps"]
    resources: ["deployments/status"]
    verbs: ["get"]

---
# Bind ให้กับ ServiceAccount
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cicd-deployer-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: cicd-deployer
    namespace: production
roleRef:
  kind: ClusterRole
  name: cicd-deployer-role
  apiGroup: rbac.authorization.k8s.io

---
# Developer Role — อ่านอย่างเดียว
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: developer-readonly
rules:
  - apiGroups: ["", "apps", "networking.k8s.io", "batch"]
    resources:
      - pods
      - pods/log
      - deployments
      - services
      - ingresses
      - jobs
      - cronjobs
    verbs: ["get", "list", "watch"]
  
  # สามารถ exec เข้า pods ใน non-production
  - apiGroups: [""]
    resources: ["pods/exec"]
    verbs: ["create"]

---
# Platform Team Role
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: platform-team
rules:
  # Full access ยกเว้น secrets
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]
  
  # ไม่สามารถดู secrets
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list", "watch"]
    # ห้าม create/update/delete secrets โดยตรง
```

### 5.3 RBAC Testing

```bash
#!/bin/bash
# scripts/test-rbac.sh
# ทดสอบ RBAC permissions ต่าง ๆ

echo "=== RBAC Permission Tests ==="

# Test 1: CI/CD Deployer
echo "--- Testing cicd-deployer ---"

# ควรทำได้
kubectl auth can-i create deployments \
    --namespace production \
    --as system:serviceaccount:production:cicd-deployer
echo "Expected: yes"

# ไม่ควรทำได้
kubectl auth can-i delete namespaces \
    --as system:serviceaccount:production:cicd-deployer
echo "Expected: no"

kubectl auth can-i get secrets \
    --namespace production \
    --as system:serviceaccount:production:cicd-deployer
echo "Expected: no"

# Test 2: Developer
echo "--- Testing developer ---"

kubectl auth can-i get pods \
    --namespace production \
    --as john@example.com
echo "Expected: yes"

kubectl auth can-i delete pods \
    --namespace production \
    --as john@example.com
echo "Expected: no"

# Test 3: ตรวจสอบ ClusterAdmin ที่ไม่ควรมี
echo "--- Checking for ClusterAdmin bindings ---"
kubectl get clusterrolebindings \
    -o json | jq -r '
    .items[] |
    select(.roleRef.name == "cluster-admin") |
    .metadata.name + ": " + 
    (.subjects[]? | .kind + "/" + .name)
'
```

---

## 6. Cluster Management Best Practices

### 6.1 Multi-tenancy ด้วย Namespace Isolation

```yaml
# k8s/multi-tenancy/tenant-a.yaml

# Namespace สำหรับ Tenant A
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-a
  labels:
    tenant: team-a
    environment: production

---
# Resource Quota
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-a-quota
  namespace: tenant-a
spec:
  hard:
    # Compute
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    
    # Objects
    pods: "20"
    services: "10"
    configmaps: "20"
    secrets: "20"
    persistentvolumeclaims: "10"
    
    # LoadBalancers
    services.loadbalancers: "2"
    services.nodeports: "0"

---
# LimitRange — default สำหรับทุก container
apiVersion: v1
kind: LimitRange
metadata:
  name: tenant-a-limits
  namespace: tenant-a
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      max:
        cpu: "2"
        memory: "2Gi"
      min:
        cpu: "50m"
        memory: "64Mi"
    
    - type: Pod
      max:
        cpu: "4"
        memory: "4Gi"
    
    - type: PersistentVolumeClaim
      max:
        storage: "10Gi"
      min:
        storage: "1Gi"
```

### 6.2 Vertical Pod Autoscaler

```yaml
# k8s/vpa/webapp-vpa.yaml

apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: webapp-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp
  
  updatePolicy:
    updateMode: "Auto"  # Auto, Recreate, Initial, Off
  
  resourcePolicy:
    containerPolicies:
      - containerName: webapp
        minAllowed:
          cpu: "100m"
          memory: "128Mi"
        maxAllowed:
          cpu: "2"
          memory: "2Gi"
        controlledResources: ["cpu", "memory"]
        controlledValues: RequestsAndLimits
```

### 6.3 Cluster Upgrade Strategy

```bash
#!/bin/bash
# scripts/cluster-upgrade.sh
# Kubernetes cluster upgrade script

set -euo pipefail

CLUSTER_NAME="${1:-production}"
NEW_VERSION="${2}"
REGION="${3:-ap-southeast-1}"

if [ -z "${NEW_VERSION}" ]; then
    echo "Usage: $0 <cluster-name> <new-version> [region]"
    exit 1
fi

echo "=== Starting Kubernetes Cluster Upgrade ==="
echo "Cluster: ${CLUSTER_NAME}"
echo "Target Version: ${NEW_VERSION}"

# 1. Check current version
CURRENT_VERSION=$(kubectl version --short 2>/dev/null | grep "Server Version" | awk '{print $3}')
echo "Current Version: ${CURRENT_VERSION}"

# 2. Backup etcd (สำหรับ self-managed clusters)
if kubectl get pods -n kube-system | grep -q etcd; then
    echo "Backing up etcd..."
    ETCD_POD=$(kubectl get pods -n kube-system -l component=etcd -o jsonpath='{.items[0].metadata.name}')
    kubectl exec -n kube-system "${ETCD_POD}" -- \
        etcdctl snapshot save /tmp/etcd-backup-$(date +%Y%m%d).db \
        --endpoints=https://127.0.0.1:2379 \
        --cacert=/etc/kubernetes/pki/etcd/ca.crt \
        --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
        --key=/etc/kubernetes/pki/etcd/healthcheck-client.key
fi

# 3. Drain nodes ทีละ node
NODES=$(kubectl get nodes --no-headers -o custom-columns=NAME:.metadata.name)
for node in ${NODES}; do
    echo "Processing node: ${node}"
    
    # Cordon node
    kubectl cordon "${node}"
    
    # Drain node (รอ 5 นาที)
    kubectl drain "${node}" \
        --ignore-daemonsets \
        --delete-emptydir-data \
        --timeout=5m \
        --force
    
    # Upgrade node (AWS EKS)
    if [ "${CLUSTER_NAME}" != "local" ]; then
        aws eks update-nodegroup-version \
            --cluster-name "${CLUSTER_NAME}" \
            --nodegroup-name "${node}" \
            --kubernetes-version "${NEW_VERSION}" \
            --region "${REGION}"
        
        # รอให้ upgrade เสร็จ
        aws eks wait nodegroup-active \
            --cluster-name "${CLUSTER_NAME}" \
            --nodegroup-name "${node}" \
            --region "${REGION}"
    fi
    
    # Uncordon node
    kubectl uncordon "${node}"
    
    echo "✅ Node ${node} upgraded"
    sleep 30
done

# 4. Verify cluster health
echo "Verifying cluster health..."
kubectl get nodes
kubectl get pods -n kube-system

echo "✅ Cluster upgrade complete!"
```

---

## 7. Workshop: Implement Custom Operator

### Lab 1: Deploy Webapp Operator

```bash
#!/bin/bash
# workshop/lab1-operator.sh

echo "=== Lab 1: Webapp Operator Workshop ==="

# ติดตั้ง prerequisites
command -v go >/dev/null || { echo "Install Go first"; exit 1; }
command -v kubectl >/dev/null || { echo "Install kubectl first"; exit 1; }

# Setup project
mkdir -p /tmp/webapp-operator-lab
cd /tmp/webapp-operator-lab

# Initialize operator
operator-sdk init \
    --domain lab.example.com \
    --repo github.com/lab/webapp-operator

# Create API
operator-sdk create api \
    --group webapp \
    --version v1alpha1 \
    --kind SimpleApp \
    --resource \
    --controller

echo "Operator project created!"
echo "Now edit api/v1alpha1/simpleapp_types.go to add your spec"
echo "Then run: make generate manifests"
```

### Lab 2: Kyverno Policies Lab

```yaml
# workshop/lab2-kyverno/test-pod.yaml
# Test pod ที่จะถูก reject โดย Kyverno policies

apiVersion: v1
kind: Pod
metadata:
  name: bad-pod
  namespace: production
  # ขาด required labels: team, environment
spec:
  containers:
    - name: app
      image: nginx:latest  # ใช้ latest tag — ไม่ดี
      # ไม่มี resource limits
      # ไม่มี security context
      # ไม่มี probes
```

```bash
# ทดสอบว่า Kyverno block pods ที่ไม่ compliant
kubectl apply -f workshop/lab2-kyverno/test-pod.yaml
# ควรจะ fail

# แก้ไข pod ให้ compliant
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: good-pod
  namespace: production
  labels:
    team: backend
    environment: production
    version: "1.0.0"
spec:
  containers:
    - name: app
      image: nginx:1.25.3  # specific version
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "512Mi"
      securityContext:
        runAsNonRoot: true
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
      livenessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 10
        periodSeconds: 10
      readinessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 5
EOF
# ควรจะ succeed
```

---

## 8. สรุปและ Best Practices

### Advanced Kubernetes Checklist

```markdown
## Kubernetes Advanced Checklist

### Operators
- [ ] ใช้ Operator pattern สำหรับ complex stateful applications
- [ ] Implement proper status conditions
- [ ] Handle edge cases และ error states
- [ ] Test controller reconciliation

### Admission Control
- [ ] ใช้ Gatekeeper หรือ Kyverno สำหรับ policy enforcement
- [ ] Start ด้วย Audit mode ก่อน Enforce
- [ ] Document ทุก policy
- [ ] Test policies ก่อน production

### RBAC
- [ ] Principle of least privilege
- [ ] Avoid cluster-admin ยกเว้นจำเป็นจริง
- [ ] Regular RBAC audits
- [ ] ใช้ Service Accounts แทน User accounts สำหรับ automation

### Cluster Management
- [ ] Resource quotas สำหรับทุก namespace
- [ ] LimitRanges สำหรับ default resources
- [ ] Regular cluster upgrades
- [ ] Backup strategy สำหรับ etcd
```

---

## อ้างอิง

- [Operator SDK Documentation](https://sdk.operatorframework.io/docs/)
- [OPA Gatekeeper](https://open-policy-agent.github.io/gatekeeper/)
- [Kyverno Documentation](https://kyverno.io/docs/)
- [Kubernetes RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Kubernetes Multi-tenancy](https://kubernetes.io/docs/concepts/security/multi-tenancy/)
- [VPA Documentation](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)
