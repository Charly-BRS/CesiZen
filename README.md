# CESIZen — Application de gestion du bien-être

[![Tests](https://github.com/Charly-BRS/CesiZen/actions/workflows/test.yml/badge.svg)](https://github.com/Charly-BRS/CesiZen/actions/workflows/test.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

Projet étudiant CESI — Application full-stack de gestion du bien-être mental avec **161 tests** automatisés et **CI/CD** GitHub Actions.

## Architecture

```
CesiZen -Test/
├── cesizen-api/        → API REST (Symfony 7.2 + API Platform 3)
├── cesizen-app/        → Frontend web (React 19 + Vite + TailwindCSS)
├── cesizen-mobile/     → Application mobile (React Native + Expo)
└── docker-compose.yml  → Orchestration globale (api + web)
```

## Prérequis

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) installé et lancé
- [Node.js 22+](https://nodejs.org/) pour le mobile
- [Expo Go](https://expo.dev/client) sur votre téléphone (optionnel)

---

## Démarrage rapide

### Option A — Tout lancer avec Docker (api + web)

```bash
# À la racine du projet
docker-compose up --build
```

- API disponible sur : http://localhost:8080
- Documentation Swagger : http://localhost:8080/api/docs
- Frontend React : http://localhost:5173

### Option B — Lancer chaque projet séparément

#### API Symfony
```bash
cd cesizen-api
cp .env.example .env   # Configurer les variables
docker-compose up --build
```

#### Frontend React
```bash
cd cesizen-app
cp .env.example .env.local
docker-compose up
# Ou sans Docker :
npm install && npm run dev
```

#### Application mobile Expo
```bash
cd cesizen-mobile
cp .env.example .env
npm install
npx expo start
# Scanner le QR code avec Expo Go sur votre téléphone
```

---

## Structure des branches (Gitflow)

```
main      → code stable, prêt pour la production
develop   → intégration des features en cours
feature/* → une branche par fonctionnalité
```

**Règle absolue : ne jamais committer directement sur `main`.**

Créer une feature :
```bash
git checkout develop
git checkout -b feature/nom-de-la-feature
# ... développer ...
git add .
git commit -m "feat: description de la feature"
```

---

## Variables d'environnement

| Projet          | Fichier source   | Fichier à créer  |
|-----------------|------------------|------------------|
| cesizen-api     | `.env.example`   | `.env`           |
| cesizen-app     | `.env.example`   | `.env.local`     |
| cesizen-mobile  | `.env.example`   | `.env`           |

---

## 🧪 Tests & CI/CD

### Couverture de tests

| Projet | Framework | Tests | Statut |
|--------|-----------|-------|--------|
| **cesizen-mobile** | Jest | 114 | ✅ Passing |
| **cesizen-api** | PHPUnit | 47 | ✅ Passing |
| **cesizen-app** | ESLint | - | ✅ Clean |
| **TOTAL** | - | **161** | ✅ **All Green** |

### Exécuter les tests localement

```bash
# Tests mobiles (Jest) - ~3s
cd cesizen-mobile
npm test

# Tests app (ESLint) - ~2s
cd cesizen-app
npm test

# Tests API (PHPUnit) en Docker - ~15s
cd cesizen-api
npm test
```

### GitHub Actions

Tous les tests s'exécutent automatiquement sur:
- ✅ Push vers `main` ou `develop`
- ✅ Pull Requests

Visualiser les résultats: [GitHub Actions](https://github.com/Charly-BRS/CesiZen/actions)

---

## Phases du projet

- [x] **Phase 1** — Infrastructure (Docker, JWT, CORS, structure de dossiers)
- [x] **Phase 2** — Entités Doctrine + Authentification JWT + Tests API
- [x] **Phase 3** — Tests (161 tests finalisés et nettoyés)
- [x] **Phase 4** — CI/CD avec GitHub Actions
- [ ] **Phase 5** — Écrans métier (Login, Dashboard, Activités)

---

## Conventions de commit

```
feat:  nouvelle fonctionnalité
fix:   correction de bug
docs:  documentation uniquement
style: formatage (pas de changement de logique)
refactor: refactorisation sans nouvelle feature
test:  ajout ou correction de tests
```
