# DevSpaces Integration Guide

## Overview

This guide explains how the multi-user workspace setup integrates with Red Hat OpenShift Dev Spaces (formerly Eclipse Che).

## How It Works

When users log into DevSpaces, they should automatically see their pre-created "llmaas" workspace in their workspace list.

### Architecture

```
User logs into DevSpaces
    ↓
DevSpaces looks for:
  - Namespaces with che.eclipse.org/username: {username}
  - DevWorkspaces with controller.devfile.io/devworkspace-creator: {username}
    ↓
DevSpaces shows workspaces that match the logged-in user
```

## Configuration

### Namespace Annotations and Labels

Each user's workspace namespace is configured with:

**Namespace:** `wksp-{username}` (e.g., `wksp-user1`)

**Labels:**
```yaml
labels:
  app.kubernetes.io/part-of: che.eclipse.org
  app.kubernetes.io/component: workspaces-namespace
  controller.devfile.io/namespace-owner: {username}
```

**Annotations:**
```yaml
annotations:
  openshift.io/display-name: "Workspace {username}"
  openshift.io/description: "Workspace Namespace"
  che.eclipse.org/username: {username}
  controller.devfile.io/namespace-owner: {username}
```

### DevWorkspace Annotations and Labels

Each user's DevWorkspace is configured with:

**Name:** `llmaas-{username}` (e.g., `llmaas-user1`)

**Annotations:**
```yaml
annotations:
  controller.devfile.io/devworkspace-creator: {username}
  che.eclipse.org/devworkspace-creator: {username}
```

**Labels:**
```yaml
labels:
  app.kubernetes.io/part-of: che.eclipse.org
  app.kubernetes.io/component: workspace
```

**Key Settings:**
- `routingClass: che` - Uses DevSpaces routing
- `started: true` - Workspace auto-starts when accessed

## User Experience

### 1. User Logs into DevSpaces

Users access DevSpaces at:
```
https://devspaces.apps.{cluster-domain}
```

### 2. Authentication

Users log in with their OpenShift credentials (e.g., `user1`, `user2`)

### 3. Workspace Appears

After login, users should see their "llmaas-{username}" workspace in the workspace list.

### 4. Open Workspace

Users click on the workspace to open it. DevSpaces will:
- Start the workspace if not already running
- Load VS Code editor with the configured repository
- Mount the project at `/projects/llmaas-workspace`

## Verification Steps

### Check Namespace Configuration

```bash
# For user1
oc get namespace wksp-user1 -o yaml

# Verify labels
oc get namespace wksp-user1 -o jsonpath='{.metadata.labels}'

# Verify annotations
oc get namespace wksp-user1 -o jsonpath='{.metadata.annotations}'
```

**Expected annotations:**
- `che.eclipse.org/username: user1`
- `controller.devfile.io/namespace-owner: user1`

### Check DevWorkspace Configuration

```bash
# For user1
oc get devworkspace -n wksp-user1

# Detailed view
oc get devworkspace llmaas-user1 -n wksp-user1 -o yaml
```

**Expected annotations:**
- `controller.devfile.io/devworkspace-creator: user1`
- `che.eclipse.org/devworkspace-creator: user1`

**Expected labels:**
- `app.kubernetes.io/part-of: che.eclipse.org`
- `app.kubernetes.io/component: workspace`

### Check DevWorkspace Status

```bash
# Check if workspace is running
oc get devworkspace llmaas-user1 -n wksp-user1 -o jsonpath='{.status.phase}'

# Should output: Running

# Get workspace URL
oc get devworkspace llmaas-user1 -n wksp-user1 -o jsonpath='{.status.mainUrl}'
```

### Check DevWorkspace Pods

```bash
# List all pods in user namespace
oc get pods -n wksp-user1

# Expected pods:
# - llmaas-user1-... (workspace pod)
# - ...other DevWorkspace operator pods
```

## Troubleshooting

### Workspace Not Appearing in DevSpaces

**Issue:** User logs into DevSpaces but doesn't see the llmaas workspace.

**Possible Causes & Solutions:**

#### 1. Username Mismatch

DevSpaces username must match the OpenShift user identity.

```bash
# Check what username DevSpaces is using
oc get devworkspace -A -o jsonpath='{range .items[*]}{.metadata.annotations.controller\.devfile\.io/devworkspace-creator}{"\n"}{end}'

# Compare with OpenShift users
oc get users
```

**Solution:** Ensure `.Values.username` in bootstrap/values.yaml matches the actual OpenShift username.

#### 2. Missing Annotations

Check if DevWorkspace has the required creator annotations.

```bash
oc get devworkspace llmaas-user1 -n wksp-user1 -o jsonpath='{.metadata.annotations}'
```

**Solution:** If missing, delete and recreate the DevWorkspace:
```bash
oc delete devworkspace llmaas-user1 -n wksp-user1
# ArgoCD will recreate it with proper annotations
```

#### 3. Namespace Not Linked to User

Check if namespace has the correct user annotation.

```bash
oc get namespace wksp-user1 -o jsonpath='{.metadata.annotations.che\.eclipse\.org/username}'
```

**Solution:** If empty or wrong, update the namespace:
```bash
oc annotate namespace wksp-user1 che.eclipse.org/username=user1 --overwrite
```

#### 4. DevWorkspace Not Started

Check workspace status:

```bash
oc get devworkspace llmaas-user1 -n wksp-user1 -o jsonpath='{.spec.started}'
# Should be: true

oc get devworkspace llmaas-user1 -n wksp-user1 -o jsonpath='{.status.phase}'
# Should be: Running or Starting
```

**Solution:** If not started, update the DevWorkspace:
```bash
oc patch devworkspace llmaas-user1 -n wksp-user1 --type merge -p '{"spec":{"started":true}}'
```

#### 5. DevSpaces Not Recognizing Namespace

DevSpaces may not be watching the user namespace.

```bash
# Check DevSpaces operator logs
oc logs -n openshift-devspaces deployment/devspaces-operator

# Look for errors related to namespace wksp-user1
```

**Solution:** Restart DevSpaces operator:
```bash
oc rollout restart deployment/devspaces-operator -n openshift-devspaces
```

#### 6. RBAC Issues

User may not have permissions to access their namespace.

```bash
# Check user permissions
oc auth can-i get devworkspace --as user1 -n wksp-user1
# Should be: yes

# Check RoleBindings
oc get rolebinding -n wksp-user1
```

**Solution:** RoleBinding should already exist via workspace chart. If missing, check ArgoCD sync status.

### Workspace Fails to Start

**Issue:** Workspace appears in DevSpaces but fails to start.

#### 1. Check Pod Logs

```bash
# Get pod name
oc get pods -n wksp-user1 | grep llmaas-user1

# Check logs
oc logs -n wksp-user1 <pod-name>
```

#### 2. Check Events

```bash
oc get events -n wksp-user1 --sort-by='.lastTimestamp'
```

#### 3. Check DevWorkspace Status

```bash
oc describe devworkspace llmaas-user1 -n wksp-user1
```

Look for error messages in the status section.

#### 4. Image Pull Issues

If the tooling image fails to pull:

```bash
# Check image
oc get devworkspace llmaas-user1 -n wksp-user1 -o yaml | grep image

# Try pulling manually
oc run test --image=registry.redhat.io/devspaces/ubi-rhel9:latest -n wksp-user1 --rm -it -- /bin/bash
```

### Git Repository Not Cloned

**Issue:** Workspace starts but the git repository is not cloned.

#### 1. Check Repository URL

```bash
oc get devworkspace llmaas-user1 -n wksp-user1 -o yaml | grep -A 5 projects
```

#### 2. Check from Inside Workspace

Once workspace is running, open a terminal in DevSpaces and check:

```bash
ls -la /projects
cd /projects/llmaas-workspace
git status
```

#### 3. Check Network Connectivity

```bash
# From workspace terminal
curl -I https://github.com/taylorjordanNC/coding-exercises
```

## User Access URLs

Once everything is working, users can access their workspace via:

### DevSpaces Dashboard
```
https://devspaces.apps.{cluster-domain}
```

Users log in with their OpenShift credentials and see their workspace.

### Direct Workspace URL

Each workspace gets a unique URL:
```bash
# Get the URL
oc get devworkspace llmaas-user1 -n wksp-user1 -o jsonpath='{.status.mainUrl}'
```

Users can bookmark this URL for direct access (still requires authentication).

## Configuration Customization

### Change Repository

To use a different git repository for a specific user:

Edit the bootstrap ApplicationSet to pass different repo per user:

```yaml
# In bootstrap/templates/applicationset-workspace.yaml
helm:
  values: |
    username: '{{ user }}'
    namespace: wksp
    devworkspace:
      projects:
        repoUrl: https://github.com/{{ user }}/my-repo  # Different per user
        revision: main
```

### Change IDE Editor

To use a different editor (e.g., IntelliJ IDEA):

Update in `bootstrap/values.yaml`:

```yaml
workspace:
  devworkspace:
    devfile: http://devspaces-dashboard.openshift-devspaces.svc.cluster.local:8080/dashboard/api/editors/devfile?che-editor=che-incubator/che-idea/latest
```

Available editors:
- VS Code: `che-incubator/che-code/latest`
- IntelliJ IDEA: `che-incubator/che-idea/latest`
- PyCharm: `che-incubator/che-pycharm/latest`

### Change Container Image

To use a different base container image:

Update in `bootstrap/values.yaml`:

```yaml
workspace:
  devworkspace:
    components:
      - name: tooling
        container:
          image: quay.io/my-org/my-custom-image:latest
          sourceMapping: /projects
```

## Best Practices

### 1. User Provisioning

- Create OpenShift users before deploying workspaces
- Ensure usernames in bootstrap/values.yaml match OpenShift user identities exactly

### 2. Testing

- Test with one user first (`user.count: 1`)
- Verify the workspace appears in DevSpaces
- Scale up to multiple users once confirmed working

### 3. Monitoring

- Monitor DevWorkspace status regularly
- Check for failed starts or pods in error state
- Review DevSpaces operator logs for issues

### 4. Resource Limits

- Set appropriate resource limits on workspace containers
- Consider cluster capacity when increasing user count

## Summary

For workspaces to appear in DevSpaces for each user:

✅ **Namespace** must have:
- `che.eclipse.org/username: {username}` annotation
- `controller.devfile.io/namespace-owner: {username}` label

✅ **DevWorkspace** must have:
- `controller.devfile.io/devworkspace-creator: {username}` annotation
- `che.eclipse.org/devworkspace-creator: {username}` annotation
- `started: true` spec

✅ **User** must:
- Exist in OpenShift
- Have edit permissions in their workspace namespace
- Log into DevSpaces with matching username

With these configurations, when user1 logs into DevSpaces, they will see "llmaas-user1" workspace ready to use!
