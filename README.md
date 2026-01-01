# Voting App — Conteneurisation Docker & Déploiement Swarm

## Présentation du projet
Ce projet est une application distribuée permettant à une audience de voter entre deux propositions (ex: Cats vs Dogs).

## Architecture
- **vote** : application web Python (Flask)
- **worker** : service .NET
- **result** : application web Node.js
- **redis** : file de messages
- **postgres** : base de données persistante

## Prérequis
- Docker ≥ 24.x
- Docker Compose ≥ 2.x
- Docker Swarm (pour le déploiement distribué)

## Lancement en local (Docker Compose)
```bash
# Première fois ou après modifications des sources
docker compose up --build

# Redémarrage ultérieur (les données sont persistées)
docker compose up

# Arrêt propre
docker compose down