# 🗳️ Projet Voting App - Déploiement Conteneurisé (Docker & Swarm)

Ce projet implémente le déploiement de l'application distribuée **Voting App** sur une infrastructure automatisée.
L'environnement repose sur **Vagrant** pour la virtualisation et **Docker** pour l'orchestration (**Compose** & **Swarm**).

---

## 🏗 Architecture de l'Infrastructure

Le cluster se compose de **3 machines virtuelles** (nœuds) provisionnées automatiquement sous **Ubuntu 24.04** :

| Hostname   | IP               | Rôle Swarm | Services hébergés |
|------------|------------------|------------|-------------------|
| **manager1** | `192.168.99.100` | Leader     | PostgreSQL, Redis, gestion du cluster |
| **worker1**  | `192.168.99.101` | Worker     | Vote, Result, Worker (.NET) |
| **worker2**  | `192.168.99.102` | Worker     | Vote, Result, Worker (.NET) |

---

## 📋 Prérequis

*   **VirtualBox** (Hyperviseur)
*   **Vagrant** (Automatisation)
*   **Docker** (Client local pour build/push les images)

> ℹ️ **Note** : Docker Engine est installé automatiquement dans les VMs par Vagrant.

---

## 🚀 Installation & Démarrage

### 1. Lancement de l'infrastructure

Ouvrez un terminal à la racine du projet et lancez :

```bash
vagrant up
```

Cette commande va :
1.  Créer les 3 VMs.
2.  Installer Docker sur chacune.
3.  Initialiser le cluster Swarm (Manager + Workers).

---

## 🛠️ Mode Développement (Docker Compose)

Ce mode permet de tester l'application rapidement sur un seul nœud (le manager).

1.  Connectez-vous au manager :
    ```bash
    vagrant ssh manager1
    ```

2.  Allez dans le dossier du projet :
    ```bash
    cd /vagrant
    ```

3.  Lancez la stack avec Compose :
    ```bash
    docker compose up --build -d
    ```

4.  **Accès à l'application** :
    *   Vote : [http://192.168.99.100:5000](http://192.168.99.100:5000)
    *   Result : [http://192.168.99.100:5001](http://192.168.99.100:5001)

---

## 🌐 Mode Production (Docker Swarm)

Le cluster Swarm est déjà actif après le `vagrant up`. Cette procédure déploie l'application de manière distribuée et résiliente.

### 1. Préparation des images (Sur votre machine hôte)

Les nœuds du cluster doivent pouvoir télécharger les images. Il faut donc les pousser sur le Docker Hub.

> ⚠️ **Important** : Remplacez `VOTRE_PSEUDO` par votre identifiant Docker Hub.

```bash
# Connexion au registre
docker login

# Build & Push
docker build -t VOTRE_PSEUDO/voting-app-vote ./vote
docker push VOTRE_PSEUDO/voting-app-vote

docker build -t VOTRE_PSEUDO/voting-app-result ./result
docker push VOTRE_PSEUDO/voting-app-result

docker build -t VOTRE_PSEUDO/voting-app-worker ./worker
docker push VOTRE_PSEUDO/voting-app-worker
```

### 2. Configuration

Modifiez le fichier `docker-stack.yml` pour utiliser vos images :
*   Remplacez `<TON_ID_DOCKERHUB>` par votre pseudo.

### 3. Déploiement

Connectez-vous au manager et déployez la stack :

```bash
vagrant ssh manager1
cd /vagrant
docker stack deploy -c docker-stack.yml vote
```

### 4. Vérification

```bash
# Voir l'état des services
docker service ls

# Voir la répartition des conteneurs
docker stack ps vote
```

---

## ⚙️ Choix Techniques & Justifications

### 1. Haute Disponibilité & Répartition
*   **Vote & Result** : Déployés avec plusieurs réplicas (`replicas: 2`) pour assurer la disponibilité même en cas de panne d'un nœud.
*   **Placement** : Les services web sont placés sur les **workers** (`node.role == worker`) pour décharger le manager.
*   **Base de données** : Placée sur le **manager** avec un volume persistant pour garantir la stabilité des données.

### 2. Réseau & Sécurité
*   Utilisation de réseaux **Overlay** (`frontend`, `backend`) pour la communication sécurisée entre les nœuds du Swarm.
*   Isolation : La base de données n'est accessible que par le backend.

### 3. Robustesse (Healthchecks)
*   Des sondes de santé (`healthcheck`) sont configurées pour **PostgreSQL** et **Redis**.
*   Les services dépendants (`vote`, `result`, `worker`) attendent que la DB et Redis soient `healthy` avant de démarrer, évitant les crashs au lancement.

---

## 🧹 Nettoyage

Pour arrêter les machines (économie de ressources) :
```bash
vagrant halt
```

Pour détruire complètement l'environnement :
```bash
vagrant destroy -f
```
