---
sidebar_position: 7
---

# Testing

ACK uses end-to-end (E2E) tests that run against real AWS APIs to verify controller behavior. These tests create actual AWS resources, verify the controller reconciles them correctly, and clean up afterward.

## How Testing Works

You can run E2E tests in two ways:

- **Locally** — Run the controller binary directly on your machine (no need to install it in a cluster), pointing it at a Kubernetes cluster (kind or otherwise). The controller runs as a regular process, making it easy to attach a debugger or add logging.
- **In CI** — Prow automatically runs E2E tests on every PR using a kind cluster with real AWS credentials.

Both approaches use the same pytest-based test framework (`acktest`).

## Test Types

### Unit Tests

```bash
cd $SERVICE-controller
make test
```

Only needed when you've added custom logic in hooks or helper functions. Generated code doesn't need unit tests.

### E2E Tests

E2E tests use pytest with the `acktest` framework from the test-infra repository.

## Setting Up the Test Environment

```bash
cd $SERVICE-controller

# Create Python virtual environment
python3 -m venv test/e2e/.venv
source test/e2e/.venv/bin/activate

# Install dependencies
pip install -r test/e2e/requirements.txt
pip install setuptools  # Required for Python 3.13+

# Set required environment variables
export AWS_ACCOUNT_ID=123456789012
export AWS_REGION=us-west-2
```

:::warning Use test accounts only
E2E tests create and delete real AWS resources. Always use a dedicated test account.
:::

## Running Tests

```bash
source test/e2e/.venv/bin/activate
python -m pytest test/e2e/tests/test_<resource>.py -v
```

## Test File Structure

```
test/e2e/
├── __init__.py              # Service constants and load helper
├── conftest.py              # pytest fixtures (boto3 client)
├── requirements.txt         # acktest dependency (pins test-infra commit)
├── bootstrap_resources.py   # Bootstrap resource loader
├── service_bootstrap.py     # Bootstrap lifecycle (create prereqs)
├── service_cleanup.py       # Cleanup lifecycle (delete prereqs)
├── resources/
│   └── <resource>.yaml      # YAML fixtures with $VARIABLE placeholders
└── tests/
    ├── __init__.py
    └── test_<resource>.py   # Test cases
```

## Writing E2E Tests

### Required Coverage

Every resource PR must include E2E tests that verify:

1. **Create** — Resource is created and reaches synced state
2. **Update** — At least one field is modified successfully
3. **Delete** — Resource is cleaned up
4. **Synced condition** — `ACK.ResourceSynced` is `True` after each operation

### Test Pattern

```python
from acktest.resources import random_suffix_name
from acktest.k8s import resource as k8s
from acktest.k8s import condition
import time

CREATE_WAIT_AFTER_SECONDS = 10

def test_create_delete(self, service_client):
    resource_name = random_suffix_name("ack-test", 24)
    replacements = {"RESOURCE_NAME": resource_name}

    resource_data = load_resource("my_resource", additional_replacements=replacements)
    ref = k8s.CustomResourceReference(
        CRD_GROUP, CRD_VERSION, RESOURCE_PLURAL,
        resource_name, namespace="default"
    )
    k8s.create_custom_resource(ref, resource_data)
    cr = k8s.wait_resource_consumed_by_controller(ref)
    assert cr is not None

    time.sleep(CREATE_WAIT_AFTER_SECONDS)
    assert k8s.wait_on_condition(
        ref, condition.CONDITION_TYPE_RESOURCE_SYNCED, "True", wait_periods=5
    )

    # Verify in AWS
    aws_resource = service_client.describe_resource(Name=resource_name)
    assert aws_resource is not None

    # Cleanup
    _, deleted = k8s.delete_custom_resource(ref)
    assert deleted
```

### AWS API Assertions

Reviewers expect tests to verify state via both the Kubernetes CR and direct AWS API calls:

```python
# Verify via CR
cr = k8s.get_resource(ref)
assert cr["spec"]["fieldName"] == "expected-value"

# Verify via AWS API
aws_resource = service_client.describe_resource(Name=resource_name)
assert aws_resource["FieldName"] == "expected-value"
```

### Bootstrap Resources

Some resources depend on other AWS resources that must exist before the test runs (e.g., a VPC for a subnet group, or an IAM role for a service). Use `service_bootstrap.py` to create these prerequisites and `service_cleanup.py` to tear them down. Bootstrap resources are created once per test session and shared across tests.

## CI Testing with Prow

ACK uses [Prow](https://prow.ack.aws.dev/) for continuous integration. When you open a PR, Prow runs several jobs:

- **unit-test** — Runs `make test`
- **verify-code-gen** — Verifies generated code is up to date
- **kind-e2e** — Spins up a kind cluster, installs the controller, and runs E2E tests against real AWS

The kind-e2e job uses a service team IAM role to authenticate with AWS. This role is registered in SSM and scoped to the specific service's APIs. If your controller is new, you'll need to onboard it to the test infrastructure — see the [test-infra onboarding guide](https://github.com/aws-controllers-k8s/test-infra/blob/main/docs/onboarding.md).

Check the Prow build logs for test failures — both the test output and controller logs are available as build artifacts.

## Common Issues

- **boto3 too old for new API fields** — Update the test-infra pin in `test/e2e/requirements.txt` to a newer commit with a current boto3
- **Slow-provisioning resources** — Pass explicit `timeout_seconds` to `wait_until()` rather than bumping global defaults
- **Python 3.13+** — Install `setuptools` separately (`pip install setuptools`)
- **Flaky tests** — Usually timing issues; add retries or increase wait times

## Next Steps

- [Releasing a controller](releasing.md)
- [Hooks and custom code](code-generation/hooks.md)
