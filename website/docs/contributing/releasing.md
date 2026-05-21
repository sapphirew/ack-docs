---
sidebar_position: 8
---

# Releasing a Controller

Each ACK service controller is released independently. A release consists of a Git tag, a container image, and a Helm chart.

## What's in a Release

| Artifact | Location |
|----------|----------|
| Container image | `public.ecr.aws/aws-controllers-k8s/$SERVICE-controller:$VERSION` |
| Helm chart | `public.ecr.aws/aws-controllers-k8s/chart:$SERVICE-$VERSION` |
| Git tag | `v$VERSION` on the service controller repository |

## Versioning

ACK follows [semantic versioning](https://semver.org/):

- **Patch** (`v1.4.1` → `v1.4.2`): Bug fixes, test-only changes
- **Minor** (`v1.4.2` → `v1.5.0`): New fields, new resources, functional changes
- **First release**: `v0.1.0`

## Release Steps

1. **Ensure your branch is up to date:**

```bash
cd $SERVICE-controller
git checkout main && git pull upstream main
```

2. **Run the release build script:**

```bash
cd ../code-generator
export RELEASE_VERSION=v0.1.0
./scripts/build-controller-release.sh $SERVICE
```

3. **Create a release branch and commit:**

```bash
cd ../$SERVICE-controller
git checkout -b release-$RELEASE_VERSION
git add -A
git commit -m "Release artifacts for release $RELEASE_VERSION"
git push origin release-$RELEASE_VERSION
```

4. **Open a PR** from your release branch to upstream `main`. Once merged, a Prow postsubmit job automatically builds and publishes the container image and Helm chart to the public ECR repositories.

## Release Branch and Commit Convention

- Branch name: `release-v$VERSION` (e.g., `release-v0.1.0`)
- Commit message: `Release artifacts for release v$VERSION`
- Single commit per release PR
