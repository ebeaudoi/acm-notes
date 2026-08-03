===============================================================================
               RED HAT ADVANCED CLUSTER MANAGEMENT (ACM) NOTES
===============================================================================

-------------------------------------------------------------------------------
1. TOOL INSTALLATION & SETUP
-------------------------------------------------------------------------------

1.1 Install Kustomize
---------------------
Ref: https://kubectl.docs.kubernetes.io/installation/kustomize/binaries/
Command:
  curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash

1.2 Install PolicyGenerator
---------------------------
Ref: https://github.com/stolostron/policy-generator-plugin#installation
Download: https://github.com/open-cluster-management-io/policy-generator-plugin/releases/tag/v1.19.0

Commands:
  chmod +x linux-amd64-PolicyGenerator
  mv linux-amd64-PolicyGenerator ${HOME}/.config/kustomize/plugin/policy.open-cluster-management.io/v1/policygenerator/PolicyGenerator
  # or
  sudo cp linux-amd64-PolicyGenerator /usr/local/bin/PolicyGenerator


-------------------------------------------------------------------------------
2. CLUSTER REMOVAL & DETACHMENT PROCEDURES
-------------------------------------------------------------------------------

2.1 Hub Self-Management Warning Note
------------------------------------
If you attempt to detach the hub cluster, which is named "local-cluster", be aware
that the default setting of disableHubSelfManagement is false. This setting 
causes the hub cluster to reimport itself and manage itself when it is detached 
and it reconciles the MultiClusterHub controller. It might take hours for the hub 
cluster to complete the detachment process and reimport.

2.2 Remove a Cluster in ACM 2.1
-------------------------------
* Command Line Removal:
    oc delete managedcluster $CLUSTER_NAME

* Console Removal:
    1. From the navigation menu, navigate to Infrastructure > Clusters.
    2. Select the option menu beside the cluster that you want to remove from management.
    3. Select "Destroy cluster" or "Detach cluster".

* Remove Remaining Resources After Removal:
    1. Check for remaining namespaces:
         oc get ns | grep open-cluster-management-agent
       Output should show two namespaces:
         open-cluster-management-agent         Active   10m
         open-cluster-management-agent-addon   Active   10m

    2. Download the cleanup-managed-cluster script from the deploy Git repository:
       Ref: https://github.com/stolostron/deploy/blob/master/hack/cleanup-managed-cluster.sh

    3. Run the script:
         ./cleanup-managed-cluster.sh

    4. Verify both namespaces are removed:
         oc get ns | grep open-cluster-management-agent

2.3 Remove a Cluster in ACM 2.7
-------------------------------
Resolving Problem: Namespace remains after deleting a cluster. Complete the 
following steps to remove the namespace manually:

1. Produce a list of remaining resources in the <cluster_name> namespace:
     oc api-resources --verbs=list --namespaced -o name | grep -E '^secrets|^serviceaccounts|^managedclusteraddons|^roles|^rolebindings|^manifestworks|^leases|^managedclusterinfo|^appliedmanifestworks|^clusteroauths' | xargs -n 1 oc get --show-kind --ignore-not-found -n <cluster_name>

2. Delete each identified resource on the list that does not have a status of Delete (replace cluster_name with the actual namespace):
     oc edit <resource_kind> <resource_name> -n <namespace>
     - Locate the finalizer attribute in the metadata.
     - Delete the non-Kubernetes finalizers by using the vi editor `dd` command.
     - Save the list and exit vi with `:wq`.

3. Delete the namespace:
     oc delete ns <cluster-name>

2.4 Backing Up Labels & Policies Before Detaching
-------------------------------------------------
Ref: https://open-cluster-management.io/docs/concepts/cluster-inventory/managedcluster/#:~:text=To%20be%20specific%2C%20the%20cluster%20has%20a,%2D%20effect%3A%20NoSelect%20key%3A%20cluster.open%2Dcluster%2Dmanagement.io%2Funavailable%20timeAdded%3A%20'2022%2D02%2D21T08%3A11%3A54Z'

Detach Workflow:
  1. Detach the managed cluster using the ACM UI.
  2. Note: Labels will be gone after detaching.
  3. Export labels before detaching (Extract labels into JSON/YAML):
       oc get managedcluster <cluster-name> -o jsonpath='{.metadata.labels}' > cluster-labels.json
  4. Export policies & governance bindings:
       oc get policies.policy.open-cluster-management.io -A -o yaml > all-policies.yaml
       oc get policyautomations.policy.open-cluster-management.io -A -o yaml > all-policy-automations.yaml
       oc get placementbindings.policy.open-cluster-management.io -A -o yaml > all-placement-bindings.yaml
       oc get placements.apps.open-cluster-management.io -A -o yaml > all-placements.yaml

5. Extract Embedded Configuration Templates Directly:
   ACM Policies wrap other Kubernetes resource templates (e.g., ConfigurationPolicy, CertificatePolicy).
   To extract underlying ConfigurationPolicy objects defining actual cluster state:
     oc get configurationpolicies.policy.open-cluster-management.io -A -o yaml > all-configuration-policies.yaml


-------------------------------------------------------------------------------
3. ACM NAMESPACES REFERENCE
-------------------------------------------------------------------------------

Checking ACM namespaces:
  oc get namespaces | grep ^open-clus
  open-cluster-management-addon-observability        Active   15h
  open-cluster-management-agent                      Active   15h
  open-cluster-management-agent-addon                Active   15h
  open-cluster-management-policies                   Active   15h

=====================================================================================================
NAMESPACE                              PURPOSE / FUNCTION                               CORE COMPONENTS
=====================================================================================================
open-cluster-management-agent          Core agent communication and cluster            - Klusterlet Registration Agent
                                       registration with the ACM Hub.                  - Klusterlet Work Agent

open-cluster-management-agent-addon    Houses add-on controllers for policies,         - Config Policy Controller
                                       search indexing, and application deployment.   - Governance Policy Framework
                                                                                       - Search Collector
                                                                                       - Application Manager

open-cluster-management-policies       Holds replicated policy definitions sent from    - Policy CRDs & Local 
                                       the Hub and local compliance status.            - Compliance Status Objects

open-cluster-management-observability  (Optional) Collects and forwards cluster        - Thanos Sidecar / Metrics Collector
                                       metrics to the ACM Hub Observability stack.

open-cluster-management-iam-addon      (Optional) Enforces and monitors IAM and        - Cert Policy Controller
                                       certificate compliance policies.                - IAM Policy Controller
=====================================================================================================

Note on open-cluster-management-policies:
  The open-cluster-management-policies namespace is automatically created on the 
  managed cluster when you deploy Governance Policies from the ACM Hub to that 
  managed cluster.
  
  Unlike the agent namespaces (open-cluster-management-agent and open-cluster-management-agent-addon),
  which store software code and controllers running the ACM agent, 
  open-cluster-management-policies is purely a data/resource namespace.

Architecture Flow:
  Hub Cluster (Central)                       Managed Cluster (Target)
  ─────────────────────                       ────────────────────────
  Creates Policy Object ──────( ACM Sync )───> Stored in "open-cluster-management-policies"
                                                           │
                                                           ▼
                                               Evaluated by Agent running in 
                                               "open-cluster-management-agent-addon"


-------------------------------------------------------------------------------
4. OPERATOR DEPLOYMENT VIA ACM GOVERNANCE (GITOPS EXAMPLE)
-------------------------------------------------------------------------------

References & Examples:
  - https://role.rhu.redhat.com/rol/app/courses/do0015l-2.13/pages/ch01s04
  - https://github.com/ebeaudoi/deploy-ocp-operators/tree/main/gitops/acm-clusterset

Step 1: Ensure User Has Cluster-Admin Privileges
------------------------------------------------
  oc adm groups new cluster-admins
  oc adm groups add-users cluster-admins admin

Step 2: Create Governance Project
---------------------------------
  oc new-project gitops-configure

Step 3: Configure ACM Cluster Set & Bindings
--------------------------------------------
  1. Go to Infrastructure -> Clusters, select "Cluster sets" tab, click "Create cluster set".
  2. Set name to "gitops-configure" and click Create.
  3. Click "Manage resource assignments", select target clusters, click Review, and Save.
  4. Bind cluster set to namespace: Actions -> Edit namespace bindings.
  5. Select "gitops-configure" namespace and click Save.

Step 4: Manifest Files Setup
----------------------------
Directory Contents:
  - ca-bundle.yaml
  - cluster-role-binding.yaml
  - gitops-operator.yaml
  - policy-generator.yaml
  - argocd-configuration.yaml

Manifest Contents:

---
### argocd-configuration.yaml
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

---
### ca-bundle.yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: cluster-root-ca-bundle
  namespace: openshift-gitops
  labels:
    config.openshift.io/inject-trusted-cabundle: 'true'

---
### cluster-role-binding.yaml
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

---
### gitops-operator.yaml
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

---
### policy-generator.yaml
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

Step 5: Apply Policy & Monitor Remediation
------------------------------------------
1. Generate and apply policies:
     PolicyGenerator policy-generator.yaml | oc apply -f -
2. Switch to RHACM web console -> Governance -> Policies tab.
3. Wait for all clusters to become compliant (up to 5 minutes).

Step 6: Import Clusters into Argo CD
------------------------------------
Apply the registration manifest (`gitops-register.yaml`):

### gitops-register.yaml
---
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

Command:
  oc apply -f gitops-register.yaml

Output:
  managedclustersetbinding.cluster.open-cluster-management.io/gitops-configure created
  placement.cluster.open-cluster-management.io/gitops-configure created
  gitopscluster.apps.open-cluster-management.io/gitops-configure created


-------------------------------------------------------------------------------
5. USEFUL OPERATOR DISCOVERY COMMANDS
-------------------------------------------------------------------------------

To find operator CSV, channel, catalog source, startingCSV details:

Commands:
  oc get packagemanifests openshift-gitops-operator -n openshift-marketplace -o yaml
  # or
  oc get packagemanifests -n openshift-marketplace

Filtering relevant fields:
  oc get packagemanifests openshift-gitops-operator -n openshift-marketplace -o yaml | grep -i -E "channel|source:|sourceNamespace|startingCSV|namespace:"

Example output:
    catalog-namespace: openshift-marketplace
  namespace: openshift-marketplace
  catalogSource: redhat-operators
  catalogSourceNamespace: openshift-marketplace
  channels:
        operatorframework.io/suggested-namespace: openshift-gitops-operator
        operatorframework.io/suggested-namespace: openshift-gitops-operator
        ...
  defaultChannel: latest
===============================================================================
