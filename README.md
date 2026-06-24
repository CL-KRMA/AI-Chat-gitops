# AI-Chat GitOps

Ce repo contient les manifests Kubernetes du projet AI-Chat, déployés automatiquement via ArgoCD selon le pattern GitOps. Toute modification poussée sur `main` est détectée et appliquée par ArgoCD sur le cluster K3s.

## 📋 Table des matières

- [Vue d'ensemble](#vue-densemble)
- [Architecture](#architecture)
- [Prérequis](#prérequis)
- [Structure du repo](#structure-du-repo)
- [Déploiement](#déploiement)
- [Manifests](#manifests)
- [Modèles IA](#modèles-ia)
- [Mise à jour de l'image](#mise-à-jour-de-limage)
- [Dépannage](#dépannage)

## 🎯 Vue d'ensemble

Ce repo GitOps orchestre le déploiement de deux composants principaux sur Kubernetes :

- **Ollama** — serveur d'inférence IA avec 3 modèles (llama3:8b, llava:7b, gemma4:12b)
- **Frontend Next.js** — interface de chat exposée via Traefik Ingress

Le pipeline CI/CD du repo [AI-Chat-app](https://github.com/CL-KRMA/AI-Chat-app) met automatiquement à jour le tag de l'image dans `frontend.yaml` à chaque push sur `main`, déclenchant un déploiement ArgoCD.

## 🏗️ Architecture

```
                        Internet
                            │
                            ▼
                   Traefik Ingress
                   (ai-chat.com:80)
                            │
                            ▼
                ┌───────────────────────┐
                │  Namespace : ai-chat  │
                │                       │
                │  ┌─────────────────┐  │
                │  │ Frontend        │  │
                │  │ Next.js         │  │
                │  │ port 3000→80    │  │
                │  └────────┬────────┘  │
                │           │           │
                │           ▼           │
                │  ┌─────────────────┐  │
                │  │ Ollama          │  │
                │  │ port 11434      │  │
                │  │                 │  │
                │  │ llama3:8b       │  │
                │  │ llava:7b        │  │
                │  │ gemma4:12b      │  │
                │  └────────┬────────┘  │
                │           │           │
                │  ┌────────▼────────┐  │
                │  │ PVC 30Gi        │  │
                │  │ (modèles IA)    │  │
                │  └─────────────────┘  │
                └───────────────────────┘
```

## 📦 Prérequis

- Cluster K3s opérationnel (3 nœuds minimum)
- ArgoCD installé sur le cluster
- Traefik installé (inclus par défaut avec K3s)
- DNS `ai-chat.com` pointant vers l'IP publique du master
- Nœud avec minimum **32GB RAM** pour Ollama

## 📁 Structure du repo

```
AI-Chat-gitops/
├── namespace.yaml    # Namespace ai-chat
├── ollama.yaml       # Deployment Ollama + PVC + Service
├── frontend.yaml     # Deployment Next.js + Service
├── ingress.yaml      # Ingress Traefik (ai-chat.com)
└── README.md
```

## 🚀 Déploiement

### Déploiement manuel (première fois)

```bash
# 1. Créer le namespace
kubectl apply -f namespace.yaml

# 2. Déployer Ollama (le pull des modèles prend 20-30 min)
kubectl apply -f ollama.yaml

# 3. Déployer le frontend
kubectl apply -f frontend.yaml

# 4. Configurer l'Ingress
kubectl apply -f ingress.yaml
```

### Déploiement automatique via ArgoCD

Après la configuration initiale, ArgoCD surveille ce repo et applique automatiquement tout changement poussé sur `main`.

**Configurer ArgoCD :**

```bash
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ai-chat
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/CL-KRMA/AI-Chat-gitops
    targetRevision: main
    path: .
  destination:
    server: https://kubernetes.default.svc
    namespace: ai-chat
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF
```

## 📄 Manifests

### namespace.yaml

Crée le namespace `ai-chat` avec les labels `project` et `environment` pour filtrer facilement les ressources.

### ollama.yaml

Contient trois ressources :

**Deployment Ollama** avec deux conteneurs :
- `initContainer (pull-models)` — télécharge les 3 modèles au premier démarrage avant de lancer le serving. Si un pull échoue, le pod reste en `Init:Error` — comportement explicite et détectable.
- `Container (ollama)` — lance `ollama serve` en foreground. Si le process crash, Kubernetes redémarre automatiquement le pod.

**PersistentVolumeClaim** — 30Gi pour stocker les modèles (~16.6GB au total + marge).

**Service** — expose Ollama sur le port `11434` en interne au cluster.

### frontend.yaml

**Deployment Next.js** avec :
- Image mise à jour automatiquement par le pipeline CI à chaque push
- Variable d'environnement `OLLAMA_API_URL=http://ollama:11434`
- `readinessProbe` — Kubernetes attend que Next.js soit prêt avant d'envoyer du trafic
- `livenessProbe` — Kubernetes redémarre le pod si Next.js ne répond plus
- Resources limitées à 512Mi RAM / 500m CPU

**Service** — mappe le port `80` vers le port `3000` du container.

### ingress.yaml

Expose le frontend sur `ai-chat.com` via Traefik (`ingressClassName: traefik`). Tout le trafic entrant sur `/` est routé vers le Service `ai-chat-frontend` sur le port `80`.

## 🤖 Modèles IA

| Modèle | Taille | Usage |
|---|---|---|
| `llama3:8b` | ~5GB | Génération de texte (route `/api/llama`) |
| `llava:7b` | ~4GB | Analyse texte + images (route `/api/llava`) |
| `gemma4:12b` | ~7.6GB | Multimodal texte/image/audio/vidéo (route `/api/gemma`) |

> **Note** : Le premier démarrage d'Ollama prend **20-30 minutes** pour télécharger les 3 modèles (~16.6GB). Les démarrages suivants sont instantanés car les modèles sont persistés dans le PVC.

## 🔄 Mise à jour de l'image

Le tag de l'image frontend est mis à jour automatiquement par le pipeline CI du repo [AI-Chat-app](https://github.com/CL-KRMA/AI-Chat-app) via ce pattern :

```bash
# Le pipeline CI exécute cette commande après un build réussi
sed -i "s|ghcr.io/cl-krma/ai-chat-app:.*|ghcr.io/cl-krma/ai-chat-app:<SHA>|g" frontend.yaml
git commit -m "chore: update ai-chat-app image to <SHA>"
git push origin main
```

ArgoCD détecte le nouveau commit et déploie automatiquement la nouvelle image.

**Flux complet :**
```
Push sur AI-Chat-app
  → GitHub Actions (lint → tests → build → scan Trivy)
  → Push image sur GHCR
  → Mise à jour frontend.yaml dans ce repo
  → ArgoCD détecte le changement
  → Déploiement sur K3s
```

## 🔧 Dépannage

### Vérifier l'état des pods

```bash
kubectl get pods -n ai-chat
kubectl get pods -n ai-chat -w  # watch en temps réel
```

### Ollama bloqué en Init:Error

```bash
# Voir les logs de l'initContainer
kubectl logs -n ai-chat -l app=ollama -c pull-models

# Causes possibles :
# - Réseau indisponible (pull impossible)
# - Disque plein (vérifier le PVC)
# - Modèle introuvable sur le registry Ollama
```

### Ollama bloqué en Pending

```bash
kubectl describe pod -n ai-chat -l app=ollama

# Causes possibles :
# - Pas assez de RAM disponible sur le nœud (besoin 16Gi)
# - PVC ne peut pas être monté (nœud différent)
```

### Frontend retourne des erreurs 500

```bash
# Vérifier qu'Ollama est bien Running
kubectl get pods -n ai-chat

# Vérifier les logs du frontend
kubectl logs -n ai-chat -l app=ai-chat-frontend

# Tester l'API Ollama directement depuis le cluster
kubectl exec -n ai-chat deploy/ollama -- curl -s http://localhost:11434/api/tags
```

### Ingress ne répond pas

```bash
# Vérifier que Traefik tourne
kubectl get pods -n kube-system | grep traefik

# Vérifier l'Ingress
kubectl describe ingress -n ai-chat ai-chat-ingress

# Vérifier que le DNS pointe vers le bon nœud
nslookup ai-chat.com
```

### Voir les modèles installés sur Ollama

```bash
kubectl exec -n ai-chat deploy/ollama -- ollama list
```

### Forcer un re-sync ArgoCD

```bash
argocd app sync ai-chat
# ou depuis l'interface ArgoCD : cliquer sur "Sync"
```

## 📚 Repos liés

- [AI-Chat-app](https://github.com/CL-KRMA/AI-Chat-app) — code source Next.js + pipeline CI/CD
- [aws-terraform-config](https://github.com/CL-KRMA/aws-terraform-config) — infrastructure AWS (K3s cluster)

## 📝 Licence

Ce projet est sous licence MIT.  
Voir le fichier [LICENSE](LICENSE) pour plus de détails.

## 👤 Auteur

Créé par Cheick — Juin 2026
