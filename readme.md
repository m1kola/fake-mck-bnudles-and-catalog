# Fake catalog and bundle

## Prerequisites

We need two repos for the bundles and for a catalog.
I recommend creating public repos so we don't have to deal with pull secrets.

1. Login with `docker` CLI to the registry (Docker Hub for this example)
1. Create a repository for the bundle.
1. Create a repository for the catalog.


## Fake bundle

```bash
# Note --platform linux/x86_64 - required if running on a Mac with ARM processor.
docker build . \
    --platform linux/x86_64 \
    -f bundle/1.32.0/bundle.Dockerfile \
    -t docker.io/mikalairadchuk070/fake-mck-operator:1.32.0
docker push docker.io/mikalairadchuk070/fake-mck-operator:1.32.0
```

## Fake catalog

mikalairadchuk070/fake-mck-catalog

```bash
docker build . \
    --platform linux/x86_64 \
    -f catalog/fake-mck-catalog.Dockerfile \
    -t docker.io/mikalairadchuk070/fake-mck-catalog:latest
docker push docker.io/mikalairadchuk070/fake-mck-catalog:latest
```
