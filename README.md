# 🗳️ Application de Vote Distribuée avec Docker

Application microservices de vote en temps réel, déployable sur Docker Compose ou Docker Swarm.

## 📋 Table des matières

- [Architecture](#architecture)
- [Technologies utilisées](#technologies-utilisées)
- [Prérequis](#prérequis)
- [Installation et Déploiement](#installation-et-déploiement)
  - [Option 1 : Docker Compose (Développement)](#option-1--docker-compose-développement)
  - [Option 2 : Docker Swarm (Production)](#option-2--docker-swarm-production)
- [Accès aux interfaces](#accès-aux-interfaces)
- [Structure du projet](#structure-du-projet)
- [Configuration](#configuration)
- [Commandes utiles](#commandes-utiles)
- [Troubleshooting](#troubleshooting)

---

## 🏗️ Architecture

L'application est composée de 5 microservices :

```
┌─────────────┐        ┌──────────────┐
│    Vote     │◄──────►│    Redis     │
│  (Python)   │        │   (Alpine)   │
│   Port 80   │        │              │
└─────────────┘        └──────────────┘
      │                       │
      │                       ▼
      │                ┌──────────────┐
      │                │    Worker    │
      │                │   (.NET 7)   │
      │                └──────────────┘
      │                       │
      ▼                       ▼
┌─────────────┐        ┌──────────────┐
│   Result    │◄──────►│  PostgreSQL  │
│  (Node.js)  │        │   (Alpine)   │
│   Port 80   │        │              │
└─────────────┘        └──────────────┘
```

### Microservices

| Service | Technologie | Rôle | Port exposé |
|---------|-------------|------|-------------|
| **vote** | Python 3.11 + Flask | Interface de vote entre deux options | 8080 |
| **result** | Node.js 18 + Socket.io | Affichage des résultats en temps réel | 8081 |
| **worker** | .NET 7.0 | Traitement des votes (Redis → PostgreSQL) | - |
| **redis** | Redis 7 Alpine | File de messages temporaire | - |
| **db** | PostgreSQL 15 Alpine | Base de données persistante | - |

### Réseaux Docker

- **frontend** : Réseaux exposé pour `vote` et `result`
- **backend** : Réseau interne pour `worker`, `redis`, et `db`

---

## 💻 Technologies utilisées

- **Docker** & **Docker Compose**
- **Python 3.11** (Flask, Gunicorn, Redis)
- **Node.js 18** (Express, Socket.io, PostgreSQL client)
- **.NET 7.0** (Worker service)
- **Redis 7** (Cache et message broker)
- **PostgreSQL 15** (Base de données)

---

## 📦 Prérequis

### Pour Docker Compose
- Docker Engine 20.10+
- Docker Compose 2.0+

### Pour Docker Swarm (VMs Vagrant)
- VirtualBox 6.1+
- Vagrant 2.2+
- Minimum 4 GB RAM disponible

---

## 🚀 Installation et Déploiement

### Option 1 : Docker Compose (Développement)

**Idéal pour** : Développement local, tests rapides

```bash
# 1. Cloner le projet
git clone <repository-url>
cd voting-app

# 2. Démarrer l'application
docker-compose up --build

# En arrière-plan
docker-compose up -d --build

# 3. Accéder aux interfaces
# Vote:   http://localhost:8080
# Result: http://localhost:8081
```

**Arrêter l'application :**
```bash
# Arrêter les conteneurs
docker-compose down

# Arrêter et supprimer les volumes
docker-compose down -v
```

---

### Option 2 : Docker Swarm (Production)

**Idéal pour** : Déploiement distribué, haute disponibilité

---

#### 🆕 Scénario A : Premier déploiement (VMs non créées)

**Suivez ces étapes si vous lancez le projet pour la première fois**

##### 1️⃣ Créer et démarrer les VMs

```powershell
# Depuis Windows, aller dans le dossier Vagrant
cd ..\Vagrant
vagrant up
```

Cette commande va automatiquement :
- ✅ Créer 3 VMs Ubuntu (manager1, worker1, worker2)
- ✅ Installer Docker sur chaque VM
- ✅ Initialiser le cluster Docker Swarm
- ✅ Connecter les workers au manager
- ⏱️ Durée : ~5-10 minutes

**IPs des VMs :**
- manager1: `192.168.99.100` (Leader Swarm)
- worker1: `192.168.99.101`
- worker2: `192.168.99.102`

##### 2️⃣ Vérifier que le cluster est opérationnel

```powershell
vagrant ssh manager1 -c "docker node ls"
```

Vous devriez voir 3 nœuds : 1 manager (Leader) et 2 workers.

##### 3️⃣ Copier les fichiers et construire les images

```powershell
# Se connecter au manager
vagrant ssh manager1
```

Dans la VM :
```bash
# Copier les fichiers depuis le dossier partagé
mkdir -p ~/voting-app
cp -r /vagrant/../voting-app/* ~/voting-app/
cd ~/voting-app

# Construire les images (peut prendre 5-10 minutes)
docker build -t voting-app-vote:latest -f ./vote/DockerFile ./vote
docker build -t voting-app-result:latest -f ./result/DockerFile ./result
docker build -t voting-app-worker:latest -f ./worker/DockerFile ./worker

# Vérifier que les images sont créées
docker images | grep voting-app
```

##### 4️⃣ Déployer la stack sur le Swarm

```bash
# Toujours dans la VM manager1
docker stack deploy -c docker-stack.yml voting-app
```

##### 5️⃣ Vérifier le déploiement

```bash
# Attendre quelques secondes puis vérifier
docker stack services voting-app
docker stack ps voting-app
```

##### 6️⃣ Accéder aux interfaces

Depuis votre navigateur Windows :
- **Vote** : http://192.168.99.100:8080
- **Result** : http://192.168.99.100:8081

---

#### ♻️ Scénario B : Redémarrage (VMs déjà configurées)

**Suivez ces étapes si les VMs existent déjà et que le cluster est configuré**

##### 1️⃣ Vérifier l'état des VMs

```powershell
cd ..\Vagrant
vagrant status
```

**Si les VMs sont arrêtées :**
```powershell
vagrant up
```

**Si les VMs tournent déjà :** Passez à l'étape suivante.

##### 2️⃣ Vérifier que le cluster Swarm est actif

```powershell
vagrant ssh manager1 -c "docker node ls"
```

##### 3️⃣ Vérifier si la stack est déjà déployée

```powershell
vagrant ssh manager1 -c "docker stack ls"
```

**Si la stack `voting-app` existe déjà :**
```powershell
# Option A : Redémarrer les services
vagrant ssh manager1 -c "docker service ls"

# Option B : Mettre à jour la stack (si vous avez modifié des fichiers)
vagrant ssh manager1 -c "cd ~/voting-app && docker stack deploy -c docker-stack.yml voting-app"
```

**Si la stack n'existe pas :**
```powershell
# Déployer la stack
vagrant ssh manager1 -c "docker stack deploy -c ~/voting-app/docker-stack.yml voting-app"
```

##### 4️⃣ Vérifier l'état des services

```powershell
vagrant ssh manager1 -c "docker stack services voting-app"
vagrant ssh manager1 -c "docker stack ps voting-app"
```

##### 5️⃣ Accéder aux interfaces

- **Vote** : http://192.168.99.100:8080
- **Result** : http://192.168.99.100:8081

---

#### 🔄 Commandes rapides

**Démarrer tout (VMs existantes) :**
```powershell
cd ..\Vagrant
vagrant up
vagrant ssh manager1 -c "docker stack deploy -c ~/voting-app/docker-stack.yml voting-app"
```

**Voir les logs :**
```powershell
vagrant ssh manager1 -c "docker service logs voting-app_vote -f"
vagrant ssh manager1 -c "docker service logs voting-app_result -f"
vagrant ssh manager1 -c "docker service logs voting-app_worker -f"
```

**Arrêter la stack (garder les VMs allumées) :**
```powershell
vagrant ssh manager1 -c "docker stack rm voting-app"
```

**Arrêter les VMs :**
```powershell
cd ..\Vagrant
vagrant halt
```

**Supprimer complètement les VMs :**
```powershell
cd ..\Vagrant
vagrant destroy -f
```

---

## 🌐 Accès aux interfaces

### Interface de Vote
- **URL** : http://localhost:8080 (Compose) ou http://192.168.99.100:8080 (Swarm)
- **Fonction** : Permet de voter entre deux options (Chats vs Chiens par défaut)

### Interface des Résultats
- **URL** : http://localhost:8081 (Compose) ou http://192.168.99.100:8081 (Swarm)
- **Fonction** : Affiche les résultats en temps réel avec mise à jour automatique

---

## 📁 Structure du projet

```
voting-app/
├── vote/                      # Service de vote (Python/Flask)
│   ├── DockerFile            # Image Docker du service vote
│   ├── app.py                # Application Flask
│   ├── requirements.txt      # Dépendances Python
│   ├── templates/            # Templates HTML
│   └── static/               # Fichiers CSS/JS
│
├── result/                    # Service de résultats (Node.js)
│   ├── DockerFile            # Image Docker du service result
│   ├── server.js             # Serveur Express + Socket.io
│   ├── package.json          # Dépendances Node.js
│   └── views/                # Fichiers HTML/CSS/JS
│
├── worker/                    # Service worker (.NET)
│   ├── DockerFile            # Image Docker du worker
│   ├── Program.cs            # Programme principal .NET
│   └── Worker.csproj         # Projet .NET
│
├── docker-compose.yml         # Configuration Docker Compose
├── .dockerignore             # Fichiers à exclure du build
├── .gitignore                # Fichiers à exclure de Git
└── README.md                 # Cette documentation
```

---

## ⚙️ Configuration

### Variables d'environnement

#### Service Vote
- `REDIS_HOST` : Hôte Redis (défaut: `redis`)

#### Service Result
- `DB_HOST` : Hôte PostgreSQL (défaut: `db`)
- `PORT` : Port d'écoute (défaut: `80`)

#### Service Worker
- `REDIS_HOST` : Hôte Redis (défaut: `redis`)
- `DB_HOST` : Hôte PostgreSQL (défaut: `db`)

#### PostgreSQL
- `POSTGRES_USER` : Utilisateur PostgreSQL (défaut: `postgres`)
- `POSTGRES_PASSWORD` : Mot de passe (défaut: `postgres`)
- `POSTGRES_DB` : Nom de la base (défaut: `postgres`)

---

## 🛠️ Commandes utiles

### Docker Compose

```bash
# Construire les images
docker-compose build

# Démarrer les services
docker-compose up -d

# Voir les logs
docker-compose logs -f

# Voir les logs d'un service spécifique
docker-compose logs -f vote

# Voir l'état des services
docker-compose ps

# Redémarrer un service
docker-compose restart vote

# Arrêter les services
docker-compose down

# Supprimer les volumes
docker-compose down -v
```

### Docker Swarm

```bash
# Depuis le manager (vagrant ssh manager1)

# Déployer/mettre à jour la stack
docker stack deploy -c docker-stack.yml voting-app

# Lister les stacks
docker stack ls

# Lister les services de la stack
docker stack services voting-app

# Voir les conteneurs de la stack
docker stack ps voting-app

# Voir les logs d'un service
docker service logs voting-app_vote

# Scaler un service
docker service scale voting-app_vote=3

# Supprimer la stack
docker stack rm voting-app

# Voir les nœuds du cluster
docker node ls
```

### Vagrant (Gestion des VMs)

```bash
# Démarrer toutes les VMs
vagrant up

# Démarrer une VM spécifique
vagrant up manager1

# Voir l'état des VMs
vagrant status

# Se connecter à une VM
vagrant ssh manager1

# Arrêter les VMs
vagrant halt

# Redémarrer les VMs
vagrant reload

# Supprimer les VMs
vagrant destroy

# Supprimer et recréer
vagrant destroy -f && vagrant up
```

---

## 🔧 Troubleshooting

### Les services ne démarrent pas

**Docker Compose :**
```bash
# Vérifier les logs
docker-compose logs

# Reconstruire les images
docker-compose build --no-cache

# Supprimer tout et recommencer
docker-compose down -v
docker-compose up --build
```

**Docker Swarm :**
```bash
# Vérifier l'état des services
docker service ls

# Voir les logs d'un service
docker service logs voting-app_vote

# Voir les conteneurs en erreur
docker stack ps voting-app --no-trunc

# Supprimer et redéployer
docker stack rm voting-app
sleep 10
docker stack deploy -c docker-stack.yml voting-app
```

### Les VMs Vagrant ne démarrent pas

```bash
# Vérifier VirtualBox
VBoxManage list vms

# Supprimer et recréer
vagrant destroy -f
vagrant up

# Vérifier les logs
vagrant up --debug
```

### Impossible de se connecter aux interfaces web

**Vérifier les ports :**
```bash
# Docker Compose
docker-compose ps

# Vérifier si les ports sont bien mappés
netstat -ano | findstr "8080"
netstat -ano | findstr "8081"
```

**Docker Swarm - Vérifier les IPs :**
```bash
vagrant ssh manager1
ip addr show
```

### Les votes ne sont pas enregistrés

1. Vérifier que Redis fonctionne :
   ```bash
   docker exec -it <redis-container> redis-cli ping
   ```

2. Vérifier que PostgreSQL fonctionne :
   ```bash
   docker exec -it <postgres-container> psql -U postgres -d postgres -c "\dt"
   ```

3. Vérifier les logs du worker :
   ```bash
   docker-compose logs worker
   # ou
   docker service logs voting-app_worker
   ```

### Nettoyage complet

**Docker Compose :**
```bash
docker-compose down -v --rmi all
docker system prune -a --volumes
```

**Docker Swarm :**
```bash
# Dans la VM manager
docker stack rm voting-app
docker system prune -a --volumes

# Depuis Windows
vagrant destroy -f
```

---

## 📝 Notes

- Les données PostgreSQL sont persistées dans un volume Docker
- Redis fonctionne en mode non-persistant (données en mémoire)
- Le worker traite les votes de manière asynchrone
- Les résultats sont mis à jour en temps réel via WebSocket (Socket.io)

---

## 📄 Licence

Projet éducatif - Libre d'utilisation

---

## 👤 Auteur

Projet réalisé dans le cadre d'une formation Docker
