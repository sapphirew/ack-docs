---
sidebar_position: 2
---

# Development Setup

This guide walks you through setting up your local environment for ACK development.

## Prerequisites

- **Go 1.23+**: `go version` ([install](https://golang.org/doc/install))
- **kubectl**: `kubectl version` ([install](https://kubernetes.io/docs/tasks/tools/))
- **AWS credentials** for the service API you're building a controller for ([configure](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html)). You don't need AWS credentials to build a controller — only to run E2E tests against real AWS resources.

Optional:
- **Docker**: Only needed if running E2E tests locally with kind
- **Helm**: For testing chart installation

## How ACK Development Works

ACK development uses a multi-repo workspace. When building a service controller, you'll use local clones of the code-generator, runtime, and test-infra repositories alongside your service controller repo. Build scripts in the code-generator reference sibling directories, and you'll frequently run commands that span repos.

## Fork and Clone

ACK uses a fork-based workflow. Fork the repositories you need to your personal GitHub account, then clone them into a workspace directory.

**Core repositories** (needed for most development):

- [`code-generator`](https://github.com/aws-controllers-k8s/code-generator) — Code generation tool, templates, and build scripts
- [`runtime`](https://github.com/aws-controllers-k8s/runtime) — Core ACK runtime library and types

**Service controllers** (fork the ones you're working on):

- `github.com/aws-controllers-k8s/$SERVICE-controller` (e.g., `s3-controller`, `ec2-controller`)

**Testing infrastructure** (needed for running E2E tests):

- [`test-infra`](https://github.com/aws-controllers-k8s/test-infra) — Testing utilities and CI config

### Clone and configure remotes

```bash
# Create a workspace directory
mkdir -p ~/ws/aws-controllers-k8s
cd ~/ws/aws-controllers-k8s

GITHUB_ID="your-github-username"

# Clone core repos
for REPO in code-generator runtime test-infra; do
    git clone git@github.com:$GITHUB_ID/$REPO.git
    cd $REPO
    git remote add upstream git@github.com:aws-controllers-k8s/$REPO
    git fetch --all
    cd ..
done

# Clone service controller(s) you're working on
SERVICE=s3  # change to your service
git clone git@github.com:$GITHUB_ID/$SERVICE-controller.git
cd $SERVICE-controller
git remote add upstream git@github.com:aws-controllers-k8s/$SERVICE-controller
git fetch --all
cd ..
```

After cloning, your workspace should look like this:

```
~/ws/aws-controllers-k8s/
├── code-generator/          # Build tool — run make build-controller from here
├── runtime/                 # Shared library — referenced by controllers
├── test-infra/              # Test utilities — acktest Python package
└── s3-controller/           # Your service controller (one per service)
```

## Build the Code Generator

```bash
cd code-generator
make build-ack-generate
```

This produces the `bin/ack-generate` binary. You only need to rebuild this when the code generator itself changes.

Verify it works:

```bash
./bin/ack-generate --help
```

## Branch and Contribute

Before starting work, check the [GitHub issues](https://github.com/aws-controllers-k8s/community/issues) for existing issues you can pick up. If you're planning significant new work, consider creating an issue first to coordinate with other contributors.

Create a feature branch from upstream main:

```bash
cd $SERVICE-controller
git fetch --all
git checkout -b my-feature upstream/main
```

Make your changes, then push to your fork:

```bash
git push origin my-feature
```

Open a pull request from your fork's branch to the upstream `main` branch.

## Next Steps

- [Build a service controller](building-controller.md)
- [Understand code generation](code-generation/overview.md)
