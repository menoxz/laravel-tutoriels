# Guide d'utilisation d'un projet Laravel dockerisé

Ce document détaille comment cloner, configurer et démarrer un projet Laravel dockerisé.

## Table des matières

1. [Prérequis](#prérequis)
2. [Clonage du projet](#clonage-du-projet)
3. [Configuration de l'environnement](#configuration-de-lenvironnement)
4. [Lancement des conteneurs Docker](#lancement-des-conteneurs-docker)
5. [Installation et configuration de Laravel](#installation-et-configuration-de-laravel)
6. [Accès à l'application](#accès-à-lapplication)
7. [Commandes utiles](#commandes-utiles)
8. [Résolution des problèmes courants](#résolution-des-problèmes-courants)

## Prérequis <a name="prérequis"></a>

Avant de commencer, assurez-vous d'avoir installé les logiciels suivants sur votre machine:

- **Docker** (avec Docker Compose): pour exécuter les conteneurs
- **Git**: pour cloner le dépôt du projet
- **Éditeur de texte** ou IDE: pour modifier les fichiers de configuration

## Clonage du projet <a name="clonage-du-projet"></a>

1. Ouvrez un terminal et clonez le projet depuis GitHub:

```bash
git clone <url_du_projet>
```

Remplacez `<url_du_projet>` par l'URL réelle du dépôt Git.

2. Accédez au répertoire du projet:

```bash
cd <nom_du_répertoire_du_projet>
```

## Configuration de l'environnement <a name="configuration-de-lenvironnement"></a>

1. Création du fichier d'environnement:

```bash
cp .env.example .env
```

2. Modifiez le fichier `.env` pour configurer les variables d'environnement:

```
# Configuration de base
APP_NAME=NomDeLApplication
APP_ENV=local
APP_KEY=
APP_DEBUG=true
APP_URL=http://localhost

# Configuration de la base de données
DB_CONNECTION=mysql
DB_HOST=db
DB_PORT=3306
DB_DATABASE=nom_de_la_base_de_donnees
DB_USERNAME=nom_utilisateur
DB_PASSWORD=mot_de_passe
```

Adaptez ces valeurs selon les besoins du projet et la configuration de votre environnement Docker.

> **Important**: Vérifiez que `DB_HOST` pointe vers le nom du service de base de données défini dans votre fichier `docker-compose.yml` (généralement `db` ou `mysql`).

## Lancement des conteneurs Docker <a name="lancement-des-conteneurs-docker"></a>

1. Démarrez les conteneurs Docker en arrière-plan:

```bash
docker-compose up -d
```

Cette commande:
- Construit les images Docker si nécessaire
- Crée et démarre les conteneurs
- Configure les réseaux et volumes définis dans le `docker-compose.yml`

2. Vérifiez que les conteneurs sont en cours d'exécution:

```bash
docker-compose ps
```

Vous devriez voir tous les services listés comme "Up".

## Installation et configuration de Laravel <a name="installation-et-configuration-de-laravel"></a>

Une fois les conteneurs démarrés, vous devez finaliser l'installation de Laravel:

1. Installation des dépendances PHP avec Composer:

```bash
docker-compose exec app composer install
```

> **Note**: Remplacez `app` par le nom du service PHP défini dans votre `docker-compose.yml` si celui-ci est différent.

2. Génération de la clé d'application Laravel:

```bash
docker-compose exec app php artisan key:generate
```

3. Exécution des migrations de base de données:

```bash
docker-compose exec app php artisan migrate
```

4. (Optionnel) Alimentation de la base de données avec des données de test:

```bash
docker-compose exec app php artisan db:seed
```

5. (Optionnel) Compilation des assets si le projet utilise Laravel Mix:

```bash
docker-compose exec app npm install
docker-compose exec app npm run dev
```

## Accès à l'application <a name="accès-à-lapplication"></a>

Une fois l'installation terminée, vous pouvez accéder à l'application Laravel:

- Frontend: http://localhost (ou le port configuré dans votre `docker-compose.yml`)
- (Si applicable) phpMyAdmin: http://localhost:8080 (si configuré)
- (Si applicable) Mailhog: http://localhost:8025 (si configuré)

## Commandes utiles <a name="commandes-utiles"></a>

Voici quelques commandes Docker utiles pour gérer votre projet Laravel:

| Commande | Description |
|----------|-------------|
| `docker-compose up -d` | Démarrer les conteneurs en arrière-plan |
| `docker-compose down` | Arrêter et supprimer les conteneurs |
| `docker-compose logs -f [service]` | Afficher les logs d'un service (ou de tous) |
| `docker-compose exec app bash` | Ouvrir un terminal dans le conteneur de l'application |
| `docker-compose exec app php artisan [commande]` | Exécuter une commande Artisan |
| `docker-compose exec db mysql -u[user] -p[password]` | Accéder directement à MySQL |

## Résolution des problèmes courants <a name="résolution-des-problèmes-courants"></a>

### Problèmes de permissions

Si vous rencontrez des problèmes de permissions:

```bash
docker-compose exec app chmod -R 777 storage bootstrap/cache
```

### Problèmes de connexion à la base de données

1. Vérifiez que le service de base de données est bien démarré:

```bash
docker-compose ps db
```

2. Vérifiez que les informations de connexion dans le fichier `.env` correspondent à celles définies dans `docker-compose.yml`.

3. Attendez quelques secondes après le démarrage des conteneurs pour que MySQL s'initialise complètement.

### Port déjà utilisé

Si Docker indique qu'un port est déjà utilisé, modifiez le mapping de port dans le fichier `docker-compose.yml`:

```yaml
ports:
  - "8080:80"  # Utilise le port 8080 au lieu de 80
```

---

Ce guide devrait vous permettre de cloner, configurer et démarrer efficacement un projet Laravel dockerisé. N'hésitez pas à consulter la documentation Docker et Laravel officielle pour des informations supplémentaires. 