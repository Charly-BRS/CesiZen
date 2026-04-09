# 02 — Architecture technique

Ce chapitre explique **pourquoi** ces technologies ont été choisies et comment elles s'articulent ensemble.

---

## Vue d'ensemble

```
┌───────────────────────────────────────────────────────────────────┐
│                        CLIENTS                                    │
│                                                                   │
│  ┌─────────────────┐          ┌──────────────────────────┐        │
│  │  Navigateur web │          │  Téléphone (Expo Go)     │        │
│  │  React 19       │          │  React Native + Expo 54  │        │
│  │  Port 5173      │          │  IP locale:8080          │        │
│  └────────┬────────┘          └────────────┬─────────────┘        │
└───────────┼────────────────────────────────┼─────────────────────┘
            │ HTTP + JWT Bearer              │ HTTP + JWT Bearer
            ▼                               ▼
┌───────────────────────────────────────────────────────────────────┐
│                    DOCKER (réseau cesizen_network)                │
│                                                                   │
│  ┌──────────────────┐    FastCGI    ┌───────────────────────┐     │
│  │  Nginx           │ ──────────→  │  PHP-FPM 8.4          │     │
│  │  Port 8080       │              │  Symfony 7.2          │     │
│  │  (reverse proxy) │              │  API Platform 3       │     │
│  └──────────────────┘              └──────────┬────────────┘     │
│                                               │ Doctrine ORM     │
│                                               ▼                  │
│                                   ┌──────────────────────┐       │
│                                   │  PostgreSQL 16       │       │
│                                   │  Port 5432           │       │
│                                   │  Volume persistant   │       │
│                                   └──────────────────────┘       │
└───────────────────────────────────────────────────────────────────┘
```

---

## Choix techniques

### Pourquoi Symfony 7.2 + API Platform 3 ?

**Symfony** est un framework PHP robuste, très utilisé en entreprise en France. Il impose une structure claire, ce qui facilite la maintenance.

**API Platform** génère automatiquement les endpoints REST CRUD à partir des annotations sur les entités. Sans lui, il faudrait écrire manuellement chaque action (liste, détail, création, modification, suppression) pour chaque entité. Avec lui :

```php
// Ces 5 lignes génèrent automatiquement GET/POST/PATCH/DELETE /api/articles
#[ApiResource(
    operations: [new GetCollection(), new Get(), new Post(), new Patch(), new Delete()]
)]
class Article { ... }
```

### Pourquoi JWT ?

L'authentification par **session** (cookies) fonctionne bien pour les sites web classiques, mais pose des problèmes avec les applications mobiles et les APIs :
- Les cookies sont liés au navigateur
- Les APIs sont souvent appelées depuis des domaines différents (CORS)
- Les applications mobiles ne gèrent pas nativement les sessions

**JWT (JSON Web Token)** résout ces problèmes :
- Le token est un simple texte, envoyé dans le header `Authorization`
- Il fonctionne partout : navigateur, mobile, Postman...
- Le serveur n'a pas besoin de stocker les sessions (stateless)
- Le token expire automatiquement (15 min dans ce projet)

### Pourquoi React 19 + Vite ?

**React** est la bibliothèque UI la plus utilisée au monde. Son modèle de composants réutilisables et son Context API facilitent la gestion de l'état (ex: l'utilisateur connecté).

**Vite** remplace Create React App. Il est **beaucoup plus rapide** au démarrage et au rechargement grâce à ES Modules natifs.

**Tailwind CSS v4** permet de styliser directement dans le JSX avec des classes utilitaires, sans écrire de CSS séparé.

### Pourquoi React Native + Expo ?

**React Native** permet d'écrire une seule codebase qui fonctionne sur iOS et Android. Les composants React (`useState`, `useEffect`, Context...) sont identiques à React web.

**Expo** simplifie le développement React Native :
- Pas besoin d'Android Studio ou Xcode pour tester
- QR code → Expo Go sur le téléphone → app instantanée
- Modules natifs prêts à l'emploi (`SecureStore`, `Camera`, etc.)

---

## Schéma des entités et relations

```
┌──────────────┐         ┌──────────────────┐         ┌────────────┐
│   User       │         │    Article       │         │ Categorie  │
│──────────────│         │──────────────────│         │────────────│
│ id           │ 1     * │ id               │ *     1 │ id         │
│ email        ├────────→│ titre            ├────────→│ nom        │
│ prenom       │         │ contenu          │         │ slug       │
│ nom          │         │ isPublie         │         │ description│
│ roles        │         │ createdAt        │         └────────────┘
│ password     │         │ auteur_id (FK)   │
│ isActif      │         │ categorie_id (FK)│
└──────┬───────┘         └──────────────────┘
       │
       │ 1
       │
       │ *
┌──────▼───────┐         ┌──────────────────────┐
│ UserSession  │         │  BreathingExercise   │
│──────────────│         │──────────────────────│
│ id           │ *     1 │ id                   │
│ status       ├────────→│ nom                  │
│ startedAt    │         │ slug                 │
│ endedAt      │         │ inspirationDuration  │
│ user_id (FK) │         │ apneaDuration        │
│ exercise_id  │         │ expirationDuration   │
└──────────────┘         │ cycles               │
                         │ isPreset             │
                         └──────────────────────┘
```

**Lecture des relations :**
- Un `User` peut avoir plusieurs `Article` (1→*)
- Un `Article` appartient à une `Categorie` (*→1)
- Un `User` peut avoir plusieurs `UserSession` (1→*)
- Une `UserSession` correspond à un `BreathingExercise` (*→1)

---

## Flux d'authentification JWT complet

```
ÉTAPE 1 — Connexion
─────────────────────────────────────────────────────
Frontend          API Symfony
   │                  │
   │─── POST /api/auth/login ──────────────────────→│
   │    { email, password }                          │
   │                                                 │
   │                  │─── Vérifie email en BDD      │
   │                  │─── Compare password haché    │
   │                  │─── Génère JWT (RSA private)  │
   │                  │    Payload: id, email, prenom,
   │                  │    nom, roles, exp (+15min)   │
   │                  │                              │
   │←── 200 OK ───────────────────────────────────── │
   │    { "token": "eyJ0eX..." }                     │


ÉTAPE 2 — Stockage
─────────────────────────────────────────────────────
   Web: localStorage.setItem('token', '...')
   Mobile: SecureStore.setItemAsync('token', '...')


ÉTAPE 3 — Requêtes authentifiées
─────────────────────────────────────────────────────
Frontend          API Symfony
   │                  │
   │─── GET /api/user_sessions ────────────────────→│
   │    Header: Authorization: Bearer eyJ0eX...      │
   │                                                 │
   │                  │─── Vérifie JWT (RSA public)  │
   │                  │─── Extrait l'utilisateur     │
   │                  │─── Applique les filtres auto │
   │                  │                              │
   │←── 200 OK + données ──────────────────────────── │


ÉTAPE 4 — Token expiré
─────────────────────────────────────────────────────
   │─── GET /api/user_sessions ────────────────────→│
   │                                                 │
   │←── 401 Unauthorized ──────────────────────────── │
   │                                                 │
   Intercepteur Axios : vide localStorage/SecureStore
   Redirige vers /login
```

---

## Structure des repos GitHub

```
github.com/Charly-BRS/
│
├── cesizen-api/          ← Repo indépendant (Symfony)
│   ├── .github/workflows/
│   │   ├── test.yml      ← Tests PHPUnit + PHPStan
│   │   └── build-and-deploy.yml  ← Build Docker → GHCR
│   └── ...code...
│
├── cesizen-app/          ← Repo indépendant (React)
│   ├── .github/workflows/
│   │   ├── test.yml      ← ESLint + TypeScript
│   │   └── build-and-deploy.yml  ← Build Vite
│   └── ...code...
│
└── cesizen-mobile/       ← Repo indépendant (Expo)
    ├── .github/workflows/
    │   ├── test.yml      ← Jest + TypeScript
    │   └── build-and-deploy.yml  ← Validation + Expo
    └── ...code...
```

---

**Suite →** [03 — Infrastructure Docker](./03-infrastructure.md)
