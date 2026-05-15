# 🎬 Épisode 01 — Pods & Namespaces

## 🎯 Ce qu'on apprend

- Créer un Namespace pour isoler nos ressources
- Créer des Pods (l'unité de base de Kubernetes)
- Comprendre labels, probes, resources
- Observer et déboguer avec kubectl

---

## 🏗️ Architecture de cet épisode

```text
Cluster Kind "kubeshop"
│
├── Namespace: default      (système)
├── Namespace: kube-system  (composants K8s)
└── Namespace: kubeshop     ← notre espace ✅
    ├── Pod: frontend       (Nginx sur port 80)
    └── Pod: backend        (http-echo sur port 5678)

# 1. Créer le namespace
kubectl apply -f namespace.yaml

# Vérifier
kubectl get namespaces
kubectl get ns kubeshop

# 2. Créer les pods
kubectl apply -f pod-frontend.yaml
kubectl apply -f pod-backend.yaml

# 3. Vérifier l'état des pods
kubectl get pods -n kubeshop
kubectl get pods -n kubeshop -o wide   # + IP et nœud

# 4. Attendre que les pods soient Running
kubectl wait --for=condition=Ready pod/frontend -n kubeshop --timeout=60s
kubectl wait --for=condition=Ready pod/backend  -n kubeshop --timeout=60s

# Voir les détails complets d'un pod (très utile pour déboguer)
kubectl describe pod frontend -n kubeshop

# Points importants dans le describe :
#   - Node:         sur quel nœud il tourne
#   - IP:           adresse IP interne du Pod
#   - Containers:   état du container
#   - Conditions:   Ready / PodScheduled / ContainersReady
#   - Events:       historique → c'est ici que se cachent les erreurs !

# Voir les logs du container
kubectl logs frontend -n kubeshop
kubectl logs backend  -n kubeshop

# Logs en live (follow)
kubectl logs -f frontend -n kubeshop

# Shell dans le container frontend
kubectl exec -it frontend -n kubeshop -- /bin/sh

# Depuis le shell : tester le backend depuis le frontend
# (communication Pod à Pod par IP)
kubectl get pod backend -n kubeshop -o jsonpath='{.status.podIP}'
# → note l'IP, par ex: 10.244.1.5

# Dans le shell du frontend :
wget -qO- http://10.244.1.5:5678
# → 🛍️ KubeShop API — épisode 01 — ça marche !
exit

# Port-forward : crée un tunnel temporaire vers le Pod
kubectl port-forward pod/frontend -n kubeshop 8080:80 &
kubectl port-forward pod/backend  -n kubeshop 8081:5678 &

# Tester
curl http://localhost:8080    # → page Nginx
curl http://localhost:8081    # → message KubeShop API

# Couper les port-forwards
kill %1 %2

# Supprimer les pods (le namespace reste)
kubectl delete pod frontend backend -n kubeshop

# OU supprimer via les fichiers
kubectl delete -f pod-frontend.yaml
kubectl delete -f pod-backend.yaml

# Vérifier
kubectl get pods -n kubeshop
# No resources found in kubeshop namespace.