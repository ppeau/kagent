# Cross-Namespace Agent Delegation Implementation

## Overview
This implementation adds support for cross-namespace agent delegation in kagent, allowing agents in one namespace to use agents in different namespaces as tools.

## Changes Made

### 1. API Types (`go/api/v1alpha2/agent_types.go`)

Added an optional `Namespace` field to the `TypedLocalReference` struct:

```go
type TypedLocalReference struct {
    // +optional
    Kind string `json:"kind"`
    // +optional
    ApiGroup string `json:"apiGroup"`
    Name     string `json:"name"`
    // Namespace is the namespace of the referenced resource.
    // If empty, the namespace of the referencing resource is used.
    // This enables cross-namespace agent delegation.
    // +optional
    Namespace string `json:"namespace,omitempty"`
}
```

**Behavior**: 
- If `Namespace` is empty, the agent is resolved in the same namespace as the referencing agent (default behavior)
- If `Namespace` is specified, the agent is resolved in the specified namespace (cross-namespace delegation)

### 2. Controller Logic (`go/internal/controller/translator/agent/adk_api_translator.go`)

Updated two locations in the agent translator to support cross-namespace resolution:

#### Location 1: `validateAgent` function (line ~202)
```go
// Support cross-namespace agent delegation
agentNamespace := agent.Namespace
if tool.Agent.Namespace != "" {
    agentNamespace = tool.Agent.Namespace
}

agentRef := types.NamespacedName{
    Namespace: agentNamespace,
    Name:      tool.Agent.Name,
}
```

#### Location 2: `buildManifest` function (line ~550)
```go
// Support cross-namespace agent delegation
agentNamespace := agent.Namespace
if tool.Agent.Namespace != "" {
    agentNamespace = tool.Agent.Namespace
}

agentRef := types.NamespacedName{
    Namespace: agentNamespace,
    Name:      tool.Agent.Name,
}
```

### 3. RBAC Permissions (`go/config/rbac/role.yaml`)

Added a comment to clarify that the existing `get` permission on agents across all namespaces is required for cross-namespace delegation:

```yaml
- apiGroups:
  - kagent.dev
  resources:
  - agents
  - modelconfigs
  - remotemcpservers
  verbs:
  - create
  - delete
  - get  # Required for cross-namespace agent delegation
  - list
  - patch
  - update
  - watch
```

**Note**: The ClusterRole already has the necessary permissions to read agents across all namespaces. No additional permissions are needed.

### 4. CRD Manifest (`go/config/crd/bases/kagent.dev_agents.yaml`)

Updated the v1alpha2 CRD schema to include the `namespace` field in the agent reference:

```yaml
agent:
  properties:
    apiGroup:
      type: string
    kind:
      type: string
    name:
      type: string
    namespace:
      description: |-
        Namespace is the namespace of the referenced resource.
        If empty, the namespace of the referencing resource is used.
        This enables cross-namespace agent delegation.
      type: string
  required:
  - name
  type: object
```

## Usage Example

After these modifications, you can use cross-namespace agent delegation as follows:

### Example 1: Same-namespace delegation (default behavior)
```yaml
apiVersion: kagent.dev/v1alpha2
kind: Agent
metadata:
  name: orchestrator-agent
  namespace: team-a
spec:
  type: Declarative
  declarative:
    systemMessage: "You are an orchestrator agent"
    tools:
    - type: Agent
      agent:
        name: worker-agent  # No namespace specified, defaults to team-a
```

### Example 2: Cross-namespace delegation
```yaml
apiVersion: kagent.dev/v1alpha2
kind: Agent
metadata:
  name: orchestrator-agent
  namespace: team-a
spec:
  type: Declarative
  declarative:
    systemMessage: "You are an orchestrator agent"
    tools:
    - type: Agent
      agent:
        name: shared-worker-agent
        namespace: shared-services  # Explicitly references agent in different namespace
```

## Benefits

✅ **Clean Solution**: Native support within the CRD, no additional infrastructure needed
✅ **Backward Compatible**: Empty namespace field defaults to same-namespace behavior
✅ **Maintains Simplicity**: No gateway or proxy required
✅ **Flexible**: Allows teams to share agents across namespaces

## Security Considerations

⚠️ **RBAC**: The kagent controller has ClusterRole permissions to read agents across all namespaces. This is necessary for cross-namespace delegation but should be understood as part of the security model.

⚠️ **Access Control**: Consider implementing additional admission webhooks if you need to restrict which namespaces can delegate to which other namespaces.

## Next Steps

1. **Testing**: Test the changes with real agent configurations
2. **Documentation**: Update user documentation to explain cross-namespace delegation
3. **Upstream Contribution**: Consider proposing these changes as a PR to the kagent-dev project

## Notes

- The deepcopy code in `go/api/v1alpha2/zz_generated.deepcopy.go` already handles the new field correctly (simple string copy)
- The Go version requirement (1.25.1) prevented running `make manifests`, but manual CRD updates were made to match the expected output
- All changes maintain backward compatibility with existing agent configurations
