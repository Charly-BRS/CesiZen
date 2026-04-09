# Documentation CESIZen

> Application de bien-être mental — Projet CESI  
> Stack : Symfony 7.2 · React 19 · React Native / Expo 54 · PostgreSQL 16 · Docker

---

## Architecture globale

```
┌─────────────────┐        ┌──────────────────────────────────────────────┐
│  Navigateur     │        │              Docker (réseau interne)         │
│  React :5173    │──────→ │  Nginx :8080 ──→ PHP-FPM :9000              │
└─────────────────┘        │                      │                       │
                           │              PostgreSQL :5432                │
┌─────────────────┐        │                      ↑                       │
│  Expo Go        │──────→ │  (même API, via IP locale)                  │
│  (téléphone)    │        └──────────────────────────────────────────────┘
└─────────────────┘
```

**Les 3 projets communiquent tous avec la même API REST.** Le frontend web et l'application mobile partagent exactement les mêmes endpoints.

---

## Ports utilisés

| Service | Port | URL d'accès |
|---------|------|-------------|
| API (Nginx) | **8080** | http://localhost:8080/api |
| Documentation Swagger | **8080** | http://localhost:8080/api/docs |
| Frontend React (dev) | **5173** | http://localhost:5173 |
| PostgreSQL | **5432** | localhost:5432 (interne Docker) |
| PHP-FPM | 9000 | Interne uniquement (Nginx → FPM) |

---

## Table des matières

| # | Chapitre | Description |
|---|---------|-------------|
| [01](./01-prerequisites.md) | **Prérequis** | Outils à installer avant de commencer |
| [02](./02-architecture.md) | **Architecture** | Choix techniques, schémas, flux JWT |
| [03](./03-infrastructure.md) | **Infrastructure Docker** | Lancer les services, base de données |
| [04](./04-api-backend.md) | **API Backend (Symfony)** | Entités, JWT, API Platform, endpoints |
| [05](./05-frontend-web.md) | **Frontend Web (React)** | Pages, auth, services API |
| [06](./06-mobile.md) | **Application Mobile (Expo)** | Navigation, SecureStore, écrans |
| [07](./07-tests.md) | **Tests** | PHPUnit, Jest, PHPStan, ESLint |
| [08](./08-cicd.md) | **CI/CD GitHub Actions** | Workflows, secrets, déploiement |
| [09](./09-reference.md) | **Référence rapide** | Cheatsheet commandes + variables |

---

## Démarrage ultra-rapide (dev)

```bash
# 1. Cloner le repo racine
git clone https://github.com/Charly-BRS/cesizen.git
cd cesizen

# 2. Lancer toute l'infrastructure
docker compose up --build

# 3. (1ère fois uniquement) Initialiser la base de données
docker exec cesizen_php php bin/console doctrine:migrations:migrate --no-interaction
docker exec cesizen_php php bin/console doctrine:fixtures:load --no-interaction

# 4. Lancer le frontend web
cd cesizen-app && npm install && npm run dev

# 5. Lancer le mobile
cd ../cesizen-mobile && npm install && npx expo start
```

**Comptes de test :**
- Admin : `admin@cesizen.fr` / `Admin1234!`
- Inscription libre depuis l'interface

---

## Structure des repos GitHub

Le projet est organisé en **3 repos indépendants** (un par sous-projet) :

| Repo | URL | Branche principale |
|------|-----|-------------------|
| cesizen-api | github.com/Charly-BRS/cesizen-api | `main` |
| cesizen-app | github.com/Charly-BRS/cesizen-app | `main` |
| cesizen-mobile | github.com/Charly-BRS/cesizen-mobile | `main` |

Chaque repo possède ses propres workflows GitHub Actions (`.github/workflows/`).
