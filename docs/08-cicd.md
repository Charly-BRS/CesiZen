# 08 — CI/CD GitHub Actions

Le CI/CD (Continuous Integration / Continuous Deployment) automatise les tests et le déploiement à chaque push de code. Chaque repo possède ses propres workflows.

---

## Architecture

```
cesizen-api/
└── .github/workflows/
    ├── test.yml              ← Déclenché sur push/PR → lance PHPUnit + PHPStan
    └── build-and-deploy.yml  ← Déclenché sur push main → build Docker + déploiement

cesizen-app/
└── .github/workflows/
    ├── test.yml              ← Déclenché sur push/PR → lance ESLint + TypeScript
    └── build-and-deploy.yml  ← Déclenché sur push main → build Vite + déploiement

cesizen-mobile/
└── .github/workflows/
    ├── test.yml              ← Déclenché sur push/PR → lance Jest + TypeScript
    └── build-and-deploy.yml  ← Déclenché sur push main → validation + déploiement Expo
```

---

## Workflow `test.yml` — Tests automatiques

### Ce qui se passe à chaque push ou Pull Request

```
Push vers main ou develop
           │
           ▼
    GitHub Actions démarre
           │
    ┌──────┴────────────────┐
    │                       │
    ▼                       ▼
[API] PHPUnit         [Mobile] Jest
      PHPStan         TypeScript check
           │                │
           └───────┬────────┘
                   ▼
           Succès ou Échec
           (visible dans l'onglet Actions)
```

### `cesizen-api` — test.yml

```yaml
jobs:
  api-tests:
    runs-on: ubuntu-latest
    services:
      postgres:              # Base de données de test temporaire
        image: postgres:16-alpine
    steps:
      - uses: actions/checkout@v6
      - uses: shivammathur/setup-php@v2    # Installe PHP 8.4
        with:
          php-version: '8.4'
          extensions: pgsql, pdo_pgsql
          coverage: xdebug
      - run: composer install
      - run: php bin/console doctrine:migrations:migrate --env=test
      - run: vendor/bin/phpunit            # Lance les tests
      - run: vendor/bin/phpstan analyse src  # Analyse statique
```

### `cesizen-mobile` — test.yml

```yaml
jobs:
  mobile-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-node@v6
        with:
          node-version: '22'
          cache: 'npm'
      - run: npm ci
      - run: npm run type-check    # Vérification TypeScript
      - run: npm test -- --coverage --passWithNoTests   # Jest
```

### `cesizen-app` — test.yml

```yaml
jobs:
  app-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-node@v6
        with:
          node-version: '22'
      - run: npm ci
      - run: npm run lint          # ESLint
      - run: npx tsc -b --noEmit  # TypeScript
```

---

## Workflow `build-and-deploy.yml` — Build et déploiement

### Ce qui se passe sur chaque push vers `main`

```
Push vers main
      │
      ▼
Build (Docker image / Vite / Expo)
      │
      ▼
Push image vers GHCR (ghcr.io/charly-brs/cesizen-api)
      │
      ▼
Déploiement staging
(commande à compléter selon ton hébergement)
```

### Build de l'API — Image Docker vers GHCR

```yaml
- name: Construire et pousser l'image Docker
  uses: docker/build-push-action@v6
  with:
    context: .
    file: docker/php/Dockerfile
    push: true
    tags: ghcr.io/charly-brs/cesizen-api:latest
```

L'image est disponible sur : `ghcr.io/charly-brs/cesizen-api`

---

## Secrets GitHub à configurer

Les secrets sont des variables d'environnement chiffrées, configurées dans **Settings → Secrets and variables → Actions** de chaque repo GitHub.

| Secret | Repo | Obligatoire | Rôle |
|--------|------|-------------|------|
| `GITHUB_TOKEN` | Tous | Automatique | Auth pour pousser vers GHCR (fourni par GitHub) |
| `CODECOV_TOKEN` | Tous | Optionnel | Upload des rapports de couverture vers Codecov |
| `EXPO_TOKEN` | cesizen-mobile | Optionnel | Déployer avec EAS Update (commenté par défaut) |
| `NETLIFY_AUTH_TOKEN` | cesizen-app | Optionnel | Déployer sur Netlify (commenté par défaut) |
| `NETLIFY_SITE_ID` | cesizen-app | Optionnel | ID du site Netlify |

> **`GITHUB_TOKEN`** est créé automatiquement par GitHub pour chaque workflow. Tu n'as pas besoin de le configurer.

---

## Lire les résultats dans GitHub

1. Va sur le repo GitHub (`github.com/Charly-BRS/cesizen-api`)
2. Clique sur l'onglet **Actions**
3. Chaque ligne = un workflow run (vert ✅ = succès, rouge ❌ = échec)
4. Clique sur un run pour voir le détail de chaque étape
5. En cas d'erreur : clique sur l'étape en rouge pour voir les logs

### Interpréter les erreurs courantes

| Erreur | Cause probable | Solution |
|--------|----------------|----------|
| `composer install failed` | `composer.lock` pas à jour | Lancer `composer install` en local |
| `doctrine:migrations:migrate failed` | Migration manquante ou cassée | Vérifier les migrations |
| `phpunit: X failures` | Tests qui échouent | Lire les messages d'erreur dans les logs |
| `phpstan: X errors` | Erreurs de typage PHP | Corriger les types dans le code |
| `npm ci failed` | `package-lock.json` pas à jour | Lancer `npm install` en local |
| `tsc: type errors` | Erreurs TypeScript | Corriger les types dans le code |

---

## Gitflow — Convention de branches

Le projet suit la convention **Gitflow** :

| Branche | Rôle |
|---------|------|
| `main` | Code en production (déclenche le build/deploy) |
| `develop` | Intégration des nouvelles fonctionnalités |
| `feature/nom-feature` | Développement d'une fonctionnalité |
| `fix/nom-bug` | Correction d'un bug |
| `test/nom-test` | Branches de tests CI/CD |

**Workflow typique :**
```bash
# Créer une branche feature depuis develop
git checkout develop
git checkout -b feature/ma-fonctionnalite

# Développer, committer
git add .
git commit -m "feat: ajouter ma fonctionnalité"
git push origin feature/ma-fonctionnalite

# Créer une Pull Request develop ← feature/ma-fonctionnalite
# → Tests automatiques se lancent
# → Merger après validation
```

---

**Suite →** [09 — Référence rapide](./09-reference.md)
