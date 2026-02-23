# Part 2: K3s and Three Simple Applications

> Objectif : Déployer 3 applications web avec routage Ingress basé sur le HOST

---

## Vue d'ensemble

Cette partie consiste à déployer **3 applications web** sur un cluster K3s et à les rendre accessibles via une seule IP (`192.168.56.110`) en utilisant le **HOST header** pour le routage.

```
                         Client
                            │
                            │ curl -H "Host: app1.com" 192.168.56.110
                            ▼
┌─────────────────────────────────────────────────────────┐
│                    VM: <login>S                          │
│                    IP: 192.168.56.110                    │
│                                                          │
│  ┌─────────────────────────────────────────────────┐   │
│  │              K3s Ingress Controller             │   │
│  │  (Traefik - inclus par défaut avec K3s)         │   │
│  └───────────────┬─────────────────────────────────┘   │
│                  │                                      │
│      ┌───────────┼───────────┐                         │
│      ▼           ▼           ▼                         │
│   ┌─────┐   ┌─────────┐   ┌─────┐                     │
│   │app1 │   │ app2    │   │ app3│                     │
│   │:8080│   │ :8080   │   │:8080│                     │
│   └─────┘   └─────────┘   └─────┘                     │
│               3 replicas                               │
└─────────────────────────────────────────────────────────┘
```

---

## Spécifications détaillées

### Machine unique

| Propriété | Valeur requise |
|-----------|----------------|
| Nom de la machine | `<login>S` (ex: `eliasS`) |
| IP | `192.168.56.110` |
| Rôle K3s | **Server** (single-node cluster) |
| Applications | **3 web apps** |

### Routage par HOST

| HOST Header | Application | Replicas |
|-------------|-------------|----------|
| `app1.com` | app1 | 1 |
| `app2.com` | app2 | **3** |
| (défaut/aucun) | app3 | 1 |

---

## Architecture K3s Ingress

### K3s Ingress (Traefik)

K3s est livré avec **Traefik** comme Ingress Controller par défaut. Il écoute sur le port 80 et 443 et route le trafic vers les services appropriés.

```
Internet
    │
    ▼
Port 80/443
    │
    ▼
┌─────────────────────────────────────┐
│         Traefik Ingress             │
│    (Ingress Controller)             │
└───────────────┬─────────────────────┘
                │
      ┌─────────┼─────────┐
      ▼         ▼         ▼
   Service   Service   Service
   app1      app2      app3
      │         │         │
      ▼         ▼         ▼
    Pods      Pods      Pods
   (1 pod)  (3 pods)  (1 pod)
```

---

## Concepts Kubernetes

### Les objets Kubernetes nécessaires

| Objet | Rôle |
|-------|------|
| **Deployment** | Gère les replicas et les mises à jour des pods |
| **Service** | Expose les pods au sein du cluster |
| **Ingress** | Règles de routage HTTP/HTTPS |

### Relation entre les objets

```
Ingress (règles de routage)
    │
    ▼
Service (point d'accès stable)
    │
    ▼
Pods (conteneurs applicatifs)
```

---

## Structure du dossier p2/

```
p2/
├── Vagrantfile              # VM unique avec K3s server
├── scripts/
│   ├── install_k3s.sh       # Installation K3s server
│   └── setup_apps.sh        # Déploiement des applications
└── confs/
    ├── app1/
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   └── ingress.yaml
    ├── app2/
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   └── ingress.yaml
    └── app3/
        ├── deployment.yaml
        ├── service.yaml
        └── ingress.yaml
```

---

## Étapes d'implémentation

### 1. Vagrantfile pour une seule VM

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"

  config.vm.define "eliasS" do |server|
    server.vm.hostname = "eliasS"
    server.vm.network "private_network", ip: "192.168.56.110"

    server.vm.provider "virtualbox" do |vb|
      vb.memory = 2048  # Plus de RAM pour les apps
      vb.cpus = 2
      vb.name = "eliasS"
    end

    # Provisioning
    server.vm.provision "shell", path: "scripts/install_k3s.sh"
    server.vm.provision "shell", path: "scripts/setup_apps.sh"
  end
end
```

### 2. Script d'installation K3s

`scripts/install_k3s.sh` :

```bash
#!/bin/bash

# Installation de K3s en mode server
curl -sfL https://get.k3s.io | sh -s - server

# Attendre que K3s soit prêt
sleep 10

# Configurer kubectl
mkdir -p /home/vagrant/.kube
sudo cp /etc/rancher/k3s/k3s.yaml /home/vagrant/.kube/config
sudo chown vagrant:vagrant /home/vagrant/.kube/config

# Installer kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

echo "K3s installed and ready!"
```

### 3. Déployment pour App1

`confs/app1/deployment.yaml` :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1
  labels:
    app: app1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
    spec:
      containers:
      - name: app1
        image: nginx:alpine
        ports:
        - containerPort: 80
```

`confs/app1/service.yaml` :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app1-service
spec:
  selector:
    app: app1
  ports:
  - port: 8080
    targetPort: 80
```

`confs/app1/ingress.yaml` :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app1-ingress
spec:
  rules:
  - host: app1.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 8080
```

### 4. Déployment pour App2 (3 replicas)

`confs/app2/deployment.yaml` :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app2
  labels:
    app: app2
spec:
  replicas: 3  # 3 replicas obligatoires
  selector:
    matchLabels:
      app: app2
  template:
    metadata:
      labels:
        app: app2
    spec:
      containers:
      - name: app2
        image: httpd:alpine
        ports:
        - containerPort: 80
```

`confs/app2/service.yaml` :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app2-service
spec:
  selector:
    app: app2
  ports:
  - port: 8080
    targetPort: 80
```

`confs/app2/ingress.yaml` :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app2-ingress
spec:
  rules:
  - host: app2.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 8080
```

### 5. Déployment pour App3 (défaut)

`confs/app3/deployment.yaml` :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app3
  labels:
    app: app3
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app3
  template:
    metadata:
      labels:
        app: app3
    spec:
      containers:
      - name: app3
        image: redis:alpine
        ports:
        - containerPort: 6379
```

`confs/app3/service.yaml` :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app3-service
spec:
  selector:
    app: app3
  ports:
  - port: 8080
    targetPort: 6379
```

`confs/app3/ingress.yaml` :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app3-ingress
spec:
  rules:
  - http:  # Pas de host = règle par défaut
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app3-service
            port:
              number: 8080
```

### 6. Script de déploiement

`scripts/setup_apps.sh` :

```bash
#!/bin/bash

# Copier les fichiers de configuration
cp -r /vagrant/confs /home/vagrant/

# Déployer app1
kubectl apply -f /home/vagrant/confs/app1/

# Déployer app2
kubectl apply -f /home/vagrant/confs/app2/

# Déployer app3
kubectl apply -f /home/vagrant/confs/app3/

# Attendre que les pods soient prêts
echo "Waiting for pods to be ready..."
sleep 30

# Afficher l'état
kubectl get pods
kubectl get svc
kubectl get ingress
```

---

## Commandes de test

### Depuis la machine hôte (votre PC)

Ajoutez les entrées dans votre `/etc/hosts` :

```bash
# Linux/Mac
sudo nano /etc/hosts

# Ajouter:
192.168.56.110 app1.com
192.168.56.110 app2.com
```

### Tester le routage

```bash
# Test app1
curl -H "Host: app1.com" http://192.168.56.110

# Test app2
curl -H "Host: app2.com" http://192.168.56.110

# Test app3 (défaut)
curl http://192.168.56.110
```

### Depuis un navigateur

```
http://app1.com         → nginx (Welcome to nginx!)
http://app2.com         → httpd (It works!)
http://192.168.56.110   → redis (réponse par défaut)
```

---

## Vérification Kubernetes

### Vérifier tous les objets

```bash
# Tous les deployments
kubectl get deployments

# Tous les pods
kubectl get pods

# Tous les services
kubectl get svc

# Tous les ingress
kubectl get ingress

# Détails d'un pod
kubectl describe pod <pod-name>

# Logs d'un pod
kubectl logs <pod-name>
```

### Résultat attendu

```bash
$ kubectl get deployments
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
app1   1/1     1            1           5m
app2   3/3     3            3           5m
app3   1/1     1            1           5m

$ kubectl get pods
NAME                    READY   STATUS    RESTARTS   AGE
app1-xxx-xxx            1/1     Running   0          5m
app2-xxx-xxx            1/1     Running   0          5m
app2-xxx-yyy            1/1     Running   0          5m
app2-xxx-zzz            1/1     Running   0          5m
app3-xxx-xxx            1/1     Running   0          5m
```

---

## Points de validation

| Vérification | Commande | Résultat attendu |
|--------------|----------|------------------|
| 3 applications déployées | `kubectl get deployments` | app1, app2, app3 |
| App2 a 3 replicas | `kubectl get pods -l app=app2` | 3 pods Running |
| Ingress configuré | `kubectl get ingress` | 3 ingress |
| App1 accessible | `curl -H "Host: app1.com" 192.168.56.110` | Réponse app1 |
| App2 accessible | `curl -H "Host: app2.com" 192.168.56.110` | Réponse app2 |
| App3 accessible (défaut) | `curl 192.168.56.110` | Réponse app3 |
| Load balancing app2 | `curl -H "Host: app2.com" 192.168.56.110` x3 | Réponses variées |

---

## Concepts clés

### Deployment vs ReplicaSet vs Pod

```
Deployment (gère les mises à jour)
    │
    ▼
ReplicaSet (garantit le nombre de replicas)
    │
    ▼
Pod (unité de déploiement)
    │
    ▼
Container (conteneur applicatif)
```

### Service types

| Type | Description |
|------|-------------|
| ClusterIP | Accès interne au cluster (défaut) |
| NodePort | Accès via port du node |
| LoadBalancer | Accès via load balancer externe |

### Ingress vs LoadBalancer

```
LoadBalancer:                     Ingress:
1 LB par service                 1 LB pour tous les services
Coûteux ($$$)                    Économique
Plus flexible                    Plus simple
```

---

## Applications web possibles

Vous pouvez utiliser n'importe quelle image Docker :

| Application | Image Docker | Port |
|-------------|--------------|------|
| Nginx | `nginx:alpine` | 80 |
| Apache | `httpd:alpine` | 80 |
| Redis | `redis:alpine` | 6379 |
| Python HTTP | `python:alpine` + `python -m http.server` | 8000 |
| Custom app | Votre propre image | Votre port |

---

## Problèmes courants

### Ingress ne fonctionne pas

```bash
# Vérifier Traefik
kubectl get pods -n kube-system

# Logs de Traefik
kubectl logs -n kube-system -l app=traefik
```

### Pods en Pending

```bash
# Décrire le pod pour voir le problème
kubectl describe pod <pod-name>

# Souvent un problème de ressources ou d'image
```

### HOST header non reconnu

```bash
# Vérifier l'Ingress
kubectl get ingress
kubectl describe ingress <ingress-name>

# Vérifier les règles dans Traefik dashboard
```

---

## Ressources utiles

- [K3s Ingress Documentation](https://docs.k3s.io/ingress)
- [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Traefik Documentation](https://doc.traefik.io/traefik/)
