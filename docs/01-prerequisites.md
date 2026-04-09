# 01 — Prérequis

Avant de commencer, installe ces outils sur ta machine. **L'ordre n'a pas d'importance** pour l'installation, mais tous sont nécessaires.

---

## Outils obligatoires

### 1. Docker Desktop
Docker permet de lancer l'API, la base de données et Nginx dans des conteneurs isolés, **sans installer PHP ou PostgreSQL directement sur ta machine**.

- Téléchargement : https://www.docker.com/products/docker-desktop/
- Version minimale : Docker 24+, Docker Compose v2
- **Windows** : activer WSL2 lors de l'installation (recommandé)

Vérification :
```bash
docker --version
# Docker version 24.x.x ou supérieur

docker compose version
# Docker Compose version v2.x.x ou supérieur
```

---

### 2. Node.js 22 (LTS)
Nécessaire pour lancer le frontend React et l'application Expo.

- Téléchargement : https://nodejs.org/ (choisir "LTS")
- Version exacte utilisée dans ce projet : **Node.js 22**

Vérification :
```bash
node -v
# v22.x.x

npm -v
# 10.x.x ou supérieur
```

> **Conseil** : utilise [nvm](https://github.com/nvm-sh/nvm) (Linux/Mac) ou [nvm-windows](https://github.com/coreybutler/nvm-windows) pour gérer plusieurs versions de Node.

---

### 3. Git
Pour cloner les repos et versionner le code.

- Téléchargement : https://git-scm.com/
- Version minimale : Git 2.x

Vérification :
```bash
git --version
# git version 2.x.x
```

Configuration minimale (si pas encore fait) :
```bash
git config --global user.name "Ton Prénom Nom"
git config --global user.email "ton.email@viacesi.fr"
```

---

### 4. Visual Studio Code (recommandé)
Éditeur de code utilisé tout au long du projet.

- Téléchargement : https://code.visualstudio.com/

Extensions recommandées :
- **PHP Intelephense** — autocomplétion PHP
- **Symfony for VS Code** — support Symfony/Twig
- **ES7+ React Snippets** — raccourcis React
- **Tailwind CSS IntelliSense** — autocomplétion Tailwind
- **Docker** — gestion des conteneurs depuis VS Code
- **GitLens** — visualisation de l'historique Git

---

### 5. Expo Go (sur ton téléphone)
Pour tester l'application mobile sur ton propre téléphone sans compilation.

- **Android** : [Expo Go sur Google Play](https://play.google.com/store/apps/details?id=host.exp.exponent)
- **iOS** : [Expo Go sur App Store](https://apps.apple.com/app/expo-go/id982107779)

> **Important** : ton téléphone et ton ordinateur doivent être sur le **même réseau Wi-Fi**.

---

## Compte GitHub

Tu as besoin d'un compte GitHub pour :
- Accéder aux repos du projet
- Déclencher les workflows CI/CD
- Pousser du code et créer des Pull Requests

Vérifie que tu es connecté :
```bash
gh auth status
# Si pas installé : https://cli.github.com/
```

---

## Résumé : vérification globale

Lance ces commandes pour confirmer que tout est installé :

```bash
docker --version          # Docker 24+
docker compose version    # Compose v2+
node -v                   # v22.x.x
npm -v                    # 10+
git --version             # 2.x.x
```

Si toutes ces commandes retournent une version sans erreur, tu es prêt pour la suite.

---

**Suite →** [02 — Architecture technique](./02-architecture.md)
