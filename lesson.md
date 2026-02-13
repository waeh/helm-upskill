# Helm Upskill Session: Creating Your First Chart (built partly with AI)

## How to Create a New Helm Chart

## Lesson 1: Getting started

### Prerequisites
Before we start, ensure you have Helm installed and configured. You can check with:
```bash
helm version
```

If not, install with on macOS:
```bash
brew install helm
```

### Step 1: Initialize a New Helm Chart
Helm provides a `helm create` command that scaffolds a basic chart structure for you. This gives you a starting point with all the necessary files and directories.

Run this command in your terminal:
```bash
helm create umbrella
```

This will create a directory called `umbrella` with the following structure:
```
umbrella/
├── Chart.yaml          # Chart metadata
├── values.yaml         # Default values
├── charts/             # Directory for subcharts
├── templates/          # Kubernetes manifest templates
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── _helpers.tpl    # Helper templates
│   └── ...
└── .helmignore         # Files to ignore when packaging
```

### Step 2: Understanding the Generated Files

**Chart.yaml**: Contains metadata about your chart
- `name`: Chart name
- `version`: Chart version (follows SemVer)
- `description`: What the chart does
- `appVersion`: Version of the app this chart deploys

**values.yaml**: Default configuration values that users can override

**templates/**: Contains Go template files that generate Kubernetes manifests

### Step 3: Customize Your Chart
1. Edit `Chart.yaml` to match your application
2. Modify `values.yaml` with your app's default settings
3. Update the templates in `templates/` to match your Kubernetes resources

### Step 4: Test Your Chart
Use `helm template` to see what manifests your chart would generate:
```bash
helm template my-release ./umbrella.chart
```

### Common Pitfalls
- Don't forget to update the `appVersion` when you update your application
- Keep `values.yaml` clean and focused on user-configurable options
- Use helper templates for reusable logic

### Next Steps
Once you've created your basic chart, we'll explore how to turn it into an umbrella chart that can manage multiple components. Try creating the chart now and let me know what you see!

## Lesson 2: Building an Umbrella Chart Structure

### What is an Umbrella Chart?
An umbrella chart is a Helm chart that doesn't deploy anything itself but orchestrates multiple subcharts. It's like a "chart of charts" that provides:

- A single point of deployment for complex applications
- Centralized configuration management
- Dependency management between components
- Clean abstraction over internal complexity

https://codefresh.io/docs/images/guides/helm-best-practices/chart-structure.png

### Step 1: Create Application Subchart
Let's create a dedicated application chart that represents your main service. This keeps the umbrella chart purely orchestrational.

Create a new Helm chart for your application:
```bash
helm create myapp
```

This creates a `myapp/` directory with the standard chart structure. You can think of this as representing your main application component (frontend, backend, or whatever your primary service is).

### Step 2: Add Subchart to Umbrella Chart
Modify `umbrella-chart/Chart.yaml` to include the application chart as a dependency:

```yaml
apiVersion: v2
name: umbrella
description: Umbrella chart for multi-component application
type: application
version: 0.1.0
appVersion: "1.0"

dependencies:
  - name: myapp
    version: "0.1.0"
    repository: "file://./charts/myapp"
  # Add more dependencies here as needed
```

Move the `myapp` chart into the `umbrella-chart/charts/` directory:
```bash
mv myapp umbrella-chart/charts/
```

### Step 3: Redesign Umbrella values.yaml for Abstraction
Replace the umbrella chart's `values.yaml` with a clean, intent-driven interface:

```yaml
# Clean, abstracted values.yaml - no Kubernetes mechanics
global:
  environment: production
```

Notice how this values.yaml:
- Hides all Kubernetes resource details
- Expresses intent clearly
- Can evolve internally without changing the interface

### Step 4: Update Umbrella Templates
The umbrella chart's `templates/` directory should contain:
- Helper templates for global configuration
- Conditional logic for enabling/disabling components
- Any cross-component resources (like network policies)

For now, you can remove the default templates from `umbrella-chart/templates/` since the umbrella won't deploy anything directly.

### Step 5: Test with helm template
```bash
# Update dependencies
cd umbrella-chart
helm dependency update

# Test rendering
helm template myapp-release .
```

### Lets compare umbrella and sub-chart 

```bash
helm template umbrella . > umbrella-output.yaml && helm template myapp ./charts/myapp > myapp-output.yaml && diff -u umbrella-output.yaml myapp-output.yaml
```

## Lesson 3: Helper Templates and Named Templates

### What Are Helper Templates?
Helper templates (defined in `_helpers.tpl`) are reusable template snippets that you can call from any template in your chart. They're like functions in programming - they encapsulate common logic and make your templates DRY (Don't Repeat Yourself).

### Why Use Helper Templates?
- **Abstraction**: Hide complex logic behind simple names
- **Reusability**: Use the same logic across multiple templates
- **Maintainability**: Change logic in one place
- **Readability**: Make main templates cleaner

### Basic Helper Template Syntax

### 1. Helpers templates file

Lets copy the _helpers.tpl file in myapp templated and into a newly create _helpers.tpl file in umbrella templates.

Lets replace the name myapp with umbrella in the _helpers.tpl file.

In the expansion of the .Chart name and in the generation of a full name
the generated _helpers.tpl file carries a template variable from the values file for name and full name override, add those to the umbrella values:

```yaml
nameOverride: ""
fullnameOverride: ""
```
### 2. Global resources

Continue with creating a global resource that can be used by all in the umbrella template:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "umbrella.fullname" . }}-config
  labels:
    {{- include "umbrella.labels" . | nindent 4 }}
data:
  # Add your configuration data here
  # For example:
  config.yaml: |-
    key: value    
```

Lets run the helm command again to verify that our resource is templated correctly using the newly created reusable template naming:

```bash 
helm template umbrella . 
```
### 3. Global values

Let's use the values defined in `umbrella-chart/values.yaml`. You have:

```yaml
global:
  environment: production
```

Insert:
```yaml
environment: {{ .Values.global.environment }}
```
In the newly create _helpers.tpl, in the labels section.

Verify with:
```bash 
helm template umbrella . 
```

Since we also want to use the environment label in myapp sub chart aswell, add the same label to the myapp _helpers.tpl file.

Verify with:
```bash 
helm template umbrella . 
```

### Best Practices
- Use the chart name as prefix: `{{ .Chart.Name }}.helperName`
- Use the chart AppVersion to controll version labels: `{{ .Chart.AppVersion | quote }}`
- Document complex helpers with comments

## Lesson 4: Conditionals

### 1. Global conditionals
We want the umbrella values to be intent-driven, meaning that it should hide kubernetes internals behind an abstraction that makes sense to developers that has limited experience in kubernetes.

In the umbrella values file, lets create a myapp definition with enable boolean:

```yaml
myapp:
  enabled: false
```

In umbrella charts.yaml, add a condition to the subchart to disable the subchart completely.

```yaml
dependencies:
  - name: myapp
    version: "0.1.0"
    repository: "file://./charts/myapp"
    condition: myapp.enabled
```

Now only the umbrella specific templates should be deployed:
```bash 
helm template umbrella . 
```

### 2. Cross-Component Resources
These are Kubernetes resources that span multiple components, like network policies, config maps shared across services, or ingress controllers that route to multiple apps.

There are some resources in the subchart that we want to add to the global resources.

Remove the ingress.yaml resource to the global scope, in umbrella/templates. Create a new ingress resource in the global resources:

```yaml
# umbrella-chart/templates/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "umbrella.fullname" . }}
spec:
  rules:
  {{- if .Values.myapp.enabled }}
  - host: {{ .Values.myapp.host | default "myapp.local" }}
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: {{ include "umbrella.fullname" . }}-myapp
            port:
              number: {{ .Values.myapp.service.port }}
  {{- end }}
  - host: frontend
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: {{ include "umbrella.fullname" . }}-frontend
            port:
              number: 8080
```

Lets also create a Global network policy and a namespace level RBAC:

```yaml
# umbrella-chart/templates/network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: {{ include "umbrella.fullname" . }}-network-policy
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  {{- if .Values.myapp.enabled }}
  - from:
    - podSelector:
        matchLabels:
          app: myapp
    ports:
    - protocol: TCP
      port: {{ .Values.myapp.port }}
  {{- end }}
  - from:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 1433
```

### Best Practices for Cross-Component Resources
- Use conditional blocks to only create resources when relevant components are enabled
- Reference subchart values through the umbrella's values.yaml
- Use consistent naming conventions
- Test with different component combinations


## Lesson 5: Namespace-Level RBAC

RBAC (Role-Based Access Control) controls **who or what can do what** inside a namespace. On AKS this is critical because pods run with default service accounts that may have too many (or too few) permissions.

There are four RBAC resources. They come in pairs:

| Scope | Role | Binding |
|-------|------|---------|
| **Namespace** | `Role` | `RoleBinding` |
| **Cluster-wide** | `ClusterRole` | `ClusterRoleBinding` |

For namespace-level RBAC (what most apps need), you use `Role` + `RoleBinding`.

### 1. Understanding the pieces

```
Role                          = "What actions are allowed on which resources"
ServiceAccount                = "An identity for pods"
RoleBinding                   = "Grants a Role to a ServiceAccount"
```

A pod runs as a ServiceAccount. The RoleBinding says: *"This ServiceAccount is allowed to do what this Role describes."*

### 2. Add RBAC values to umbrella values.yaml

Add an `rbac` section to your umbrella `values.yaml`:

```yaml
# umbrella-chart/values.yaml
rbac:
  create: true

myapp:
  enabled: true
  # ... existing values ...
```

### 3. Create a Role

A Role lists **rules** — each rule says which API groups, resources, and verbs are permitted.

Create this file:

```yaml
# umbrella-chart/templates/role.yaml
{{- if .Values.rbac.create }}
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: {{ include "umbrella-chart.fullname" . }}-app-role
  labels:
    {{- include "umbrella-chart.labels" . | nindent 4 }}
rules:
  # Allow pods to read ConfigMaps in this namespace
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]

  # Allow pods to read their own service endpoints
  - apiGroups: [""]
    resources: ["services", "endpoints"]
    verbs: ["get", "list"]

  # No write access, no pod exec, no escalation
{{- end }}
```

**Key concepts:**
- `apiGroups: [""]` — the core API group (pods, services, configmaps, secrets, etc.)
- `apiGroups: ["apps"]` — would target deployments, replicasets, etc.
- `verbs` — the allowed actions: `get`, `list`, `watch`, `create`, `update`, `patch`, `delete`
- **Least privilege:** only grant what the app actually needs

### 4. Create a RoleBinding

The RoleBinding **connects** the Role to a ServiceAccount:

```yaml
# umbrella/templates/rolebinding.yaml
{{- if .Values.rbac.create }}
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: {{ include "umbrella.fullname" . }}-app-binding
  labels:
    {{- include "umbrella.labels" . | nindent 4 }}
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: {{ include "umbrella.fullname" . }}-app-role
subjects:
  # Bind to myapp's service account
  {{- if .Values.myapp.enabled }}
  - kind: ServiceAccount
    name: {{ .Release.Name }}-myapp    # subchart's auto-generated SA
    namespace: {{ .Release.Namespace }}
  {{- end }}
  # Add more subjects here as you add more subcharts:
  # {{- if .Values.frontend.enabled }}
  # - kind: ServiceAccount
  #   name: {{ .Release.Name }}-frontend
  #   namespace: {{ .Release.Namespace }}
  # {{- end }}
{{- end }}
```

**Why this lives in the umbrella:** RBAC is a cross-component concern. Multiple subcharts share the same Role. The umbrella owns the policy; subcharts just own their ServiceAccounts.

### 5. Verify with helm template

```bash
# See the RBAC resources rendered:
helm template umbrella . -s templates/role.yaml
helm template umbrella . -s templates/rolebinding.yaml

# Confirm they disappear when rbac.create is false:
helm template umbrella . --set rbac.create=false -s templates/role.yaml
# Should output empty
```


### 7. AKS-specific notes

- **Azure AD integration:** On AKS with Azure AD enabled, you can also bind to Azure AD groups using `ClusterRoleBinding` with `kind: Group` subjects. That's cluster-level, not namespace.
- **Default SA:** Every namespace has a `default` ServiceAccount. Without RBAC, pods using it may have more access than intended.
- **Pod Identity:** For accessing Azure resources (Key Vault, Storage), we use **Workload Identity** (the modern replacement for pod-managed identity). That's a separate concern from Kubernetes RBAC — covered in Lesson 6 below.
- **Pod disruption budget:** We want to be able to allow the AKS worker node-pools (where your container lives) to survive node upgrades.

---

## Lesson 6: Azure Workload Identity 

Workload Identity lets pods authenticate to Azure services (Key Vault, Storage, SQL) **without secrets**. Instead of storing credentials, the pod exchanges a Kubernetes service account token for an Azure AD token.


1. **Annotate the ServiceAccount** with the Managed Identity's client ID
2. **Label the pod** with `azure.workload.identity/use: "true"`

### 1. Add workload identity values to umbrella values.yaml

Add the Workload Identity config under the `myapp:` section in the umbrella `values.yaml`:

```yaml
# umbrella/values.yaml
myapp:
  enabled: true

  # Workload Identity — set clientId to your Azure Managed Identity
  serviceAccount:
    create: true
    annotations:
      azure.workload.identity/client-id: "<your-managed-identity-client-id>"

  podLabels:
    azure.workload.identity/use: "true"
```

Verify that the serviceaccount is created correctly with the workload identiy annotations and pod labels: 

```bash
helm template umbrella . -s charts/myapp/templates/serviceaccount.yaml
```

```bash
helm template umbrella . -s charts/myapp/templates/deployment.yaml
```

## Lesson 7: Pod Disruption Budgets

### 1. Add PDB values to myapp subchart

The PDB belongs to the **subchart** because each component has its own availability requirements. Add to the myapp `values.yaml`:

```yaml
# umbrella-chart/charts/myapp/values.yaml
podDisruptionBudget:
  enabled: true
  minAvailable: 1
  # maxUnavailable: 1  # Alternative to minAvailable
```

### 2. Create the PDB template

Create `umbrella-chart/charts/myapp/templates/poddisruptionbudget.yaml`:

```yaml
{{- if .Values.podDisruptionBudget.enabled }}
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  {{- if .Values.podDisruptionBudget.minAvailable }}
  minAvailable: {{ .Values.podDisruptionBudget.minAvailable }}
  {{- end }}
  {{- if .Values.podDisruptionBudget.maxUnavailable }}
  maxUnavailable: {{ .Values.podDisruptionBudget.maxUnavailable }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
{{- end }}
```

### 3. Test with helm template

```bash
# Render the PDB:
helm template umbrella . -s charts/myapp/templates/poddisruptionbudget.yaml
```

## Lesson 8: Push helm charts to repos

We are ready with our charts and would like other make it consumable, you need to package and publish it so others can install it with `helm install`.

### 1. Login to registry

Login to the registry you want to use for Helm chart hosting.

```bash
helm registry login myregistry.azurecr.io
```

### 2. Package the chart

```bash
# Package the umbrella chart (creates .tgz file)
helm package umbrella
```

### 2. Push to the repo

```bash
helm push umbrella-0.1.0.tgz oci://myregistry.azurecr.io/helm
```

### 3. Install from ACR

```bash
helm install umbrella oci://myregistry.azurecr.io/helm/umbrella --version 0.1.0
```

### 3. Commit and push to GitHub

The code should live separate from the artifacts, lets
push them to GH.

```bash
git init

git add .

git commit -m "Release umbrella chart v0.1.0"

git push origin main
```