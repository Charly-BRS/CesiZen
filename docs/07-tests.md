# 07 — Tests

Le projet dispose de **~72 tests automatisés** répartis sur les 3 projets. Les tests garantissent que le code fonctionne correctement à chaque modification.

---

## Vue d'ensemble

| Projet | Framework | Tests | Ce qui est testé |
|--------|-----------|-------|-----------------|
| `cesizen-api` | PHPUnit (unitaires) | ~26 | Entités, StateProcessors |
| `cesizen-api` | PHPUnit (fonctionnels) | 18 | Endpoints HTTP réels (BDD test) |
| `cesizen-app` | Vitest + Testing Library | 18 | Services, Contexte Auth, Formulaires |
| `cesizen-mobile` | Jest + jest-expo | 10 | Services, Contexte Auth |
| **Total** | | **~72** | |

---

## 7.1 Tests unitaires API — PHPUnit

### Lancer les tests

```bash
# Tous les tests (unitaires + fonctionnels)
docker exec cesizen_php vendor/bin/phpunit

# Uniquement les tests unitaires
docker exec cesizen_php vendor/bin/phpunit tests/Unit/

# Uniquement les tests fonctionnels
docker exec cesizen_php vendor/bin/phpunit tests/Functional/
```

> **Prérequis fonctionnels** : une base de données de test `cesizen_test` doit exister avec les migrations jouées.
> ```bash
> php bin/console doctrine:database:create --env=test
> php bin/console doctrine:migrations:migrate --env=test
> ```

### Structure des tests

```
tests/
├── Unit/
│   ├── Entity/
│   │   ├── UserTest.php                      ← Tests de l'entité User
│   │   ├── ArticleTest.php                   ← Tests de l'entité Article
│   │   └── BreathingExerciseTest.php         ← 10 tests : durées, cycles, calculs
│   └── State/
│       └── UserSessionStateProcessorTest.php ← 7 tests : injection utilisateur auto
└── Functional/
    ├── AuthEndpointTest.php                  ← 7 tests : register, login
    ├── BreathingExerciseEndpointTest.php     ← 5 tests : liste publique, CRUD protégé
    ├── ArticleEndpointTest.php               ← 5 tests : visibilité, droits admin
    └── ResetPasswordEndpointTest.php         ← 9 tests : forgot-password + reset-with-token
```

### Ce qui est testé (exemples)

**`BreathingExerciseTest`** — Vérifie les calculs de durée :
```php
$exercise->setInspirationDuration(4);
$exercise->setApneaDuration(7);
$exercise->setExpirationDuration(8);
$this->assertEquals(19, $exercise->getDureeCycleSecondes());
$exercise->setCycles(4);
$this->assertEquals(76, $exercise->getDureeTotaleSecondes());
```

**`ArticleEndpointTest`** — Vérifie que les brouillons sont invisibles :
```php
// L'admin crée un article non publié → 201
// L'admin peut le lire → 200
// Un user normal ne peut pas le lire → 404 (filtré par ArticlePublieExtension)
$this->assertSame(404, $client->getResponse()->getStatusCode());
```

---

## 7.2 Tests unitaires Frontend Web — Vitest

### Lancer les tests

```bash
cd cesizen-app

# Lancer tous les tests
npm test

# Mode watch (relance automatiquement lors des modifications)
npm run test:watch
```

### Structure des tests

```
src/test/
├── setup.ts               ← Configuration globale (jest-dom matchers)
├── authService.test.ts    ← Tests du service d'authentification
├── AuthContext.test.tsx   ← Tests du contexte global d'auth
└── LoginPage.test.tsx     ← Tests du composant LoginPage
```

### Mocks utilisés

Les tests mockent le module `api.ts` (Axios) pour ne jamais faire de vraies requêtes HTTP :

```ts
vi.mock('../services/api', () => ({
  default: { post: vi.fn(), get: vi.fn(), ... }
}));

// Configurer la réponse simulée
vi.mocked(apiClient.post).mockResolvedValueOnce({ data: { token: 'fake-token' } });
```

### Configuration Vitest (`vite.config.ts`)

```ts
test: {
  globals: true,          // describe/it/expect disponibles sans import
  environment: 'jsdom',   // simule un navigateur (window, document…)
  setupFiles: ['./src/test/setup.ts'],
}
```

---

## 7.3 Tests unitaires Mobile — Jest + jest-expo

### Lancer les tests

```bash
cd cesizen-mobile

# Lancer tous les tests
npm test
```

### Structure des tests

```
src/__tests__/
├── authService.test.ts    ← Tests de seConnecter() et sInscrire()
└── AuthContext.test.tsx   ← Tests du contexte global d'auth mobile
```

### Mocks utilisés

Les tests mockent les modules qui dépendent de modules natifs React Native :

```ts
// Mock SecureStore (non disponible en environnement Node.js)
jest.mock('../services/storage', () => ({
  recupererToken: jest.fn().mockResolvedValue(null),
  sauvegarderToken: jest.fn().mockResolvedValue(undefined),
  // ...
}));

// Mock de l'instance Axios
jest.mock('../services/api', () => ({
  default: { post: jest.fn(), ... },
  definirToken: jest.fn(),
  definirCallbackDeconnexion: jest.fn(),
}));
```

### Configuration Jest (`package.json`)

```json
"jest": {
  "preset": "jest-expo",
  "transformIgnorePatterns": [
    "node_modules/(?!((jest-)?react-native|@react-native(-community)?)|expo(nent)?|...)"
  ]
}
```

> Le `transformIgnorePatterns` est obligatoire avec jest-expo : beaucoup de packages React Native/Expo sont distribués en ESM et doivent être transformés par Babel avant d'être exécutés dans Node.js.

---

## 7.4 Ajouter un nouveau test

### PHPUnit unitaire (entité)

```php
<?php
// tests/Unit/Entity/CategorieTest.php
namespace App\Tests\Unit\Entity;
use App\Entity\Categorie;
use PHPUnit\Framework\TestCase;

class CategorieTest extends TestCase
{
    public function testSlugEstEnregistre(): void
    {
        $categorie = new Categorie();
        $categorie->setSlug('bien-etre');
        $this->assertEquals('bien-etre', $categorie->getSlug());
    }
}
```

### PHPUnit fonctionnel (endpoint HTTP)

```php
// tests/Functional/CategorieEndpointTest.php
public function testListerCategoriesRetourne200(): void
{
    $client = static::createClient();
    $client->request('GET', '/api/categories', [], [],
        ['HTTP_ACCEPT' => 'application/ld+json']
    );
    $this->assertSame(200, $client->getResponse()->getStatusCode());
}
```

### Vitest (service web)

```ts
// src/test/articleService.test.ts
import { describe, it, expect, vi } from 'vitest';
import { getArticles } from '../services/articleService';
import apiClient from '../services/api';

vi.mock('../services/api', () => ({ default: { get: vi.fn(), ... } }));

it('getArticles() retourne la liste des articles', async () => {
  vi.mocked(apiClient.get).mockResolvedValueOnce({ data: { 'hydra:member': [] } });
  const resultat = await getArticles();
  expect(apiClient.get).toHaveBeenCalledWith('/articles');
});
```

---

**Suite →** [08 — CI/CD GitHub Actions](./08-cicd.md)
