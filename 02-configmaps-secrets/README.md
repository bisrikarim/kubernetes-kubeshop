# 🎬 Épisode 02 — ConfigMaps & Secrets

## 🎯 Ce qu'on apprend

- Comprendre la différence entre ConfigMap et Secret
- Créer un ConfigMap pour la configuration non sensible
- Créer un Secret pour les credentials
- Injecter la config dans un Pod (2 méthodes)
- Les pièges classiques : interpolation dans args, doublons, secrets en Git
- Observer et déboguer avec kubectl

---

## 🧠 Concepts clés

### Le problème qu'on résout

Sans ConfigMap ni Secret, la configuration est **hardcodée dans l'image Docker** :

```
❌ Sans ConfigMap/Secret              ✅ Avec ConfigMap/Secret

┌─────────────────────┐              ┌─────────────────────┐
│ Image Docker        │              │ Image Docker        │
│  DB_HOST=localhost  │              │  (image générique)  │
│  DB_PORT=5432       │              └──────────┬──────────┘
│  DB_PASS=s3cr3t     │                         │ injecte
│  APP_ENV=prod       │              ┌──────────▼──────────┐
└─────────────────────┘              │ ConfigMap           │
  → rebuild si changement            │  DB_HOST, APP_ENV   │
  → secret visible dans image        ├─────────────────────┤
  → impossible multi-env             │ Secret              │
                                     │  DB_PASS (base64)   │
                                     └─────────────────────┘
                                       → config séparée du code
                                       → même image, configs différentes
```

### ConfigMap vs Secret

| | ConfigMap | Secret |
|---|---|---|
| **Usage** | Config non sensible | Données sensibles |
| **Exemples** | APP_ENV, DB_HOST, PORT | passwords, tokens, clés |
| **Stockage etcd** | En clair | Encodé base64 |
| **Committer en Git** | ✅ OK | ❌ JAMAIS |

> ⚠️ **Important** : base64 n'est PAS du chiffrement. C'est juste un encodage.
> La vraie sécurité des Secrets vient du chiffrement etcd + RBAC (épisode 10).

---

### ⚠️ Piège #1 — Interpolation des variables dans `args`

```yaml
# ❌ NE FONCTIONNE PAS
# envFrom n'est PAS disponible pour $(VAR) dans args/command
envFrom:
  - configMapRef:
      name: backend-config
args:
  - "-text=$(APP_NAME)"   # → affiche littéralement "$(APP_NAME)"

# ✅ FONCTIONNE
# La variable DOIT être déclarée dans env: pour être interpolée dans args
env:
  - name: APP_NAME
    valueFrom:
      configMapKeyRef:
        name: backend-config
        key: APP_NAME
args:
  - "-text=$(APP_NAME)"   # → affiche "KubeShop API" ✅
```

### ⚠️ Piège #2 — Doublon entre ConfigMap et Secret

```yaml
# ❌ CONFLIT SILENCIEUX
# Si DB_NAME est dans ConfigMap ET dans Secret
# → le Secret écrase le ConfigMap sans aucun avertissement

# ✅ BONNE PRATIQUE — séparation claire :
# ConfigMap → DB_HOST, DB_PORT, DB_NAME, APP_ENV, LOG_LEVEL  (pas sensible)
# Secret    → DB_USER, DB_PASSWORD, APP_SECRET_KEY            (sensible)
```

### ⚠️ Piège #3 — Images ultra-minimales sans shell

```bash
# Les images "distroless" (comme hashicorp/http-echo) n'ont pas de shell
# kubectl exec backend -- env    → executable file not found

# Solution : pod de debug temporaire avec busybox
kubectl apply -f /tmp/pod-debug.yaml
kubectl logs debug-env -n kubeshop
kubectl delete pod debug-env -n kubeshop
```

---

## 🏗️ Architecture de cet épisode

```
Namespace: kubeshop
│
├── ConfigMap: backend-config
│     APP_ENV=production
│     APP_NAME=KubeShop API
│     APP_PORT=5678
│     DB_HOST=postgres
│     DB_PORT=5432
│     DB_NAME=kubeshop
│     LOG_LEVEL=info
│     app-message=(texte multi-lignes)
│
├── Secret: db-secret
│     DB_USER=admin              (stocké encodé en base64)
│     DB_PASSWORD=kubeshop123    (stocké encodé en base64)
│     APP_SECRET_KEY=***         (stocké encodé en base64)
│
├── Pod: frontend                (inchangé — épisode 01)
│
└── Pod: backend (v2)
      env: APP_NAME, APP_ENV, APP_PORT  ← depuis ConfigMap (interpolation args)
      env: DB_HOST, DB_PORT, DB_NAME    ← depuis ConfigMap
      env: DB_USER, DB_PASSWORD         ← depuis Secret
      env: APP_SECRET_KEY               ← depuis Secret
```

---

## 📁 Fichiers de l'épisode

```
02-configmaps-secrets/
├── configmap-backend.yaml   ← configuration non sensible
├── secret-db.yaml           ← credentials (base64)
├── pod-backend-v2.yaml      ← pod backend qui consomme la config
├── .gitignore               ← protection fichiers secrets locaux
└── README.md                ← ce fichier
```

---

## 🚀 Commandes — dans l'ordre

### 1. Créer le ConfigMap

```bash
kubectl apply -f 02-configmaps-secrets/configmap-backend.yaml

# Vérifier
kubectl get configmap -n kubeshop
# NAME             DATA   AGE
# backend-config   8      Xs
```

### 2. Créer le Secret

```bash
kubectl apply -f 02-configmaps-secrets/secret-db.yaml

# Vérifier (les valeurs sont masquées)
kubectl get secret -n kubeshop
# NAME        TYPE     DATA   AGE
# db-secret   Opaque   3      Xs
```

### 3. Encoder/décoder en base64

```bash
# Encoder une valeur
echo -n "monpassword" | base64
# → bW9ucGFzc3dvcmQ=

# Décoder une valeur
echo "bW9ucGFzc3dvcmQ=" | base64 --decode
# → monpassword

# Décoder directement depuis le cluster
kubectl get secret db-secret -n kubeshop \
  -o jsonpath='{.data.DB_PASSWORD}' | base64 --decode
# → kubeshop123
```

### 4. Remplacer le pod backend (v1 → v2)

```bash
# Supprimer l'ancien pod (épisode 01)
kubectl delete pod backend -n kubeshop

# Créer le nouveau pod avec la config injectée
kubectl apply -f 02-configmaps-secrets/pod-backend-v2.yaml

# Attendre qu'il soit Running
kubectl get pods -n kubeshop -w
# Ctrl+C quand STATUS = Running
```

### 5. Vérifier que tout fonctionne

```bash
# Test HTTP — vérifier l'interpolation des variables dans args
kubectl port-forward pod/backend -n kubeshop 8082:5678 &
sleep 2
curl http://localhost:8082
# ✅ Attendu : 🛍️ KubeShop API — ENV=production — Port=5678
kill $(pgrep -f "kubectl port-forward")
```

---

## 🔍 Observer ConfigMap et Secret

```bash
# Voir le contenu complet du ConfigMap
kubectl get configmap backend-config -n kubeshop -o yaml

# Voir le Secret (valeurs en base64)
kubectl get secret db-secret -n kubeshop -o yaml

# Voir les variables injectées dans le pod via describe
kubectl describe pod backend -n kubeshop | grep -A 40 "Environment:"
# Tu verras :
# Environment:
#   APP_NAME:      <set to the key 'APP_NAME' of config map 'backend-config'>
#   DB_PASSWORD:   <set to the key 'DB_PASSWORD' in secret 'db-secret'>
#   ...
```

---

## 🐛 Pod de debug temporaire

Quand l'image du pod n'a pas de shell, utiliser un pod busybox :

```yaml
# /tmp/pod-debug.yaml
apiVersion: v1
kind: Pod
metadata:
  name: debug-env
  namespace: kubeshop
spec:
  restartPolicy: Never
  containers:
    - name: debug
      image: busybox
      command: ["sh", "-c", "env | grep -E 'APP_|DB_|LOG_' && sleep 3600"]
      envFrom:
        - configMapRef:
            name: backend-config
        - secretRef:
            name: db-secret
```

```bash
# Appliquer
kubectl apply -f /tmp/pod-debug.yaml

# Voir les variables injectées
kubectl logs debug-env -n kubeshop
# APP_ENV=production
# APP_NAME=KubeShop API
# APP_PORT=5678
# DB_HOST=postgres
# DB_NAME=kubeshop
# DB_PASSWORD=kubeshop123
# DB_PORT=5432
# DB_USER=admin
# LOG_LEVEL=info

# Nettoyage
kubectl delete pod debug-env -n kubeshop
```

---

## 🐛 Erreurs classiques & Solutions

| Erreur | Cause | Solution |
|--------|-------|----------|
| `$(VAR)` affiché littéralement | Variable dans `envFrom` au lieu de `env:` | Déclarer la variable dans `env:` avec `valueFrom` |
| `CreateContainerConfigError` | ConfigMap ou Secret introuvable | Vérifier le nom et le namespace |
| Variable non mise à jour | ConfigMap modifié sans restart du Pod | Supprimer et recréer le Pod |
| Conflit silencieux de valeur | Même clé dans ConfigMap ET Secret | Chaque clé dans une seule source |
| `executable file not found` | Image sans shell (distroless) | Utiliser un pod de debug busybox |
| Port déjà utilisé (port-forward) | Ancien port-forward non tué | `kill $(pgrep -f "kubectl port-forward")` |

---

## ✅ Bonnes pratiques

```bash
# 1. Ne JAMAIS committer de vrais secrets
# Ajouter dans .gitignore :
*-real.yaml
*-prod.yaml
*.env

# 2. Toujours pin une version d'image
image: nginx:1.25      # ✅
image: nginx:latest    # ❌

# 3. Séparer clairement ConfigMap et Secret
# ConfigMap → tout ce qui n'est PAS sensible
# Secret    → UNIQUEMENT les credentials

# 4. En production → ne pas utiliser de Secrets "nus"
# Utiliser : Sealed Secrets, HashiCorp Vault, AWS Secrets Manager

# 5. Vérifier l'encodage base64 avant d'appliquer
echo -n "mavaleur" | base64   # encoder
echo "bWF2YWxldXI=" | base64 --decode   # vérifier
```

---

## 🧹 Nettoyage (optionnel — fin de l'épisode seulement)

> ⚠️ Ne pas exécuter si tu enchaînes avec l'épisode 03 !

```bash
kubectl delete pod backend -n kubeshop
kubectl delete configmap backend-config -n kubeshop
kubectl delete secret db-secret -n kubeshop
```

---

## 📊 État du cluster en fin d'épisode

```
Namespace: kubeshop
│
├── ConfigMap : backend-config   ✅
├── Secret    : db-secret        ✅
├── Pod       : frontend         ✅ (épisode 01 — inchangé)
└── Pod       : backend          ✅ (épisode 02 — args interpolés correctement)
```

---

## ➡️ Prochain épisode

**Épisode 03 — Deployments & ReplicaSets**

On abandonne les Pods "nus" et on passe à la vraie façon de déployer en Kubernetes :
- Self-healing automatique
- Rolling updates sans downtime
- Rollback en une commande
- Gestion des ReplicaSets