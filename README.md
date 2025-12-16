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

install -Dm644 ~/.config/containers/registries.conf.d/ ./podman/50-k3s-local-registry.conf
install -Dm644 ~/.config/containers/registries.conf.d/ ./podman/50-search-docker-io.conf
```


## Explainer

I want to customize the default `k3s` install with some enhancements and my opinions,
for the local development use-case. We gonna run a local docker image registry too.

### `cluster-domain.yaml`

This config file changes the `.local` <abbr title="Top-Level Domain">TLD</abbr> with
`.internal` for the cluster domain of k3s. With that, the domain used by the cluster
is `cluster.internal` instead of `cluster.local`.

Reason is, `.local` is reserved for the mDNS protocol, and really shouldn't be used for anything
else (see [RFC 6762](https://datatracker.ietf.org/doc/html/rfc6762#section-3)).
Instead, the `.internal` private TLD has been standardized exactly for this purpose.

This will change how pods and services are named and resolved.


### `kubeconfig.yaml`

Allow anyone in the `wheel` unix group to have access to the k3s `k3s.yaml` kube-config file.
By default `/etc/rancher/k3s/k3s.yaml` is readable by root only.
Let's have it generated in `/run`, readable by the `wheel` group too.

Anyone in the `wheel` group can now have `export KUBECONFIG=/run/k3s.yaml` and run any
kubernetes tool, like `kubectl`, `k9s`, etc…


### Running a local registry - `registries.yaml`

For local development, we need a way to run our own applications in the local cluster.
So we need to build our own custom images and provide them to the cluster. Kubernetes/k3s
can't use images from the local docker or podman storage, so we need to push these
images to a local registry instead, and make kubernetes use them from there.

We can run the registry inside kubernetes 😉, and a suitable one is the 
[Distribution Registry](https://distribution.github.io/distribution/) project
(see the `./manifests` directory).

The `registries.yaml` config file tells the `k3s` cluster that the local
registry (`registry.localhost`) will not need https, and is running on port 80/http.


#### Using the local registry:

##### With `podman`:
```
podman build -t demo .
podman push demo docker://registry.localhost/demo:latest
```

##### With `skopeo`:
```
skopeo copy docker://docker.io/alpine:3 docker://registry.localhost/alpine:3

k3s kubectl run hello -it --restart=Never --image=registry.localhost/alpine:3
k3s kubectl delete pod hello
```

> [!NOTE]
> Configure `registry.localhost` as an [insecure registry](https://github.com/containers/image/blob/main/docs/containers-registries.conf.5.md) in podman.

##### With `docker`:
```
docker build -t demo .
docker tag demo registry.localhost/demo:latest
docker push registry.localhost/demo:latest
```
> [!NOTE]
> Configure `registry.localhost` as an [insecure registry](https://docs.docker.com/registry/insecure/)


## References:

Alternatives to a local registry are:
- [ttl.sh](https://ttl.sh) - anonymous & ephemeral docker image registry
- [hub.docker.com](https://hub.docker.com) - Docker Hub
- [quay.io](https://quay.io) - Redhat supported registry, alternative to docker hub
- [ghcr.io](https://docs.github.com/en/packages) - Github Packages provides an image registry
