# 10 — Cahier de tests CESIZen

Ce document recense l'ensemble des tests du projet : stratégie, tests automatisés et scénarios manuels.

---

## 10.1 Stratégie de tests

Le projet utilise **3 niveaux de tests** complémentaires :

| Niveau | Outil | Portée | Quand |
|---|---|---|---|
| **Unitaire** | PHPUnit | Entités + StateProcessors PHP | À chaque commit |
| **Fonctionnel** | PHPUnit WebTestCase | Endpoints HTTP réels (BDD test) | À chaque commit |
| **Unitaire JS** | Vitest (web) + Jest (mobile) | Services + Contextes + Composants | À chaque commit |
| **Manuel** | Navigateur / Expo Go | Parcours utilisateur complets | Avant livraison |

---

## 10.2 Tests unitaires PHPUnit — API

**Commande :** `vendor/bin/phpunit tests/Unit/`

| Fichier | Nb tests | Ce qui est vérifié |
|---|---|---|
| `tests/Unit/Entity/UserTest.php` | ~5 | Constructeur, setters/getters, hashage MDP |
| `tests/Unit/Entity/ArticleTest.php` | ~4 | Setters, isPublie par défaut |
| `tests/Unit/Entity/BreathingExerciseTest.php` | 10 | `getDureeCycleSecondes()`, `getDureeTotaleSecondes()`, isPreset/isActive |
| `tests/Unit/State/UserSessionStateProcessorTest.php` | 7 | Injection auto de l'utilisateur connecté dans la session |

---

## 10.3 Tests fonctionnels PHPUnit — API

**Commande :** `vendor/bin/phpunit tests/Functional/`

Ces tests envoient de vraies requêtes HTTP via un client Symfony simulé et touchent une vraie base de données (`cesizen_test`).

| Fichier | Endpoint | Condition | Résultat attendu |
|---|---|---|---|
| `AuthEndpointTest` | POST `/api/auth/register` | Données valides | 201 Created + email dans réponse |
| `AuthEndpointTest` | POST `/api/auth/register` | Email déjà utilisé | 400+ (erreur) |
| `AuthEndpointTest` | POST `/api/auth/register` | Email invalide | 422 Unprocessable |
| `AuthEndpointTest` | POST `/api/auth/register` | Prénom manquant | 422 Unprocessable |
| `AuthEndpointTest` | POST `/api/auth/login` | Identifiants valides | 200 + champ `token` |
| `AuthEndpointTest` | POST `/api/auth/login` | Mauvais mot de passe | 401 Unauthorized |
| `AuthEndpointTest` | POST `/api/auth/login` | Email inconnu | 401 Unauthorized |
| `BreathingExerciseEndpointTest` | GET `/api/breathing_exercises` | Sans token | 200 (liste publique) |
| `BreathingExerciseEndpointTest` | GET `/api/breathing_exercises/99999` | Sans token | 404 Not Found |
| `BreathingExerciseEndpointTest` | POST `/api/breathing_exercises` | Token admin | 201 Created |
| `BreathingExerciseEndpointTest` | POST `/api/breathing_exercises` | Token user | 403 Forbidden |
| `BreathingExerciseEndpointTest` | POST `/api/breathing_exercises` | Sans token | 401 Unauthorized |
| `ArticleEndpointTest` | GET `/api/articles` | Sans token | 401 Unauthorized |
| `ArticleEndpointTest` | GET `/api/articles` | Token user | 200 + `hydra:member` |
| `ArticleEndpointTest` | GET `/api/articles/{id}` | Brouillon vu par admin | 200 |
| `ArticleEndpointTest` | GET `/api/articles/{id}` | Brouillon vu par user | 404 (filtré) |
| `ArticleEndpointTest` | POST `/api/articles` | Token admin | 201 Created |
| `ArticleEndpointTest` | POST `/api/articles` | Token user | 403 Forbidden |

**Total : 18 tests fonctionnels**

---

## 10.4 Tests unitaires JavaScript — Application Web (Vitest)

**Commande :** `npm test` dans `cesizen-app/`

| Fichier | Ce qui est testé | Nb assertions clés |
|---|---|---|
| `authService.test.ts` | `login()` retourne le token JWT | Appel API, retour token |
| `authService.test.ts` | `login()` propage l'erreur 401 | rejects.toThrow |
| `authService.test.ts` | `register()` retourne utilisateur et message | Appel API, structure réponse |
| `authService.test.ts` | `register()` propage l'erreur 422 | rejects.toThrow |
| `AuthContext.test.tsx` | `estConnecte` = false sans token | État initial |
| `AuthContext.test.tsx` | `connecter()` met estConnecte à true | State + localStorage |
| `AuthContext.test.tsx` | `connecter()` sauvegarde les rôles | roles dans utilisateur |
| `AuthContext.test.tsx` | `deconnecter()` remet à zéro | State + localStorage effacé |
| `AuthContext.test.tsx` | `mettreAJourUtilisateur()` modifie partiellement | nom mis à jour, prénom intact |
| `AuthContext.test.tsx` | `mettreAJourUtilisateur()` persiste en localStorage | localStorage.getItem |
| `LoginPage.test.tsx` | Champ email affiché | getByLabelText('Email') |
| `LoginPage.test.tsx` | Champ mot de passe affiché | getByLabelText('Mot de passe') |
| `LoginPage.test.tsx` | Bouton "Se connecter" affiché | getByRole('button') |
| `LoginPage.test.tsx` | Titre CESIZen affiché | getByText(/CESIZen/) |
| `LoginPage.test.tsx` | Lien "Créer un compte" affiché | getByText(/Créer un compte/) |
| `LoginPage.test.tsx` | Erreur 401 → message explicite | waitFor, getByText |
| `LoginPage.test.tsx` | Erreur 500 → message générique | waitFor, getByText |
| `LoginPage.test.tsx` | Bouton désactivé pendant chargement | toBeDisabled |

**Total : 18 tests Vitest**

---

## 10.5 Tests unitaires JavaScript — Application Mobile (Jest + jest-expo)

**Commande :** `npm test` dans `cesizen-mobile/`

| Fichier | Ce qui est testé | Nb assertions clés |
|---|---|---|
| `authService.test.ts` | `seConnecter()` décode le JWT et retourne utilisateur | token, email, rôles |
| `authService.test.ts` | `seConnecter()` propage l'erreur | rejects.toThrow |
| `authService.test.ts` | `sInscrire()` appelle /auth/register | toHaveBeenCalledWith |
| `authService.test.ts` | `sInscrire()` propage l'erreur | rejects.toThrow |
| `AuthContext.test.tsx` | `estConnecte` = false au démarrage | State initial |
| `AuthContext.test.tsx` | `chargement` = false après vérification session | chargement = false |
| `AuthContext.test.tsx` | `connecter()` met estConnecte à true | State, token, utilisateur |
| `AuthContext.test.tsx` | `connecter()` sauvegarde les rôles | roles ROLE_ADMIN |
| `AuthContext.test.tsx` | `deconnecter()` remet à zéro | State token = null |
| `AuthContext.test.tsx` | `mettreAJourUtilisateur()` modifie partiellement | nom mis à jour, prénom intact |

**Total : 10 tests Jest**

---

## 10.6 Récapitulatif général

| Projet | Outil | Tests | Statut |
|---|---|---|---|
| `cesizen-api` | PHPUnit (unitaires) | ~26 | ✅ |
| `cesizen-api` | PHPUnit (fonctionnels) | 18 | ✅ |
| `cesizen-app` | Vitest | 18 | ✅ |
| `cesizen-mobile` | Jest + jest-expo | 10 | ✅ |
| **Total** | | **~72** | **✅** |

---

## 10.7 Scénarios de tests manuels

Ces tests sont effectués manuellement dans le navigateur et sur l'application mobile avant la présentation jury.

### Inscription et connexion

| # | Scénario | Application | Résultat attendu | Testé le | Statut |
|---|---|---|---|---|---|
| 1 | Inscription avec données valides | Web | Compte créé, redirection login | | ☐ |
| 2 | Inscription email déjà utilisé | Web | Message d'erreur explicite | | ☐ |
| 3 | Connexion avec identifiants valides | Web | Token JWT stocké, redirection dashboard | | ☐ |
| 4 | Connexion avec mauvais mot de passe | Web | Message "Email ou mot de passe incorrect" | | ☐ |
| 5 | Inscription avec données valides | Mobile | Compte créé, redirection connexion | | ☐ |
| 6 | Connexion valide | Mobile | Accès aux onglets, token en SecureStore | | ☐ |

### Navigation et contenu

| # | Scénario | Application | Résultat attendu | Testé le | Statut |
|---|---|---|---|---|---|
| 7 | Accès au dashboard sans être connecté | Web | Redirection vers /login | | ☐ |
| 8 | Accès /admin sans ROLE_ADMIN | Web | Redirection (403 ou login) | | ☐ |
| 9 | Lecture liste des articles | Web + Mobile | Articles publiés visibles, brouillons masqués | | ☐ |
| 10 | Lecture liste des exercices | Web + Mobile | Exercices actifs visibles | | ☐ |
| 11 | Lancer un exercice de respiration | Web + Mobile | Animation de respiration qui se lance | | ☐ |
| 12 | Session terminée → visible dans historique | Web + Mobile | Session enregistrée et affichée | | ☐ |

### Administration

| # | Scénario | Application | Résultat attendu | Testé le | Statut |
|---|---|---|---|---|---|
| 13 | Créer un article (brouillon) | Web admin | Article créé avec isPublie=false, non visible aux users | | ☐ |
| 14 | Publier un article | Web admin | isPublie=true, visible aux users | | ☐ |
| 15 | Désactiver un exercice | Web admin + Mobile admin | isActive=false, exercice non visible aux users | | ☐ |
| 16 | Réactiver un exercice | Mobile admin | isActive=true, exercice visible à nouveau | | ☐ |
| 17 | Supprimer un exercice sans session | Web admin | Exercice supprimé définitivement | | ☐ |
| 18 | Supprimer un exercice avec sessions | Web admin | Exercice désactivé (pas supprimé), message 422 | | ☐ |
| 19 | Désactiver un compte utilisateur | Web admin | L'utilisateur ne peut plus se connecter | | ☐ |

### Profil

| # | Scénario | Application | Résultat attendu | Testé le | Statut |
|---|---|---|---|---|---|
| 20 | Modifier prénom/nom | Web + Mobile | Changements sauvegardés, affichés immédiatement | | ☐ |
| 21 | Changer mot de passe | Web + Mobile | Nouveau MDP valide, ancienMDP requis | | ☐ |
| 22 | Déconnexion | Web + Mobile | Token supprimé, redirection login | | ☐ |

---

## 10.8 Résultats des tests automatisés (à remplir avant jury)

| Commande | Résultat | Date |
|---|---|---|
| `vendor/bin/phpunit` (api) | __ / __ tests passés | |
| `npm test` (cesizen-app) | 18 / 18 tests passés | |
| `npm test` (cesizen-mobile) | 10 / 10 tests passés | |
| GitHub Actions test.yml | ✅ / ❌ | |
