# Épisode 00 — Installation & Setup

## Objectif
Installer Kind, kubectl, k9s et Helm sur WSL2 Ubuntu 24.04.

## Commandes

```bash
# Installer kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# Installer Kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind

# Créer le cluster
kind create cluster --config kind-kubeshop-cluster.yaml

# Vérifier
kubectl get nodes
```

## Résultat attendu
NAME                      STATUS   ROLES           AGE
kubeshop-control-plane    Ready    control-plane   2m
kubeshop-worker           Ready    <none>          2m
kubeshop-worker2          Ready    <none>          2m
