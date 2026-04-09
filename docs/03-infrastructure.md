# 03 — Infrastructure Docker

Docker orchestre tous les services du projet. Un seul fichier `docker-compose.yml` à la **racine du projet** permet de tout lancer en une commande.

---

## Vue d'ensemble des services

```
docker-compose.yml (racine)
│
├── nginx          ← Serveur web, reçoit les requêtes HTTP sur :8080
│     └── proxy → php-fpm:9000
│
├── php-fpm        ← Exécute le code PHP Symfony
│     └── connexion → postgres:5432
│
└── postgres       ← Base de données PostgreSQL 16
      └── volume postgres_data (persistant)
```

---

## Commandes essentielles

### Démarrer le projet

```bash
# 1ère fois : construire les images et démarrer
docker compose up --build

# Les fois suivantes : démarrer sans reconstruire
docker compose up

# Démarrer en arrière-plan (pas de logs dans le terminal)
docker compose up -d
```

### Arrêter le projet

```bash
# Arrêter les conteneurs (garde les données)
docker compose down

# Arrêter ET supprimer les volumes (REMET LA BDD À ZÉRO)
docker compose down -v
```

### Voir les logs

```bash
# Tous les services
docker compose logs -f

# Un service spécifique
docker compose logs -f php-fpm
docker compose logs -f nginx
docker compose logs -f postgres
```

### Exécuter des commandes dans un conteneur

```bash
# Ouvrir un shell dans le conteneur PHP
docker exec -it cesizen_php bash

# Lancer une commande Symfony sans ouvrir de shell
docker exec cesizen_php php bin/console <commande>
```

---

## Initialisation de la base de données (1ère fois)

Ces commandes ne s'exécutent **qu'une seule fois** lors du premier démarrage :

```bash
# Jouer toutes les migrations (crée les tables)
docker exec cesizen_php php bin/console doctrine:migrations:migrate --no-interaction

# Charger les données de démo (admin, catégories, exercices)
docker exec cesizen_php php bin/console doctrine:fixtures:load --no-interaction
```

> **Pourquoi les fixtures ?** Elles créent le compte administrateur (`admin@cesizen.fr`) et les données de base (catégories, exercices de respiration) sans lesquelles l'application ne peut pas fonctionner correctement.

---

## Génération des clés JWT (1ère fois)

L'authentification JWT nécessite une paire de clés RSA. Si elles n'existent pas encore :

```bash
docker exec cesizen_php php bin/console lexik:jwt:generate-keypair
```

Cela crée `config/jwt/private.pem` et `config/jwt/public.pem`. Ces fichiers **ne doivent jamais être commités** (déjà dans `.gitignore`).

---

## Variables d'environnement (`cesizen-api/.env`)

Copie `.env.example` vers `.env` et adapte si nécessaire :

```bash
cd cesizen-api
cp .env.example .env
```

| Variable | Valeur par défaut | Explication |
|----------|-------------------|-------------|
| `APP_ENV` | `dev` | Environnement Symfony (`dev`, `prod`, `test`) |
| `APP_SECRET` | *(chaîne aléatoire)* | Clé secrète Symfony, **ne jamais partager** |
| `DATABASE_URL` | `postgresql://cesizen_user:cesizen_password@postgres:5432/cesizen` | URL complète de connexion PostgreSQL. `postgres` = hostname interne Docker |
| `POSTGRES_DB` | `cesizen` | Nom de la base de données |
| `POSTGRES_USER` | `cesizen_user` | Utilisateur PostgreSQL |
| `POSTGRES_PASSWORD` | `cesizen_password` | Mot de passe PostgreSQL |
| `JWT_PASSPHRASE` | *(hash 64 chars)* | Mot de passe pour chiffrer les clés RSA JWT |
| `JWT_TTL` | `900` | Durée de vie du token JWT en secondes (900 = 15 minutes) |
| `CORS_ALLOW_ORIGIN` | `^https?://(localhost\|127\.0\.0\.1)(:[0-9]+)?$` | Origines autorisées pour les requêtes CORS depuis le navigateur |

> **Attention** : Dans `DATABASE_URL`, le hostname est `postgres` (nom du service Docker), **pas** `localhost`. Si tu lances Symfony hors Docker, remplace par `localhost`.

---

## Configuration Nginx (`cesizen-api/docker/nginx/default.conf`)

Nginx sert de **reverse proxy** : il reçoit les requêtes HTTP et les transmet à PHP-FPM.

```nginx
server {
    listen 80;
    root /var/www/html/public;   # Dossier public/ de Symfony

    # Toutes les URLs passent par index.php (Front Controller Symfony)
    location / {
        try_files $uri /index.php$is_args$args;
    }

    # Transmet les requêtes PHP à PHP-FPM
    location ~ ^/index\.php(/|$) {
        fastcgi_pass php-fpm:9000;   # Hostname interne Docker
        fastcgi_split_path_info ^(.+\.php)(/.*)$;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        fastcgi_read_timeout 300;
        internal;
    }

    # Bloque l'accès direct aux autres fichiers .php (sécurité)
    location ~ \.php$ {
        return 404;
    }
}
```

---

## Réinitialiser complètement la base de données

Utile si tu veux repartir de zéro (ex: tu as cassé les migrations) :

```bash
# 1. Supprimer la base et la recréer
docker exec cesizen_php php bin/console doctrine:database:drop --force --env=dev
docker exec cesizen_php php bin/console doctrine:database:create --env=dev

# 2. Rejouer toutes les migrations
docker exec cesizen_php php bin/console doctrine:migrations:migrate --no-interaction

# 3. Recharger les fixtures
docker exec cesizen_php php bin/console doctrine:fixtures:load --no-interaction
```

---

## Accès à la base de données en ligne de commande

```bash
# Se connecter à PostgreSQL depuis le conteneur
docker exec -it cesizen_postgres psql -U cesizen_user -d cesizen

# Quelques commandes utiles dans psql :
\dt          # lister les tables
\d articles  # décrire la table articles
SELECT * FROM utilisateurs;
\q           # quitter
```

---

---

## Docker Compose — Production

Pour la production, un fichier séparé `docker-compose.prod.yml` est disponible à la racine du projet.

### Différences avec le dev

| Aspect | Développement | Production |
|--------|--------------|------------|
| Frontend | Hot-reload Node.js | Build statique Nginx (multi-stage) |
| PostgreSQL | Port 5432 exposé | Port non exposé (sécurité) |
| Code PHP | Monté en volume | Copié dans l'image au build |
| Restart | Non configuré | `unless-stopped` sur tous les services |
| Variables | `.env` monté | `${VAR}` via fichier `.env.prod` |

### Lancer en production

```bash
# 1. Copier et renseigner le fichier de variables
cp .env.prod.example .env.prod
# Éditer .env.prod : mots de passe, secrets, URLs

# 2. Générer les clés JWT
docker compose -f docker-compose.prod.yml run --rm php-fpm \
  php bin/console lexik:jwt:generate-keypair --env=prod

# 3. Lancer l'infrastructure
docker compose -f docker-compose.prod.yml --env-file .env.prod up -d --build

# 4. Jouer les migrations
docker compose -f docker-compose.prod.yml exec php-fpm \
  php bin/console doctrine:migrations:migrate --env=prod --no-interaction
```

### Ports exposés (prod)

| Service | Port hôte | Description |
|---------|-----------|-------------|
| Nginx API | 8080 | API REST (`/api`) et Swagger (`/api/docs`) |
| Frontend | 3000 | Interface React (statique Nginx) |
| PostgreSQL | *(non exposé)* | Accessible uniquement via Docker network |

---

**Suite →** [04 — API Backend Symfony](./04-api-backend.md)
