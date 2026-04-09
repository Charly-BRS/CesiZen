# 05 — Frontend Web (React 19 + Vite)

L'application web est un **Single Page Application (SPA)** construite avec React 19, Vite et Tailwind CSS. Elle consomme l'API Symfony et offre une interface complète pour les utilisateurs et les administrateurs.

---

## 5.1 Installation

```bash
cd cesizen-app

# Copier les variables d'environnement
cp .env.example .env.local

# Installer les dépendances
npm install

# Lancer en développement
npm run dev
# → http://localhost:5173
```

### Variable d'environnement (`.env.local`)

```env
# URL de base de l'API (le préfixe VITE_ est obligatoire pour exposer au navigateur)
VITE_API_URL=http://localhost:8080/api
```

---

## 5.2 Structure `src/`

```
src/
├── main.tsx                    ← Point d'entrée React (monte <App /> dans le DOM)
├── App.tsx                     ← Routeur principal (React Router v7)
├── index.css                   ← Styles globaux (import Tailwind)
│
├── context/
│   └── AuthContext.tsx         ← Gestion de l'authentification (état global)
│
├── components/
│   ├── Navbar.tsx              ← Barre de navigation (liens + déconnexion)
│   ├── PrivateRoute.tsx        ← Redirige vers /login si non connecté
│   └── AdminRoute.tsx          ← Redirige vers /dashboard si pas admin
│
├── pages/
│   ├── LoginPage.tsx           ← Formulaire de connexion
│   ├── RegisterPage.tsx        ← Formulaire d'inscription
│   ├── DashboardPage.tsx       ← Accueil utilisateur
│   ├── ProfilPage.tsx          ← Voir/modifier son profil
│   ├── ArticlesPage.tsx        ← Liste des articles
│   ├── ArticleDetailPage.tsx   ← Détail d'un article
│   ├── ExercisesPage.tsx       ← Liste des exercices de respiration
│   ├── ExerciseDetailPage.tsx  ← Détail + animation d'un exercice
│   ├── SessionHistoryPage.tsx  ← Historique des sessions
│   └── admin/
│       ├── AdminDashboardPage.tsx   ← Tableau de bord admin
│       ├── AdminUsersPage.tsx       ← Gestion des utilisateurs
│       ├── AdminArticlesPage.tsx    ← Gestion des articles
│       └── AdminExercisesPage.tsx   ← Gestion des exercices
│
└── services/
    ├── api.ts                  ← Instance Axios centralisée
    ├── authService.ts          ← Login, register, profil, changement MDP
    ├── articleService.ts       ← CRUD articles
    ├── exerciseService.ts      ← Exercices + sessions
    └── adminService.ts         ← Endpoints admin
```

---

## 5.3 Authentification

### AuthContext (`src/context/AuthContext.tsx`)

Le `AuthContext` est un **contexte React** partagé dans toute l'application. Il centralise l'état d'authentification.

**Ce qu'il stocke :**
- `utilisateur` : objet `{ id, email, prenom, nom, roles }` (extrait du token JWT)
- `token` : la chaîne JWT brute
- `estConnecte` : booléen calculé

**Ce qu'il stocke dans `localStorage` :**
- `token` → le JWT reçu de l'API
- `utilisateur` → les infos utilisateur (pour restaurer la session au rechargement)

**Hook d'utilisation :**
```tsx
// Dans n'importe quel composant
import { useAuth } from '../context/AuthContext';

const { utilisateur, estConnecte, connecter, deconnecter } = useAuth();
```

### Protection des routes

**`PrivateRoute`** — Si l'utilisateur n'est pas connecté, redirige vers `/login` :
```tsx
// Dans App.tsx
<Route path="/dashboard" element={
    <PrivateRoute>
        <DashboardPage />
    </PrivateRoute>
} />
```

**`AdminRoute`** — Si l'utilisateur n'a pas le rôle `ROLE_ADMIN`, redirige vers `/dashboard` :
```tsx
<Route path="/admin" element={
    <AdminRoute>
        <AdminDashboardPage />
    </AdminRoute>
} />
```

---

## 5.4 Services API

### `api.ts` — Instance Axios centralisée

```ts
// Toutes les requêtes passent par cette instance
const api = axios.create({
    baseURL: import.meta.env.VITE_API_URL,
    headers: { 'Content-Type': 'application/json' }
});

// Intercepteur : ajoute automatiquement le token Bearer à chaque requête
api.interceptors.request.use((config) => {
    const token = localStorage.getItem('token');
    if (token) {
        config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
});

// Intercepteur : si le token est expiré (401), déconnecter et rediriger
api.interceptors.response.use(
    (response) => response,
    (error) => {
        if (error.response?.status === 401) {
            localStorage.clear();
            window.location.href = '/login';
        }
        return Promise.reject(error);
    }
);
```

### Services disponibles

| Fichier | Fonctions | Endpoints appelés |
|---------|-----------|------------------|
| `authService.ts` | `login()`, `register()`, `getMonProfil()`, `modifierProfil()`, `changerMotDePasse()` | `/auth/login`, `/auth/register`, `/users/{id}` |
| `articleService.ts` | `getArticles()`, `getArticle()`, `creerArticle()`, `modifierArticle()`, `supprimerArticle()` | `/articles`, `/articles/{id}` |
| `exerciseService.ts` | `getExercises()`, `getExercise()`, `demarrerSession()`, `terminerSession()`, `getSessions()` | `/breathing_exercises`, `/user_sessions` |
| `adminService.ts` | `getUsers()`, `toggleUserStatus()`, `creerExercice()`, `modifierExercice()`, etc. | Tous les endpoints admin |

---

## 5.5 Pages et routes

| URL | Composant | Accès |
|-----|-----------|-------|
| `/` | Redirige → `/dashboard` ou `/login` | — |
| `/login` | `LoginPage` | Public |
| `/register` | `RegisterPage` | Public |
| `/dashboard` | `DashboardPage` | Connecté |
| `/profil` | `ProfilPage` | Connecté |
| `/articles` | `ArticlesPage` | Connecté |
| `/articles/:id` | `ArticleDetailPage` | Connecté |
| `/exercises` | `ExercisesPage` | Connecté |
| `/exercises/:id` | `ExerciseDetailPage` | Connecté |
| `/sessions` | `SessionHistoryPage` | Connecté |
| `/admin` | `AdminDashboardPage` | ROLE_ADMIN |
| `/admin/users` | `AdminUsersPage` | ROLE_ADMIN |
| `/admin/articles` | `AdminArticlesPage` | ROLE_ADMIN |
| `/admin/exercises` | `AdminExercisesPage` | ROLE_ADMIN |

---

## 5.6 Dépendances principales

| Package | Version | Rôle |
|---------|---------|------|
| `react` | ^19.2.4 | Framework UI |
| `react-dom` | ^19.2.4 | Rendu DOM |
| `react-router-dom` | ^7.13.2 | Navigation SPA (routes) |
| `axios` | ^1.14.0 | Requêtes HTTP vers l'API |
| `vite` | ^8.0.1 | Bundler ultra-rapide |
| `tailwindcss` | ^4.2.2 | Framework CSS utilitaire |
| `typescript` | ~5.9.3 | Typage statique |
| `eslint` | ^9.39.4 | Linter qualité de code |

---

## 5.7 Scripts disponibles

```bash
npm run dev      # Démarrer en développement (hot reload)
npm run build    # Construire pour la production (dossier dist/)
npm run lint     # Vérifier la qualité du code avec ESLint
npm run preview  # Aperçu du build de production
```

---

**Suite →** [06 — Application Mobile Expo](./06-mobile.md)
