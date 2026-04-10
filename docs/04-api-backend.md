# 04 — API Backend (Symfony 7.2)

L'API est le **cœur du projet**. Elle expose tous les endpoints REST consommés par le frontend web et l'application mobile.

---

## 4.1 Installation

```bash
cd cesizen-api

# Copier le fichier d'environnement
cp .env.example .env

# Installer les dépendances PHP (via Docker, pas besoin de PHP local)
docker compose up --build

# Initialiser la base de données (1ère fois uniquement)
docker exec cesizen_php php bin/console doctrine:migrations:migrate --no-interaction
docker exec cesizen_php php bin/console doctrine:fixtures:load --no-interaction
docker exec cesizen_php php bin/console lexik:jwt:generate-keypair
```

---

## 4.2 Structure du projet

```
cesizen-api/src/
├── Controller/
│   └── AuthController.php         ← Routes d'inscription et changement de mot de passe
├── Entity/
│   ├── User.php                   ← Utilisateurs (table: utilisateurs)
│   ├── Article.php                ← Articles de bien-être (table: articles)
│   ├── Categorie.php              ← Catégories d'articles (table: categories)
│   ├── BreathingExercise.php      ← Exercices de respiration (table: breathing_exercises)
│   └── UserSession.php            ← Historique des sessions (table: user_sessions)
├── Repository/
│   └── *Repository.php            ← Requêtes Doctrine pour chaque entité
├── State/
│   ├── ArticleStateProcessor.php  ← Injecte l'auteur automatiquement à la création
│   └── UserSessionStateProcessor.php ← Injecte l'utilisateur automatiquement
├── Doctrine/
│   ├── ArticlePublieExtension.php ← Filtre auto: affiche uniquement les articles publiés
│   └── UserSessionExtension.php   ← Filtre auto: affiche uniquement les sessions de l'utilisateur connecté
├── Security/
│   ├── JWTCreatedListener.php     ← Enrichit le token JWT avec les données utilisateur
│   └── UserHashPasswordSubscriber.php ← Hache le mot de passe avant sauvegarde
└── DataFixtures/
    ├── AdminFixtures.php          ← Crée le compte admin
    ├── CategorieFixtures.php      ← Crée les 5 catégories
    ├── BreathingExerciseFixtures.php ← Crée les 3 exercices preset
    └── AppFixtures.php            ← Point d'entrée fixtures
```

---

## 4.3 Les 5 entités

### Entité `User` (table: `utilisateurs`)

Représente un utilisateur de l'application.

| Champ | Type SQL | Description | Règles |
|-------|----------|-------------|--------|
| `id` | INT (PK) | Identifiant unique | Auto-généré |
| `email` | VARCHAR(180) | Email de connexion | Unique, obligatoire |
| `roles` | JSON | Tableau des rôles | Ex: `["ROLE_USER"]` ou `["ROLE_ADMIN"]` |
| `password` | VARCHAR(255) | Mot de passe **haché** | Jamais stocké en clair |
| `prenom` | VARCHAR(100) | Prénom | Obligatoire |
| `nom` | VARCHAR(100) | Nom de famille | Obligatoire |
| `isActif` | BOOLEAN | Compte actif/désactivé | Défaut: `true` |
| `dateDesactivation` | TIMESTAMP | Date de désactivation RGPD | Nullable — rempli à la désactivation |
| `resetPasswordToken` | VARCHAR(64) | Token hex de réinitialisation | Nullable — valable 1 heure |
| `resetPasswordTokenExpiry` | TIMESTAMP | Expiration du token | Nullable |
| `createdAt` | TIMESTAMP | Date de création | Auto-rempli |

**Relations :**
- `articles` → un utilisateur peut rédiger plusieurs articles (OneToMany)
- `sessions` → un utilisateur peut avoir plusieurs sessions d'exercices (OneToMany)

**Sécurité API :**
- Créer un compte (`POST /api/users`) : **public** (inscription)
- Voir tous les comptes (`GET /api/users`) : **admin uniquement**
- Modifier/supprimer : **admin** ou **l'utilisateur lui-même**

---

### Entité `Article` (table: `articles`)

Articles de bien-être rédigés par les admins.

| Champ | Type SQL | Description | Règles |
|-------|----------|-------------|--------|
| `id` | INT (PK) | Identifiant unique | Auto-généré |
| `titre` | VARCHAR(255) | Titre de l'article | Obligatoire |
| `contenu` | TEXT | Corps de l'article | Obligatoire |
| `auteur_id` | INT (FK) | Référence vers `utilisateurs.id` | Obligatoire |
| `categorie_id` | INT (FK) | Référence vers `categories.id` | Obligatoire |
| `isPublie` | BOOLEAN | Publié ou brouillon | Défaut: `false` |
| `createdAt` | TIMESTAMP | Date de création | Auto-rempli |
| `updatedAt` | TIMESTAMP | Date de modification | Nullable |

> **Important :** Le champ `auteur_id` n'est **jamais envoyé par le frontend**. Le `ArticleStateProcessor` l'injecte automatiquement depuis le token JWT. Cela empêche un utilisateur de créer un article au nom d'un autre.

**Filtrage automatique :** L'extension `ArticlePublieExtension` ajoute `WHERE isPublie = true` à toutes les requêtes, **sauf pour les admins** qui voient aussi les brouillons.

---

### Entité `Categorie` (table: `categories`)

Catégories pour classer les articles.

| Champ | Type SQL | Description | Règles |
|-------|----------|-------------|--------|
| `id` | INT (PK) | Identifiant unique | Auto-généré |
| `nom` | VARCHAR(100) | Nom affiché | Unique, obligatoire |
| `slug` | VARCHAR(100) | Identifiant URL | Unique, format `^[a-z0-9-]+$` |
| `description` | TEXT | Description courte | Nullable |

**Catégories créées par les fixtures :**

| Nom | Slug |
|-----|------|
| Gestion du stress | `gestion-du-stress` |
| Méditation & relaxation | `meditation-relaxation` |
| Activité physique | `activite-physique` |
| Alimentation & bien-être | `alimentation-bien-etre` |
| Sommeil & récupération | `sommeil-recuperation` |

---

### Entité `BreathingExercise` (table: `breathing_exercises`)

Exercices de respiration guidée.

| Champ | Type SQL | Description | Règles |
|-------|----------|-------------|--------|
| `id` | INT (PK) | Identifiant unique | Auto-généré |
| `nom` | VARCHAR(150) | Nom affiché | Obligatoire |
| `slug` | VARCHAR(150) | Identifiant URL | Unique |
| `description` | TEXT | Bienfaits et description | Nullable |
| `inspirationDuration` | INT | Durée inspiration (secondes) | > 0, défaut: 4 |
| `apneaDuration` | INT | Durée apnée/rétention (secondes) | >= 0, défaut: 0 |
| `expirationDuration` | INT | Durée expiration (secondes) | > 0, défaut: 4 |
| `cycles` | INT | Nombre de cycles | > 0, défaut: 5 |
| `isPreset` | BOOLEAN | Exercice fourni par défaut | Défaut: `false` |
| `isActive` | BOOLEAN | Exercice visible | Défaut: `true` |

**Méthodes calculées :**
- `getDureeCycleSecondes()` = inspiration + apnée + expiration
- `getDureeTotaleSecondes()` = durée cycle × cycles

**Exercices preset créés par les fixtures :**

| Nom | Inspiration | Apnée | Expiration | Cycles | Durée totale |
|-----|-------------|-------|------------|--------|--------------|
| Cohérence cardiaque 5-5 | 5s | 0s | 5s | 6 | 60s |
| Relaxation 4-7-8 | 4s | 7s | 8s | 4 | 76s |
| Respiration carrée | 4s | 4s | 4s | 5 | 60s |

---

### Entité `UserSession` (table: `user_sessions`)

Historique des sessions d'exercices effectuées par les utilisateurs.

| Champ | Type SQL | Description | Règles |
|-------|----------|-------------|--------|
| `id` | INT (PK) | Identifiant unique | Auto-généré |
| `user_id` | INT (FK) | Référence vers `utilisateurs.id` | Obligatoire |
| `breathing_exercise_id` | INT (FK) | Exercice effectué | Obligatoire |
| `status` | VARCHAR(20) | État de la session | `started`, `completed`, `abandoned` |
| `startedAt` | TIMESTAMP | Heure de début | Auto-rempli |
| `endedAt` | TIMESTAMP | Heure de fin | Nullable |

**Filtrage automatique :** `UserSessionExtension` limite les résultats à **l'utilisateur connecté**. Un utilisateur ne peut jamais voir les sessions d'un autre. Les admins voient tout.

---

## 4.4 Authentification JWT

### Pourquoi JWT ?
JWT (JSON Web Token) est un système d'authentification **stateless** : le serveur ne stocke pas de session. Le token contient toutes les informations nécessaires, signé cryptographiquement. C'est idéal pour une API consommée par un navigateur **et** une application mobile.

### Flux complet

```
1. POST /api/auth/login  { "email": "...", "password": "..." }
         │
         ▼
2. Symfony vérifie email + mot de passe haché
         │
         ▼
3. LexikJWT génère un token signé (RSA private key)
   Payload: { id, email, prenom, nom, roles, exp }
         │
         ▼
4. Réponse: { "token": "eyJ0eX..." }
         │
         ▼
5. Le frontend stocke le token (localStorage / SecureStore)
         │
         ▼
6. Chaque requête suivante envoie :
   Authorization: Bearer eyJ0eX...
         │
         ▼
7. Nginx/Symfony valide le token (RSA public key)
   Si valide → accès autorisé
   Si expiré → 401 Unauthorized
```

### Configuration (`config/packages/lexik_jwt_authentication.yaml`)

```yaml
lexik_jwt_authentication:
    secret_key: '%kernel.project_dir%/config/jwt/private.pem'
    public_key: '%kernel.project_dir%/config/jwt/public.pem'
    pass_phrase: '%env(JWT_PASSPHRASE)%'
    token_ttl: '%env(int:JWT_TTL)%'   # 900 secondes = 15 minutes
```

### Enrichissement du token (`src/Security/JWTCreatedListener.php`)

Par défaut, le token JWT ne contient que le `username` (email). Le listener ajoute :
```json
{
  "id": 1,
  "email": "user@example.com",
  "prenom": "Jean",
  "nom": "Dupont",
  "roles": ["ROLE_USER"],
  "exp": 1234567890
}
```

Ainsi, le frontend n'a pas besoin d'un appel API supplémentaire pour afficher le nom de l'utilisateur.

### Hashage automatique des mots de passe

`UserHashPasswordSubscriber` écoute les événements Doctrine `prePersist` et `preUpdate`. Quand un `User` est sauvegardé avec un `plainPassword` non nul, il est automatiquement haché et stocké dans `password`. Le champ `plainPassword` n'est jamais persisté en base.

---

## 4.5 API Platform — Endpoints automatiques

API Platform 3 génère automatiquement tous les endpoints CRUD à partir des annotations sur les entités. **Tu n'as pas à écrire les actions CRUD à la main.**

### Tableau complet des endpoints

| Méthode | URL | Accès | Description |
|---------|-----|-------|-------------|
| **Auth** ||||
| POST | `/api/auth/login` | Public | Connexion → retourne un token JWT |
| POST | `/api/auth/register` | Public | Inscription d'un nouvel utilisateur |
| POST | `/api/auth/change-password` | Connecté | Changer son mot de passe |
| POST | `/api/auth/forgot-password` | Public | Envoie un email avec un lien de réinitialisation (valable 1h) |
| POST | `/api/auth/reset-with-token` | Public | Réinitialise le MDP avec le token reçu par email |
| POST | `/api/auth/set-role` | Admin | Définir les rôles d'un utilisateur |
| POST | `/api/auth/reset-password` | Admin | Réinitialiser le MDP d'un utilisateur sans token |
| POST | `/api/auth/desactiver-compte` | Connecté | Désactiver son compte (RGPD — suppression définitive après 30 jours) |
| **Utilisateurs** ||||
| GET | `/api/users` | Admin | Liste tous les utilisateurs |
| GET | `/api/users/{id}` | Admin ou soi-même | Détail d'un utilisateur |
| POST | `/api/users` | Public | Créer un compte (inscription alternative) |
| PATCH | `/api/users/{id}` | Admin ou soi-même | Modifier un utilisateur |
| DELETE | `/api/users/{id}` | Admin | Supprimer un utilisateur |
| **Articles** ||||
| GET | `/api/articles` | Public (publiés seulement) | Liste des articles publiés |
| GET | `/api/articles/{id}` | Public (publiés seulement) | Détail d'un article |
| POST | `/api/articles` | Admin | Créer un article |
| PATCH | `/api/articles/{id}` | Admin ou auteur | Modifier un article |
| DELETE | `/api/articles/{id}` | Admin | Supprimer un article |
| **Catégories** ||||
| GET | `/api/categories` | Public | Liste des catégories |
| GET | `/api/categories/{id}` | Public | Détail d'une catégorie |
| POST | `/api/categories` | Admin | Créer une catégorie |
| PATCH | `/api/categories/{id}` | Admin | Modifier une catégorie |
| DELETE | `/api/categories/{id}` | Admin | Supprimer une catégorie |
| **Exercices de respiration** ||||
| GET | `/api/breathing_exercises` | Public | Liste des exercices |
| GET | `/api/breathing_exercises/{id}` | Public | Détail d'un exercice |
| POST | `/api/breathing_exercises` | Admin | Créer un exercice |
| PATCH | `/api/breathing_exercises/{id}` | Admin | Modifier un exercice |
| DELETE | `/api/breathing_exercises/{id}` | Admin | Supprimer un exercice |
| **Sessions utilisateur** ||||
| GET | `/api/user_sessions` | Connecté | Liste ses propres sessions |
| GET | `/api/user_sessions/{id}` | Connecté (propriétaire) | Détail d'une session |
| POST | `/api/user_sessions` | Connecté | Démarrer une session |
| PATCH | `/api/user_sessions/{id}` | Connecté (propriétaire) | Mettre à jour le statut |
| **Documentation** ||||
| GET | `/api/docs` | Public | Swagger UI (documentation interactive) |

### State Processors

Les State Processors interceptent les opérations POST/PATCH avant l'enregistrement en base.

**`ArticleStateProcessor`** — Pourquoi ? Si le frontend envoyait `auteur_id`, n'importe qui pourrait créer un article au nom d'un autre utilisateur. Le processor injecte automatiquement l'utilisateur connecté :

```php
// Avant sauvegarde : injecter l'auteur depuis le token JWT
if ($data instanceof Article && $data->getAuteur() === null) {
    $data->setAuteur($this->security->getUser());
}
```

**`UserSessionStateProcessor`** — Même principe : injecte automatiquement l'utilisateur connecté comme propriétaire de la session.

---

## 4.6 Migrations Doctrine

Les migrations sont des fichiers PHP qui décrivent les changements de structure de la base de données. Elles s'appliquent dans l'ordre chronologique.

### Migrations existantes

| Fichier | Date | Contenu |
|---------|------|---------|
| `Version20260402135832.php` | 02/04/2026 | Crée les tables `utilisateurs`, `categories`, `articles` |
| `Version20260403075519.php` | 03/04/2026 | Crée les tables `breathing_exercises`, `user_sessions` |

### Commandes utiles

```bash
# Appliquer toutes les migrations en attente
docker exec cesizen_php php bin/console doctrine:migrations:migrate --no-interaction

# Voir le statut des migrations
docker exec cesizen_php php bin/console doctrine:migrations:status

# Créer une nouvelle migration après modification d'une entité
docker exec cesizen_php php bin/console doctrine:migrations:diff

# Annuler la dernière migration (revenir en arrière)
docker exec cesizen_php php bin/console doctrine:migrations:execute --down "App\Migrations\Version..."
```

---

## 4.7 Fixtures (données de démo)

Les fixtures permettent de pré-remplir la base avec des données de test.

```bash
# Charger toutes les fixtures (EFFACE les données existantes)
docker exec cesizen_php php bin/console doctrine:fixtures:load --no-interaction

# Charger uniquement un groupe de fixtures
docker exec cesizen_php php bin/console doctrine:fixtures:load --group=categories
```

**Ce qui est créé :**
- 1 compte administrateur : `admin@cesizen.fr` / `Admin1234!`
- 5 catégories d'articles
- 3 exercices de respiration preset (non supprimables)

---

**Suite →** [05 — Frontend Web React](./05-frontend-web.md)
