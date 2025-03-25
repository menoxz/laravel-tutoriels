# Guide complet de dockerisation d'un projet Laravel

Ce document détaille le processus complet pour dockeriser une application Laravel, en expliquant chaque fichier nécessaire et leurs configurations.

## Table des matières

1. [Introduction à Docker et Laravel](#introduction)
2. [Structure des fichiers](#structure-des-fichiers)
3. [Dockerfile](#dockerfile)
4. [Docker Compose](#docker-compose)
5. [Configuration Nginx](#configuration-nginx)
6. [Configuration PHP](#configuration-php)
7. [Scripts d'initialisation](#scripts-dinitialisation)
8. [Déploiement](#déploiement)
9. [Optimisations et bonnes pratiques](#optimisations)

## Introduction <a name="introduction"></a>

Docker permet de conteneuriser les applications pour garantir qu'elles fonctionnent de manière identique dans tous les environnements. Pour une application Laravel, cela implique généralement plusieurs services:

- PHP-FPM pour exécuter l'application Laravel
- Nginx comme serveur web
- MySQL/PostgreSQL pour la base de données
- Redis pour la mise en cache (optionnel)
- Mailhog pour tester les emails en développement (optionnel)

## Structure des fichiers <a name="structure-des-fichiers"></a>

Voici l'organisation recommandée des fichiers Docker dans un projet Laravel:

```
projet-laravel/
├── .env
├── docker/
│   ├── nginx/
│   │   ├── default.conf
│   │   └── Dockerfile
│   ├── php/
│   │   ├── Dockerfile
│   │   └── php.ini
│   └── scripts/
│       └── start.sh
├── docker-compose.yml
└── Dockerfile
```

## Dockerfile <a name="dockerfile"></a>

Le Dockerfile principal, situé à la racine du projet, définit comment construire l'image de l'application Laravel:

```dockerfile
FROM php:8.2-fpm

# Arguments définis dans docker-compose.yml
ARG user
ARG uid

# Installation des dépendances système
RUN apt-get update && apt-get install -y \
    git \
    curl \
    libpng-dev \
    libonig-dev \
    libxml2-dev \
    libzip-dev \
    zip \
    unzip

# Nettoyer le cache
RUN apt-get clean && rm -rf /var/lib/apt/lists/*

# Installation des extensions PHP
RUN docker-php-ext-install pdo_mysql mbstring exif pcntl bcmath gd zip

# Installation de Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Création d'un utilisateur système pour exécuter les commandes Composer et Artisan
RUN useradd -G www-data,root -u $uid -d /home/$user $user
RUN mkdir -p /home/$user/.composer && \
    chown -R $user:$user /home/$user

# Définition du répertoire de travail
WORKDIR /var/www

# Copie des fichiers de l'application
COPY . /var/www

# Copie des autorisations de dossier
COPY --chown=$user:$user . /var/www

# Changement vers l'utilisateur non-root
USER $user

# Exposition du port 9000
EXPOSE 9000

# Démarrage de PHP-FPM
CMD ["php-fpm"]
```

### Explication du Dockerfile:

- **Image de base**: `php:8.2-fpm` - Contient PHP 8.2 avec PHP-FPM pour traiter les requêtes PHP
- **Arguments**: `user` et `uid` pour créer un utilisateur non-root correspondant à l'utilisateur local
- **Dépendances système**: Installation des bibliothèques nécessaires pour les extensions PHP
- **Extensions PHP**: Installation des extensions requises par Laravel
- **Composer**: Copie de l'exécutable Composer depuis son image officielle
- **Utilisateur système**: Création d'un utilisateur avec les mêmes privilèges que l'utilisateur local
- **Répertoire de travail**: Configuration de `/var/www` comme répertoire de l'application
- **Copie de l'application**: Transfert des fichiers de l'application dans le conteneur
- **Permissions**: Configuration des permissions appropriées
- **Port**: Exposition du port 9000 utilisé par PHP-FPM
- **Commande de démarrage**: Lancement de PHP-FPM

## Docker Compose <a name="docker-compose"></a>

Le fichier `docker-compose.yml` définit et configure tous les services nécessaires pour l'application:

```yaml
version: '3'

services:
  # Service PHP Laravel
  app:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        user: laravel
        uid: 1000
    image: laravel-app
    container_name: laravel-app
    restart: unless-stopped
    working_dir: /var/www/
    volumes:
      - ./:/var/www
    networks:
      - laravel

  # Service NGINX
  nginx:
    build:
      context: ./docker/nginx
      dockerfile: Dockerfile
    image: laravel-nginx
    container_name: laravel-nginx
    restart: unless-stopped
    ports:
      - "80:80"
    volumes:
      - ./:/var/www
      - ./docker/nginx/default.conf:/etc/nginx/conf.d/default.conf
    networks:
      - laravel

  # Service MySQL
  db:
    image: mysql:8.0
    container_name: laravel-db
    restart: unless-stopped
    environment:
      MYSQL_DATABASE: ${DB_DATABASE}
      MYSQL_ROOT_PASSWORD: ${DB_PASSWORD}
      MYSQL_PASSWORD: ${DB_PASSWORD}
      MYSQL_USER: ${DB_USERNAME}
      SERVICE_TAGS: dev
      SERVICE_NAME: mysql
    volumes:
      - dbdata:/var/lib/mysql
    ports:
      - "3306:3306"
    networks:
      - laravel

  # Service Redis (optionnel pour la mise en cache)
  redis:
    image: redis:alpine
    container_name: laravel-redis
    restart: unless-stopped
    ports:
      - "6379:6379"
    networks:
      - laravel

# Volumes
volumes:
  dbdata:
    driver: local

# Réseaux
networks:
  laravel:
    driver: bridge
```

### Explication du docker-compose.yml:

- **Version**: Spécifie la version de la syntaxe Docker Compose
- **Services**: Définit les conteneurs à exécuter:
  - **app**: Le service PHP-FPM qui exécute l'application Laravel
  - **nginx**: Le serveur web qui reçoit les requêtes et les renvoie à PHP-FPM
  - **db**: Le service de base de données MySQL
  - **redis**: Service de cache (optionnel)
- **Volumes**: 
  - Montage du code source dans les conteneurs
  - Volume persistant pour les données MySQL
  - Configuration Nginx montée depuis le système hôte
- **Réseaux**: 
  - Réseau bridge pour la communication entre conteneurs

## Configuration Nginx <a name="configuration-nginx"></a>

Le fichier `docker/nginx/default.conf` configure Nginx pour servir l'application Laravel:

```nginx
server {
    listen 80;
    index index.php index.html;
    error_log  /var/log/nginx/error.log;
    access_log /var/log/nginx/access.log;
    root /var/www/public;

    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass app:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }

    location / {
        try_files $uri $uri/ /index.php?$query_string;
        gzip_static on;
    }
}
```

### Explication de la configuration Nginx:

- **Port d'écoute**: Configuration du port 80
- **Fichiers d'index**: Priorité à index.php, puis index.html
- **Journaux**: Configuration des fichiers de log
- **Racine du site**: Répertoire `/var/www/public` (dossier public de Laravel)
- **Traitement PHP**: 
  - Redirection vers PHP-FPM sur `app:9000` (le nom du service dans docker-compose)
  - Configuration des paramètres FastCGI
- **Routage Laravel**: 
  - Configuration pour le routage de Laravel avec `try_files`
  - Activation de la compression gzip

Le Dockerfile pour Nginx est simple:

```dockerfile
FROM nginx:1.21-alpine

WORKDIR /var/www

CMD ["nginx", "-g", "daemon off;"]
```

## Configuration PHP <a name="configuration-php"></a>

Le fichier `docker/php/php.ini` permet de personnaliser la configuration PHP:

```ini
upload_max_filesize = 40M
post_max_size = 40M
memory_limit = 256M
max_execution_time = 600
max_input_time = 600
```

## Scripts d'initialisation <a name="scripts-dinitialisation"></a>

Le script `docker/scripts/start.sh` peut être utilisé pour initialiser l'application au démarrage:

```bash
#!/bin/bash

# Attendre que la base de données soit prête
echo "Attente de la base de données..."
sleep 10

# Installation des dépendances PHP
composer install

# Génération de la clé d'application
php artisan key:generate

# Exécution des migrations
php artisan migrate

# Démarrage du serveur
php artisan serve --host=0.0.0.0

# Pour utiliser ce script, ajoutez-le comme CMD dans le Dockerfile
# ou comme commande de démarrage dans docker-compose.yml
```

## Déploiement <a name="déploiement"></a>

Pour déployer l'application dockerisée:

1. **Construction des images**:
```bash
docker-compose build
```

2. **Démarrage des services**:
```bash
docker-compose up -d
```

3. **Installation des dépendances**:
```bash
docker-compose exec app composer install
```

4. **Configuration de Laravel**:
```bash
docker-compose exec app php artisan key:generate
docker-compose exec app php artisan migrate
```

5. **Vérification du fonctionnement**:
Accédez à `http://localhost` dans votre navigateur.

## Optimisations et bonnes pratiques <a name="optimisations"></a>

### Images multi-étapes

Pour optimiser davantage, utilisez des builds multi-étapes:

```dockerfile
# Étape de build
FROM composer:2 as build

WORKDIR /app

COPY composer.json composer.lock ./
RUN composer install --prefer-dist --no-scripts --no-dev --no-autoloader

COPY . /app
RUN composer dump-autoload --no-scripts --no-dev --optimize

# Étape finale
FROM php:8.2-fpm

# ... installation des dépendances et configuration ...

COPY --from=build /app /var/www
```

### Environnements de développement vs production

Pour la production, ajoutez ces optimisations:

```dockerfile
# Optimisations Laravel pour la production
RUN php artisan config:cache && \
    php artisan route:cache && \
    php artisan view:cache
```

### Sécurité

- Utilisez des images de base minimales (Alpine quand possible)
- Ne stockez jamais de secrets dans les images
- Scannez régulièrement les images pour les vulnérabilités
- Limitez les droits d'accès au minimum nécessaire

### Surveillance

Intégrez des outils de surveillance:

```yaml
# Dans docker-compose.yml
prometheus:
  image: prom/prometheus
  volumes:
    - ./monitoring/prometheus:/etc/prometheus
  ports:
    - "9090:9090"

grafana:
  image: grafana/grafana
  ports:
    - "3000:3000"
  volumes:
    - grafana-storage:/var/lib/grafana
```

Cette documentation couvre les aspects essentiels de la dockerisation d'une application Laravel, depuis la configuration des conteneurs jusqu'aux bonnes pratiques de déploiement et d'optimisation. 