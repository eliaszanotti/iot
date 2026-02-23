# Part 3: K3d and Argo CD

> Objectif : Mettre en place un pipeline GitOps avec K3d et Argo CD

---

## Vue d'ensemble

Cette partie est la plus avancée : vous allez découvrir **K3d** (K3s dans Docker) et mettre en place un pipeline **GitOps** avec **Argo CD**.

```
┌─────────────────────────────────────────────────────────────────────┐
│                         GitHub Repository                            │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  - deployment.yaml                                            │  │
│  │  - service.yaml                                               │  │
│  │  - ingress.yaml                                               │  │
│  └───────────────────────────────────────────────────────────────┘  │
└───────────────────────────────┬─────────────────────────────────────┘
                                │ git push
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      Argo CD (GitOps Operator)                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  - Watch GitHub repository                                    │  │
│  │  - Detect changes                                             │  │
│  │  - Sync automatically                                         │  │
│  └───────────────────────────────────────────────────────────────┘  │
└───────────────────────────────┬─────────────────────────────────────┘
                                │ kubectl apply
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      K3d Cluster (K3s in Docker)                    │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  Namespace: dev                                                │  │
│  │  ┌─────────────────────────────────────────────────────────┐  │  │
│  │  │  Application Pod (wil42/playground:v1 or v2)             │  │  │
│  │  │  Port: 8888                                              │  │  │
│  │  └─────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Spécifications détaillées

### Infrastructure

| Composant | Description |
|-----------|-------------|
| **K3d** | K3s exécuté dans des conteneurs Docker |
| **Docker** | Requis pour faire tourner K3d |
| **Argo CD** | Outil de GitOps pour le déploiement continu |
| **GitHub** | Repository public stockant les manifests |

### Namespaces obligatoires

| Namespace | Contenu |
|-----------|---------|
| `argocd` | Argo CD lui-même |
| `dev` | Votre application |

### Application

- Doit avoir **2 versions** (tags `v1` et `v2`)
- Options :
  - Utiliser `wil42/playground` (Docker Hub)
  - Créer votre propre application

---

## Concepts fondamentaux

### K3s vs K3d

| Aspect | K3s | K3d |
|--------|-----|-----|
| **Type** | Binaire Linux | Conteneurs Docker |
| **Installation** | Curl + script | Docker + binaire k3d |
| **Usage** | Production | Développement local |
| **Légèreté** | Léger | Ultra léger |
| **Démarrage** | Secondes | Millisecondes |
| **Multi-cluster** | Complexe | Très simple |

### GitOps avec Argo CD

```
Déclaration d'état désiré (Git)
        │
        │ git push
        ▼
    Argo CD détecte le changement
        │
        ▼
    Argo CD compare l'état actuel vs désiré
        │
        ▼
    Argo CD applique les changements
        │
        ▼
    État du cluster synchronisé avec Git
```

**Avantages du GitOps** :
- **Versionnage** : Toute modification est tracée dans Git
- **Audit** : Historique complet des changements
- **Rollback** : Retour facile à une version précédente
- **Automatisation** : Déploiement automatique
- **Collaboration** : PR, reviews, approbations

---

## Structure du dossier p3/

```
p3/
├── scripts/
│   ├── install_tools.sh        # Docker, K3d, kubectl, helm
│   ├── create_cluster.sh       # Création du cluster K3d
│   ├── install_argocd.sh       # Installation Argo CD
│   └── deploy_app.sh           # Déploiement initial application
└── confs/
    ├── argocd/
    │   ├── namespace.yaml
    │   └── install.yaml        # Manifest Argo CD
    ├── dev/
    │   ├── namespace.yaml
    │   ├── deployment.yaml     # Application à déployer
    │   ├── service.yaml
    │   └── ingress.yaml
    └── github/
        └── app.yaml            # Argo CD Application manifest
```

---

## Étapes d'implémentation

### 1. Script d'installation des outils

`scripts/install_tools.sh` :

```bash
#!/bin/bash

set -e

echo "=== Installing Docker ==="
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker vagrant

echo "=== Installing kubectl ==="
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

echo "=== Installing K3d ==="
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash

echo "=== Installing Helm ==="
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

echo "=== Verifying installations ==="
docker --version
kubectl version --client
k3d version
helm version

echo "=== Tools installation complete! ==="
```

### 2. Script de création du cluster K3d

`scripts/create_cluster.sh` :

```bash
#!/bin/bash

echo "=== Creating K3d cluster ==="

# Créer un cluster avec 2 serveurs pour haute disponibilité
k3d cluster create iot-cluster \
    --servers 1 \
    --agents 1 \
    --port "8080:80@loadbalancer" \
    --port "8888:8888@loadbalancer" \
    --wait

# Vérifier le cluster
echo "=== Cluster nodes ==="
k3d node list

echo "=== Kubernetes info ==="
kubectl get nodes

echo "=== Cluster created successfully! ==="
```

### 3. Script d'installation Argo CD

`scripts/install_argocd.sh` :

```bash
#!/bin/bash

echo "=== Installing Argo CD ==="

# Créer le namespace argocd
kubectl create namespace argocd

# Installer Argo CD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Attendre qu'Argo CD soit prêt
echo "Waiting for Argo CD pods to be ready..."
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s

# Afficher l'état
kubectl get pods -n argocd

# Récupérer le password initial
echo "=== Argo CD initial password ==="
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

echo ""
echo "=== Argo CD installed! ==="
echo "Access Argo CD UI: kubectl port-forward svc/argocd-server -n argocd 8080:443"
```

### 4. Namespace dev

`confs/dev/namespace.yaml` :

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

### 5. Deployment de l'application

`confs/dev/deployment.yaml` :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wil-playground
  namespace: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wil-playground
  template:
    metadata:
      labels:
        app: wil-playground
    spec:
      containers:
      - name: wil-playground
        image: wil42/playground:v1  # ou v2
        ports:
        - containerPort: 8888
```

### 6. Service de l'application

`confs/dev/service.yaml` :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: wil-playground-service
  namespace: dev
spec:
  selector:
    app: wil-playground
  ports:
  - port: 8888
    targetPort: 8888
```

### 7. Ingress pour l'application

`confs/dev/ingress.yaml` :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wil-playground-ingress
  namespace: dev
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: wil-playground-service
            port:
              number: 8888
```

### 8. Application Argo CD

`confs/github/app.yaml` :

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: wil-playground
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<votre-login>/iot-k3d-argocd.git
    targetRevision: HEAD
    path: confs/dev
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

### 9. Script de déploiement initial

`scripts/deploy_app.sh` :

```bash
#!/bin/bash

echo "=== Creating namespaces ==="
kubectl apply -f /vagrant/confs/dev/namespace.yaml
kubectl apply -f /vagrant/confs/argocd/namespace.yaml

echo "=== Deploying application ==="
kubectl apply -f /vagrant/confs/dev/

echo "=== Creating Argo CD Application ==="
kubectl apply -f /vagrant/confs/github/app.yaml

echo "=== Waiting for application pods ==="
kubectl wait --for=condition=ready pod -l app=wil-playground -n dev --timeout=300s

echo "=== Application deployed! ==="
kubectl get pods -n dev
kubectl get svc -n dev
```

---

## GitHub Repository

### Structure du repository

```
iot-k3d-argocd/
├── confs/
│   └── dev/
│       ├── namespace.yaml
│       ├── deployment.yaml
│       ├── service.yaml
│       └── ingress.yaml
├── .gitignore
└── README.md
```

### Commandes Git

```bash
# Cloner le repository
git clone https://github.com/<votre-login>/iot-k3d-argocd.git
cd iot-k3d-argocd

# Copier les fichiers
cp -r /vagrant/confs/dev/* confs/dev/

# Commiter et pousser
git add .
git commit -m "Initial commit: v1 application"
git push origin main
```

### Workflow de mise à jour

```bash
# Pour passer de v1 à v2

# 1. Modifier le deployment
sed -i 's/wil42\/playground:v1/wil42\/playground:v2/g' confs/dev/deployment.yaml

# 2. Vérifier le changement
cat confs/dev/deployment.yaml | grep image

# 3. Commiter et pousser
git add confs/dev/deployment.yaml
git commit -m "Upgrade to v2"
git push origin main

# 4. Argo CD détecte et applique automatiquement
```

---

## Commandes utiles

### Cluster K3d

```bash
# Créer un cluster
k3d cluster create iot-cluster

# Lister les clusters
k3d cluster list

# Supprimer un cluster
k3d cluster delete iot-cluster

# Accéder au cluster
kubectl get nodes
```

### Argo CD CLI (optionnel)

```bash
# Installer Argo CD CLI
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd

# Se connecter à Argo CD
argocd login localhost:8080

# Lister les applications
argocd app list

# Synchroniser une application
argocd app sync wil-playground

# Watcher une application
argocd app get wil-playground --watch
```

### Argo CD UI

```bash
# Port forward pour accéder à l'UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Ouvrir le navigateur
# URL: https://localhost:8080
# User: admin
# Password: (récupérer avec la commande ci-dessous)
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### Vérification du déploiement

```bash
# Vérifier les namespaces
kubectl get ns

# Vérifier les pods
kubectl get pods -n dev
kubectl get pods -n argocd

# Vérifier la synchro Argo CD
argocd app list
# ou dans l'UI

# Tester l'application
curl http://localhost:8888/
# Réponse attendue pour v1: {"status":"ok", "message":"v1"}
# Réponse attendue pour v2: {"status":"ok", "message":"v2"}
```

---

## Points de validation

| Vérification | Commande | Résultat attendu |
|--------------|----------|------------------|
| Cluster K3d créé | `k3d cluster list` | iot-cluster running |
| Namespaces créés | `kubectl get ns` | argocd, dev |
| Argo CD installé | `kubectl get pods -n argocd` | argocd-* pods Running |
| Application déployée | `kubectl get pods -n dev` | wil-playground-* Running |
| Argo CD app créée | `argocd app list` | wil-playground |
| Application accessible | `curl localhost:8888` | {"status":"ok", "message":"v1"} |
| Sync automatique | `git push` (v1→v2) | Argo CD sync automatique |
| Application mise à jour | `curl localhost:8888` | {"status":"ok", "message":"v2"} |

---

## Démo complète (workflow)

```bash
# 1. État initial - v1
$ curl http://localhost:8888/
{"status":"ok", "message":"v1"}

$ cat confs/dev/deployment.yaml | grep image
    image: wil42/playground:v1

# 2. Modifier pour v2
$ sed -i 's/wil42\/playground:v1/wil42\/playground:v2/g' confs/dev/deployment.yaml

$ git add confs/dev/deployment.yaml
$ git commit -m "Upgrade to v2"
$ git push origin main

# 3. Vérifier dans Argo CD (UI ou CLI)
$ argocd app get wil-playground
Name:         wil-playground
Status:       Synced  # ← Argo CD a synchronisé automatiquement

# 4. Vérifier que l'application a changé
$ curl http://localhost:8888/
{"status":"ok", "message":"v2"}  # ← Version mise à jour !
```

---

## Concepts avancés Argo CD

### Sync Policy

```yaml
syncPolicy:
  automated:
    prune: true        # Supprime les ressources orphelines
    selfHeal: true     # Répare les modifications manuelles
  syncOptions:
  - CreateNamespace=true
```

### Health Checks

Argo CD vérifie la santé des ressources :
- **Pods** : Running et ready
- **Services** : Ont des endpoints
- **Deployments** : Tous les replicas disponibles

### Rollback avec GitOps

```bash
# Pour revenir à v1
git checkout HEAD~1  # Dernier commit
git push origin main --force

# Argo CD va détecter et appliquer v1 automatiquement
```

---

## Alternatives : Votre propre application

### Créer une application avec 2 versions

**Application v1** :
```python
# app.py
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello():
    return '{"status":"ok", "message":"v1"}'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8888)
```

**Application v2** (différente) :
```python
# app.py
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello():
    return '{"status":"ok", "message":"v2", "features":["new_feature"]}'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8888)
```

### Dockerfile

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY app.py .
RUN pip install flask
CMD ["python", "app.py"]
```

### Build et push

```bash
# Build v1
docker build -t <votre-login>/myapp:v1 .
docker push <votre-login>/myapp:v1

# Build v2 (avec code modifié)
docker build -t <votre-login>/myapp:v2 .
docker push <votre-login>/myapp:v2
```

---

## Problèmes courants

### Argo CD ne se synchronise pas

```bash
# Vérifier les logs Argo CD
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-repo-server

# Vérifier la connexion au repo Git
argocd repo get https://github.com/<votre-login>/iot-k3d-argocd.git
```

### L'application reste en version v1

```bash
# Forcer la synchronisation
argocd app sync wil-playground

# Vérifier le déploiement
kubectl describe deployment wil-playground -n dev
```

### Accès à l'application impossible

```bash
# Vérifier le service
kubectl get svc -n dev

# Vérifier l'ingress
kubectl get ingress -n dev

# Vérifier le port mapping K3d
k3d cluster list
```

---

## Ressources utiles

- [K3d Documentation](https://k3d.io/)
- [Argo CD Documentation](https://argoproj.github.io/argo-cd/)
- [GitOps Principles](https://www.open-gitops.org/)
- [Wil's Playground Docker Hub](https://hub.docker.com/r/wil42/playground)
