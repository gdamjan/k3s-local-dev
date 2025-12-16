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
```
