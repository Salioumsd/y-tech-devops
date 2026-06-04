# Y-Tech — Base documentaire industrialisée

> Projet de fin de module DevOps — B3  
> Industrialisation du déploiement de la base documentaire Y-Tech

---

## Sommaire

1. [Architecture](#architecture)
2. [Prérequis](#prérequis)
3. [Démarrage rapide (développement)](#démarrage-rapide)
4. [Structure du projet](#structure-du-projet)
5. [Sécurité](#sécurité)
6. [Pipeline CI/CD GitLab](#pipeline-cicd)
7. [Docker Swarm (production)](#docker-swarm)
8. [Persistance des données](#persistance-des-données)
9. [Variables d'environnement](#variables-denvironnement)

---

## Architecture

```
Internet
    │
    ▼
┌─────────┐      ┌───────────────────────────┐
│  Nginx  │─────▶│  App Flask (Gunicorn x4)  │
│  :80    │      │  :5000                    │
└─────────┘      └────────────┬──────────────┘
                              │ TCP 3306
                   ┌──────────▼──────────┐
                   │  MariaDB 11.3       │
                   │  ytech_db           │
                   └─────────────────────┘
```

### Services

| Service | Image | Rôle |
|---------|-------|------|
| `nginx` | `nginx:1.27-alpine` | Reverse proxy, terminaison HTTP, headers de sécurité |
| `app` | build local (`python:3.12-slim`) | Application Flask + Gunicorn |
| `db` | `mariadb:11.3` | Base de données relationnelle |

### Volumes persistants

| Volume | Contenu |
|--------|---------|
| `db_data` | Données MariaDB (`/var/lib/mysql`) |
| `app_uploads` | Fichiers uploadés PDF, PNG… (`/app/uploads`) |

---

## Prérequis

- Docker ≥ 26
- Docker Compose ≥ 2.27
- (Pour Swarm) Docker Swarm initialisé sur le serveur de prod

---

## Démarrage rapide

### 1. Cloner le dépôt

```bash
git clone https://gitlab.com/<votre-groupe>/ytech-docs.git
cd ytech-docs
```

### 2. Configurer l'environnement

```bash
cp .env.example .env
# Éditer .env et renseigner vos valeurs
nano .env
```

### 3. Lancer en développement

```bash
docker compose up -d --build
```

L'application est accessible sur **http://localhost**

**Identifiants par défaut :**
- Utilisateur : `admin`
- Mot de passe : valeur de `ADMIN_PASSWORD` dans `.env` (défaut : `Admin1234!`)

### 4. Vérifier les logs

```bash
docker compose logs -f app
docker compose logs -f db
```

### 5. Arrêter

```bash
docker compose down          # Arrêt sans supprimer les données
docker compose down -v       # Arrêt + suppression des volumes (attention !)
```

---

## Structure du projet

```
ytech/
├── app/
│   ├── Dockerfile            # Image multi-stage Python
│   ├── entrypoint.sh         # Wait-for-DB + init + Gunicorn
│   ├── requirements.txt      # Dépendances Python
│   ├── app.py                # Application Flask principale
│   └── templates/            # Templates Jinja2
│       ├── base.html
│       ├── login.html
│       ├── register.html
│       ├── index.html
│       ├── pages_list.html
│       ├── page_view.html
│       ├── page_form.html
│       ├── documents_list.html
│       ├── document_upload.html
│       └── admin_users.html
├── nginx/
│   └── nginx.conf            # Configuration reverse proxy
├── mysql-init/
│   └── 01_init.sql           # Script SQL d'initialisation
├── docker-compose.yml        # Environnement de développement
├── docker-stack.yml          # Déploiement Docker Swarm (prod)
├── .gitlab-ci.yml            # Pipeline CI/CD GitLab
├── .env.example              # Template des variables d'env
├── .gitignore
└── README.md
```

---

## Sécurité

### Conteneur applicatif

- **Utilisateur non-root** : l'application tourne avec l'utilisateur `ytech` (UID dédié, pas de shell)
- **Build multi-stage** : l'image finale ne contient pas les outils de compilation (gcc, headers…)
- **Pas d'exposition directe** : le port 5000 de Flask n'est jamais exposé sur l'hôte, seul Nginx est accessible

### Réseau

- **Réseau Docker isolé** : tous les services communiquent sur `ytech_net`, la base de données n'est pas accessible depuis l'extérieur
- **Headers HTTP de sécurité** via Nginx : `X-Frame-Options`, `X-Content-Type-Options`, `X-XSS-Protection`, `Referrer-Policy`
- `server_tokens off` sur Nginx (pas de divulgation de version)

### Secrets

- **Développement** : variables d'environnement via `.env` (non commité)
- **Production Swarm** : `docker secret` (les secrets sont montés comme fichiers en mémoire, jamais visibles dans `docker inspect`)

### Application

- Mots de passe hashés avec **Werkzeug** (PBKDF2-SHA256)
- Validation des extensions de fichiers uploadés (liste blanche)
- Noms de fichiers UUID (pas de chemin relatif possible)
- Taille maximale d'upload limitée (16 Mo Flask + 20 Mo Nginx)
- Authentification requise sur toutes les routes (décorateur `@login_required`)

---

## Pipeline CI/CD

### Stages

```
test ──▶ build ──▶ publish ──▶ deploy
```

| Stage | Job | Déclencheur |
|-------|-----|-------------|
| `test` | `lint` — analyse statique flake8 | MR, `develop`, `main` |
| `test` | `unit_tests` — init DB + smoke test | MR, `develop`, `main` |
| `build` | `build_image` — `docker build` multi-stage | `develop`, `main` |
| `publish` | `push_image` — push vers GitLab Container Registry | `develop`, `main` |
| `deploy` | `deploy_staging` — `docker compose up` | `develop` (auto) |
| `deploy` | `deploy_production` — `docker stack deploy` | `main` (**manuel**) |

### Flux Git recommandé

```
feature/* ──▶ develop ──▶ main
                │              │
            (staging)     (production)
             auto           manuel
```

### Variables GitLab à configurer

Dans **Settings > CI/CD > Variables** :

| Variable | Type | Description |
|----------|------|-------------|
| `SSH_PRIVATE_KEY` | File | Clé SSH privée pour accéder au serveur de prod |
| `DEPLOY_HOST` | Variable | IP / DNS du serveur cible |
| `DEPLOY_USER` | Variable | Utilisateur SSH |
| `DB_ROOT_PASSWORD` | Variable (masked) | Mot de passe root MariaDB |
| `DB_PASS` | Variable (masked) | Mot de passe utilisateur DB |
| `SECRET_KEY` | Variable (masked) | Clé secrète Flask |
| `ADMIN_PASSWORD` | Variable (masked) | Mot de passe admin initial |

---

## Docker Swarm

Docker Swarm permet de **scaler horizontalement** l'application et d'assurer la **haute disponibilité**.

### Initialisation du cluster

```bash
# Sur le serveur manager
docker swarm init --advertise-addr <IP_DU_MANAGER>

# Ajouter des workers (commande affichée par docker swarm init)
docker swarm join --token <TOKEN> <IP_DU_MANAGER>:2377
```

### Créer les secrets

```bash
echo "SuperSecret!"           | docker secret create db_root_password -
echo "YtechDbPass!"           | docker secret create db_password -
openssl rand -hex 32           | docker secret create app_secret_key -
echo "AdminPass!"             | docker secret create admin_password -
```

### Déployer le stack

```bash
docker stack deploy --with-registry-auth -c docker-stack.yml ytech
```

### Commandes utiles

```bash
# État du stack
docker stack ps ytech

# Liste des services
docker service ls

# Scaler l'application (ex: 4 réplicas)
docker service scale ytech_app=4

# Logs d'un service
docker service logs -f ytech_app

# Mettre à jour l'image (rolling update)
docker service update --image registry.gitlab.com/<path>/ytech-app:nouvelleverison ytech_app

# Supprimer le stack
docker stack rm ytech
```

### Rolling update automatique

Le fichier `docker-stack.yml` est configuré pour les mises à jour progressives :

- `parallelism: 1` — une réplica à la fois
- `order: start-first` — le nouveau conteneur démarre avant que l'ancien s'arrête (zéro downtime)
- `failure_action: rollback` — retour automatique en cas d'échec

---

## Persistance des données

Les données sont persistées via deux volumes Docker nommés :

- **`db_data`** → `/var/lib/mysql` — toutes les données MariaDB
- **`app_uploads`** → `/app/uploads` — tous les fichiers uploadés

En cas de redémarrage, de mise à jour ou de redéploiement sur une autre machine, les données sont conservées tant que les volumes existent.

**Backup recommandé :**

```bash
# Dump de la base de données
docker exec ytech_db mysqldump -u root -p"$DB_ROOT_PASSWORD" ytech_db > backup_$(date +%Y%m%d).sql

# Backup des uploads
docker run --rm -v app_uploads:/data -v $(pwd):/backup alpine \
  tar czf /backup/uploads_$(date +%Y%m%d).tar.gz /data
```

---

## Variables d'environnement

| Variable | Description | Défaut |
|----------|-------------|--------|
| `DB_HOST` | Hostname de MariaDB | `db` |
| `DB_NAME` | Nom de la base | `ytech_db` |
| `DB_USER` | Utilisateur MariaDB | `ytech` |
| `DB_PASS` | Mot de passe | — |
| `DB_ROOT_PASSWORD` | Mot de passe root | — |
| `SECRET_KEY` | Clé secrète Flask (sessions) | — |
| `ADMIN_PASSWORD` | Mot de passe admin initial | `Admin1234!` |
| `UPLOAD_FOLDER` | Chemin des uploads | `/app/uploads` |
