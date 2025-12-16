# `K3s for local development`

> [K3s](https://docs.k3s.io/) - Lightweight Kubernetes.
> Easy to install, half the memory, all in a binary of less than 100 MB.


## Quickstart
```
sudo install -Dm644 -t /etc/rancher/k3s/config.yaml.d/ ./k3s/cluster-domain.yaml
sudo install -Dm644 -t /etc/rancher/k3s/config.yaml.d/ ./k3s/kubeconfig.yaml
sudo install -Dm644 -t /etc/rancher/k3s/ ./k3s/registries.yaml

sudo systemctl restart k3s

k3s kubectl create namespace docker-registry
k3s kubectl apply -f manifests/ --namespace docker-registry

# configure podman/skopeo
install -Dm644 ~/.config/containers/registries.conf.d/ ./podman/50-k3s-local-registry.conf
install -Dm644 ~/.config/containers/registries.conf.d/ ./podman/50-search-docker-io.conf
```


## tl;dr;

I want to customize the default k3s installation by applying a set of enhancements and
opinionated choices aimed at improving the local development experience. As part of this
setup, we will also run a local container image registry. For building container images,
I prefer using Podman instead of Docker.

# Explainer

## Don't use `.local` for the cluster domain - [`cluster-domain.yaml`](./k3s/cluster-domain.yaml)

This configuration file replaces the default `.local` <abbr title="Top-Level Domain">TLD</abbr>
used by k3s with `.internal` as the cluster domain. As a result, all cluster DNS names will be
rooted under `cluster.internal` rather than `cluster.local`.

The rationale behind this change is that `.local` is formally reserved for the mDNS protocol and
should not be used for conventional DNS resolution or other purposes
(see [RFC 6762](https://datatracker.ietf.org/doc/html/rfc6762#section-3)).
Using `.local` outside of mDNS can lead to name resolution conflicts, unexpected behavior, and
hard-to-debug networking issues—especially on developer machines where mDNS is commonly enabled.

By contrast, `.internal` is a private TLD specifically intended for internal networking and
non-public DNS namespaces, making it a better and more standards-compliant choice for Kubernetes clusters.

Note that changing the cluster domain affects how pod and service DNS names are constructed and
resolved throughout the cluster, so all workloads will transparently use the new
`cluster.internal` suffix.

## Make the cluster available to non-root users - [`kubeconfig.yaml`](./k3s/kubeconfig.yaml)

This configuration allows members of the `wheel` Unix group to access the k3s kubeconfig file.
By default, the k3s kubeconfig (`/etc/rancher/k3s/k3s.yaml`) is readable only by the root user,
which unnecessarily restricts day-to-day cluster interaction.

Instead, the kubeconfig is generated under `/run` with read permissions granted to the `wheel` group.
This makes the cluster accessible to trusted non-root users without compromising overall system security.

As a result, any user in the `wheel` group can simply set:
```
export KUBECONFIG=/run/k3s.yaml
```
and use standard Kubernetes tooling such as `kubectl`, `k9s`, and similar clients.

## Running a local registry – [`registries.yaml`](./k3s/registries.yaml)

For local development, we need a reliable way to run our own applications inside the cluster.
This typically involves building custom container images and making them available to k3s.
Since Kubernetes (and k3s) cannot directly consume images from the local Docker or Podman
image stores, these images must be pushed to a container registry that the cluster can pull from.

To keep everything self-contained, we can run a registry inside the Kubernetes cluster itself 😉.
A well-suited option for this purpose is the [Distribution Registry](https://distribution.github.io/distribution/)
project, with the required manifests located in the `./manifests` directory.

The `registries.yaml` configuration file informs k3s about this local registry. Specifically,
it declares that the registry at `registry.localhost` is reachable over plain HTTP on port 80 and
does not require HTTPS, allowing the cluster to pull images from it without additional TLS configuration.


### Use the local registry:

#### With `podman`:
```
podman build -t demo .
podman push demo docker://registry.localhost/demo:latest
```

#### With `skopeo`:
`skopeo` is a low-level cli utility that performs various operations on container images and image repositories.

```
skopeo copy docker://docker.io/alpine:3 docker://registry.localhost/alpine:3

k3s kubectl run hello -it --restart=Never --image=registry.localhost/alpine:3
k3s kubectl delete pod hello
```

> [!NOTE]
> To configure `registry.localhost` as an
> [insecure registry](https://github.com/containers/image/blob/main/docs/containers-registries.conf.5.md)
> in podman/skopeo, we have used the [`50-k3s-local-registry.conf`](./podman/50-k3s-local-registry.conf)
> config file.


#### With `docker`:
```
docker build -t demo .
docker tag demo registry.localhost/demo:latest
docker push registry.localhost/demo:latest
```
> [!NOTE]
> Configure `registry.localhost` as an [insecure registry](https://docs.docker.com/registry/insecure/).


## Accessing k3s

While k3s comes with its own kubectl (`k3s kubectl`) that works out of the box, sometimes we need
to use 3rd-party kubernetes tools.

#### k9s / kubectl

Using the `--kubeconfig` cli option or exporting the `KUBECONFIG` environment variable, works for both:

```
k9s --kubeconfig /run/k3s.yaml
kubectl --kubeconfig /run/k3s.yaml get pods -A
```

```
export KUBECONFIG=/run/k3s.yaml
kubectl get pods -A
k9s
```

## References:

Alternatives to a local registry are:
- [ttl.sh](https://ttl.sh) - anonymous & ephemeral docker image registry
- [hub.docker.com](https://hub.docker.com) - Docker Hub
- [quay.io](https://quay.io) - Redhat supported registry, alternative to docker hub
- [ghcr.io](https://docs.github.com/en/packages) - Github Packages provides an image registry

Tools:
- [k3s](https://k3s.io/)
- [k9s](https://k9scli.io/)
- [kubectl](https://kubernetes.io/docs/reference/kubectl/)
- [podman](https://podman.io/)
- [skopeo](https://github.com/containers/skopeo)

> [!WARNING]
> Tested on Arch Linux only. Arch Linux provides k3s and podman in pristine state, so the only customizations
> done, are by my config files. Other distros might apply their own opinions out from the package.
