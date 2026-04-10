# 06 — Application Mobile (React Native + Expo 54)

L'application mobile est construite avec **React Native** via **Expo**, ce qui permet de développer pour iOS et Android avec le même code JavaScript/TypeScript.

---

## 6.1 Installation

```bash
cd cesizen-mobile

# Copier les variables d'environnement
cp .env.example .env

# Installer les dépendances
npm install

# Lancer Expo
npx expo start
```

Expo affiche un **QR code** dans le terminal. Scanne-le avec l'application **Expo Go** sur ton téléphone pour voir l'app en temps réel.

### Variable d'environnement (`.env`)

```env
# ⚠️ IMPORTANT : utilise l'IP locale de ta machine, pas "localhost"
# "localhost" sur un téléphone pointe vers le téléphone lui-même, pas ton PC
EXPO_PUBLIC_API_URL=http://192.168.1.XX:8080/api
```

> **Comment trouver ton IP locale ?**
> - Windows : `ipconfig` → "Adresse IPv4"
> - Mac/Linux : `ifconfig` → `inet` (interface Wi-Fi)

Le préfixe `EXPO_PUBLIC_` est obligatoire pour qu'Expo expose la variable au code JavaScript.

> **Timeout réseau** : l'API Docker en développement peut être lente (premier démarrage, machine peu puissante). Le timeout Axios est fixé à **30 secondes** dans `src/services/api.ts`. Si tu vois des erreurs `ECONNABORTED`, vérifie d'abord que Docker est démarré et que les containers sont bien en route (`docker compose up -d`).

---

## 6.2 Structure `src/`

```
cesizen-mobile/
├── App.tsx                         ← Point d'entrée (AuthProvider + AppNavigator)
├── index.ts                        ← Entrée Expo
├── app.json                        ← Configuration Expo (nom, icône, plugins)
│
└── src/
    ├── context/
    │   └── AuthContext.tsx         ← Auth avec SecureStore (persistance chiffrée)
    │
    ├── navigation/
    │   ├── AppNavigator.tsx        ← Routeur principal (Auth ou Onglets)
    │   ├── NavigateurOnglets.tsx   ← Bottom Tabs (4 onglets)
    │   ├── NavigateurArticles.tsx  ← Stack Articles (liste → détail)
    │   ├── NavigateurExercices.tsx ← Stack Exercices (liste → animation)
    │   └── NavigateurProfil.tsx    ← Stack Profil (profil → sous-pages)
    │
    ├── screens/
    │   ├── auth/
    │   │   ├── EcranConnexion.tsx      ← Formulaire de connexion
    │   │   └── EcranInscription.tsx    ← Formulaire d'inscription
    │   ├── tabs/
    │   │   ├── EcranAccueil.tsx        ← Dashboard (onglet 1)
    │   │   ├── EcranArticles.tsx       ← Liste articles (onglet 2)
    │   │   └── EcranExercices.tsx      ← Liste exercices (onglet 3)
    │   ├── EcranProfil.tsx             ← Profil utilisateur (onglet 4)
    │   ├── articles/
    │   │   └── EcranDetailArticle.tsx  ← Détail d'un article
    │   ├── exercises/
    │   │   └── EcranAnimationExercice.tsx ← Animation guidée de respiration
    │   ├── profil/
    │   │   ├── EcranModifierProfil.tsx
    │   │   ├── EcranChangerMotDePasse.tsx
    │   │   └── EcranHistoriqueSessions.tsx
    │   └── admin/
    │       └── EcranAdmin.tsx          ← Interface admin
    │
    └── services/
        ├── api.ts                  ← Instance Axios + cache token en mémoire
        ├── storage.ts              ← Abstraction SecureStore
        ├── secureStorage.ts        ← Compatibilité web/native
        ├── authService.ts          ← Auth API + décodage JWT
        ├── articleService.ts       ← Articles API
        ├── exerciseService.ts      ← Exercices + sessions API
        ├── adminService.ts         ← Admin API
        └── profilService.ts        ← Profil utilisateur
```

---

## 6.3 Schéma de navigation

```
AppNavigator
│
├── [Non connecté] → NavigateurAuth (Stack)
│     ├── EcranConnexion
│     └── EcranInscription
│
└── [Connecté] → NavigateurOnglets (Bottom Tabs)
      │
      ├── Onglet 1 : Accueil
      │     └── EcranAccueil
      │
      ├── Onglet 2 : Articles → NavigateurArticles (Stack)
      │     ├── EcranArticles (liste)
      │     └── EcranDetailArticle (détail)
      │
      ├── Onglet 3 : Exercices → NavigateurExercices (Stack)
      │     ├── EcranExercices (liste)
      │     └── EcranAnimationExercice (animation guidée)
      │
      └── Onglet 4 : Profil → NavigateurProfil (Stack)
            ├── EcranProfil
            ├── EcranModifierProfil
            ├── EcranChangerMotDePasse
            └── EcranHistoriqueSessions
```

**Comment ça fonctionne :**
- `AppNavigator` vérifie si l'utilisateur est connecté (via `AuthContext`)
- Si connecté → affiche les onglets
- Si non connecté → affiche les écrans de connexion/inscription
- La navigation entre les deux états est automatique lors du login/logout

---

## 6.4 Authentification mobile

### Pourquoi `SecureStore` et pas `localStorage` ?

Sur mobile, `localStorage` n'est **pas chiffré**. N'importe quelle application peut lire son contenu. `expo-secure-store` utilise le **trousseau iOS** ou le **KeyStore Android** pour chiffrer les données. C'est la méthode recommandée pour stocker des tokens JWT sur mobile.

### Architecture du stockage

```
AuthContext
│
├── SecureStore (persistant, chiffré)
│     ├── "token" → JWT brut
│     └── "utilisateur" → JSON des infos utilisateur
│
└── Cache mémoire (variable JS)
      └── tokenEnCache → copie du token pour accès synchrone
```

**Pourquoi le cache mémoire ?** SecureStore est asynchrone (renvoie une `Promise`). Or, Axios a besoin du token de façon synchrone dans son intercepteur. La solution : stocker le token dans une variable JS en mémoire, synchronisée avec SecureStore.

### Vérification d'expiration JWT

Au démarrage de l'app, `AuthContext` :
1. Lit le token depuis SecureStore
2. Décode le payload JWT (base64)
3. Vérifie si `exp` (timestamp d'expiration) est dans le passé
4. Si expiré → déconnecte l'utilisateur automatiquement
5. Si valide → restaure la session

---

## 6.5 Écrans clés

### `EcranAnimationExercice`

Affiche une animation de respiration guidée avec un timer :
- Compte les phases (inspiration → apnée → expiration)
- Affiche un cercle animé qui s'agrandit/rétrécit
- Enregistre une `UserSession` via `POST /api/user_sessions`
- Met à jour le statut (`completed` ou `abandoned`) via `PATCH /api/user_sessions/{id}`

### `EcranHistoriqueSessions`

Affiche la liste des sessions avec :
- L'exercice effectué
- La durée
- Le statut : `started` (commencé), `completed` (terminé), `abandoned` (abandonné)

---

## 6.6 Dépendances principales

| Package | Version | Rôle |
|---------|---------|------|
| `expo` | ~54.0.33 | Framework React Native |
| `react-native` | 0.81.5 | Rendu mobile natif |
| `@react-navigation/native` | ^7.2.2 | Navigation |
| `@react-navigation/bottom-tabs` | ^7.15.9 | Onglets du bas |
| `@react-navigation/stack` | ^7.8.9 | Navigation par pile |
| `expo-secure-store` | ~15.0.8 | Stockage chiffré des tokens |
| `@expo/vector-icons` | ^15.0.3 | Icônes (Ionicons) |
| `axios` | ^1.14.0 | Requêtes HTTP |
| `typescript` | ~5.9.2 | Typage statique |

---

## 6.7 Configuration Expo (`app.json`)

Fichier de configuration principal de l'application :

```json
{
  "expo": {
    "name": "cesizen-mobile",
    "slug": "cesizen-mobile",
    "version": "1.0.0",
    "orientation": "portrait",
    "plugins": ["expo-secure-store"],
    "android": {
      "edgeToEdgeEnabled": true,
      "adaptiveIcon": { ... }
    },
    "ios": {
      "infoPlist": {
        "NSAppTransportSecurity": {
          "NSAllowsArbitraryLoads": true
        }
      }
    }
  }
}
```

> **`NSAllowsArbitraryLoads: true`** est nécessaire en développement pour autoriser les connexions HTTP (non HTTPS) vers l'API locale. En production, utilise HTTPS et retire cette option.

---

## 6.8 Scripts disponibles

```bash
npx expo start          # Démarrer (QR code pour Expo Go)
npx expo start --android  # Démarrer sur émulateur Android
npx expo start --ios      # Démarrer sur simulateur iOS (Mac uniquement)
npm run type-check        # Vérifier les types TypeScript
npm test                  # Lancer les tests Jest
npm run test:coverage     # Tests avec rapport de couverture
```

---

**Suite →** [07 — Tests](./07-tests.md)
