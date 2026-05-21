---
sidebar_position: 4
---

# Field Configuration

This page covers the field-level configuration options in `generator.yaml` that control how individual fields are generated, compared, and presented.

## Renaming Fields

AWS API field names often include the resource name as a prefix, creating redundant "stutter" in the CRD. For example, ECR's `Repository` resource has a field called `RepositoryName`. In Kubernetes, this becomes `Repository.Spec.RepositoryName` — the word "Repository" appears twice. Renaming it to `Name` gives you the cleaner `Repository.Spec.Name`.

```yaml
resources:
  Repository:
    renames:
      operations:
        CreateRepository:
          input_fields:
            RepositoryName: Name
        DeleteRepository:
          input_fields:
            RepositoryName: Name
        DescribeRepositories:
          input_fields:
            RepositoryName: Name
```

:::warning Add renames for ALL operations
You must add renames for every operation that uses the field (Create, Read, Update, Delete, List). Missing any operation causes code generation errors like `could not find field with path`.
:::

For fields with different names in input and output shapes (e.g., `EnableEncryption` in the input vs `EncryptionEnabled` in the output), use both `input_fields` and `output_fields` to tell the code generator they're the same field:

```yaml
renames:
  operations:
    CreateResource:
      input_fields:
        OldInputName: NewName
      output_fields:
        OldOutputName: NewName
```

## Immutability

Some fields cannot be changed after a resource is created — primary keys, identifiers, and certain configuration options that AWS doesn't allow updating. Marking these as immutable causes the controller to return a terminal error if a user tries to modify them, rather than silently failing or causing unexpected behavior.

```yaml
resources:
  DBInstance:
    fields:
      AvailabilityZone:
        is_immutable: true
```

**How to verify immutability:** A field should be marked immutable if:
- AWS API docs say "cannot be changed" or "immutable"
- The field is a primary key or lookup identifier
- Testing with the AWS CLI confirms the field cannot be updated

:::tip Required in Update ≠ mutable
A field being "required" in the Update API doesn't mean it's mutable. Primary keys are often required in Update calls as identifiers but cannot actually be changed.
:::

## Cross-Resource References

Kubernetes users often manage related resources together — a database instance that needs a subnet group, or an API integration that references a VPC link. Cross-resource references let users express these relationships declaratively, so ACK automatically resolves the reference and populates the ID/ARN field.

```yaml
resources:
  Integration:
    fields:
      ApiId:
        references:
          resource: API
          path: Status.APIID
```

This generates an `APIRef` field alongside `ApiId`. Users can specify either the ID directly or a reference to another ACK-managed resource. The controller resolves the reference at reconciliation time.

**Cross-service references** — reference resources managed by a different controller:

```yaml
resources:
  DBCluster:
    fields:
      KmsKeyId:
        references:
          resource: Key
          service_name: kms
          path: Status.ACKResourceMetadata.ARN
```

:::warning Same-service references
When referencing a resource in the same controller, do NOT set `service_name`. Setting it (even correctly) causes an unresolved import alias and a compile error.
:::

## Read-Only Fields

During API inference, the code generator automatically determines which fields belong in Spec vs Status. Sometimes you need to override this — for example, when a field has different Go types in the Create input vs the Describe output, or when you're creating a custom status field.

```yaml
resources:
  Function:
    fields:
      LayerStatuses:
        is_read_only: true
```

## Required Fields

Override whether a field is required. Useful when the AWS API marks a field as required but your controller handles it differently:

```yaml
resources:
  Instance:
    fields:
      MaxCount:
        is_required: false
```

## Custom Field Types

Add fields that don't exist in the AWS API model. This is useful when you need to expose a concept that the API handles through separate operations (e.g., IAM policy attachments) as a simple field on the CRD:

```yaml
resources:
  Role:
    fields:
      Policies:
        type: "[]*string"
```

## Primary Key Fields

The code generator usually identifies primary keys automatically by looking for fields named `Name`, `ID`, or variants with the resource name prefix. When a resource's primary key doesn't follow this convention, tell the code generator explicitly:

```yaml
resources:
  PullThroughCacheRule:
    fields:
      ECRRepositoryPrefix:
        is_primary_key: true
```

## Field Source (`from`)

Sometimes the same conceptual field has different shapes in the Create response vs the Describe response. For example, Lambda's `Code` field in `CreateFunction` contains upload parameters (`S3Bucket`, `S3Key`), but in `GetFunction` it contains result metadata (`Location`, `RepositoryType`). The `from` option lets you pull a field's definition from a different API operation:

```yaml
resources:
  Function:
    fields:
      CodeLocation:
        is_read_only: true
        from:
          operation: GetFunction
          path: Code.Location
```

Prefer `from` over fully custom fields — it adapts automatically to API model changes, including documentation updates.

## Comparison Control

The controller compares desired and observed state to decide whether an update is needed. Sometimes the auto-generated comparison produces false positives — for example, unordered slices where `reflect.DeepEqual` fails non-deterministically. Exclude these fields from comparison and handle them with a custom [comparison hook](hooks.md#comparison-hook-points):

```yaml
resources:
  Role:
    fields:
      Tags:
        compare:
          is_ignored: true
```

## Late Initialization

Some AWS resources have fields where the service assigns a default value after creation. For example, if you create an RDS instance without specifying an availability zone, RDS picks one for you. Without late initialization, the controller would see a mismatch between the desired state (`nil`) and the observed state (`us-west-2a`) and try to "fix" it on every reconciliation loop.

Late initialization tells the controller to accept the server-assigned value and adopt it as the desired state:

```yaml
resources:
  DBInstance:
    fields:
      AvailabilityZone:
        late_initialize: {}
        is_immutable: true
```

## Tags

Most AWS resources support tagging via a `TagResource` API. The code generator handles tag conversion between the ACK standard format (`map[string]string`) and the service-specific format automatically.

**For resources that do NOT support tags** (check the AWS API docs first):

```yaml
resources:
  PullThroughCacheRule:
    tags:
      ignore: true
```

Without this, the code generator will fail with an error about the tag field not existing on the resource.

## Printer Columns

Add fields to `kubectl get` output so users can see important state at a glance:

```yaml
resources:
  ElasticIPAddress:
    fields:
      PublicIp:
        print:
          name: PUBLIC-IP
```

## Secrets

Mark sensitive fields to store them in Kubernetes Secrets rather than directly in the CR spec:

```yaml
resources:
  User:
    fields:
      Passwords:
        is_secret: true
```

## Next Steps

- [Configuration reference](configuration.md) — Full generator.yaml structure
- [Hooks and custom code](hooks.md) — Injecting custom logic
