# 08 — CI/CD GitHub Actions

Le CI/CD automatise les tests et le build à chaque push de code. Les workflows sont centralisés dans le **repo racine** (monorepo `.github/workflows/`).

---

## Architecture

```
CesiZen-Test/  (repo racine)
└── .github/workflows/
    ├── test.yml              ← Tests automatiques (push + PR)
    └── build-and-deploy.yml  ← Build de production (push main)
```

> Les 3 projets (api, app, mobile) sont testés dans un seul workflow grâce aux `working-directory`.

---

## Workflow `test.yml` — Tests automatiques

### Déclenchement

- Sur chaque `push` vers `main` ou `develop`
- Sur chaque `pull_request` vers `main` ou `develop`

### 3 jobs en parallèle

```
Push / PR
    │
    ├── job: api ────────────────────────────────────
    │   PHP 8.4 + PostgreSQL service
    │   composer install → migrations → phpunit
    │
    ├── job: app ────────────────────────────────────
    │   Node 22
    │   npm ci → eslint → tsc → vitest run
    │
    └── job: mobile ─────────────────────────────────
        Node 22
        npm ci → tsc --noEmit → jest
```

### Job `api` — PHPUnit + BDD test

```yaml
api:
  runs-on: ubuntu-latest
  services:
    postgres:
      image: postgres:16-alpine
      env:
        POSTGRES_DB: cesizen_test
        POSTGRES_USER: cesizen_user
        POSTGRES_PASSWORD: cesizen_password
      # ⚠️ Utiliser 127.0.0.1 (pas localhost) pour les health-checks Docker
      options: >-
        --health-cmd pg_isready
        --health-interval 10s
  steps:
    - uses: actions/checkout@v4
    - uses: shivammathur/setup-php@v2
      with:
        php-version: '8.4'
        extensions: pdo_pgsql
    - run: composer install
      working-directory: cesizen-api
    - run: php bin/console doctrine:migrations:migrate --env=test --no-interaction
      working-directory: cesizen-api
      env:
        DATABASE_URL: postgresql://cesizen_user:cesizen_password@127.0.0.1:5432/cesizen_test
    - run: vendor/bin/phpunit
      working-directory: cesizen-api
```

### Job `app` — ESLint + TypeScript + Vitest

```yaml
app:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with:
        node-version: '22'
        cache: 'npm'
        cache-dependency-path: cesizen-app/package-lock.json
    - run: npm ci
      working-directory: cesizen-app
    - run: npm run lint
      working-directory: cesizen-app
    - run: npx tsc -b --noEmit
      working-directory: cesizen-app
    - run: npm test
      working-directory: cesizen-app
```

### Job `mobile` — TypeScript + Jest

```yaml
mobile:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with:
        node-version: '22'
        cache: 'npm'
        cache-dependency-path: cesizen-mobile/package-lock.json
    - run: npm ci --legacy-peer-deps
      working-directory: cesizen-mobile
    - run: npm run type-check
      working-directory: cesizen-mobile
    - run: npm test
      working-directory: cesizen-mobile
```

---

## Workflow `build-and-deploy.yml` — Build production

### Déclenchement

- Sur chaque `push` vers `main` uniquement

### 2 jobs

```
Push main
    │
    ├── job: build-app ──────────────────────────────
    │   Node 22
    │   npm ci → npm run build → artifact cesizen-app/dist
    │
    └── job: build-api ──────────────────────────────
        Docker
        docker build cesizen-api/docker/php/Dockerfile
```

### Job `build-app` — Build Vite + artifact

```yaml
build-app:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with:
        node-version: '22'
    - run: npm ci
      working-directory: cesizen-app
    - run: npm run build
      working-directory: cesizen-app
      env:
        VITE_API_URL: http://localhost:8080/api
    - uses: actions/upload-artifact@v4
      with:
        name: cesizen-app-dist
        path: cesizen-app/dist/
        retention-days: 7
```

L'artifact `cesizen-app-dist` est disponible dans l'onglet **Actions** → run → **Artifacts** pendant 7 jours.

### Job `build-api` — Build Docker image

```yaml
build-api:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    - name: Build de l'image Docker PHP-FPM
      run: docker build -t cesizen-api:latest -f docker/php/Dockerfile .
      working-directory: cesizen-api
```

> L'image est construite localement dans le runner CI pour vérifier que le Dockerfile est valide. Elle n'est pas poussée vers un registre (pas de déploiement automatique pour ce projet étudiant).

---

## Lire les résultats dans GitHub

1. Va sur le repo GitHub (`github.com/Charly-BRS/cesizen`)
2. Clique sur l'onglet **Actions**
3. Chaque ligne = un workflow run (vert ✅ = succès, rouge ❌ = échec)
4. Clique sur un run pour voir les 3 jobs
5. En cas d'erreur : clique sur l'étape en rouge pour voir les logs

### Interpréter les erreurs courantes

| Erreur | Cause probable | Solution |
|--------|----------------|----------|
| `composer install failed` | `composer.lock` pas à jour | Lancer `composer install` en local |
| `doctrine:migrations:migrate failed` | Migration cassée ou BDD inaccessible | Vérifier `DATABASE_URL` dans le job |
| `phpunit: X failures` | Tests qui échouent | Lire les messages dans les logs |
| `npm ci failed` | `package-lock.json` pas à jour | Lancer `npm install` en local |
| `tsc: type errors` | Erreurs TypeScript | Corriger les types dans le code |
| `vitest: X failures` | Tests Vitest qui échouent | Lire les messages dans les logs |

---

## Gitflow — Convention de branches

| Branche | Rôle |
|---------|------|
| `main` | Code stable (déclenche le build) |
| `develop` | Intégration des nouvelles fonctionnalités |
| `feature/nom` | Développement d'une fonctionnalité |
| `fix/nom` | Correction d'un bug |

```bash
# Workflow typique
git checkout develop
git checkout -b feature/ma-fonctionnalite
# ... développer, committer ...
git push origin feature/ma-fonctionnalite
# Créer une PR develop ← feature → tests se lancent automatiquement
```

---

**Suite →** [09 — Référence rapide](./09-reference.md)
