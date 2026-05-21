---
sidebar_position: 3
---

# Hooks and Custom Code

The code generator produces controller logic that handles the common patterns in AWS APIs — creating resources, reading their state, updating them, and deleting them. But AWS APIs are inconsistent, and many resources need custom logic that can't be expressed through declarative configuration alone.

Hooks let you inject Go code at specific points in the generated controller logic, bridging the gap between imperative AWS APIs and Kubernetes' declarative reconciliation model. Common uses include:

- Setting fields that the code generator can't automatically map
- Adding custom validation or defaulting logic
- Implementing partial updates (only sending changed fields)
- Setting custom status conditions based on resource state
- Wiring in fields that live outside an API wrapper struct

## How Hooks Work

Hooks are Go template files (`.go.tpl`) placed in `templates/hooks/<resource>/` in the service controller repository. You reference them in `generator.yaml`:

```yaml
resources:
  BackupPlan:
    hooks:
      sdk_update_post_build_request:
        template_path: hooks/backup_plan/sdk_update_post_build_request.go.tpl
```

Alternatively, for short snippets, use inline code:

```yaml
resources:
  User:
    hooks:
      sdk_create_post_set_output:
        code: "rm.setSyncedCondition(resp.Status, &resource{ko})"
```

After adding or modifying hooks, run `make build-controller` to regenerate the controller code. Custom templates are picked up automatically.

:::info Use renamed field names in hooks
If you renamed fields in `generator.yaml`, use the new names in your hook templates (e.g., `r.ko.Spec.Name` not `r.ko.Spec.BackupVaultName`).
:::

## Hook Variable Names

Different hook points provide different variable names for the resource. Using the wrong variable is a common source of bugs:

| Hook point | Resource variable | Notes |
|---|---|---|
| `sdk_create_post_build_request` | `desired` | Input resource |
| `sdk_read_one_post_set_output` | `ko` | Output resource being built |
| `sdk_delete_post_build_request` | `r` | Resource to delete |
| `sdk_update_*` | `desired`, `latest` | Desired and current state |

:::note Avoid `latest` in delete hooks
In `sdkDelete`, `latest` is the return variable (initialized to `nil`), not the input resource. Using it causes a nil pointer panic. Use `r` instead.
:::

## SDK Hook Points

These hooks inject code into the generated `sdk.go` file at specific points in the CRUD lifecycle. As a developer, you care about these when the auto-generated SDK call needs adjustment — for example, when you need to set a field that the code generator missed, or when you need to transform the API response before it's stored in the CR.

### Create hooks

| Hook | When it runs |
|------|-------------|
| `sdk_create_pre_build_request` | Before building the Create API input |
| `sdk_create_post_build_request` | After building the Create API input, before calling AWS |
| `sdk_create_post_set_output` | After setting output fields from the Create response |

### Read hooks

| Hook | When it runs |
|------|-------------|
| `sdk_read_one_post_set_output` | After setting output fields from the Describe/Get response |
| `sdk_read_many_post_set_output` | After setting output fields from a List response |

### Update hooks

| Hook | When it runs |
|------|-------------|
| `sdk_update_pre_build_request` | Before building the Update API input |
| `sdk_update_post_build_request` | After building the Update API input, before calling AWS |
| `sdk_update_post_set_output` | After setting output fields from the Update response |

### Delete hooks

| Hook | When it runs |
|------|-------------|
| `sdk_delete_pre_build_request` | Before building the Delete API input |
| `sdk_delete_post_build_request` | After building the Delete API input, before calling AWS |

### File-level hooks

| Hook | When it runs |
|------|-------------|
| `sdk_file_end` | Appended to the end of `sdk.go` (useful for helper functions) |

## Comparison Hook Points

When the controller reconciles a resource, it compares the desired state (from the CR's Spec) with the latest observed state (from the AWS API). The result is a `Delta` struct listing which fields differ. Sometimes the auto-generated comparison produces false positives — for example, when AWS returns a field in a different format than the user specified, or when an unordered list causes `reflect.DeepEqual` to fail non-deterministically.

Comparison hooks let you filter or adjust the delta:

| Hook | When it runs |
|------|-------------|
| `delta_pre_compare` | Before the auto-generated field comparisons |
| `delta_post_compare` | After the auto-generated field comparisons |

Example — filtering out a non-meaningful difference:

```yaml
resources:
  User:
    hooks:
      delta_post_compare:
        code: "filterDelta(delta, a, b)"
```

## Late Initialization Hook Points

Late initialization handles fields where AWS assigns a default value after resource creation (e.g., RDS choosing an availability zone when the user didn't specify one). The controller needs to recognize that the `nil` desired value and the AWS-assigned value aren't actually a conflict.

| Hook | When it runs |
|------|-------------|
| `late_initialize_pre_read_one` | Before reading the resource during late initialization |
| `late_initialize_post_read_one` | After reading the resource during late initialization |

## Common Hook Patterns

### Setting fields outside a wrapper

When using `input_wrapper_field_path`, fields outside the wrapper need manual wiring:

```go
// templates/hooks/backup_plan/sdk_update_post_build_request.go.tpl
if desired.ko.Status.BackupPlanID != nil {
    input.BackupPlanId = desired.ko.Status.BackupPlanID
}
```

### Custom synced condition

Setting `ACK.ResourceSynced` based on the resource's AWS status:

```go
if resp.Status != nil && *resp.Status == "active" {
    syncedCondition.Status = corev1.ConditionTrue
} else {
    syncedCondition.Status = corev1.ConditionFalse
}
```

### Custom update logic

When the Update API requires only changed fields (not the full spec), use a hook to build a selective payload from the delta:

```yaml
resources:
  User:
    hooks:
      sdk_update_post_build_request:
        code: "rm.populateUpdatePayload(input, desired, delta)"
```

## Next Steps

- [Field configuration](field-config.md) — Renames, immutability, references
- [Configuration reference](configuration.md) — Full generator.yaml options
