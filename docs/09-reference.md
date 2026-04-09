# 09 — Référence rapide (Cheatsheet)

Toutes les commandes et informations essentielles en un seul endroit.

---

## Démarrage du projet (tout lancer)

```bash
# 1. Infrastructure (API + BDD + Nginx)
cd "CesiZen -Test"
docker compose up --build

# 2. Initialisation BDD (1ère fois uniquement)
docker exec cesizen_php php bin/console doctrine:migrations:migrate --no-interaction
docker exec cesizen_php php bin/console doctrine:fixtures:load --no-interaction
docker exec cesizen_php php bin/console lexik:jwt:generate-keypair

# 3. Frontend web (dans un nouveau terminal)
cd cesizen-app && npm install && npm run dev

# 4. Mobile (dans un nouveau terminal)
cd cesizen-mobile && npm install && npx expo start
```

---

## Commandes Docker

```bash
# Démarrage / Arrêt
docker compose up --build        # 1ère fois (construit les images)
docker compose up                # Démarrage sans rebuild
docker compose up -d             # Démarrage en arrière-plan
docker compose down              # Arrêt (garde les données)
docker compose down -v           # Arrêt + suppression volumes (REMET BDD À ZÉRO)
docker compose restart           # Redémarrer tous les services

# Logs
docker compose logs -f           # Tous les logs en temps réel
docker compose logs -f php-fpm   # Logs PHP uniquement
docker compose logs -f nginx     # Logs Nginx uniquement
docker compose logs -f postgres  # Logs PostgreSQL uniquement

# Accéder aux conteneurs
docker exec -it cesizen_php bash          # Shell PHP
docker exec -it cesizen_postgres bash     # Shell PostgreSQL
docker exec cesizen_php <commande>        # Exécuter sans shell interactif

# État des conteneurs
docker compose ps                # Voir les conteneurs actifs
docker ps                        # Tous les conteneurs Docker
docker images                    # Images Docker locales
```

---

## Commandes Symfony (dans le conteneur PHP)

```bash
# Préfixe obligatoire : docker exec cesizen_php

# Migrations
php bin/console doctrine:migrations:migrate --no-interaction   # Appliquer toutes les migrations
php bin/console doctrine:migrations:status                     # État des migrations
php bin/console doctrine:migrations:diff                       # Générer une migration depuis les entités
php bin/console doctrine:migrations:execute --down "Version..."  # Annuler une migration

# Base de données
php bin/console doctrine:database:create                       # Créer la BDD
php bin/console doctrine:database:drop --force                 # Supprimer la BDD
php bin/console doctrine:fixtures:load --no-interaction        # Charger les fixtures

# Clés JWT
php bin/console lexik:jwt:generate-keypair                     # Générer les clés RSA

# Cache
php bin/console cache:clear                                    # Vider le cache Symfony
php bin/console cache:warmup                                   # Préchauffer le cache

# Debug
php bin/console debug:router                                   # Lister toutes les routes
php bin/console debug:container                                # Lister tous les services
```

---

## Commandes npm

### `cesizen-app` (React web)

```bash
npm run dev          # Démarrer en développement → http://localhost:5173
npm run build        # Construire pour production (dossier dist/)
npm run lint         # Vérifier la qualité (ESLint)
npm run preview      # Aperçu du build de production
npx tsc -b --noEmit  # Vérifier les types TypeScript
```

### `cesizen-mobile` (Expo)

```bash
npx expo start              # Démarrer (QR code)
npx expo start --android    # Démarrer sur émulateur Android
npx expo start --ios        # Démarrer sur simulateur iOS
npm run type-check           # Vérifier les types TypeScript
npm test                     # Lancer les tests Jest
npm run test:watch            # Tests en mode watch
npm run test:coverage         # Tests + rapport de couverture
```

---

## Commandes Tests

```bash
# API — PHPUnit
docker exec cesizen_php vendor/bin/phpunit
docker exec cesizen_php vendor/bin/phpunit --coverage-clover=var/coverage/clover.xml
docker exec cesizen_php vendor/bin/phpunit tests/Entity/BreathingExerciseTest.php

# API — PHPStan
docker exec cesizen_php vendor/bin/phpstan analyse src --level=6 --no-progress

# Mobile — Jest
cd cesizen-mobile && npm test
cd cesizen-mobile && npm run test:coverage

# Frontend — Lint + TypeScript
cd cesizen-app && npm run lint
cd cesizen-app && npx tsc -b --noEmit
```

---

## Variables d'environnement

### `cesizen-api/.env`

| Variable | Valeur dev | Description |
|----------|-----------|-------------|
| `APP_ENV` | `dev` | Environnement Symfony |
| `APP_SECRET` | *(32 chars aléatoires)* | Clé secrète Symfony |
| `DATABASE_URL` | `postgresql://cesizen_user:cesizen_password@postgres:5432/cesizen` | Connexion PostgreSQL (hostname `postgres` = Docker) |
| `POSTGRES_DB` | `cesizen` | Nom de la base |
| `POSTGRES_USER` | `cesizen_user` | Utilisateur PostgreSQL |
| `POSTGRES_PASSWORD` | `cesizen_password` | Mot de passe PostgreSQL |
| `JWT_PASSPHRASE` | *(hash 64 chars)* | Passphrase des clés RSA JWT |
| `JWT_TTL` | `900` | Durée du token JWT (secondes) |
| `CORS_ALLOW_ORIGIN` | `^https?://(localhost\|127\.0\.0\.1)(:[0-9]+)?$` | Origines CORS autorisées |

### `cesizen-app/.env.local`

| Variable | Valeur dev | Description |
|----------|-----------|-------------|
| `VITE_API_URL` | `http://localhost:8080/api` | URL de base de l'API |

### `cesizen-mobile/.env`

| Variable | Valeur dev | Description |
|----------|-----------|-------------|
| `EXPO_PUBLIC_API_URL` | `http://192.168.X.X:8080/api` | URL de l'API (IP locale, pas localhost !) |

---

## Comptes de test

| Compte | Email | Mot de passe | Rôle |
|--------|-------|--------------|------|
| Administrateur | `admin@cesizen.fr` | `Admin1234!` | `ROLE_ADMIN` |
| Utilisateur | *(à créer via l'inscription)* | *(au choix)* | `ROLE_USER` |

---

## Endpoints API (résumé)

| Méthode | URL | Accès |
|---------|-----|-------|
| POST | `/api/auth/login` | Public |
| POST | `/api/auth/register` | Public |
| POST | `/api/auth/change-password` | Connecté |
| GET | `/api/articles` | Public (publiés) |
| GET/POST/PATCH/DELETE | `/api/articles/{id}` | Voir chapitre 04 |
| GET | `/api/categories` | Public |
| GET | `/api/breathing_exercises` | Public |
| GET/POST/PATCH | `/api/user_sessions` | Connecté |
| GET/PATCH/DELETE | `/api/users/{id}` | Admin / soi-même |
| GET | `/api/docs` | Public (Swagger) |

---

## Pièges courants à éviter

### 1. `localhost` vs IP locale sur mobile
```
❌ EXPO_PUBLIC_API_URL=http://localhost:8080/api
   → "localhost" sur le téléphone = le téléphone lui-même, pas ton PC

✅ EXPO_PUBLIC_API_URL=http://192.168.1.42:8080/api
   → Utilise l'IP de ton PC sur le réseau Wi-Fi local
```

### 2. Clés JWT manquantes
```
❌ Erreur : "Unable to load the private key"
✅ Solution :
   docker exec cesizen_php php bin/console lexik:jwt:generate-keypair
```

### 3. Migrations non jouées
```
❌ Erreur : "Table 'utilisateurs' doesn't exist"
✅ Solution :
   docker exec cesizen_php php bin/console doctrine:migrations:migrate --no-interaction
```

### 4. CORS bloqué en développement
```
❌ Erreur navigateur : "CORS policy: No 'Access-Control-Allow-Origin' header"
✅ Vérifier dans cesizen-api/config/packages/nelmio_cors.yaml :
   allow_origin: ['^https?://(localhost|127\.0\.0\.1)(:[0-9]+)?$']
```

### 5. Variable `VITE_` manquante
```
❌ import.meta.env.VITE_API_URL = undefined
✅ Les variables Vite DOIVENT commencer par VITE_
   Et le fichier doit s'appeler .env.local (pas .env)
```

### 6. `docker compose` vs `docker-compose`
```
❌ docker-compose up   (ancienne syntaxe, plugin séparé)
✅ docker compose up   (nouvelle syntaxe, intégrée à Docker)
```

### 7. Fixtures effacent les données
```
⚠️ doctrine:fixtures:load supprime TOUTES les données avant de recharger
   Utiliser --append pour ajouter sans supprimer :
   docker exec cesizen_php php bin/console doctrine:fixtures:load --append
```

---

## Ports et URLs de référence

| Service | URL | Notes |
|---------|-----|-------|
| API REST | http://localhost:8080/api | Tous les endpoints |
| Swagger UI | http://localhost:8080/api/docs | Documentation interactive |
| Frontend React | http://localhost:5173 | Hot reload activé |
| PostgreSQL | localhost:5432 | Accès direct BDD |
| Expo DevTools | http://localhost:8081 | Quand `npx expo start` |

---

← [Retour à l'index](./README.md)
