# 🚀 KubeShop — Apprends Kubernetes par la pratique

## 📺 La série

| Épisode | Dossier | Concepts |
|---------|---------|----------|
| 00 | [00-setup](./00-setup/) | Kind, kubectl, k9s, Helm |
| 01 | [01-pods-namespaces](./01-pods-namespaces/) | Pod, Namespace |
| 02 | [02-configmaps-secrets](./02-configmaps-secrets/) | ConfigMap, Secret |
| 03 | [03-deployments](./03-deployments/) | Deployment, ReplicaSet |
| 04 | [04-services](./04-services/) | ClusterIP, NodePort |
| 05 | [05-volumes](./05-volumes/) | PV, PVC, StorageClass |
| 06 | [06-ingress](./06-ingress/) | Ingress, Nginx Controller |
| 07 | [07-scaling](./07-scaling/) | HPA, Resources |
| 08 | [08-scheduling](./08-scheduling/) | Affinity, Taints |
| 09 | [09-network-policies](./09-network-policies/) | NetworkPolicy |
| 10 | [10-rbac](./10-rbac/) | RBAC, ServiceAccount |
| 11 | [11-crd-operator](./11-crd-operator/) | CRD, Operator |
| 12 | [12-gateway-api](./12-gateway-api/) | Gateway API |
| 13 | [13-helm](./13-helm/) | Helm Charts |
| 14 | [14-kustomize](./14-kustomize/) | Kustomize |

## 🛠️ Prérequis

- WSL2 Ubuntu 24.04
- Docker Desktop (intégration WSL2 activée)
- Kind, kubectl, Helm, k9s

## 🚀 Démarrage rapide

```bash
# Cloner le repo
git clone https://github.com/TON_USERNAME/kubernetes-kubeshop.git
cd kubernetes-kubeshop

# Créer le cluster
kind create cluster --config 00-setup/kind-kubeshop-cluster.yaml

# Vérifier
kubectl get nodes
```

