# 07 — Tests

Le projet dispose de **161 tests automatisés** répartis sur les 3 projets. Les tests garantissent que le code fonctionne correctement à chaque modification.

---

## Vue d'ensemble

| Projet | Framework | Tests | Ce qui est testé |
|--------|-----------|-------|-----------------|
| `cesizen-api` | PHPUnit + PHPStan | 47 | Entités, controllers, state processors |
| `cesizen-mobile` | Jest | 114 | Services, context, écrans |
| `cesizen-app` | ESLint + TypeScript | — | Qualité et typage du code |
| **Total** | | **161** | |

---

## 7.1 Tests API — PHPUnit

### Lancer les tests

```bash
# Lancer tous les tests PHPUnit
docker exec cesizen_php vendor/bin/phpunit

# Lancer un fichier de test spécifique
docker exec cesizen_php vendor/bin/phpunit tests/Entity/BreathingExerciseTest.php

# Lancer avec rapport de couverture HTML
docker exec cesizen_php vendor/bin/phpunit --coverage-html=var/coverage/html

# Lancer avec rapport de couverture XML (pour CI/CD)
docker exec cesizen_php vendor/bin/phpunit --coverage-clover=var/coverage/clover.xml
```

> **Prérequis** : les tests utilisent une base de données de test séparée (`cesizen_test`). Assure-toi que la variable `DATABASE_URL` dans `.env.test` pointe vers cette base.

### Structure des tests

```
tests/
├── bootstrap.php                         ← Charge .env.test et vendor/autoload.php
├── Controller/
│   └── AuthControllerTest.php            ← Tests des endpoints d'authentification
├── Entity/
│   ├── UserTest.php                      ← Tests de l'entité User
│   ├── ArticleTest.php                   ← Tests de l'entité Article
│   └── BreathingExerciseTest.php         ← 10 tests : durations, cycles, calculs
└── State/
    └── UserSessionStateProcessorTest.php ← 7 tests : injection utilisateur auto
```

### Ce qui est testé (exemples)

**`BreathingExerciseTest`** — Vérifie les calculs de durée :
```php
// Vérifie que getDureeCycleSecondes() = inspiration + apnée + expiration
$exercise->setInspirationDuration(4);
$exercise->setApneaDuration(7);
$exercise->setExpirationDuration(8);
$this->assertEquals(19, $exercise->getDureeCycleSecondes());

// Vérifie que getDureeTotaleSecondes() = durée cycle * cycles
$exercise->setCycles(4);
$this->assertEquals(76, $exercise->getDureeTotaleSecondes());
```

**`UserSessionStateProcessorTest`** — Vérifie que l'utilisateur connecté est injecté automatiquement :
```php
// Crée une session sans utilisateur
$session = new UserSession();
// Le processor doit injecter l'utilisateur connecté
$processor->process($session, $operation);
$this->assertSame($utilisateurMock, $session->getUser());
```

### Configuration PHPUnit (`phpunit.dist.xml`)

```xml
<phpunit
    colors="true"
    failOnDeprecation="true"
    bootstrap="tests/bootstrap.php"
>
    <php>
        <server name="APP_ENV" value="test" force="true" />
    </php>
    <testsuites>
        <testsuite name="Project Test Suite">
            <directory>tests</directory>
        </testsuite>
    </testsuites>
</phpunit>
```

---

## 7.2 Analyse statique — PHPStan

PHPStan analyse le code PHP **sans l'exécuter** pour détecter les bugs potentiels (variables mal typées, méthodes inexistantes, etc.).

### Lancer PHPStan

```bash
# Analyse avec la configuration par défaut (niveau 6)
docker exec cesizen_php vendor/bin/phpstan analyse src

# Affiche les erreurs formatées pour GitHub Actions
docker exec cesizen_php vendor/bin/phpstan analyse src --error-format=github

# Sans barre de progression (plus rapide en CI)
docker exec cesizen_php vendor/bin/phpstan analyse src --no-progress
```

### Niveaux PHPStan (0 = permissif → 9 = très strict)

Le projet est configuré au **niveau 6** : bon équilibre entre rigueur et praticité pour un projet étudiant.

### Configuration (`phpstan.neon`)

```neon
parameters:
    level: 6
    paths:
        - src
    phpVersion: 80400   # PHP 8.4
```

---

## 7.3 Tests Mobile — Jest

### Lancer les tests

```bash
cd cesizen-mobile

# Lancer tous les tests
npm test

# Mode watch (relance automatiquement lors des modifications)
npm run test:watch

# Avec rapport de couverture
npm run test:coverage
```

### Structure des tests

```
src/
├── context/
│   └── __tests__/
│       └── AuthContext.test.ts       ← Tests du contexte d'authentification
├── screens/
│   └── __tests__/                    ← Tests des écrans principaux
└── services/
    └── __tests__/
        ├── api.test.ts               ← Tests de l'instance Axios
        ├── authService.test.ts       ← Tests du service d'authentification
        ├── storage.test.ts           ← Tests du stockage SecureStore
        └── ...
```

### Mocks utilisés

Les tests mobiles utilisent des **mocks** pour simuler les modules natifs (non disponibles dans l'environnement de test Node.js) :

| Mock | Simule | Fichier |
|------|--------|---------|
| `expo-secure-store` | Stockage chiffré mobile | `jest.mocks/expo-secure-store.js` |
| `AsyncStorage` | Stockage async React Native | `jest.mocks/react-native-async-storage.js` |

### Configuration Jest (`jest.config.js`)

```js
module.exports = {
    preset: 'jest-expo',
    testEnvironment: 'node',
    transform: { '^.+\\.tsx?$': 'ts-jest' },
    moduleNameMapper: {
        '^@/(.*)$': '<rootDir>/src/$1'  // Alias @ → src/
    },
    coverageThreshold: {
        global: {
            branches: 1,
            functions: 10,
            lines: 8,
        }
    }
};
```

---

## 7.4 Qualité Frontend — ESLint + TypeScript

Le frontend web n'a pas de tests unitaires, mais est vérifié par :

### ESLint

Vérifie les règles de qualité de code (variables inutilisées, hooks React mal utilisés, etc.).

```bash
cd cesizen-app

# Linter tout le code
npm run lint

# Afficher les erreurs avec détails
npx eslint src/ --format=verbose
```

### TypeScript

Vérifie que tous les types sont corrects sans compiler les fichiers.

```bash
cd cesizen-app

# Vérification des types (sans générer de fichiers)
npx tsc -b --noEmit
```

---

## 7.5 Ajouter un nouveau test

### PHPUnit (API)

```bash
# Créer un fichier de test
touch tests/Entity/CategorieTest.php
```

```php
<?php
namespace App\Tests\Entity;

use App\Entity\Categorie;
use PHPUnit\Framework\TestCase;

class CategorieTest extends TestCase
{
    public function testSlugEstUnique(): void
    {
        $categorie = new Categorie();
        $categorie->setSlug('bien-etre');
        $this->assertEquals('bien-etre', $categorie->getSlug());
    }
}
```

### Jest (Mobile)

```ts
// src/services/__tests__/monService.test.ts
import { maFonction } from '../monService';

describe('monService', () => {
    it('devrait retourner la bonne valeur', () => {
        const result = maFonction('test');
        expect(result).toBe('valeur attendue');
    });
});
```

---

**Suite →** [08 — CI/CD GitHub Actions](./08-cicd.md)
