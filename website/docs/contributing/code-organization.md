---
sidebar_position: 3
---

# Code Organization

ACK is organized into multiple repositories under the [`aws-controllers-k8s`](https://github.com/aws-controllers-k8s) GitHub organization.

## Core Repositories

| Repository | Purpose |
|-----------|---------|
| [`runtime`](https://github.com/aws-controllers-k8s/runtime) | Core ACK runtime library, types, and reconciliation framework |
| [`code-generator`](https://github.com/aws-controllers-k8s/code-generator) | `ack-generate` CLI tool, Go templates, and build scripts |
| [`test-infra`](https://github.com/aws-controllers-k8s/test-infra) | `acktest` Python package, Prow CI config, and test scripts |
| [`community`](https://github.com/aws-controllers-k8s/community) | Issue tracking and project management |
| [`docs`](https://github.com/aws-controllers-k8s/docs) | Documentation website (this site) |

## Service Controller Repositories

Each AWS service has its own controller repository following the naming pattern `$SERVICE-controller`:

- [`s3-controller`](https://github.com/aws-controllers-k8s/s3-controller)
- [`ec2-controller`](https://github.com/aws-controllers-k8s/ec2-controller)
- [`rds-controller`](https://github.com/aws-controllers-k8s/rds-controller)
- ... and many more

## Service Controller Directory Structure

```
$SERVICE-controller/
├── apis/v1alpha1/           # Generated API types, CRD structs, enums
│   ├── generator.yaml       # Code generation configuration (edit this)
│   ├── types.go             # Generated shared types
│   ├── $resource.go         # Generated per-resource types
│   └── zz_generated.deepcopy.go
├── cmd/controller/          # Generated controller entrypoint
├── config/
│   ├── crd/bases/           # Generated CRD YAML manifests
│   └── rbac/                # Generated RBAC manifests
├── helm/                    # Generated Helm chart
├── pkg/resource/
│   └── $resource/           # Generated resource manager
│       ├── sdk.go           # AWS SDK integration
│       ├── delta.go         # Field comparison logic
│       ├── hooks.go         # Custom hook implementations (edit this)
│       └── ...
├── templates/hooks/         # Custom hook templates (edit this)
├── test/e2e/                # E2E tests (edit this)
│   ├── tests/
│   └── resources/
└── generator.yaml           # Symlink to apis/v1alpha1/generator.yaml
```

**Editable files:** `generator.yaml`, `templates/hooks/`, `test/e2e/`, and `pkg/resource/$resource/hooks.go`

**Everything else is generated** by `make build-controller` and should not be manually edited.