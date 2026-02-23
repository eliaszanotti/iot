# Bonus Part: GitLab Integration

> Objectif : Ajouter GitLab à l'infrastructure de la Partie 3

---

## Vue d'ensemble

Le bonus consiste à intégrer **GitLab** à votre infrastructure K3d/Argo CD existante. Au lieu d'utiliser GitHub comme source GitOps, vous utiliserez votre instance GitLab locale.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    GitLab Local Instance                            │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  - Container Registry (Docker images)                         │  │
│  │  - Git Repository (Kubernetes manifests)                      │  │
│  │  - CI/CD Pipelines                                            │  │
│  │  - GitLab Agent (K8s integration)                             │  │
│  └───────────────────────────────────────────────────────────────┘  │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      Argo CD (GitOps)                               │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  - Watch GitLab repository                                    │  │
│  │  - Deploy from GitLab                                         │  │
│  └───────────────────────────────────────────────────────────────┘  │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      K3d Cluster                                    │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  │
│  │  Namespace: argocd│  │  Namespace: dev  │  │  Namespace: gitlab│  │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Spécifications détaillées

### GitLab Requirements

| Composant | Description |
|-----------|-------------|
| **Version** | Dernière version disponible |
| **Local** | Doit tourner localement (pas GitLab.com) |
| **Namespace** | `gitlab` dédié |
| **Intégration** | Doit fonctionner avec le cluster K3d |

### Fonctionnalités requises

- GitLab UI accessible
- Registry Docker fonctionnelle
- Repository Git pour les manifests Kubernetes
- Intégration avec le cluster (via agent ou certificats)
- **Tout de la Partie 3 doit continuer à fonctionner**

---

## Architecture GitLab sur Kubernetes

```
GitLab Components (namespace: gitlab)
├── gitlab-webservice (UI/API)
├── gitlab-workhorse (Reverse proxy)
├── gitlab-sidekiq (Background jobs)
├── gitlab-gitaly (Git storage)
├── gitlab-rails (Main application)
├── postgresql (Database)
├── redis (Cache)
├── registry (Docker registry)
└── cert-manager (TLS certificates)
```

---

## Approches d'installation

### Option 1: Helm Chart (Recommandé)

**Avantages** :
- Installation standardisée
- Mises à jour facilitées
- Configuration flexible
- Support officiel

**Inconvénients** :
- Configuration complexe
- Beaucoup de paramètres

### Option 2: Docker Compose + K3d

**Avantages** :
- Plus simple à comprendre
- GitLab isolé du cluster

**Inconvénients** :
- Moins intégré
- Plus difficile à faire évaluer

---

## Structure du dossier bonus/

```
bonus/
├── Vagrantfile              # VM avec plus de ressources
├── scripts/
│   ├── install_tools.sh     # Docker, K3d, Helm, kubectl...
│   ├── create_cluster.sh    # Cluster K3d avec ressources suffisantes
│   ├── install_gitlab.sh    # Installation GitLab via Helm
│   ├── configure_gitlab.sh  # Configuration post-install
│   └── setup_integration.sh # Intégration GitLab + Argo CD
└── confs/
    ├── gitlab/
    │   ├── namespace.yaml
    │   ├── helm-values.yaml  # Configuration Helm GitLab
    │   └── ingress.yaml      # Ingress pour GitLab
    ├── argocd/
    │   └── gitlab-repo.yaml  # Repo GitLab dans Argo CD
    └── dev/
        └── ...               # Même que Partie 3
```

---

## Vagrantfile pour le bonus

GitLab nécessite plus de ressources :

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"

  config.vm.define "ezanottiS" do |server|
    server.vm.hostname = "ezanottiS"
    server.vm.network "private_network", ip: "192.168.56.110"

    server.vm.provider "virtualbox" do |vb|
      vb.memory = 8192     # 8 GB RAM minimum recommandé
      vb.cpus = 4          # 4 CPU recommandés
      vb.name = "ezanottiS-bonus"
    end

    # Provisioning
    server.vm.provision "shell", path: "scripts/install_tools.sh"
    server.vm.provision "shell", path: "scripts/create_cluster.sh"
    server.vm.provision "shell", path: "scripts/install_gitlab.sh"
  end
end
```

---

## Installation avec Helm

### 1. Ajouter le repo Helm GitLab

```bash
helm repo add gitlab https://charts.gitlab.io
helm repo update
```

### 2. Créer le namespace

`confs/gitlab/namespace.yaml` :

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: gitlab
```

### 3. Helm Values pour GitLab

`confs/gitlab/helm-values.yaml` (version simplifiée) :

```yaml
# Configuration minimale pour GitLab sur K3d

global:
  hosts:
    domain: gitlab.local
    # K3d ingress va router vers ces hosts
  ingress:
    configureCertmanager: false  # Pas de cert-manager pour simplifier

gitlab:
  webservice:
    minReplicas: 1
    maxReplicas: 1
  sidekiq:
    minReplicas: 1
    maxReplicas: 1

# External URL pour GitLab
externalUrl: http://gitlab.local

# Désactiver certains composants pour économiser des ressources
redis:
  install: true
postgresql:
  install: true
registry:
  enabled: true

# Ingress configuration
nginx-ingress:
  enabled: false  # Utiliser l'ingress Traefik de K3d

# Certificates
certmanager:
  install: false
```

### 4. Script d'installation GitLab

`scripts/install_gitlab.sh` :

```bash
#!/bin/bash

set -e

echo "=== Installing NGINX Ingress Controller for GitLab ==="
# GitLab a besoin de son propre ingress controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml

echo "Waiting for NGINX ingress..."
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s

echo "=== Installing GitLab via Helm ==="
helm repo add gitlab https://charts.gitlab.io
helm repo update

# Créer le namespace gitlab
kubectl create namespace gitlab

# Installer GitLab
helm upgrade --install gitlab gitlab/gitlab \
  --namespace gitlab \
  -f /vagrant/confs/gitlab/helm-values.yaml \
  --timeout 15m

echo "Waiting for GitLab to be ready..."
kubectl wait --for=condition=ready pod -l app=webservice -n gitlab --timeout=600s

echo "=== GitLab installed! ==="
echo "Getting initial password..."

# Le password initial est généré automatiquement
kubectl get secret -n gitlab gitlab-initial-root-password -ojsonpath='{.data.password}' | base64 -d

echo ""
echo "Access GitLab at: http://gitlab.local"
echo "Don't forget to add gitlab.local to your /etc/hosts!"
```

### 5. Configuration Ingress GitLab

`confs/gitlab/ingress.yaml` :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gitlab-ingress
  namespace: gitlab
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  rules:
  - host: gitlab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: gitlab-webservice-default
            port:
              number: 8080
```

---

## Intégration GitLab + Argo CD

### 1. Créer le projet GitLab

```bash
# Se connecter à GitLab UI
# URL: http://gitlab.local
# User: root
# Password: (récupéré avec la commande ci-dessus)

# Créer un nouveau projet:
# - Name: iot-k3d-gitlab
# - Visibility: Public (important pour Argo CD)
# - Clone method: HTTPS
```

### 2. Pusher les manifests sur GitLab

```bash
# Cloner le projet
git clone http://gitlab.local/root/iot-k3d-gitlab.git
cd iot-k3d-gitlab

# Copier les manifests
cp -r /vagrant/confs/dev/* .

# Commit et push
git add .
git commit -m "Initial commit: v1 application"
git push origin main

# Note: GitLab va demander des identifiants
# User: root
# Password: (votre password GitLab)
```

### 3. Configuration Argo CD pour GitLab

`confs/argocd/gitlab-repo.yaml` :

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-repo-secret
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repo-credential
stringData:
  url: http://gitlab.local/root/iot-k3d-gitlab.git
  username: root
  password: <gitlab-password>
```

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: wil-playground-gitlab
  namespace: argocd
spec:
  project: default
  source:
    repoURL: http://gitlab.local/root/iot-k3d-gitlab.git
    targetRevision: HEAD
    path: .  # Les manifests sont à la racine
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

### 4. Script d'intégration

`scripts/setup_integration.sh` :

```bash
#!/bin/bash

echo "=== Setting up GitLab + Argo CD integration ==="

# Créer le secret pour le repo GitLab
kubectl apply -f /vagrant/confs/argocd/gitlab-repo-secret.yaml

# Créer l'application Argo CD
kubectl apply -f /vagrant/confs/argocd/gitlab-app.yaml

echo "Waiting for sync..."
sleep 30

echo "=== Integration complete! ==="
kubectl get pods -n dev
argocd app list
```

---

## Workflow GitOps avec GitLab

### Mise à jour de l'application (v1 → v2)

```bash
# 1. Modifier le fichier
cd iot-k3d-gitlab
sed -i 's/wil42\/playground:v1/wil42\/playground:v2/g' deployment.yaml

# 2. Commit et push sur GitLab
git add deployment.yaml
git commit -m "Upgrade to v2"
git push origin main

# 3. Argo CD détecte automatiquement le changement
# (via polling ou webhook GitLab)

# 4. Vérifier
argocd app get wil-playground-gitlab
curl http://localhost:8888/
```

### Webhook GitLab (optionnel)

Pour une synchronisation en temps réel :

```bash
# Dans GitLab UI:
# Settings → Webhooks
# URL: http://argocd-server.argocd.svc/api/webhook
# Secret token: (généré dans Argo CD)

# Triggers: Push events
```

---

## GitLab Container Registry

### Pusher votre propre image

```bash
# 1. Créer un accès token dans GitLab
# User Settings → Access Tokens
# Scopes: write_registry, read_registry

# 2. Se connecter au registry
docker login gitlab.local:5050
# Username: root
# Password: <access-token>

# 3. Tagger et pusher
docker tag myapp:v1 gitlab.local:5050/root/iot-k3d-gitlab/myapp:v1
docker push gitlab.local:5050/root/iot-k3d-gitlab/myapp:v1

# 4. Utiliser dans le deployment
image: gitlab.local:5050/root/iot-k3d-gitlab/myapp:v1
```

---

## Vérifications

### GitLab est installé

```bash
# Namespace créé
kubectl get ns | grep gitlab

# Pods en cours d'exécution
kubectl get pods -n gitlab

# Services
kubectl get svc -n gitlab

# Ingress
kubectl get ingress -n gitlab
```

### GitLab est accessible

```bash
# Ajouter à /etc/hosts sur votre machine hôte:
echo "192.168.56.110 gitlab.local" | sudo tee -a /etc/hosts

# Accéder via navigateur:
# http://gitlab.local
# User: root
# Password: <récupéré du secret>
```

### Argo CD utilise GitLab

```bash
# Repository configuré
argocd repo list

# Application créée
argocd app list

# Application synchronisée
argocd app get wil-playground-gitlab
```

### Application fonctionne

```bash
# Pod en cours d'exécution
kubectl get pods -n dev

# Application accessible
curl http://localhost:8888/
```

---

## Points de validation

| Vérification | Commande | Résultat attendu |
|--------------|----------|------------------|
| Namespace gitlab créé | `kubectl get ns` | gitlab présent |
| GitLab pods running | `kubectl get pods -n gitlab` | Tous Running |
| GitLab accessible | Navigateur `http://gitlab.local` | UI fonctionnelle |
| GitLab registry | `docker login gitlab.local:5050` | Connexion réussie |
| Projet créé | GitLab UI | Projet iot-k3d-gitlab |
| Argo CD connecté | `argocd repo list` | GitLab repo listé |
| App déployée | `kubectl get pods -n dev` | Pod Running |
| Sync automatique | `git push` (v1→v2) | Argo CD sync |
| App mise à jour | `curl localhost:8888` | v2 déployée |

---

## Problèmes courants

### GitLab ne démarre pas (manque de ressources)

```bash
# Augmenter les ressources de la VM
# Vagrantfile: vb.memory = 12288, vb.cpus = 6

# Vérifier les pods
kubectl describe pod -n gitlab <pod-name>
```

### Argo CD ne peut pas se connecter à GitLab

```bash
# Vérifier le secret
kubectl get secret gitlab-repo-secret -n argocd -o yaml

# Tester manuellement
curl -u root:<password> http://gitlab.local/root/iot-k3d-gitlab.git
```

### GitLab Ingress ne fonctionne pas

```bash
# Vérifier l'ingress Traefik
kubectl get ingress -n gitlab

# Vérifier les hosts
kubectl get ingress -n gitlab -o yaml

# Logs Traefik
kubectl logs -n kube-system -l app=traefik
```

### Password GitLab perdu

```bash
# Récupérer le password
kubectl get secret -n gitlab gitlab-initial-root-password -ojsonpath='{.data.password}' | base64 -d

# Réinitialiser si nécessaire
kubectl exec -it -n gitlab <gitlab-rails-pod> -- gitlab-rails runner "User.find_by_username('root').update(password='newpassword')"
```

---

## Alternatives

### GitLab Runner (CI/CD)

Vous pouvez ajouter un GitLab Runner pour exécuter des pipelines CI/CD :

```bash
# Installer le runner
helm repo add gitlab https://charts.gitlab.io
helm install gitlab-runner -n gitlab-runner --create-namespace gitlab/gitlab-runner

# Enregistrer le runner
# GitLab UI → Settings → CI/CD → Runners
```

### Pipeline GitLab pour déploiement automatique

`.gitlab-ci.yml` :

```yaml
stages:
  - build
  - deploy

build:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker build -t gitlab.local:5050/root/iot-k3d-gitlab/myapp:$CI_COMMIT_SHA .
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD gitlab.local:5050
    - docker push gitlab.local:5050/root/iot-k3d-gitlab/myapp:$CI_COMMIT_SHA

deploy:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl set image deployment/wil-playground wil-playground=gitlab.local:5050/root/iot-k3d-gitlab/myapp:$CI_COMMIT_SHA -n dev
```

---

## Ressources utiles

- [GitLab Helm Charts](https://docs.gitlab.com/charts/)
- [GitLab Container Registry](https://docs.gitlab.com/ee/user/packages/container_registry/)
- [Argo CD Git Integration](https://argoproj.github.io/argo-cd/user-guide/private-repositories/)
- [GitLab Kubernetes Agent](https://docs.gitlab.com/ee/user/clusters/agent/)
