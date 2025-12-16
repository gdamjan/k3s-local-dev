# `K3s for local development`

> [K3s](https://docs.k3s.io/) - Lightweight Kubernetes. Easy to install, half the memory, all in a binary of less than 100 MB.

## Quickstart
```
sudo install -m644 ./k3s/cluster-domain.yaml /etc/rancher/k3s/config.yaml.d/
sudo install -m644 ./k3s/kubeconfig.yaml /etc/rancher/k3s/config.yaml.d/
sudo install -m644 ./k3s/registries.yaml /etc/rancher/k3s/

sudo systemctl restart k3s

k3s kubectl create namespace docker-registry
k3s kubectl apply -f manifests/ --namespace docker-registry
```
