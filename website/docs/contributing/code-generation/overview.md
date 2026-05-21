---
sidebar_position: 1
---

# How Code Generation Works

ACK uses a custom code generator that reads AWS API model files and produces Kubernetes Custom Resource Definitions (CRDs) and the controllers that reconcile them. This page explains the architecture and process.

## Overview

```
AWS API Model → ack-generate → Generated Code
     ↓              ↓              ↓
  service.json  generator.yaml  CRDs + Go types + controller logic
```

The code generator combines three inputs:

1. **AWS API model** — JSON files from the AWS SDK that describe every operation, shape, and error for a service
2. **generator.yaml** — Your configuration that controls which resources to generate, how fields are mapped, and what customizations to apply
3. **Go templates** — The code generator's built-in templates (plus any custom hook templates you provide)

## What Becomes a Resource?

AWS APIs can have hundreds of operations. The code generator identifies resources by looking for CRUD operation patterns:

- A **Create** operation (e.g., `CreateBucket`, `CreateRepository`)
- A **Delete** operation (e.g., `DeleteBucket`, `DeleteRepository`)
- A **Describe/Get/List** operation for reading current state

When these patterns are found, the code generator infers a Kubernetes Custom Resource.

Not every Create or Put API becomes its own resource. Some APIs that look like they create something are actually setting a property on a parent resource (e.g., `PutBucketVersioning` is a field on the `Bucket` resource, not a separate resource). The code generator handles the common cases, but you may need to use `ignore.resource_names` to skip operations that shouldn't become standalone resources, and use [field configuration](field-config.md) or [hooks](hooks.md) to wire sub-resource APIs into the parent resource's reconciliation.

## Spec vs Status Assignment

The code generator assigns fields based on where they appear in the AWS API:

- Fields that appear in the **Create input** land in `Spec` — these represent desired state that the user controls.
- Fields that appear **only in the output** (Create response or Describe response) land in `Status` — these represent observed state reported by AWS.
- Fields that appear in both input and output land in `Spec`, with the output value used to keep `Status` current.

## Updates

The code generator looks for a matching Update or Modify operation for each resource (e.g., `ModifyReplicationGroup` for `ReplicationGroup`). When found, it generates an `sdkUpdate` method that builds the Update API input from the resource's `Spec`.

Not all resources have a straightforward Update API. Some require custom logic — for example, only sending changed fields, or calling multiple APIs to update different aspects of the resource. In these cases, you use [hooks](hooks.md) to customize the update behavior or provide a fully custom update implementation.

## Generation Pipeline

When you run `make build-controller`, the following happens in order:

1. **API types** (`ack-generate apis`) — Generates Go structs for CRD Spec and Status from the API model
2. **DeepCopy** — Generates `zz_generated.deepcopy.go` for all custom types
3. **CRDs** — Generates CRD YAML manifests from the Go types
4. **Controller** (`ack-generate controller`) — Generates resource managers with SDK integration
5. **RBAC** — Generates ClusterRole manifests based on required AWS API permissions
6. **Helm** — Packages everything into a Helm chart
7. **Formatting** — Runs `gofmt` and `go mod tidy`

## Customization

The code generator handles common AWS API patterns automatically, but every service has quirks. Customization is done through:

- **[generator.yaml configuration](configuration.md)** — Declarative config for field behavior, error codes, operation settings
- **[Custom hooks](hooks.md)** — Go template snippets injected at specific points in the generated code
- **[Field configuration](field-config.md)** — Renames, immutability, cross-resource references, tags

## Next Steps

- [generator.yaml configuration reference](configuration.md)
- [Custom hooks and templates](hooks.md)
- [Field configuration](field-config.md)
