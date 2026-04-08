# CESIZen — Application de gestion du bien-être

Projet étudiant CESI — Application full-stack de gestion du bien-être mental.

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

## Phases du projet

- [x] **Phase 1** — Infrastructure (Docker, JWT, CORS, structure de dossiers)
- [ ] **Phase 2** — Entités Doctrine + Authentification JWT
- [ ] **Phase 3** — Écrans métier (Login, Dashboard, Activités)
- [ ] **Phase 4** — Tests et déploiement

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
