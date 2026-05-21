---
sidebar_position: 2
---

# generator.yaml Configuration Reference

The `generator.yaml` file is the primary way you control how the code generator builds a service controller. It lives at the root of each service controller repository and tells the code generator which resources to generate, how to map AWS API fields to Kubernetes CRD fields, and where to inject custom behavior.

Every service controller has one. When you first bootstrap a controller, all resources are listed in `ignore.resource_names`. As you add support for each resource, you remove it from the ignore list and add its configuration.

## Top-Level Structure

```yaml
ignore:
  resource_names:
    - ResourceToSkip
  field_paths:
    - CreateInputShape.FieldToSkip

resources:
  ResourceName:
    # Resource-level configuration (exceptions, hooks, fields, renames, tags, reconcile)

operations:
  OperationName:
    # Operation-level configuration (wrappers, custom methods)
```

The three top-level sections are:
- **`ignore`** — What to skip during generation
- **`resources`** — Per-resource configuration (this is where most of your config lives)
- **`operations`** — Per-API-operation configuration (wrappers, custom output methods)

## Ignore Configuration

### `ignore.resource_names`

List resources that should not be generated. Used in bootstrap PRs to scaffold a controller without generating any resource-specific code. Remove a resource from this list when you're ready to add it.

### `ignore.field_paths`

Skip specific fields during code generation. Use the format `ShapeName.FieldName` to target fields in specific API operation shapes:

```yaml
ignore:
  field_paths:
    - VersioningConfiguration.MFADelete
    - CreateUserInput.AuthenticationMode
```

## Resource Configuration

### `exceptions`

Configure how the controller handles AWS API errors.

**404 error codes** — Many AWS APIs return non-standard error codes for "not found":

```yaml
resources:
  Table:
    exceptions:
      errors:
        404:
          code: ResourceNotFoundException
```

**Terminal codes** — Error codes that indicate the resource cannot be reconciled without user intervention:

```yaml
resources:
  DBCluster:
    exceptions:
      terminal_codes:
        - DBClusterQuotaExceededFault
        - InvalidParameterValue
        - InvalidParameterCombination
```

### `reconcile`

Control reconciliation behavior:

```yaml
resources:
  NotebookInstance:
    reconcile:
      requeue_on_success_seconds: 60
```

By default, successfully reconciled resources are requeued after 10 hours. Set `requeue_on_success_seconds` for resources whose state changes frequently outside of Kubernetes (e.g., via the AWS console).

### `is_arn_primary_key`

Set to `true` when a resource has no `Name` or `ID` field and can only be identified by ARN:

```yaml
resources:
  ModelPackage:
    is_arn_primary_key: true
```

### `renames`, `fields`, `hooks`, `tags`

These are covered in their own pages:
- [Field configuration](field-config.md) — Renames, immutability, references, tags, custom types
- [Hooks and custom code](hooks.md) — Injecting custom logic at specific points

## Operation Configuration

### `output_wrapper_field_path`

Some AWS APIs wrap responses in a nested object. Use this to flatten the wrapper so its fields land directly in Spec/Status:

```yaml
operations:
  CreateReplicationGroup:
    output_wrapper_field_path: ReplicationGroup
  DescribeReplicationGroups:
    output_wrapper_field_path: ReplicationGroups  # plural for list operations
```

### `input_wrapper_field_path`

Some AWS APIs wrap input fields in a nested structure. This flattens the wrapper's fields into the CRD Spec:

```yaml
operations:
  CreateBackupPlan:
    input_wrapper_field_path: BackupPlan
    output_wrapper_field_path: BackupPlan
```

:::warning Fields outside the wrapper
Fields on the API input that are outside the wrapper struct are excluded from the CRD spec. If those fields are required for CRUD operations, you need custom hooks to wire them in. See [Hooks and Custom Code](hooks.md) for details.
:::

### `set_output_custom_method_name`

Specify a custom Go method to call when setting output fields from an API response:

```yaml
operations:
  CreateUser:
    set_output_custom_method_name: CustomCreateUserSetOutput
```

### `custom_implementation`

Replace the entire generated SDK call with a custom implementation:

```yaml
operations:
  ModifyUser:
    custom_implementation: CustomModifyUser
```

## Next Steps

- [Field configuration](field-config.md) — Renames, immutability, references, tags
- [Hooks and custom code](hooks.md) — Injecting custom logic
