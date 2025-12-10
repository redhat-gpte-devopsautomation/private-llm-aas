# Multi-User Setup Documentation

## Overview

This repository has been refactored to support multiple users with isolated workspaces and LLama Stack instances, following the pattern from etx-agentic-ai-gitops.

## Architecture

### Three-Tier Deployment Model

1. **Cluster-Wide Resources** (deployed once)
   - LLama Stack Operator
   - Shared MCP servers
   - Shared namespace: `lls-demo`

2. **Per-User Resources** (deployed N times)
   - DevWorkspace in `wksp-{username}` namespace
   - LLama Stack instance in `lls-{username}` namespace
   - LLama Stack Playground UI

3. **Bootstrap Layer**
   - ArgoCD ApplicationSets that generate per-user Applications

## Directory Structure

```
private-llmaas-multitenant/
├── bootstrap/                      # ArgoCD bootstrap (deploy once)
│   ├── templates/
│   │   ├── application-llama-stack-operator.yaml    # Operator (cluster-wide)
│   │   ├── application-llama-stack-shared.yaml      # MCP servers (shared)
│   │   ├── applicationset-workspace.yaml            # Per-user workspaces
│   │   └── applicationset-llama-stack-instance.yaml # Per-user LLama Stack
│   └── values.yaml                 # Central configuration
│
├── llama-stack-operator/          # Cluster-wide operator
│   ├── templates/
│   │   ├── crd.yaml               # LlamaStackDistribution CRD
│   │   ├── deployment.yaml        # Operator deployment
│   │   └── ...                    # RBAC, ServiceAccount, etc.
│   └── Chart.yaml
│
├── llama-stack-instance/          # Per-user LLama Stack
│   ├── templates/
│   │   ├── namespace.yaml         # lls-{username}
│   │   ├── configmap.yaml         # LLama Stack config
│   │   ├── llamastack-distribution.yaml  # LlamaStackDistribution CR
│   │   └── llama-stack-playground/       # Playground UI
│   └── Chart.yaml
│
├── llama-stack/                   # Shared resources (MCP)
│   ├── templates/
│   │   ├── namespace-lls-demo.yaml
│   │   └── openshift-mcp/         # Shared MCP server
│   └── Chart.yaml
│
└── workspace/                     # Per-user DevWorkspace
    ├── templates/
    │   ├── namespaces.yaml        # wksp-{username}
    │   ├── devworkspace.yaml      # IDE workspace
    │   └── rbac.yaml              # User permissions
    └── Chart.yaml
```

## Configuration

### User Count

Edit `bootstrap/values.yaml`:

```yaml
user:
  count: 2      # Number of users
  prefix: user  # Username prefix (generates user1, user2, ...)
```

### Model Endpoints

Configure in `bootstrap/values.yaml`:

```yaml
llamaStackInstance:
  granite:
    modelName: granite-3dot2-8b-instruct
    endpoint: "https://maas.apps.ocp.example.com/llm/granite-3dot2-8b-instruct"
    apiToken: "your-api-token"
    tlsverify: true
  llamaguard:
    modelName: llama-guard-3-1b
    endpoint: "https://maas.apps.ocp.example.com/llm/llama-guard-3-1b"
    apiToken: "your-api-token"
    tlsverify: true
```

### MCP Server Endpoints

```yaml
llamaStackInstance:
  mcp:
    openshiftServerUrl: http://ocp-mcp-server.lls-demo.svc.cluster.local:8000/sse
    slackServerUrl: http://slack-mcp-server.lls-demo.svc.cluster.local:80/sse
```

## Deployment

### Step 1: Deploy Bootstrap

```bash
# Deploy the bootstrap chart (creates all ArgoCD Applications/ApplicationSets)
helm install bootstrap ./bootstrap -n openshift-gitops
```

This creates:
- 1 Application for llama-stack-operator (cluster-wide)
- 1 Application for llama-stack-shared (MCP servers)
- N Applications for workspaces (one per user)
- N Applications for llama-stack-instances (one per user)

### Step 2: Verify Deployments

```bash
# Check ArgoCD Applications
oc get applications -n openshift-gitops

# Expected output:
# llama-stack-operator        Synced
# llama-stack-shared          Synced
# workspace-user1             Synced
# workspace-user2             Synced
# llama-stack-instance-user1  Synced
# llama-stack-instance-user2  Synced
```

### Step 3: Access User Resources

Each user gets:

**Namespaces:**
- `wksp-{username}` - DevWorkspace (IDE)
- `lls-{username}` - LLama Stack instance

**Routes:**
- LLama Stack Playground: `https://llama-stack-playground-{username}.apps.cluster.example.com`

## What Changed from Single-User to Multi-User

### Before (Single-User)

- Single `lls-demo` namespace
- One LLama Stack instance
- One playground
- All resources shared

### After (Multi-User)

- Shared `lls-demo` namespace (MCP servers only)
- Per-user `lls-{username}` namespace
- Per-user LLama Stack instance
- Per-user playground
- Isolated resources per user

## Key Features

### 1. Authentication Fix

Changed vLLM provider from `remote::vllm` to `remote::openai` with custom headers:

```yaml
providers:
  inference:
  - provider_id: vllm
    provider_type: remote::openai  # Changed!
    config:
      url: ${env.GRANITE_URL}
      api_token: ${env.GRANITE_TOKEN}
      default_headers:
        api-key: ${env.GRANITE_TOKEN}  # Custom header for vLLM
```

This fixes the 401 authentication errors by sending the correct `api-key` header instead of `Authorization: Bearer`.

### 2. Shared MCP Servers

MCP servers are deployed once in `lls-demo` namespace and referenced by all users via fully qualified service names:

```yaml
mcp_endpoint:
  uri: http://ocp-mcp-server.lls-demo.svc.cluster.local:8000/sse
```

### 3. Per-User Service Discovery

Each user's playground connects to their own LLama Stack instance:

```yaml
env:
  - name: LLAMA_STACK_ENDPOINT
    value: "http://llamastack-{{ .Values.username }}:8321"
```

## Namespaces

### Cluster-Wide

- `llama-stack-k8s-operator-controller-manager` - Operator
- `lls-demo` - Shared MCP servers

### Per-User (Example: user1, user2)

- `wksp-user1`, `wksp-user2` - DevWorkspaces
- `lls-user1`, `lls-user2` - LLama Stack instances

## Troubleshooting

### Check ApplicationSet Generation

```bash
# View generated Applications
oc get applications -n openshift-gitops -l argocd.argoproj.io/application-set-name=llama-stack-instance
```

### Check User Namespace

```bash
# List resources in user namespace
oc get all -n lls-user1
```

### View LLama Stack Logs

```bash
# Get logs from user's LLama Stack pod
oc logs -n lls-user1 deployment/llamastack-user1
```

### Check MCP Server

```bash
# Verify MCP server is running
oc get pods -n lls-demo
oc logs -n lls-demo deployment/ocp-mcp-server
```

## Adding More Users

1. Edit `bootstrap/values.yaml`:
   ```yaml
   user:
     count: 5  # Increase from 2 to 5
   ```

2. Upgrade bootstrap:
   ```bash
   helm upgrade bootstrap ./bootstrap -n openshift-gitops
   ```

3. ArgoCD will automatically create `user3`, `user4`, `user5` resources

## Removing Users

1. Decrease user count in `bootstrap/values.yaml`
2. Manually delete excess Applications:
   ```bash
   oc delete application workspace-user3 -n openshift-gitops
   oc delete application llama-stack-instance-user3 -n openshift-gitops
   ```
3. Delete user namespaces:
   ```bash
   oc delete namespace wksp-user3 lls-user3
   ```

## Compatibility Notes

- LLama Stack version: 0.3.0
- Playground version: Updated to use `latest` (ensure compatibility)
- Operator version: v0.3.0

## Next Steps

1. Configure actual model endpoints in `bootstrap/values.yaml`
2. Deploy bootstrap chart
3. Verify all Applications are synced
4. Test user1 and user2 access
5. Distribute playground URLs to users
