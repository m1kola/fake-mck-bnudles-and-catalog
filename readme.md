# Fake catalog and bundle

## Prerequisites

We need two repos for the bundles and for a catalog.
I recommend creating public repos so we don't have to deal with pull secrets.

1. Login with `docker` CLI to the registry (Docker Hub for this example)
1. Create a repository for the bundle.
1. Create a repository for the catalog.


## Fake bundle

```bash
docker buildx build . \
    --platform linux/arm64/v8,linux/amd64 \
    -f bundle/1.32.0/bundle.Dockerfile \
    -t docker.io/mikalairadchuk070/fake-mck-operator:1.32.0
docker push docker.io/mikalairadchuk070/fake-mck-operator:1.32.0
```

## Fake catalog

mikalairadchuk070/fake-mck-catalog

```bash
docker buildx build . \
    --platform linux/arm64/v8,linux/amd64 \
    -f catalog/fake-mck-catalog.Dockerfile \
    -t docker.io/mikalairadchuk070/fake-mck-catalog:latest
docker push docker.io/mikalairadchuk070/fake-mck-catalog:latest
```

# Tests

Tested this with a kind cluster and [OLM v0.31.0 upstream release](https://github.com/operator-framework/operator-lifecycle-manager/releases/tag/v0.31.0) installed using the install script.

Tests assume OLM installed the `olm` namespace.

**Note:** tests share names. They require cleanup.
Normally you can run `kubectl delete -f FILE` for each file from the example.

## Fresh fake MCK installation

This is is just to verify that fake MCK catalog and bundle is installable.

```
kubectl apply -f tests/test-fresh-install/fake-mck.yml
```

You can check status of the OLM resources like this:

```
kubectl wait -n test-fake-mck sub fake-mck --for jsonpath='{.status.state}'="AtLatestKnown"
kubectl wait -n test-fake-mck csv fake-mck-operator.v1.32.0 --for jsonpath='{.status.phase}'="Succeeded"
```

You should see the operator pod running. Also crds, RBAC and other resources installed.

## Install MEKO and fake MCK together

This is to demonstrate CRD conflict during resolution (before the operator is actually installed).

### Install MEKO
```
kubectl apply -f tests/meko-and-fake-mck/01-meko.yml
```

### Wait for MEKO to be installed
```
kubectl wait -n test-fake-mck sub mongodb-enterprise --for jsonpath='{.status.state}'="AtLatestKnown"
kubectl wait -n test-fake-mck csv mongodb-enterprise.v1.32.0 --for jsonpath='{.status.phase}'="Succeeded"
```

### Install fake MCK

```
kubectl apply -f tests/meko-and-fake-mck/02-fake-mck.yml
```

You will see that OLM fails to install MCK. On the `Subscription` CR for both operators there will be the `ResolutionFailed=True` condition.

When running this:
```
kubectl -n test-fake-mck get sub mongodb-enterprise -o yaml | yq ".status.conditions"
```

You should see something like this:

```yaml
- message: 'constraints not satisfiable: subscription mongodb-enterprise requires @existing/test-fake-mck//mongodb-enterprise.v1.32.0, subscription mongodb-enterprise exists, subscription fake-mck exists, subscription fake-mck requires fake-mck-catalog/olm/stable-v1/fake-mck-operator.v1.32.0, @existing/test-fake-mck//mongodb-enterprise.v1.32.0 and fake-mck-catalog/olm/stable-v1/fake-mck-operator.v1.32.0 provide MongoDBOpsManager (mongodb.com/v1)'
  reason: ConstraintsNotSatisfiable
  status: "True"
  type: ResolutionFailed
```

## Uninstall MEKO and install fake MCK

In this test we try the proposed migration scenario when we uninstall MEKO
and install MCK which takes over existing CRs.

### Install MEKO

To simulate existing setup.

```
kubectl apply -f tests/meko-and-fake-mck/01-meko.yml
```

### Wait for MEKO to be installed
```
kubectl wait -n test-fake-mck sub mongodb-enterprise --for jsonpath='{.status.state}'="AtLatestKnown"
kubectl wait -n test-fake-mck csv mongodb-enterprise.v1.32.0 --for jsonpath='{.status.phase}'="Succeeded"
```

**Note:** Ideally at this point for complete testing we should also deploy workloads.

### Uninstall MEKO

```
kubectl -n test-fake-mck delete sub mongodb-enterprise
kubectl -n test-fake-mck delete csv mongodb-enterprise.v1.32.0
```

### Install fake MCK

```
kubectl apply -f tests/meko-and-fake-mck/02-fake-mck.yml
```

### Wait for the fake MCK to be installed
```
kubectl wait -n test-fake-mck sub fake-mck --for jsonpath='{.status.state}'="AtLatestKnown"
kubectl wait -n test-fake-mck csv fake-mck-operator.v1.32.0 --for jsonpath='{.status.phase}'="Succeeded"
```
