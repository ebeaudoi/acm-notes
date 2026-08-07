# Red Hat Advanced Cluster Management (ACM) Notes

---

## 1. Tool Installation & Setup

### 1.1 Install Kustomize
* **Reference:** [Kustomize Binaries Installation](https://kubectl.docs.kubernetes.io/installation/kustomize/binaries/)
* **Command:**
  ```bash
  curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
  ```

### 1.2 Install PolicyGenerator
* **Reference:** [PolicyGenerator Plugin Installation](https://github.com/stolostron/policy-generator-plugin#installation)
* **Download:** [PolicyGenerator v1.19.0 Release](https://github.com/open-cluster-management-io/policy-generator-plugin/releases/tag/v1.19.0)
* **Commands:**
  ```bash
  chmod +x linux-amd64-PolicyGenerator
  mv linux-amd64-PolicyGenerator ${HOME}/.config/kustomize/plugin/policy.open-cluster-management.io/v1/policygenerator/PolicyGenerator
  # Alternatively, copy to system path:
  sudo cp linux-amd64-PolicyGenerator /usr/local/bin/PolicyGenerator
  ```

### 1.3 Add a Kustomize Plugin to OpenShift GitOps in RHOCP4
* **Article:** Add a Kustomize Plugin to OpenShift GitOps in RHOCP4
* **Reference:** [https://access.redhat.com/solutions/6997069](https://access.redhat.com/solutions/6997069)
* **Title:** In order to make the Kustomize plugin available to OpenShift GitOps, it should be injected in the OpenShift GitOps repo server pod.

#### Resolution
In order to make the Kustomize plugin available to OpenShift GitOps, it should be injected in the OpenShift GitOps repo server pod.

Create a patch file for the default ArgoCD instance running in the namespace `openshift-gitops` with the configuration below:

1. Define an environment variable for the Kustomize plugin home directory.
2. Define an init container to copy the Kustomize plugin from a custom container image which contains the plugin.
3. Define a volume which is shared by both the init container and the repo server container.

`argo_patch.yaml`:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ArgoCD
metadata:
  name: openshift-gitops
  namespace: openshift-gitops
spec:
  repo:
    env:
    - name: KUSTOMIZE_PLUGIN_HOME                                             # ---> 1
      value: /etc/kustomize/plugin                                                    
    initContainers:                                                           # ---> 2
    - args:
      - -c
      - cp /etc/kustomize/plugin/my-plugin /kustomize-plugin
      command:
      - /bin/bash
      image: registry.example.com/kustomize/kustomize-plugin:latest
      name: kustomize-plugin-install
      volumeMounts:
      - mountPath: /kustomize-plugin
        name: kustomize-plugin-dir
    volumeMounts:
    - mountPath: /etc/kustomize/plugin
      name: kustomize-plugin-dir
    volumes:                                                                  # ---> 3
    - emptyDir: {}
      name: kustomize-plugin-dir
  kustomizeBuildOptions: --enable-alpha-plugins
```

Apply the patch and wait for the `openshift-gitops-repo-server` pod to get recreated with the init container which copies the Kustomize plugin to the repo server container:

```bash
oc apply -f argo_patch.yaml
```

---

## 2. Cluster Removal & Detachment Procedures

### 2.1 Hub Self-Management Warning Note
> **Warning:** If you attempt to detach the hub cluster (named `local-cluster`), be aware that the default setting of `disableHubSelfManagement` is `false`. This setting causes the hub cluster to reimport and manage itself when it is detached and reconciles the `MultiClusterHub` controller. It might take hours for the hub cluster to complete the detachment process and reimport.

### 2.2 Remove a Cluster in ACM 2.1

* **Remove via Command Line:**
  ```bash
  oc delete managedcluster $CLUSTER_NAME
  ```

* **Remove via Web Console:**
  1. From the navigation menu, navigate to **Infrastructure > Clusters**.
  2. Select the option menu (`⋮`) beside the cluster you want to remove.
  3. Select **Destroy cluster** or **Detach cluster**.

* **Remove Remaining Resources After Removal:**
  1. Verify active agent namespaces:
     ```bash
     oc get ns | grep open-cluster-management-agent
     ```
     *Expected output:*
     ```text
     open-cluster-management-agent         Active   10m
     open-cluster-management-agent-addon   Active   10m
     ```
  2. Download the cleanup script from the [stolostron/deploy GitHub repository](https://github.com/stolostron/deploy/blob/master/hack/cleanup-managed-cluster.sh).
  3. Run the script:
     ```bash
     ./cleanup-managed-cluster.sh
     ```
  4. Confirm that both namespaces are completely removed:
     ```bash
     oc get ns | grep open-cluster-management-agent
     ```

### 2.3 Remove a Cluster in ACM 2.7
If namespaces remain stuck after deleting a cluster, follow these steps to clear resources manually:

1. List all remaining resources inside the `<cluster_name>` namespace:
   ```bash
   oc api-resources --verbs=list --namespaced -o name | grep -E '^secrets|^serviceaccounts|^managedclusteraddons|^roles|^rolebindings|^manifestworks|^leases|^managedclusterinfo|^appliedmanifestworks|^clusteroauths' | xargs -n 1 oc get --show-kind --ignore-not-found -n <cluster_name>
   ```

2. For each identified resource that does not show a status of `Delete`, remove non-Kubernetes finalizers:
   ```bash
   oc edit <resource_kind> <resource_name> -n <namespace>
   ```
   * Locate the `finalizers` attribute under `metadata`.
   * Delete the non-Kubernetes finalizers (e.g., using `dd` in Vim).
   * Save and exit (`:wq`).

3. Delete the namespace:
   ```bash
   oc delete ns <cluster-name>
   ```

### 2.4 Backing Up Labels & Policies Before Detaching
* **Reference:** [OpenShift Cluster Inventory - ManagedCluster](https://open-cluster-management.io/docs/concepts/cluster-inventory/managedcluster/#:~:text=To%20be%20specific%2C%20the%20cluster%20has%20a,%2D%20effect%3A%20NoSelect%20key%3A%20cluster.open%2Dcluster%2Dmanagement.io%2Funavailable%20timeAdded%3A%20'2022%2D02%2D21T08%3A11%3A54Z')

> **Note:** Labels associated with the cluster are deleted upon detachment. Export labels before initiating detachment.

1. **Export Labels:**
   ```bash
   oc get managedcluster <cluster-name> -o jsonpath='{.metadata.labels}' > cluster-labels.json
   ```

2. **Export Governance Policies & Placements:**
   ```bash
   # Export Policies
   oc get policies.policy.open-cluster-management.io -A -o yaml > all-policies.yaml

   # Export Policy Automations (Ansible integrations)
   oc get policyautomations.policy.open-cluster-management.io -A -o yaml > all-policy-automations.yaml

   # Export Governance Placements & PlacementBindings
   oc get placementbindings.policy.open-cluster-management.io -A -o yaml > all-placement-bindings.yaml
   oc get placements.apps.open-cluster-management.io -A -o yaml > all-placements.yaml
   ```

3. **Extract Embedded Configuration Templates Directly:**
   ACM Policies wrap underlying Kubernetes resource templates (such as `ConfigurationPolicy` or `CertificatePolicy`). To extract these underlying objects directly:
   ```bash
   oc get configurationpolicies.policy.open-cluster-management.io -A -o yaml > all-configuration-policies.yaml
   ```

---

## 3. ACM Namespaces Reference

To view active ACM namespaces:
```bash
oc get namespaces | grep ^open-clus
```

*Example Output:*
```text
open-cluster-management-addon-observability        Active   15h
open-cluster-management-agent                      Active   15h
open-cluster-management-agent-addon                Active   15h
open-cluster-management-policies                   Active   15h
```

### Namespace Breakdown

| Namespace | Purpose / Function | Core Components |
| :--- | :--- | :--- |
| `open-cluster-management-agent` | Core agent communication and cluster registration with the ACM Hub. | - Klusterlet Registration Agent<br>- Klusterlet Work Agent |
| `open-cluster-management-agent-addon` | Houses add-on controllers for policies, search indexing, and application deployment. | - Config Policy Controller<br>- Governance Policy Framework<br>- Search Collector<br>- Application Manager |
| `open-cluster-management-policies` | Holds replicated policy definitions sent from the Hub and local compliance status. | - Policy CRDs<br>- Local Compliance Status Objects |
| `open-cluster-management-observability` | *(Optional)* Collects and forwards cluster metrics to the ACM Hub Observability stack. | - Thanos Sidecar / Metrics Collector |
| `open-cluster-management-iam-addon` | *(Optional)* Enforces and monitors IAM and certificate compliance policies. | - Cert Policy Controller<br>- IAM Policy Controller |

### Details on `open-cluster-management-policies`
The `open-cluster-management-policies` namespace is automatically created on a managed cluster when you deploy Governance Policies from the ACM Hub to that cluster.

> **Note:** Unlike `open-cluster-management-agent` and `open-cluster-management-agent-addon` (which store binary/controller software running the ACM agent), `open-cluster-management-policies` serves purely as a **data/resource namespace**.

```text
Hub Cluster (Central)                       Managed Cluster (Target)
─────────────────────                       ────────────────────────
Creates Policy Object ──────( ACM Sync )───> Stored in "open-cluster-management-policies"
                                                         │
                                                         ▼
                                             Evaluated by Agent running in 
                                             "open-cluster-management-agent-addon"
```

---

## 4. Operator Deployment via ACM Governance (GitOps Example)

* **References & Examples:**
  * [Red Hat Training Course - DO0015L 2.13](https://role.rhu.redhat.com/rol/app/courses/do0015l-2.13/pages/ch01s04)
  * [deploy-ocp-operators GitHub Repository](https://github.com/ebeaudoi/deploy-ocp-operators/tree/main/gitops/acm-clusterset)

### Step 1: Ensure User Has `cluster-admin` Privileges
```bash
oc adm groups new cluster-admins
oc adm groups add-users cluster-admins admin
```

### Step 2: Create Governance Project
```bash
oc new-project gitops-configure
```

### Step 3: Configure ACM Cluster Set & Bindings
1. In the ACM Console, navigate to **Infrastructure → Clusters**, select the **Cluster sets** tab, and click **Create cluster set**.
2. Set the name to `gitops-configure` and click **Create**.
3. Click **Manage resource assignments**, select the target clusters, click **Review**, and click **Save**.
4. Bind the cluster set to the namespace by clicking **Actions → Edit namespace bindings**.
5. Select the `gitops-configure` namespace and click **Save**.

### Step 4: Manifest Files Setup

Required files in your working directory:
* `ca-bundle.yaml`
* `cluster-role-binding.yaml`
* `gitops-operator.yaml`
* `policy-generator.yaml`
* `argocd-configuration.yaml`

#### File Definitions:

`argocd-configuration.yaml`
```yaml
apiVersion: argoproj.io/v1beta1
kind: ArgoCD
metadata:
  name: openshift-gitops
  namespace: openshift-gitops
spec:
  rbac:
    defaultPolicy: ""
    policy: |
      g, system:cluster-admins, role:admin
      g, cluster-admins, role:admin
      g, ocpadmins, role:admin
    scopes: '[groups]'
  repo:
    volumeMounts:
      - mountPath: /etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem
        subPath: ca-bundle.crt
        name: cluster-root-ca-bundle
    volumes:
      - configMap:
          name: cluster-root-ca-bundle
        name: cluster-root-ca-bundle
  applicationSet:
    resources:
      limits:
        cpu: "2"
        memory: 1Gi
      requests:
        cpu: 250m
        memory: 512Mi
    webhookServer:
      ingress:
        enabled: false
      route:
        enabled: false
  resourceExclusions: |
    - apiGroups:
      - tekton.dev
      clusters:
      - '*'
      kinds:
      - TaskRun
      - PipelineRun
  server:
    route:
      enabled: true
  sso:
    dex:
      openShiftOAuth: true
      resources:
        limits:
          cpu: 500m
          memory: 256Mi
        requests:
          cpu: 250m
          memory: 128Mi
    provider: dex
```

`ca-bundle.yaml`
```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: cluster-root-ca-bundle
  namespace: openshift-gitops
  labels:
    config.openshift.io/inject-trusted-cabundle: 'true'
```

`cluster-role-binding.yaml`
```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: argo-admin
subjects:
  - kind: ServiceAccount
    name: openshift-gitops-argocd-application-controller
    namespace: openshift-gitops
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
```

`gitops-operator.yaml`
```yaml
apiVersion: policy.open-cluster-management.io/v1beta1
kind: OperatorPolicy
metadata:
  name: install-gitops-operator
spec:
  remediationAction: enforce
  severity: critical
  complianceType: musthave
  subscription:
    name: openshift-gitops-operator
    namespace: openshift-gitops-operator
    channel: latest
    source: redhat-operators
    sourceNamespace: openshift-marketplace
    startingCSV: openshift-gitops-operator.v1.21.1
  upgradeApproval: Automatic
  versions: []
```

`policy-generator.yaml`
```yaml
apiVersion: policy.open-cluster-management.io/v1
kind: PolicyGenerator
metadata:
  name: gitops-policy-generator
policyDefaults:
  namespace: gitops-configure
  orderManifests: true
  consolidateManifests: false
  remediationAction: enforce
  placement:
    name: gitops-configure
    labelSelector:
      vendor: OpenShift
placementBindingDefaults:
  name: gitops-configure
policies:
  - name: gitops-configure
    manifests:
      - path: ca-bundle.yaml
      - path: gitops-operator.yaml
      - path: cluster-role-binding.yaml
      - path: argocd-configuration.yaml
```

### Step 5: Apply Policy & Monitor Remediation
1. Generate and apply governance policies:
   ```bash
   PolicyGenerator policy-generator.yaml | oc apply -f -
   ```
2. Navigate to the ACM web console: **Governance → Policies**.
3. Monitor status until all targeted clusters reach **Compliant** state (typically takes up to 5 minutes).

### Step 6: Import Clusters into Argo CD
To register target clusters with Argo CD, apply `gitops-register.yaml`:

`gitops-register.yaml`
```yaml
apiVersion: cluster.open-cluster-management.io/v1beta2
kind: ManagedClusterSetBinding
metadata:
  name: gitops-configure
  namespace: openshift-gitops
spec:
  clusterSet: gitops-configure
---
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: gitops-configure
  namespace: openshift-gitops
spec:
  tolerations:
    - key: cluster.open-cluster-management.io/unreachable
      operator: Exists
    - key: cluster.open-cluster-management.io/unavailable
      operator: Exists
  clusterSets:
    - gitops-configure
---
apiVersion: apps.open-cluster-management.io/v1beta1
kind: GitOpsCluster
metadata:
  name: gitops-configure
  namespace: openshift-gitops
spec:
  argoServer:
    argoNamespace: openshift-gitops
  placementRef:
    kind: Placement
    apiVersion: cluster.open-cluster-management.io/v1beta1
    name: gitops-configure
    namespace: openshift-gitops
```

Apply the registration manifest:
```bash
oc apply -f gitops-register.yaml
```

*Expected output:*
```text
managedclustersetbinding.cluster.open-cluster-management.io/gitops-configure created
placement.cluster.open-cluster-management.io/gitops-configure created
gitopscluster.apps.open-cluster-management.io/gitops-configure created
```

---

## 5. Useful Operator Discovery Commands

To inspect CSVs, subscription channels, catalog sources, and package metadata for operators:

```bash
# Query package details for the OpenShift GitOps operator
oc get packagemanifests openshift-gitops-operator -n openshift-marketplace -o yaml

# List all available package manifests
oc get packagemanifests -n openshift-marketplace
```

Filter specific subscription fields (`channel`, `source`, `startingCSV`):
```bash
oc get packagemanifests openshift-gitops-operator -n openshift-marketplace -o yaml | grep -i -E "channel|source:|sourceNamespace|startingCSV|namespace:"
```

*Example Output:*
```yaml
    catalog-namespace: openshift-marketplace
  namespace: openshift-marketplace
  catalogSource: redhat-operators
  catalogSourceNamespace: openshift-marketplace
  channels:
        operatorframework.io/suggested-namespace: openshift-gitops-operator
        ...
  defaultChannel: latest
```
