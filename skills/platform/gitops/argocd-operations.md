# Platform — GitOps: ArgoCD Operations

RBAC, SSO, sync policies, notifications, CLI usage, Talos bootstrap, and monitoring for ArgoCD. For installation, Application CRD patterns, and ApplicationSet, see the [main ArgoCD guide](argocd.md).

---

## 1. RBAC Configuration

ArgoCD RBAC uses a Casbin-based policy model. Policies are defined in the `argocd-rbac-cm` ConfigMap.

### Policy Format

```
p, <subject>, <resource>, <action>, <object>, <effect>
g, <user/group>, <role>
```

Resources: `applications`, `clusters`, `repositories`, `projects`, `logs`, `exec`, `extensions`
Actions: `get`, `create`, `update`, `delete`, `sync`, `override`, `action/*`
Effect: `allow` or `deny`

### Example RBAC Policies

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:readonly
  policy.csv: |
    # Platform admins — full access
    p, role:platform-admin, applications, *, */*, allow
    p, role:platform-admin, clusters, *, *, allow
    p, role:platform-admin, repositories, *, *, allow
    p, role:platform-admin, projects, *, *, allow
    p, role:platform-admin, logs, get, *, allow
    p, role:platform-admin, exec, create, */*, allow

    # Developers — view all, sync own project apps
    p, role:developer, applications, get, */*, allow
    p, role:developer, applications, sync, apps/*, allow
    p, role:developer, logs, get, */*, allow

    # Team-specific — full access to their project only
    p, role:team-alpha, applications, *, apps-alpha/*, allow
    p, role:team-alpha, logs, get, apps-alpha/*, allow
    p, role:team-alpha, exec, create, apps-alpha/*, allow

    # Map OIDC groups to roles
    g, platform-admins, role:platform-admin
    g, developers, role:developer
    g, team-alpha, role:team-alpha

  # Scopes controls which OIDC claims are used for group mapping
  scopes: "[groups, email]"
```

### AppProject for Tenant Isolation

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: apps-alpha
  namespace: argocd
spec:
  description: "Team Alpha applications"
  sourceRepos:
    - https://github.com/your-org/team-alpha-*.git
    - https://charts.bitnami.com/bitnami
  destinations:
    - namespace: team-alpha-*
      server: https://kubernetes.default.svc
  clusterResourceWhitelist: []
  namespaceResourceBlacklist:
    - group: ""
      kind: ResourceQuota
    - group: ""
      kind: LimitRange
    - group: ""
      kind: NetworkPolicy
  roles:
    - name: admin
      description: "Team Alpha admin"
      policies:
        - p, proj:apps-alpha:admin, applications, *, apps-alpha/*, allow
      groups:
        - team-alpha
```

---

## 2. SSO Configuration

ArgoCD supports SSO via OIDC (direct) or Dex (bundled identity broker). Dex supports OIDC, SAML, LDAP, GitHub, GitLab, and more.

### OIDC Direct (e.g., Keycloak, Okta, Azure AD)

```yaml
# In argocd-cm ConfigMap or Helm values
configs:
  cm:
    url: https://argocd.example.com
    oidc.config: |
      name: Keycloak
      issuer: https://keycloak.example.com/realms/platform
      clientID: argocd
      clientSecret: $oidc.keycloak.clientSecret
      requestedScopes:
        - openid
        - profile
        - email
        - groups
```

```yaml
# Store the client secret in argocd-secret
apiVersion: v1
kind: Secret
metadata:
  name: argocd-secret
  namespace: argocd
type: Opaque
stringData:
  oidc.keycloak.clientSecret: "your-client-secret-here"
```

### Dex with GitHub

```yaml
configs:
  cm:
    url: https://argocd.example.com
    dex.config: |
      connectors:
        - type: github
          id: github
          name: GitHub
          config:
            clientID: your-github-oauth-app-client-id
            clientSecret: $dex.github.clientSecret
            orgs:
              - name: your-org
                teams:
                  - platform-admins
                  - developers
```

### Dex with LDAP

```yaml
configs:
  cm:
    dex.config: |
      connectors:
        - type: ldap
          id: ldap
          name: Active Directory
          config:
            host: ldap.example.com:636
            insecureNoSSL: false
            insecureSkipVerify: false
            rootCAData: <base64-encoded-ca-cert>
            bindDN: cn=argocd,ou=service-accounts,dc=example,dc=com
            bindPW: $dex.ldap.bindPW
            userSearch:
              baseDN: ou=users,dc=example,dc=com
              filter: "(objectClass=person)"
              username: sAMAccountName
              idAttr: sAMAccountName
              emailAttr: mail
              nameAttr: displayName
            groupSearch:
              baseDN: ou=groups,dc=example,dc=com
              filter: "(objectClass=group)"
              userMatchers:
                - userAttr: DN
                  groupAttr: member
              nameAttr: cn
```

---

## 3. Sync Policies

### Automated Sync

```yaml
syncPolicy:
  automated:
    prune: true        # Delete resources removed from Git
    selfHeal: true     # Revert manual changes made outside Git
    allowEmpty: false   # Prevent sync if source produces no resources (safety)
```

### Sync Options

```yaml
syncPolicy:
  syncOptions:
    - CreateNamespace=true           # Create target namespace if missing
    - PrunePropagationPolicy=foreground  # Wait for dependents before pruning
    - PruneLast=true                 # Prune after all other resources sync
    - ServerSideApply=true           # Use server-side apply (handles large CRDs)
    - ApplyOutOfSyncOnly=true        # Only apply resources that differ
    - RespectIgnoreDifferences=true  # Respect ignoreDifferences during sync
    - Replace=false                  # Use apply instead of replace
```

### Retry Policy

```yaml
syncPolicy:
  retry:
    limit: 5
    backoff:
      duration: 5s
      factor: 2
      maxDuration: 3m
```

### Ignore Differences

For resources that are modified by controllers at runtime (e.g., HPA modifying replicas):

```yaml
spec:
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
    - group: admissionregistration.k8s.io
      kind: MutatingWebhookConfiguration
      jqPathExpressions:
        - .webhooks[]?.clientConfig.caBundle
```

---

## 4. Notifications

ArgoCD notifications are configured via the `argocd-notifications-cm` ConfigMap (or via Helm values).

### Slack Notifications

```yaml
configs:
  notifications:
    enabled: true
    secret:
      items:
        slack-token: xoxb-xxxxxxxxxxxx-xxxxxxxxxxxx-xxxxxxxxxxxxxxxxxxxxxxxx
    notifiers:
      service.slack: |
        token: $slack-token
    triggers:
      trigger.on-sync-succeeded: |
        - when: app.status.operationState.phase in ['Succeeded']
          send: [app-sync-succeeded]
      trigger.on-sync-failed: |
        - when: app.status.operationState.phase in ['Error', 'Failed']
          send: [app-sync-failed]
      trigger.on-health-degraded: |
        - when: app.status.health.status == 'Degraded'
          send: [app-health-degraded]
    templates:
      template.app-sync-succeeded: |
        slack:
          attachments: |
            [{
              "color": "#18be52",
              "title": "{{ .app.metadata.name }} synced successfully",
              "text": "Application {{ .app.metadata.name }} sync succeeded.\nRevision: {{ .app.status.sync.revision }}"
            }]
      template.app-sync-failed: |
        slack:
          attachments: |
            [{
              "color": "#E96D76",
              "title": "{{ .app.metadata.name }} sync failed",
              "text": "Application {{ .app.metadata.name }} sync failed.\nMessage: {{ .app.status.operationState.message }}"
            }]
      template.app-health-degraded: |
        slack:
          attachments: |
            [{
              "color": "#f4c030",
              "title": "{{ .app.metadata.name }} health degraded",
              "text": "Application {{ .app.metadata.name }} is degraded."
            }]
    subscriptions:
      - recipients:
          - slack:platform-alerts
        triggers:
          - on-sync-succeeded
          - on-sync-failed
          - on-health-degraded
```

### Annotate Applications for Notifications

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-a
  namespace: argocd
  annotations:
    notifications.argoproj.io/subscribe.on-sync-succeeded.slack: platform-alerts
    notifications.argoproj.io/subscribe.on-sync-failed.slack: platform-alerts
```

---

## 5. CLI Commands

### Installation

```bash
# macOS
brew install argocd

# Linux
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd
sudo mv argocd /usr/local/bin/
```

### Authentication

```bash
# Login to ArgoCD server
argocd login argocd.example.com --username admin --password <password>

# Login with SSO (opens browser)
argocd login argocd.example.com --sso

# Check current context
argocd context
```

### Application Management

```bash
# List all applications
argocd app list

# Get detailed status of an application
argocd app get app-a

# Show diff between live state and desired state
argocd app diff app-a

# Sync an application (reconcile from Git)
argocd app sync app-a

# Sync with prune (delete resources removed from Git)
argocd app sync app-a --prune

# Sync specific resources only
argocd app sync app-a --resource apps:Deployment:nginx

# Force sync (even if already in sync)
argocd app sync app-a --force

# Hard refresh (clear cache and re-render manifests)
argocd app get app-a --hard-refresh

# View application logs
argocd app logs app-a --follow

# View application events
argocd app resources app-a

# Delete an application (and optionally its resources)
argocd app delete app-a
argocd app delete app-a --cascade   # Also delete managed resources

# Create an application from CLI
argocd app create nginx \
  --repo https://github.com/your-org/cluster-config.git \
  --path apps/nginx \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace nginx \
  --sync-policy automated \
  --auto-prune \
  --self-heal

# Set application parameters
argocd app set app-a --values-literal-file values-override.yaml
argocd app set app-a -p image.tag=v2.0.0
```

### Project Management

```bash
# List projects
argocd proj list

# Get project details
argocd proj get apps-alpha

# Add a source repo to a project
argocd proj add-source apps-alpha https://github.com/your-org/new-repo.git

# Add a destination to a project
argocd proj add-destination apps-alpha https://kubernetes.default.svc team-alpha-new
```

### Cluster Management

```bash
# List registered clusters
argocd cluster list

# Add a cluster (uses current kubeconfig context)
argocd cluster add my-other-cluster --name production-west

# Remove a cluster
argocd cluster rm https://10.0.0.1:6443
```

### Repository Management

```bash
# List registered repositories
argocd repo list

# Add a private repository with HTTPS
argocd repo add https://github.com/your-org/private-repo.git \
  --username git \
  --password ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# Add a private repository with SSH
argocd repo add git@github.com:your-org/private-repo.git \
  --ssh-private-key-path ~/.ssh/id_ed25519

# Add a Helm repository
argocd repo add https://charts.bitnami.com/bitnami \
  --type helm \
  --name bitnami
```

### Account Management

```bash
# List accounts
argocd account list

# Get current user info
argocd account get-user-info

# Generate an API token for automation
argocd account generate-token --account automation

# Update password
argocd account update-password --account admin
```

---

## 6. ArgoCD on Talos — Bootstrap via inlineManifests

To bootstrap ArgoCD directly from the Talos machine config, include the ArgoCD install manifests and a root Application in `cluster.inlineManifests`.

### Generate ArgoCD Install Manifests

```bash
# Download the ArgoCD install manifest for a specific version
curl -sSL -o argocd-install.yaml \
  https://raw.githubusercontent.com/argoproj/argo-cd/v2.14.3/manifests/install.yaml

# Check the size
wc -l argocd-install.yaml

# For HA installation
curl -sSL -o argocd-install-ha.yaml \
  https://raw.githubusercontent.com/argoproj/argo-cd/v2.14.3/manifests/ha/install.yaml
```

### Talos Machine Config

```yaml
# controlplane.yaml (relevant section only)
cluster:
  inlineManifests:
    - name: argocd-namespace
      contents: |
        apiVersion: v1
        kind: Namespace
        metadata:
          name: argocd
          labels:
            pod-security.kubernetes.io/warn: restricted
            pod-security.kubernetes.io/warn-version: latest
    - name: argocd-install
      contents: |
        # Paste the entire contents of argocd-install.yaml here.
        # The standard install is ~10000 lines containing:
        #   - CRDs: Application, ApplicationSet, AppProject
        #   - Deployments: argocd-server, argocd-repo-server,
        #                  argocd-application-controller, argocd-applicationset-controller,
        #                  argocd-notifications-controller, argocd-redis, argocd-dex-server
        #   - Services, ServiceAccounts, ConfigMaps, Secrets
        #   - ClusterRoles, ClusterRoleBindings, Roles, RoleBindings
        #   - NetworkPolicies
    - name: argocd-repo-secret
      contents: |
        apiVersion: v1
        kind: Secret
        metadata:
          name: private-repo
          namespace: argocd
          labels:
            argocd.argoproj.io/secret-type: repository
        type: Opaque
        stringData:
          type: git
          url: https://github.com/your-org/cluster-config.git
          username: git
          password: ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
    - name: argocd-root-app
      contents: |
        apiVersion: argoproj.io/v1alpha1
        kind: Application
        metadata:
          name: root
          namespace: argocd
          finalizers:
            - resources-finalizer.argocd.argoproj.io
        spec:
          project: default
          source:
            repoURL: https://github.com/your-org/cluster-config.git
            targetRevision: main
            path: clusters/production
          destination:
            server: https://kubernetes.default.svc
            namespace: argocd
          syncPolicy:
            automated:
              prune: true
              selfHeal: true
            retry:
              limit: 5
              backoff:
                duration: 5s
                factor: 2
                maxDuration: 3m
```

After Talos boots the cluster, ArgoCD starts, connects to the Git repository, and begins syncing all applications defined under `clusters/production/`. ArgoCD then manages itself — its own Helm release or manifests are included in the repository and reconciled like any other application.

---

## 7. Monitoring ArgoCD

### Prometheus ServiceMonitor

The Helm chart enables ServiceMonitors when `metrics.serviceMonitor.enabled: true` is set (shown in the Helm values in the [main ArgoCD guide](argocd.md)). If installing from raw manifests, create ServiceMonitors manually:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-metrics
  namespace: argocd
  labels:
    release: kube-prometheus-stack
spec:
  selector:
    matchLabels:
      app.kubernetes.io/part-of: argocd
  endpoints:
    - port: metrics
      interval: 30s
```

### Key Metrics

| Metric | Description |
|---|---|
| `argocd_app_info` | Application metadata (name, project, sync status, health) |
| `argocd_app_sync_total` | Total sync operations per application |
| `argocd_app_reconcile_count` | Reconciliation count per application |
| `argocd_app_health_status` | Current health status (Healthy, Degraded, Missing, Unknown) |
| `argocd_cluster_api_resource_objects` | Number of API resource objects per cluster |
| `argocd_git_request_total` | Total Git requests (fetch, ls-remote) |
| `argocd_git_request_duration_seconds` | Git request latency |

### Grafana Dashboard

Import ArgoCD's official Grafana dashboard: ID `14584` (ArgoCD) from grafana.com.
