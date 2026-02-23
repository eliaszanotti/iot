# Part 1: K3s and Vagrant

> Objectif : Mettre en place un cluster K3s avec 2 machines virtuelles

---

## Vue d'ensemble

Cette première partie consiste à créer une infrastructure de base pour découvrir **K3s**, une distribution légère de Kubernetes.

```
┌─────────────────────────────────────────────────────────┐
│                    Host Machine                         │
│  ┌──────────────────┐      ┌──────────────────┐        │
│  │   VM: <login>S   │      │ VM: <login>SW    │        │
│  │  IP: .56.110     │      │  IP: .56.111     │        │
│  │  Role: Controller│◄─────┤  Role: Agent     │        │
│  │      K3s         │      │      K3s         │        │
│  └──────────────────┘      └──────────────────┘        │
└─────────────────────────────────────────────────────────┘
```

---

## Prérequis

### Outils nécessaires
- **VirtualBox** (provider pour Vagrant)
- **Vagrant** (outil de gestion de VMs)
- Une distribution Linux (ex: Ubuntu Server, Debian, Rocky Linux)

### Sur votre machine hôte
```bash
# Vérifier Vagrant
vagrant --version

# Vérifier VirtualBox
VBoxManage --version
```

---

## Spécifications détaillées

### Machine 1 : Controller

| Propriété | Valeur requise |
|-----------|----------------|
| Nom de la machine | `<login>S` (ex: `ezanottiS`) |
| Hostname | `<login>S` |
| IP | `192.168.56.110` |
| CPU | **1** |
| RAM | **512 Mo** ou **1024 Mo** |
| Rôle K3s | **Controller** (server) |
| SSH | Sans mot de passe |

### Machine 2 : Agent

| Propriété | Valeur requise |
|-----------|----------------|
| Nom de la machine | `<login>SW` (ex: `ezanottiSW`) |
| Hostname | `<login>SW` |
| IP | `192.168.56.111` |
| CPU | **1** |
| RAM | **512 Mo** ou **1024 Mo** |
| Rôle K3s | **Agent** (worker) |
| SSH | Sans mot de passe |

---

## Architecture K3s

### K3s Controller (Server)
- Gère le cluster
- Expose l'API Kubernetes
- Gère le scheduler et le controller-manager
- Héberge la base de données (SQLite par défaut)

### K3s Agent (Worker)
- Exécute les conteneurs (Pods)
- Se connecte au controller
- Gère le kubelet et kube-proxy

### Communication
```
Agent (192.168.56.111)
        │
        │ Join avec token
        ▼
Controller (192.168.56.110)
        │
        │ kubectl
        ▼
    Vous (Admin)
```

---

## Structure du dossier p1/

```
p1/
├── Vagrantfile          # Configuration des 2 VMs
├── scripts/
│   ├── install_k3s_controller.sh  # Installation K3s controller
│   ├── install_k3s_agent.sh       # Installation K3s agent
│   └── setup_kubectl.sh           # Installation kubectl + config
└── confs/
    └── kube_config.yaml  # Configuration kubectl (optionnel)
```

---

## Étapes d'implémentation

### 1. Créer le Vagrantfile

Le Vagrantfile doit définir les 2 machines avec leurs configurations :

```ruby
Vagrant.configure("2") do |config|
  # Distribution Linux de votre choix
  config.vm.box = "ubuntu/jammy64"  # Exemple avec Ubuntu 22.04

  # Machine Controller
  config.vm.define "ezanottiS" do |control|
    control.vm.hostname = "ezanottiS"
    control.vm.network "private_network", ip: "192.168.56.110"

    control.vm.provider "virtualbox" do |vb|
      vb.memory = 1024
      vb.cpus = 1
      vb.name = "ezanottiS"
    end

    # Provisioning
    control.vm.provision "shell", path: "scripts/install_k3s_controller.sh"
    control.vm.provision "shell", path: "scripts/setup_kubectl.sh"
  end

  # Machine Agent
  config.vm.define "ezanottiSW" do |worker|
    worker.vm.hostname = "ezanottiSW"
    worker.vm.network "private_network", ip: "192.168.56.111"

    worker.vm.provider "virtualbox" do |vb|
      vb.memory = 1024
      vb.cpus = 1
      vb.name = "ezanottiSW"
    end

    worker.vm.provision "shell", path: "scripts/install_k3s_agent.sh"
  end
end
```

### 2. Script d'installation K3s Controller

`scripts/install_k3s_controller.sh` :

```bash
#!/bin/bash

# Installation de K3s en mode controller
curl -sfL https://get.k3s.io | sh - \
  -s - server \
  --node-name ezanottiS \
  --bind-address 192.168.56.110 \
  --advertise-address 192.168.56.110 \
  --tls-san 192.168.56.110

# Récupérer le token de join
TOKEN=$(sudo cat /var/lib/rancher/k3s/server/node-token)
echo "K3S_TOKEN=$TOKEN" > /vagrant/shared_token

# Attendre que K3s soit prêt
sleep 10

# Vérifier l'installation
sudo k3s kubectl get nodes
```

### 3. Script d'installation K3s Agent

`scripts/install_k3s_agent.sh` :

```bash
#!/bin/bash

# Récupérer le token
source /vagrant/shared_token

# Installation de K3s en mode agent
curl -sfL https://get.k3s.io | sh - \
  -s - agent \
  --node-name ezanottiSW \
  --server https://192.168.56.110:6443 \
  --token $K3S_TOKEN
```

### 4. Script de configuration kubectl

`scripts/setup_kubectl.sh` :

```bash
#!/bin/bash

# Installer kubectl sur le controller
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Créer un alias pour kubectl
echo 'alias k=kubectl' >> /home/vagrant/.bashrc
echo 'source <(kubectl completion bash)' >> /home/vagrant/.bashrc

# Rendre kubectl accessible pour l'utilisateur vagrant
mkdir -p /home/vagrant/.kube
sudo cp /etc/rancher/k3s/k3s.yaml /home/vagrant/.kube/config
sudo chown vagrant:vagrant /home/vagrant/.kube/config
```

---

## Commandes utiles

### Lancement des VMs
```bash
cd p1/
vagrant up
```

### Accès SSH aux VMs
```bash
# Controller
vagrant ssh ezanottiS

# Agent
vagrant ssh ezanottiSW
```

### Vérification du cluster
```bash
# Depuis le controller
sudo k3s kubectl get nodes
# ou
kubectl get nodes  # si configuré
```

**Résultat attendu** :
```
NAME     STATUS   ROLES                       AGE   VERSION
ezanottiS   Ready    control-plane,etcd,master   5m    v1.XX.X
ezanottiSW  Ready    <none>                      2m    v1.XX.X
```

### Vérification des pods
```bash
kubectl get pods -A
```

### Arrêter les VMs
```bash
vagrant halt
```

### Détruire les VMs
```bash
vagrant destroy -f
```

---

## Points de validation

| Vérification | Commande | Résultat attendu |
|--------------|----------|------------------|
| VM Controller accessible | `vagrant ssh ezanottiS` | Connexion réussie |
| VM Agent accessible | `vagrant ssh ezanottiSW` | Connexion réussie |
| IP Controller | `ip a` | `192.168.56.110` |
| IP Agent | `ip a` | `192.168.56.111` |
| K3s controller installé | `sudo systemctl status k3s` | Active (running) |
| K3s agent installé | `sudo systemctl status k3s-agent` | Active (running) |
| Cluster prêt | `kubectl get nodes` | 2 nodes Ready |
| SSH sans mot de passe | `ssh ezanottiSW` | Pas de password |

---

## Concepts clés à comprendre

### Vagrant
- **Outil d'automatisation** de VMs
- **Vagrantfile** = code infrastructure
- **Provisioning** = scripts exécutés au démarrage

### K3s
- **Kubernetes léger** (< 100 MB binaire)
- **Binaire unique** qui remplace plusieurs composants K8s
- **SQLite** par défaut (pas de etcd externe)

### Kubernetes Architecture
- **Control Plane** : Controller (API, scheduler, manager)
- **Data Plane** : Workers (exécutent les pods)
- **kubectl** : CLI pour interagir avec le cluster

---

## Ressources utiles

- [K3s Documentation](https://docs.k3s.io/)
- [Vagrant Documentation](https://www.vagrantup.com/docs)
- [Kubernetes Basics](https://kubernetes.io/docs/tutorials/kubernetes-basics/)
