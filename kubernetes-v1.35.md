# Kubernetes v1.35 "Timbernetes" - Comprehensive Feature Analysis for LinkedIn Article

**Release Date:** December 17, 2025
**Release Cycle:** 14 weeks (Sept 15 - Dec 17, 2025)
**Total Enhancements:** 60 (17 Stable/GA, 19 Beta, 22 Alpha)
**Go Version:** 1.25.4

---

## 🚨 BREAKING CHANGES & MIGRATION REQUIRED

### 1. **kubelet `--pod-infra-container-image` Flag REMOVED**
**Impact:** HIGH - kubelet will fail to start if this flag is present

**How to check if you're affected:**
```bash
# Check kubelet configuration
ps aux | grep kubelet | grep "pod-infra-container-image"

# For systemd-managed kubelet
systemctl cat kubelet | grep "pod-infra-container-image"
```

**Action Required:**
- **Non-kubeadm clusters:** Manually remove `--pod-infra-container-image` flag from kubelet configuration
- **kubeadm clusters:** Remove from `extraArgs` in kubelet configuration file
- **Timeline:** Must be done BEFORE upgrading to v1.35

**Reference:** PR #133779

---

### 2. **cgroup v1 Validation Changes - kubeadm Now Errors**
**Impact:** CRITICAL - kubeadm will throw ERROR (not warning) if cgroup v1 detected with kubelet v1.35+

**How to check if you're affected:**
```bash
# Check if your system is using cgroup v1
stat -fc %T /sys/fs/cgroup/
# If output is "tmpfs" → cgroup v1 (AFFECTED)
# If output is "cgroup2fs" → cgroup v2 (OK)

# Alternative check
grep cgroup /proc/filesystems
```

**Action Required - Option 1 (Recommended):**
- Migrate all nodes to systems with cgroup v2 enabled
- Update Linux kernel (requires kernel 4.5+ with cgroup v2 support)
- Test workloads on cgroup v2 before upgrading

**Action Required - Option 2 (Workaround):**
- Ignore SystemVerification preflight check in kubeadm
- Add `failCgroupV1: false` to `kube-system/kubelet-config` ConfigMap
- **Note:** cgroup v1 is deprecated and will be removed in future

**Reference:** PR #134744, #134298

---

### 3. **StorageVersionMigration API v1alpha1 REMOVED**
**Impact:** HIGH - v1alpha1 API no longer supported

**How to check if you're affected:**
```bash
# Check if you have v1alpha1 StorageVersionMigration resources
kubectl get storageversionmigrations.migration.k8s.io/v1alpha1 --all-namespaces
```

**Action Required:**
- Remove all v1alpha1 StorageVersionMigration resources BEFORE upgrading
- Migrate to v1beta1 API
- **Timeline:** Must be done BEFORE upgrading to v1.35

**Reference:** PR #134784

---

### 4. **kubectl exec Syntax Change**
**Impact:** MEDIUM - Old syntax no longer supported

**Breaking Change:**
```bash
# OLD SYNTAX (NO LONGER WORKS)
kubectl exec my-pod ls /app

# NEW SYNTAX (REQUIRED)
kubectl exec my-pod -- ls /app
```

**Action Required:**
- Update all scripts, CI/CD pipelines, and automation tools
- Add `--` before command in all kubectl exec invocations
- **Timeline:** Effective immediately in v1.35

**Reference:** PR #133841

---

### 5. **Pods/exec, pods/attach, pods/portforward Permission Changes**
**Impact:** MEDIUM - RBAC permissions need updating

**What Changed:**
- Websocket requests (kubectl exec, attach, portforward) now require `create` permission
- Previously only required `get` permission
- Gated by `AuthorizePodWebsocketUpgradeCreatePermission` (enabled by default)

**How to check if you're affected:**
```bash
# Check your ClusterRoles and Roles
kubectl get clusterrole -o yaml | grep -A 5 "pods/exec"
kubectl get role -A -o yaml | grep -A 5 "pods/exec"
```

**Action Required:**
```yaml
# Update ClusterRoles/Roles to include 'create' verb
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-executor
rules:
- apiGroups: [""]
  resources: ["pods/exec", "pods/attach", "pods/portforward"]
  verbs: ["get", "create"]  # Add 'create' verb
```

**Reference:** PR #134577

---

### 6. **Partitionable Devices (DRA) - Backwards Incompatible Changes**
**Impact:** LOW - Only affects alpha DRA partitionable devices users

**Action Required:**
- If using partitionable devices: Remove ResourceSlices before upgrading/downgrading between v1.34 and v1.35
- **Timeline:** Before upgrading to v1.35

**Reference:** PR #134189

---

## ⚠️ DEPRECATIONS (Still Available but with Warnings)

### 1. **cgroup v1 Deprecated**
**Status:** Deprecated (default behavior changed)

**What Changed:**
- `failCgroupV1` set to `true` by default in v1.35
- Nodes will not start on cgroup v1 by default (can be overridden with `failCgroupV1: false`)

**Action:** Begin planning migration to cgroup v2

**Reference:** PR #134298

---

### 2. **IPVS Mode in kube-proxy Deprecated**
**Status:** Deprecated (still functional but emits warnings)

**What Changed:**
- kube-proxy emits warnings on startup when configured to use IPVS mode
- Will be removed in a future Kubernetes version

**How to check if you're using IPVS:**
```bash
# Check kube-proxy mode
kubectl get configmap kube-proxy -n kube-system -o yaml | grep mode

# Check running kube-proxy pods
kubectl logs -n kube-system -l k8s-app=kube-proxy | grep "Using ipvs"
```

**Replacement:** `nftables` mode

**Migration steps:**
```bash
# Update kube-proxy ConfigMap
kubectl edit configmap kube-proxy -n kube-system
# Change mode: "ipvs" to mode: "nftables"

# Restart kube-proxy pods
kubectl rollout restart daemonset kube-proxy -n kube-system
```

**Reference:** PR #134539

---

### 3. **Service trafficDistribution `PreferClose` Deprecated**
**Status:** Deprecated (backward compatible)

**What Changed:**
- `PreferClose` deprecated in favor of more explicit `PreferSameZone`
- Old value still works but should be updated

**Action:**
```yaml
# OLD (Deprecated)
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.kubernetes.io/traffic-distribution: PreferClose

# NEW (Recommended)
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.kubernetes.io/traffic-distribution: PreferSameZone
```

**Reference:** PR #134457

---

### 4. **kubeadm etcd Subphase Deprecated**
**Status:** Deprecated (hidden, replaced by etcd-join)

**What Changed:**
- `kubeadm join phase control-plane-join etcd` subphase deprecated
- Replaced by `etcd-join` with identical functionality

**Action:**
```bash
# OLD (Deprecated)
kubeadm join phase control-plane-join etcd

# NEW (Recommended)
kubeadm join phase control-plane-join etcd-join
```

**Reference:** PR #134106

---

## 🗑️ FEATURES & APIs REMOVED

### **Feature Gates Removed (Now Locked)**
The following feature gates have been removed and are locked to their default values:

| Feature Gate | Status | Reference |
|-------------|--------|-----------|
| `StrictCostEnforcementForVAP` | Locked since v1.32 | PR #134994 |
| `StrictCostEnforcementForWebhooks` | Locked since v1.32 | PR #134994 |
| `SizeMemoryBackedVolumes` | GA | PR #133720 |
| `ComponentSLIs` | GA in v1.32 | PR #133742 |
| `UserNamespacesPodSecurityStandards` | Removed | PR #132157 |
| `WaitForAllControlPlaneComponents` | kubeadm GA in v1.34 | PR #134781 |
| `SystemdWatchdog` | Locked to default | PR #134691 |

**Action:** Remove these feature gates from your kubelet, kube-apiserver, and other component configurations.

---

### **APIs Removed**

1. **VolumeAttributesClass from storage.k8s.io/v1alpha1** - PR #134625
2. **StorageVersionMigration v1alpha1 API** - PR #134784 (see Breaking Changes)

---

### **kubectl API Support Removed**

The following beta APIs are no longer supported by kubectl:

| API | Replacement | Reference |
|-----|-------------|-----------|
| `certificates/v1beta1` CertificateSigningRequest | `certificates.k8s.io/v1` | PR #134782 |
| `discovery/v1beta1` EndpointSlice | `discovery.k8s.io/v1` | PR #134913 |
| `networking/v1beta1` Ingress | `networking.k8s.io/v1` | PR #135108, #135176 |
| `networking/v1beta1` IngressClass | `networking.k8s.io/v1` | PR #135108 |
| `policy/v1beta1` PodDisruptionBudget | `policy/v1` | PR #134685 |

**Action Required:**
- Update all YAML manifests to use GA APIs
- Update kubectl commands to use GA API versions

---

### **Environment Variables Removed**

- **`KUBECTL_OPENAPIV3_PATCH`** environment variable removed - PR #134130

---

### **Other Removals**

1. **Deprecated gogo protocol definitions** removed from `k8s.io/kubelet/pkg/apis/dra` - PR #133026
2. **Kubernetes API Go types no longer implement `ProtoMessage()` by default**
   - Can be re-enabled temporarily with build tag
   - PR #134256
3. **rsync dependency** for building Kubernetes removed - PR #134656

---

## ⭐ MAJOR FEATURES PROMOTED TO GA (ENABLED BY DEFAULT)

Developers using alpha/beta versions must **remove feature gates** from configurations.

### 1. **In-Place Pod Resource Updates** (InPlacePodVerticalScaling)
**What it does:** Adjust CPU and memory resources without restarting Pods or containers

**Developer Impact:**
- **Remove feature gate:** `InPlacePodVerticalScaling`
- API fields now stable in `PodSpec`
- No other code changes needed

**Use cases:**
- Nondisruptive vertical scaling for stateful applications
- Batch job resource optimization without Pod recreation
- Dynamic resource adjustment based on load

**Example:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: app
    resources:
      requests:
        cpu: "500m"
        memory: "512Mi"
      limits:
        cpu: "1"
        memory: "1Gi"
    resizePolicy:
    - resourceName: cpu
      restartPolicy: NotRequired
    - resourceName: memory
      restartPolicy: RestartContainer
```

**How to use:**
```bash
# Resize resources in-place
kubectl set resources pod my-pod --containers=app --requests=cpu=1,memory=1Gi
```

**Reference:** PR #134949

---

### 2. **Pod Generation and Observation** (PodObservedGenerationTracking)
**What it does:** Adds `.metadata.generation` field to Pods for reliable change tracking

**Developer Impact:**
- **Remove feature gate:** `PodObservedGenerationTracking`
- New fields available:
  - `.metadata.generation`
  - `.status.observedGeneration`
  - Individual Pod conditions get their own `observedGeneration` fields

**Use case:** Verify if kubelet processed latest Pod spec changes

**Example:**
```bash
# Check if kubelet processed latest changes
kubectl get pod my-pod -o jsonpath='{.metadata.generation}{" "}{.status.observedGeneration}'
# If values match → kubelet processed changes
```

**Reference:** PR #134948

---

### 3. **Service Traffic Distribution - PreferSameNode** (ServiceTrafficDistribution)
**What it does:** Enhanced Service traffic routing with node-level affinity

**Developer Impact:**
- **Remove feature gate:** `ServiceTrafficDistribution`
- **RENAMED:** `PreferClose` → `PreferSameZone` (backward compatible)
- **NEW:** `PreferSameNode` - strictly prioritize local node endpoints

**Example:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  annotations:
    service.kubernetes.io/traffic-distribution: PreferSameNode  # New option
spec:
  selector:
    app: my-app
  ports:
  - port: 80
```

**Benefits:**
- Reduced network latency
- Lower cross-node traffic costs
- Improved performance for node-local services

**Reference:** PR #134457

---

### 4. **Job API managed-by Mechanism** (JobManagedBy)
**What it does:** Enables external controllers to manage Job status synchronization

**Developer Impact:**
- **Feature gate locked to true** (cannot be disabled)
- Use stable `managedBy` field in Job spec
- Controllers can cleanly delegate Job synchronization

**Example:**
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: my-job
spec:
  managedBy: example.com/my-controller
  template:
    spec:
      containers:
      - name: worker
        image: my-app:latest
```

**Reference:** PR #135080

---

### 5. **Fine-grained SupplementalGroups Control** (KEP-3619)
**What it does:** Granular control over supplemental group IDs

**Developer Impact:**
- **Remove feature gate:** `SupplementalGroupsPolicy`
- New supplemental groups control available in Pod security context

**Reference:** PR #135088

---

### 6. **Topology Manager Max Allowable NUMA Nodes**
**What it does:** Configurable NUMA node limit (supports 8+ NUMA nodes)

**Developer Impact:**
- **Remove feature gate:** `TopologyManagerMaxAllowableNUMANodes`
- Can now support servers with >8 NUMA nodes
- Configure via `--topology-manager-policy-options=max-allowable-numa-nodes=N`

**Reference:** PR #134614

---

### 7. **kubeadm ControlPlaneKubeletLocalMode**
**What it does:** Optimized kubelet communication for control plane

**Developer Impact:**
- **Feature gate locked to enabled by default**
- No configuration changes needed
- Improved performance on control plane nodes

**Reference:** PR #134106

---

## 🎯 FEATURES PROMOTED TO BETA (ENABLED BY DEFAULT)

Developers using alpha versions must **remove alpha feature gates** from configurations.

### 1. **HPA Configurable Tolerance** (HPAConfigurableTolerance)
**What it does:** Per-resource custom tolerance for scaling actions

**Developer Impact:**
- **Remove feature gate:** `HPAConfigurableTolerance` (now enabled by default)
- Previous: Hardcoded 10% global tolerance
- Now: Configure via HPA `behavior` field

**Example:**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      tolerance: 0.05  # 5% tolerance instead of default 10%
```

**Reference:** PR #133128

---

### 2. **Deployment Terminating Replicas Status** (DeploymentReplicaSetTerminatingReplicas)
**What it does:** Track Pods with deletion timestamp in Deployment status

**Developer Impact:**
- **Remove feature gate:** `DeploymentReplicaSetTerminatingReplicas` (now enabled by default)
- New status field: `.status.terminatingReplicas`

**Example:**
```bash
# Check terminating replicas
kubectl get deployment my-app -o jsonpath='{.status.terminatingReplicas}'
```

**Benefit:** Better visibility into Pod lifecycle and rollout progress

**Reference:** PR #133087

---

### 3. **StatefulSet maxUnavailable** (MaxUnavailableStatefulSet)
**What it does:** Control maximum unavailable pods during StatefulSet rolling updates

**Developer Impact:**
- **Remove feature gate:** `MaxUnavailableStatefulSet` (now enabled by default)
- New field: `spec.updateStrategy.rollingUpdate.maxUnavailable`

**Example:**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 10
  podManagementPolicy: Parallel  # Recommended with maxUnavailable
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 3  # or "30%" for percentage
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
```

**Benefits:**
- Faster updates for stateful apps
- Reduced update time from O(n) to O(n/maxUnavailable)
- Tolerates multiple Pod downtime

**Reference:** PR #133153

---

### 4. **Enforced kubelet Credential Verification** (KubeletEnsureSecretPulledImages)
**What it does:** Verify Pod credentials before using cached images

**Developer Impact:**
- **Remove feature gate:** `KubeletEnsureSecretPulledImages` (now enabled by default)
- Previous vulnerability: `imagePullPolicy: IfNotPresent` allowed unauthorized access to cached private images
- Now: kubelet verifies credentials before using cached images
- To disable (NOT recommended): Set feature gate to false

**Security impact:** Prevents unauthorized access to cached private container images

**Reference:** PR #135228

---

### 5. **Fine-grained Container Restart Rules** (ContainerRestartRules)
**What it does:** Per-container restart policies based on exit codes

**Developer Impact:**
- **Remove feature gate:** `ContainerRestartRules` (now enabled by default)
- New fields: `restartPolicy` and `restartPolicyRules` at container level

**Example:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: granular-restart-pod
spec:
  restartPolicy: Always  # Pod-level default
  containers:
  - name: app
    image: myapp:latest
    restartPolicy: OnFailure  # Container-level override
    restartPolicyRules:
    - action: RestartContainer
      onExitCodes:
      - 1
      - 2
    - action: IgnoreError
      onExitCodes:
      - 143  # SIGTERM - graceful shutdown
```

**Benefits:**
- Faster recovery for transient failures
- Reduced resource usage (no Pod rescheduling)
- Better control over failure handling

**Reference:** PR #134631

---

### 6. **Pod Topology Labels Admission** (PodTopologyLabelsAdmission)
**What it does:** Auto-label Pods with node topology (zone/region)

**Developer Impact:**
- **Remove feature gate:** `PodTopologyLabelsAdmission` (now enabled by default)
- Kubernetes **automatically injects** topology labels into Pods
- Labels: `topology.kubernetes.io/zone`, `topology.kubernetes.io/region`

**Example:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: topology-aware-pod
spec:
  containers:
  - name: app
    env:
    - name: NODE_ZONE
      valueFrom:
        fieldRef:
          fieldPath: metadata.labels['topology.kubernetes.io/zone']
```

**Benefits:**
- Topology-aware workloads without API server queries
- Improved security (no additional permissions needed)

**Reference:** PR #135158

---

### 7. **Image Volume Source (OCI Artifacts)** (ImageVolume)
**What it does:** Pull and unpack OCI container images into volumes

**Developer Impact:**
- **Remove feature gate:** `ImageVolume` (now enabled by default)
- Requires compatible container runtime: **containerd v2.1+**

**Use cases:**
- Package data-only artifacts (configs, binaries, ML models)
- Eliminates custom init containers

**Example:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: oci-volume-pod
spec:
  volumes:
  - name: data-volume
    image:
      reference: docker.io/myorg/data-artifact:v1.0.0
      pullPolicy: IfNotPresent
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: data-volume
      mountPath: /data
```

**Reference:** PR #135195

---

### 8. **EnvFiles Support** (EnvFiles)
**What it does:** Load environment variables from files (similar to Docker's --env-file)

**Developer Impact:**
- **Remove feature gate:** `EnvFiles` (now enabled by default)
- New Pod spec field for loading env vars from ConfigMaps/Secrets as files

**Reference:** PR #134414

---

### 9. **Hostname Override** (HostnameOverride)
**What it does:** Override Pod hostname independently of Pod name

**Developer Impact:**
- **Remove feature gate:** `HostnameOverride` (now enabled by default)
- Enhanced hostname control for Pods

**Reference:** PR #134729

---

### 10. **Kubelet CrashLoopBackOff Max** (KubeletCrashLoopBackOffMax)
**What it does:** Configurable maximum backoff time for crash looping containers

**Developer Impact:**
- **Remove feature gate:** `KubeletCrashLoopBackOffMax` (now enabled by default)
- Configure maximum backoff time to prevent indefinite waiting

**Reference:** PR #135044

---

### 11. **CSI ServiceAccount Tokens via Secrets** (CSIServiceAccountTokenSecrets)
**What it does:** Secure token delivery to CSI drivers

**Developer Impact:**
- **Remove feature gate:** `CSIServiceAccountTokenSecrets` (now enabled by default)
- Previous: Tokens in `volume_context` (logged in plaintext) - **SECURITY RISK**
- Now: Opt-in `serviceAccountTokenInSecrets` field in CSIDriver

**Example:**
```yaml
apiVersion: storage.k8s.io/v1
kind: CSIDriver
metadata:
  name: secrets-store.csi.k8s.io
spec:
  serviceAccountTokenInSecrets: true  # Enable secure token delivery
  podInfoOnMount: true
```

**Security benefit:** Prevents credential exposure in logs

**Reference:** PR #134826

---

### 12. **Pod Certificate Request** (PodCertificateRequest)
**What it does:** Native workload identity with automated certificate rotation

**Developer Impact:**
- **Disabled by default** in beta (opt-in)
- Enable with feature gate: `PodCertificateRequest=true`
- No need for external cert-manager or SPIFFE/SPIRE controllers

**How it works:**
1. Kubelet generates keys and requests certificates
2. Credentials written directly to Pod filesystem
3. Pure mTLS flows without bearer tokens

**Reference:** PR #134624

---

### 13. **Mutable CSI Node Allocatable Count** (MutableCSINodeAllocatableCount)
**What it does:** Dynamically update node volume attachment capacity

**Developer Impact:**
- **Remove feature gate:** `MutableCSINodeAllocatableCount` (now enabled by default)
- Note: Was disabled by default in v1.34, now enabled in v1.35
- CSI drivers can update `CSINode.spec.drivers[*].allocatable.count` dynamically

**Example:**
```yaml
apiVersion: storage.k8s.io/v1
kind: CSINode
metadata:
  name: node-1
spec:
  drivers:
  - name: ebs.csi.aws.com
    nodeID: i-0123456789abcdef0
    allocatable:
      count: 39  # Can now be updated dynamically
```

**Reference:** PR #134647

---

### 14. **Watch List Client** (WatchListClient)
**What it does:** Efficient list-watch operations for informers

**Developer Impact:**
- **Remove feature gate:** `WatchListClient` (now enabled by default)
- Improved performance for controllers using informers
- No code changes needed

**Reference:** PR #134180

---

### 15. **Opportunistic Scheduler Batching** (KEP-5598)
**What it does:** Optimize scheduling for Pods with identical requirements

**Developer Impact:**
- **Enabled by default**
- No configuration changes needed
- Significant performance improvements for large Pod deployments (40-60% latency reduction)

**How it works:**
- Groups Pods with same "scheduling signature"
- Shares filtering/scoring results across batches
- Reduces O(num pods × num nodes) redundant computation

**Reference:** PR #135231

---

## 🔬 NEW FEATURES IN ALPHA (OPT-IN)

### 1. **Gang Scheduling** (GangScheduling)
**What it does:** All-or-nothing scheduling for interdependent workloads using Workload API

**Enable:**
```bash
--feature-gates=GangScheduling=true
```

**Use cases:**
- AI/ML training jobs requiring all workers
- HPC simulations with interdependent tasks
- Prevents deadlocks and resource fragmentation

**Example:**
```yaml
apiVersion: scheduling.k8s.io/v1alpha1
kind: Workload
metadata:
  name: training-job
spec:
  minMember: 10  # All 10 pods must be scheduled together
  scheduleTimeoutSeconds: 300
---
apiVersion: v1
kind: Pod
metadata:
  name: worker-1
  annotations:
    scheduling.k8s.io/workload: training-job
spec:
  containers:
  - name: worker
    image: ml-training:latest
```

**Reference:** PR #134722

---

### 2. **Node Declared Features**
**What it does:** Nodes declare supported Kubernetes features to prevent incompatible scheduling

**Enable:**
```bash
--feature-gates=NodeDeclaredFeatures=true
```

**Components:**
- New `Node.Status.DeclaredFeatures` field
- `NodeDeclaredFeatures` scheduler plugin
- `NodeDeclaredFeatureValidator` admission plugin

**Use case:** Prevents scheduling Pods on nodes when control plane version > node version

**Example:**
```bash
# Check node declared features
kubectl get node my-node -o jsonpath='{.status.declaredFeatures}'
```

**Reference:** PR #133389

---

### 3. **scheduling.k8s.io/v1alpha1 Workload API**
**What it does:** Express workload-level scheduling requirements (used by gang scheduling)

**Enable:**
```bash
--feature-gates=WorkloadAPI=true
```

**Use case:** Group related Pods for coordinated scheduling decisions

**Reference:** PR #134564

---

### 4. **Mutable Scheduling Directives for Suspended Jobs** (MutableSchedulingDirectivesForSuspendedJobs)
**What it does:** Update Job's scheduling directives (node selectors, affinity, tolerations) while suspended

**Enable:**
```bash
--feature-gates=MutableSchedulingDirectivesForSuspendedJobs=true
```

**Use case:** Adjust Job placement without recreating Job

**Example:**
```bash
# Suspend Job
kubectl patch job my-job -p '{"spec":{"suspend":true}}'

# Update node selector
kubectl patch job my-job --type=merge -p '
spec:
  template:
    spec:
      nodeSelector:
        disktype: ssd
'

# Resume Job
kubectl patch job my-job -p '{"spec":{"suspend":false}}'
```

**Reference:** PR #135104

---

### 5. **Resource Resizing for Suspended Jobs** (KEP-5440)
**What it does:** Update Pod resource requests/limits while Job is suspended

**Enable:**
```bash
--feature-gates=MutableJobPodResourcesForSuspendedJobs=true
```

**Use case:** Recover from OOM/CPU errors without recreating Job

**Reference:** PR #132441

---

### 6. **DRA Device Taints with None Effect**
**What it does:** Preview what DeviceTaintRule would do without actual enforcement

**Enable:**
```bash
--feature-gates=DRADeviceTaintRules=true
```

**Use case:** Dry-run testing for DRA device scheduling policies

**Reference:** PR #134152, #135068

---

### 7. **Structured JSON Response for statusz Endpoint**
**What it does:** `/statusz` endpoint supports structured v1alpha1 JSON format

**Enable:**
```bash
--feature-gates=StatuszJSONOutput=true
```

**Example:**
```bash
# Automated debugging and monitoring
curl -s http://localhost:10250/statusz?format=json | jq .
```

**Reference:** PR #134313

---

### 8. **Structured JSON Response for flagz Endpoint**
**What it does:** `/flagz` endpoint supports structured v1alpha1 JSON format

**Enable:**
```bash
--feature-gates=FlagzJSONOutput=true
```

**Example:**
```bash
# Programmatic component configuration auditing
curl -s http://localhost:10250/flagz?format=json | jq .
```

**Reference:** PR #134995

---

### 9. **Change Container Status on Kubelet Restart** (ChangeContainerStatusOnKubeletRestart)
**What it does:** Controls whether kubelet changes Pod status during restart

**Enable:**
```bash
--feature-gates=ChangeContainerStatusOnKubeletRestart=true
```

**Default:** Disabled by default

**Use case:** Fine-tune kubelet restart behavior impact on Pod status

**Reference:** PR #134746

---

### 10. **User Namespaces with Host Network** (UserNamespacesHostNetworkSupport)
**What it does:** Allow hostNetwork Pods to use user namespaces

**Enable:**
```bash
--feature-gates=UserNamespacesHostNetworkSupport=true
```

**Default:** Disabled by default

**Security benefit:** Enhanced isolation for host network Pods

**Reference:** PR #134893

---

### 11. **Watch-Based Routes Reconciliation for Cloud Controller** (CloudControllerManagerWatchBasedRoutesReconciliation)
**What it does:** Cloud Controller Manager uses informers instead of polling for route updates

**Enable:**
```bash
--feature-gates=CloudControllerManagerWatchBasedRoutesReconciliation=true
```

**Benefits:**
- Reduced cloud provider API calls
- Lowers rate limit risks
- Faster route updates

**Reference:** PR #131220

---

### 12. **Constrained Impersonation** (KEP-5284)
**What it does:** Fine-grained impersonation authorization (principle of least privilege)

**Enable:**
```bash
--feature-gates=ConstrainedImpersonation=true
```

**Previous:** All-or-nothing impersonation
**Now:** Secondary check for specific actions using `impersonate-on:<mode>:<verb>`

**Example RBAC:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: support-viewer
rules:
- apiGroups: [""]
  resources: ["users"]
  verbs: ["impersonate-on:logs:get"]  # Only view logs, not exec
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
```

**Use case:** Support engineer can view logs only, no shell access

**Reference:** PR #134803

---

### 13. **Toleration Numeric Comparison Operators**
**What it does:** Support `Gt` (greater than) and `Lt` (less than) operators in tolerations

**Enable:**
```bash
--feature-gates=TolerationNumericComparison=true
```

**Example:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: critical-app
spec:
  tolerations:
  - key: "sla-tier"
    operator: "Gt"  # Greater than
    value: "99.9"
    effect: NoSchedule
  containers:
  - name: app
    image: critical-service:latest
```

**Use case:** Schedule critical workloads only on high-SLA nodes

**Reference:** PR #134665

---

### 14. **Pod-Level Resource In-Place Resizing**
**What it does:** Track Pod-level resource allocation in status

**Enable:**
```bash
--feature-gates=PodLevelResourceResize=true
```

**New fields:**
- `PodStatus.Resources`
- `PodStatus.AllocatedResources`

**Use case:** Better visibility into Pod resource allocation

**Reference:** PR #132919

---

### 15. **Restart All Containers on Container Exit** (RestartAllContainersOnContainerExit)
**What it does:** Restart all containers when source container exits with matching restart policy rule

**Enable:**
```bash
--feature-gates=RestartAllContainersOnContainerExit=true
```

**Use case:** Coordinated container lifecycle management within Pods

**Reference:** PR #134345

---

### 16. **DRA (Dynamic Resource Allocation) Enhancements**

Core DRA graduated to GA in v1.34 and is **locked to enabled** (cannot be disabled) in v1.35.

**New DRA Alpha Features:**
- **DRAExtendedResourceRequests:** Scoring and init container device reuse
- **DRADeviceTaintsAndTolerations:** Device-level taints and tolerations
- **DRAPartitionableDevices:** Devices defined across ResourceSlices
- **DRAConsumableCapacity:** Consumable capacity tracking
- **DRADeviceBindingConditions:** Device binding condition improvements

**Reference:** PR #134452 (core DRA locked), various PRs for enhancements

---

### 17. **Comparable Resource Version Semantics** (KEP-5504)
**What changed:** Resource versions now support semantic comparison (not just string equality)

**Enable:**
```bash
--feature-gates=ComparableResourceVersion=true
```

**Benefits:**
- Detecting lost updates on reconnection
- Storage version migration improvements
- Performance improvements for informers
- Improved controller reliability

**Technical detail:** Resource versions are decimal numbers that can be compared

---

## 📊 KUBECTL CHANGES & ENHANCEMENTS

### **New Commands**

1. **`kubectl kuberc view` and `kubectl kuberc set`**
   - Manage kubeconfig credential plugin policies
   - Reference: PR #135003

### **New Features**

2. **KYAML Output Format (Beta - Enabled by Default)**
   - Safer YAML subset designed for Kubernetes
   - To enable: `kubectl get -o kyaml` (default in v1.35)
   - To disable: `export KUBECTL_KYAML=false`
   - Addresses whitespace sensitivity and type coercion issues
   - Reference: PR #133327

3. **`--profile=trace` Flag**
   - Tracing support for kubectl commands
   - Reference: PR #134709

4. **`-n` Shorthand for `--namespace`**
   - Available in `kubectl config set-context`
   - Example: `kubectl config set-context --current -n default`
   - Reference: PR #134384

5. **`--as-user-extra` Persistent Flag**
   - Impersonation with extra user attributes
   - Reference: PR #134378

6. **`--chunk-size` Flag Promoted to Stable**
   - Available in: `kubectl describe`, `kubectl get`, `kubectl drain`, `kubectl events`
   - Reference: PR #134481

7. **kubectl Command Headers Promoted to Stable**
   - kubectl automatically sends command metadata in HTTP headers
   - Better audit trails and debugging
   - Reference: PR #134777

### **Breaking Changes**

8. **kubectl exec Requires `--` Before Command**
   - See Breaking Changes section above
   - Reference: PR #133841

### **Improvements**

9. **`kubectl describe pods` Includes `fieldPath` in Events**
   - Better debugging with field-specific event messages
   - Reference: PR #133627

10. **`kubectl auth reconcile` Retries on Conflict**
    - Improved reliability during concurrent RBAC updates
    - Reference: PR #133323

11. **`kubectl scale` Consistent Error Messages**
    - Standardized error message format
    - Reference: PR #134017

12. **`kubectl wait` No Longer Shows "Experimental"**
    - Command promoted to stable
    - Reference: PR #133731, #133907

13. **`kubectl get` and `kubectl describe` Token/Secret Count Removed**
    - No longer shows counts for referenced tokens and secrets
    - Improved security by not exposing secret counts
    - Reference: PR #117160

14. **`kubectl api-resources` Panic Fix**
    - Fixed panic when Discovery Client fails
    - Reference: PR #134833

### **New Fields in Output**

```bash
# View Pod generation and observed generation
kubectl get pod my-pod -o jsonpath='{.metadata.generation}{" "}{.status.observedGeneration}'

# View Deployment terminating replicas
kubectl get deployment my-app -o jsonpath='{.status.terminatingReplicas}'

# View node declared features (alpha - requires feature gate)
kubectl get node my-node -o jsonpath='{.status.declaredFeatures}'
```

---

## 🔄 STABILITY & RELIABILITY IMPROVEMENTS

### **Improved Pod Stability During kubelet Restarts**

**What changed:**
- Previous: Restarting kubelet marked healthy Pods as `NotReady`, removed from load balancers
- Now: kubelet properly restores container state from runtime
- Pods remain `Ready` during kubelet restarts
- Traffic continues uninterrupted

**Benefits:**
- Seamless node maintenance
- No service disruption during kubelet updates
- Improved production stability

---

### **Feature Gate Dependencies Now Explicit**

**What changed:**
- Feature gate dependencies now validated at startup
- `AllAlpha=true` won't work without enabling disabled-by-default beta dependencies
- Prevents misconfiguration and startup issues

**Reference:** PR #133697

---

### **Nomination Improvements in Scheduler**

**Features enabled by default:**
- `NominatedNodeNameForExpectation` - Better pod nomination tracking
- `ClearingNominatedNodeNameAfterBinding` - Cleanup after successful binding

**Benefits:**
- Improved scheduler performance
- Better handling of preemption scenarios

**Reference:** PR #135103

---

## 📈 RELEASE STATISTICS

- **Release cycle:** 14 weeks (Sept 15 - Dec 17, 2025)
- **Contributing companies:** 85
- **Individual contributors:** 419
- **Cloud native ecosystem:** 281 companies, 1,769 contributors
- **Release lead:** Drew Hagen
- **Go version:** 1.25.4

---

## 🗓️ MIGRATION TIMELINE SUMMARY

| Action Item | Priority | Deadline | Reference |
|------------|----------|----------|-----------|
| **Remove `--pod-infra-container-image` flag** | CRITICAL | Before v1.35 upgrade | PR #133779 |
| **Review cgroup v1 usage** | CRITICAL | Before v1.35 upgrade | PR #134744 |
| **Remove StorageVersionMigration v1alpha1** | HIGH | Before v1.35 upgrade | PR #134784 |
| **Update kubectl exec scripts (add `--`)** | HIGH | Before v1.35 upgrade | PR #133841 |
| **Update RBAC for pods/exec (add create verb)** | HIGH | Before v1.35 upgrade | PR #134577 |
| **Upgrade containerd to v2.0+** | HIGH | Before v1.36 | Multiple sources |
| **Migrate from IPVS to nftables** | MEDIUM | Before future release | PR #134539 |
| **Update Service trafficDistribution** | LOW | Anytime | PR #134457 |
| Remove GA feature gates | LOW | After v1.35 upgrade | Multiple PRs |
| Remove beta feature gates | LOW | After v1.35 upgrade | Multiple PRs |

---

## 🎓 UPCOMING EVENTS

- **January 14, 2026:** Kubernetes v1.35 Release Webinar
- **March 23-26, 2026:** KubeCon Europe (Amsterdam)
- **Feb-Jul 2026:** Multiple regional Kubernetes Community Days (KCD)

---

## 📚 KEY RESOURCES

**Official Documentation:**
- [Kubernetes v1.35 Release Blog](https://kubernetes.io/blog/2025/12/17/kubernetes-v1-35-release/)
- [CHANGELOG-1.35.md](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.35.md)
- [Release Notes (Searchable)](https://relnotes.k8s.io)
- [cgroup v2 Documentation](https://kubernetes.io/docs/concepts/architecture/cgroups/)

---

## 💡 SUMMARY FOR LINKEDIN ARTICLE

### ✅ Immediate Actions Required:
1. **Remove `--pod-infra-container-image` flag** from kubelet configs
2. **Verify cgroup v2** compatibility (or add `failCgroupV1: false` workaround)
3. **Remove StorageVersionMigration v1alpha1 resources**
4. **Update kubectl exec** scripts to use `--` syntax
5. **Update RBAC** to add `create` verb for pods/exec, pods/attach, pods/portforward
6. **Remove feature gates** for GA features from all component configurations

### 🎯 Major Features to Leverage:
1. **In-place Pod resource updates** - No more Pod restarts for vertical scaling
2. **Pod generation tracking** - Reliable change verification
3. **StatefulSet maxUnavailable** - Faster stateful app updates
4. **Service PreferSameNode** - Optimized node-local traffic routing
5. **Enhanced container restart rules** - Fine-grained failure recovery
6. **Image volumes (OCI artifacts)** - Package data artifacts without init containers
7. **Enforced credential verification** - Better security for private images

### 🔬 Exciting Alpha Features to Watch:
1. **Gang scheduling** - All-or-nothing scheduling for ML/HPC workloads
2. **Constrained impersonation** - Fine-grained RBAC for support scenarios
3. **Toleration numeric operators** - SLA-aware scheduling
4. **Workload API** - Express workload-level scheduling requirements
5. **Resource resizing for suspended Jobs** - Easier OOM recovery

### ⚠️ Plan Migrations:
1. **Start IPVS → nftables migration** (IPVS deprecated)
2. **Update to containerd v2.0+** before v1.36
3. **Migrate to cgroup v2** (or explicitly allow v1 with workaround)
4. **Update beta API manifests** (networking/v1beta1, policy/v1beta1, etc.)

---

**Last Updated:** December 26, 2025
**Author:** Sreedhar Kodem
**Kubernetes Version:** v1.35 "Timbernetes (The World Tree Release)"

---

## Sources
- [Kubernetes v1.35 Release Blog](https://kubernetes.io/blog/2025/12/17/kubernetes-v1-35-release/)
- [Kubernetes 1.35 CHANGELOG](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.35.md)
- [Kubernetes Release Notes](https://kubernetes.io/releases/notes/)
- [Kubernetes 1.35 Upgrade Guide (ScaleOps)](https://scaleops.com/blog/kubernetes-1-35-release-overview/)
- [Kubernetes v1.35 Overview (Medium)](https://user-cube.medium.com/kubernetes-v1-35-timbernetes-whats-new-and-what-s-changing-654817df7e95)
