---
sidebar_position: 4
---

# Building a Controller

This guide covers how to generate and compile an ACK service controller, including its Custom Resource Definitions (CRDs), RBAC manifests, and Helm chart.

## Prerequisites

You should have completed the [development setup](setup.md) and have the `ack-generate` binary built.

:::note Code-generator and runtime versions
CI builds controllers with both `code-generator` and `runtime` at HEAD of `main`. Keep both at HEAD for dev work to match CI.
:::

## Generate a Service Controller

The `make build-controller` command in the code-generator repository handles the entire generation pipeline in one shot:

- Go types and CRD structs for each resource
- Controller and resource manager code
- DeepCopy implementations
- CRD YAML manifests
- Kustomize overlays
- RBAC manifests
- Helm chart (templates, values, CRDs)

```bash
cd code-generator
SERVICE=s3 AWS_SDK_GO_VERSION=v1.41.0 make build-controller
```

The generated code is written to the service controller repository (e.g., `../s3-controller/`). You can override this with the `SERVICE_CONTROLLER_SOURCE_PATH` environment variable.

:::info AWS SDK version
Use the version currently pinned in the controller's `go.mod`, unless you need to bump it to pick up new API fields or resources. Bumping the SDK version can introduce new resources that the code generator will attempt to generate — if you're not ready to support them, add them to `ignore.resource_names` in `generator.yaml`.

Use the core `github.com/aws/aws-sdk-go-v2` version, not the service-specific module version.
:::

## What Gets Generated

After running `make build-controller`, the service controller repository will contain:

| Directory | Contents |
|-----------|----------|
| `apis/v1alpha1/` | Go types, CRD structs, deepcopy, enums |
| `pkg/resource/$RESOURCE/` | Resource manager, SDK integration, delta comparison |
| `config/crd/bases/` | CRD YAML manifests |
| `config/rbac/` | RBAC role and binding manifests |
| `config/` | Kustomize overlays for deployment |
| `helm/` | Helm chart (templates, values, CRDs) |
| `cmd/controller/main.go` | Controller entrypoint |

:::warning Never edit generated files
All files listed above are generated and will be overwritten on the next build. To change behavior, edit `generator.yaml` and rebuild. The only hand-edited files should be `generator.yaml`, hook templates in `templates/hooks/`, and E2E tests in `test/e2e/`.
:::

## Verify the Build

After generation, compile the controller to check for errors:

```bash
cd ../s3-controller
go build -o bin/controller ./cmd/controller
```

Check that all expected files were generated:

```bash
git status
```

You should see changes in `apis/`, `pkg/resource/`, `config/`, and `helm/`.

## Next Steps

- [Understand how code generation works](code-generation/overview.md)
- [Configure generator.yaml](code-generation/configuration.md)
- [Write E2E tests](testing.md)
