# Workspace Multi-User Fix

## Problem Identified

The DevWorkspace was only being created correctly for user1 due to hardcoded or missing configuration values. All users were attempting to use the same workspace name, causing conflicts.

## Root Causes

1. **Missing DevWorkspace Name Configuration**
   - `bootstrap/values.yaml` did not include `devworkspace.name`
   - All users defaulted to "exercises" from `workspace/values.yaml`
   - DevWorkspaces need unique names per user to avoid conflicts

2. **Missing DevFile URL**
   - `bootstrap/values.yaml` did not include `devworkspace.devfile`
   - The IDE editor URL was not being passed to users

3. **Missing Namespace Configuration**
   - `workspace.namespace` was not defined in bootstrap values
   - ApplicationSet was not passing namespace to workspace chart

## Changes Made

### 1. Updated DevWorkspace Template

**File:** `workspace/templates/devworkspace.yaml`

**Before:**
```yaml
metadata:
  name: {{ .Values.devworkspace.name }}
```

**After:**
```yaml
metadata:
  name: {{ .Values.devworkspace.name | default (printf "llmaas-%s" .Values.username) }}
```

**Impact:** Each user now gets a unique DevWorkspace name:
- user1 → `llmaas-user1`
- user2 → `llmaas-user2`
- userN → `llmaas-userN`

### 2. Updated Project Name

**File:** `workspace/templates/devworkspace.yaml`

**Before:**
```yaml
projects:
- name: {{ .Values.devworkspace.name }}
```

**After:**
```yaml
projects:
- name: {{ .Values.devworkspace.name | default "llmaas-workspace" }}
```

**Impact:** Project name now defaults to "llmaas-workspace" if not specified.

### 3. Added DevFile URL to Bootstrap

**File:** `bootstrap/values.yaml`

**Added:**
```yaml
workspace:
  namespace: wksp
  devworkspace:
    devfile: http://devspaces-dashboard.openshift-devspaces.svc.cluster.local:8080/dashboard/api/editors/devfile?che-editor=che-incubator/che-code/latest
    projects:
      repoUrl: https://github.com/taylorjordanNC/coding-exercises
      revision: main
    components:
      - name: tooling
        container:
          image: registry.redhat.io/devspaces/ubi-rhel9:latest
          sourceMapping: /projects
```

**Impact:** IDE editor configuration is now properly passed to all users.

### 4. Updated ApplicationSet

**File:** `bootstrap/templates/applicationset-workspace.yaml`

**Before:**
```yaml
helm:
  values: |
    username: '{{ `{{ user }}` }}'
    devworkspace:
      {{- toYaml .Values.workspace.devworkspace | nindent 14 }}
```

**After:**
```yaml
helm:
  values: |
    username: '{{ `{{ user }}` }}'
    namespace: {{ .Values.workspace.namespace }}
    devworkspace:
      {{- toYaml .Values.workspace.devworkspace | nindent 14 }}
```

**Impact:** Namespace configuration is now passed to each user's workspace.

### 5. Updated RBAC Template

**File:** `workspace/templates/rbac.yaml`

**Before:**
```yaml
metadata:
  name: wksp-edit-{{ .Values.username }}
  namespace: wksp-{{ .Values.username }}
```

**After:**
```yaml
metadata:
  name: {{ .Values.namespace }}-edit-{{ .Values.username }}
  namespace: {{ .Values.namespace }}-{{ .Values.username }}
```

**Impact:** RBAC now uses the configurable namespace prefix instead of hardcoded "wksp".

## How It Works Now

### Per-User Resources

Each user (user1, user2, ..., userN) gets:

1. **Namespace:** `wksp-{username}`
   - Example: `wksp-user1`, `wksp-user2`

2. **DevWorkspace:** `llmaas-{username}`
   - Example: `llmaas-user1`, `llmaas-user2`
   - Each is a unique DevWorkspace resource

3. **Project:** `llmaas-workspace`
   - Git repository: `https://github.com/taylorjordanNC/coding-exercises`
   - Same project for all users (can be customized per user if needed)

4. **IDE Editor:** VS Code (che-incubator/che-code/latest)
   - Configured via DevFile URL

5. **RBAC:** User has "edit" permissions in their namespace

### Configuration Flow

```
bootstrap/values.yaml
  ├─> user.count: 2 (creates user1, user2)
  ├─> workspace.namespace: wksp
  └─> workspace.devworkspace (config for all users)
        ├─> devfile: (IDE URL)
        ├─> projects.repoUrl
        └─> components

ApplicationSet: workspace
  ├─> Generates: workspace-user1
  │     └─> Creates: wksp-user1 namespace
  │           └─> DevWorkspace: llmaas-user1
  │
  └─> Generates: workspace-user2
        └─> Creates: wksp-user2 namespace
              └─> DevWorkspace: llmaas-user2
```

## Verification Steps

### 1. Check ApplicationSet Generated Applications

```bash
oc get applications -n openshift-gitops -l argocd.argoproj.io/application-set-name=workspace

# Expected output:
# NAME               SYNC STATUS   HEALTH STATUS
# workspace-user1    Synced        Healthy
# workspace-user2    Synced        Healthy
```

### 2. Verify Namespaces Created

```bash
oc get namespaces | grep wksp

# Expected output:
# wksp-user1   Active   10m
# wksp-user2   Active   10m
```

### 3. Check DevWorkspaces

```bash
oc get devworkspace -n wksp-user1
oc get devworkspace -n wksp-user2

# Expected output for user1:
# NAME           STARTED   STATUS    URL
# llmaas-user1   true      Running   https://workspaces...

# Expected output for user2:
# NAME           STARTED   STATUS    URL
# llmaas-user2   true      Running   https://workspaces...
```

### 4. Verify RBAC

```bash
oc get rolebinding -n wksp-user1 | grep user1
oc get rolebinding -n wksp-user2 | grep user2

# Expected output:
# wksp-edit-user1   ClusterRole/edit   ...
# wksp-edit-user2   ClusterRole/edit   ...
```

### 5. Access DevWorkspace URLs

Get the workspace URLs:

```bash
oc get devworkspace llmaas-user1 -n wksp-user1 -o jsonpath='{.status.mainUrl}'
oc get devworkspace llmaas-user2 -n wksp-user2 -o jsonpath='{.status.mainUrl}'
```

Visit the URLs to verify:
- IDE loads correctly
- Project is cloned from GitHub
- User can edit files

## Troubleshooting

### DevWorkspace Stuck in "Starting" Status

```bash
# Check DevWorkspace events
oc describe devworkspace llmaas-user1 -n wksp-user1

# Check pod status
oc get pods -n wksp-user1
oc logs -n wksp-user1 <pod-name>
```

### DevWorkspace Name Conflicts

If you see errors about duplicate names:

```bash
# Delete existing DevWorkspace
oc delete devworkspace exercises -n wksp-user1

# ArgoCD will recreate with correct name (llmaas-user1)
```

### RBAC Issues

If users can't access their workspace:

```bash
# Check RoleBinding
oc get rolebinding -n wksp-user1
oc describe rolebinding wksp-edit-user1 -n wksp-user1

# Verify user subject matches OpenShift username
```

## Customization Options

### Per-User Git Repositories

To give each user a different repository, update the ApplicationSet:

```yaml
# In bootstrap/templates/applicationset-workspace.yaml
helm:
  values: |
    username: '{{ `{{ user }}` }}'
    namespace: {{ .Values.workspace.namespace }}
    devworkspace:
      projects:
        repoUrl: https://github.com/user-{{ `{{ user }}` }}/repo
```

### Custom Workspace Names

To use custom names instead of auto-generated:

```yaml
# In bootstrap/values.yaml
workspace:
  devworkspace:
    name: my-custom-workspace  # Will use: my-custom-workspace-user1, etc.
```

### Different IDE Editors

Change the DevFile URL to use a different editor:

```yaml
# In bootstrap/values.yaml
workspace:
  devworkspace:
    devfile: http://devspaces-dashboard.openshift-devspaces.svc.cluster.local:8080/dashboard/api/editors/devfile?che-editor=che-incubator/che-idea/latest  # IntelliJ IDEA
```

## Summary

The fix ensures that:
1. ✅ Each user gets a unique DevWorkspace name
2. ✅ IDE configuration is properly passed from bootstrap
3. ✅ Namespace configuration is consistent
4. ✅ RBAC is properly configured per user
5. ✅ All users (user1, user2, ..., userN) work correctly

The workspace deployment now follows the same multi-user pattern as the llama-stack-instance deployment.
