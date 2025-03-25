# Guide complet d'authentification avec Laravel Breeze

Ce document détaille l'installation, la configuration et l'utilisation de Laravel Breeze pour mettre en place un système d'authentification moderne et sécurisé dans votre application Laravel.

## Table des matières

1. [Présentation de Laravel Breeze](#présentation)
2. [Installation](#installation)
3. [Configuration](#configuration)
4. [Personnalisation](#personnalisation)
5. [Fonctionnalités d'authentification](#fonctionnalités)
6. [Intégration d'API](#intégration-api)
7. [Authentification avec SPA](#authentification-spa)
8. [Dépannage](#dépannage)

## Présentation de Laravel Breeze <a name="présentation"></a>

Laravel Breeze est une implémentation minimaliste et élégante du système d'authentification de Laravel, incluant:

- Connexion et inscription
- Réinitialisation du mot de passe 
- Vérification d'email
- Interface utilisateur avec Tailwind CSS
- Stack frontend au choix (Blade, React, Vue, API)

Breeze est particulièrement adapté aux nouveaux projets et offre une solution légère contrairement à Laravel Jetstream qui propose davantage de fonctionnalités (équipes, deux facteurs, etc.).

## Installation <a name="installation"></a>

### Prérequis

- Laravel 9.x ou supérieur installé
- Composer
- Node.js et NPM

### Étapes d'installation

1. Créez un nouveau projet Laravel (si ce n'est pas déjà fait):

```bash
composer create-project laravel/laravel nom-du-projet
cd nom-du-projet
```

2. Installez Laravel Breeze via Composer:

```bash
composer require laravel/breeze --dev
```

3. Publiez les ressources de Breeze en choisissant l'une des stacks suivantes:

#### Pour une stack Blade (traditionnel):

```bash
php artisan breeze:install blade
```

#### Pour une stack React (Inertia.js):

```bash
php artisan breeze:install react
```

#### Pour une stack Vue (Inertia.js):

```bash
php artisan breeze:install vue
```

#### Pour une API uniquement (sans interface frontend):

```bash
php artisan breeze:install api
```

4. Installez les dépendances frontend et compilez les assets:

```bash
npm install
npm run dev
```

5. Configurez votre base de données dans le fichier `.env`:

```
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=root
DB_PASSWORD=
```

6. Exécutez les migrations pour créer les tables nécessaires:

```bash
php artisan migrate
```

## Configuration <a name="configuration"></a>

### Structure des fichiers

Après l'installation, Breeze ajoute plusieurs fichiers à votre projet:

```
app/
├── Http/
│   ├── Controllers/
│   │   ├── Auth/
│   │   │   ├── AuthenticatedSessionController.php
│   │   │   ├── ConfirmablePasswordController.php
│   │   │   ├── EmailVerificationNotificationController.php
│   │   │   ├── EmailVerificationPromptController.php
│   │   │   ├── NewPasswordController.php
│   │   │   ├── PasswordController.php
│   │   │   ├── PasswordResetLinkController.php
│   │   │   ├── RegisteredUserController.php
│   │   │   └── VerifyEmailController.php
│   │   └── ...
│   ├── Requests/
│   │   ├── Auth/
│   │   │   ├── LoginRequest.php
│   │   │   └── ...
├── Providers/
│   └── RouteServiceProvider.php
```

### Routes d'authentification

Breeze génère plusieurs routes d'authentification dans le fichier `routes/auth.php`:

- `/login` - Page de connexion
- `/register` - Page d'inscription
- `/forgot-password` - Page de récupération de mot de passe
- `/reset-password` - Page de réinitialisation de mot de passe
- `/verify-email` - Vérification d'email
- `/confirm-password` - Confirmation de mot de passe
- `/profile` - Page de profil utilisateur

### Redirection après connexion

Par défaut, les utilisateurs sont redirigés vers `/dashboard` après s'être connectés. Vous pouvez modifier ce comportement dans le fichier `app/Providers/RouteServiceProvider.php`:

```php
public const HOME = '/dashboard';
```

Changez cette valeur pour rediriger les utilisateurs vers une autre URL.

## Personnalisation <a name="personnalisation"></a>

### Personnalisation des vues

Les vues d'authentification se trouvent dans les dossiers suivants (selon le stack choisi):

#### Pour Blade:
```
resources/views/auth/
├── confirm-password.blade.php
├── forgot-password.blade.php
├── login.blade.php
├── register.blade.php
├── reset-password.blade.php
└── verify-email.blade.php
```

#### Pour Inertia (React/Vue):
```
resources/js/Pages/Auth/
├── ConfirmPassword.jsx
├── ForgotPassword.jsx
├── Login.jsx
├── Register.jsx
├── ResetPassword.jsx
└── VerifyEmail.jsx
```

### Personnalisation du modèle User

Le modèle `App\Models\User` contient déjà les traits nécessaires pour l'authentification:

```php
use Laravel\Sanctum\HasApiTokens;
use Illuminate\Notifications\Notifiable;
use Illuminate\Contracts\Auth\MustVerifyEmail;
```

Pour activer la vérification d'email, votre modèle User doit implémenter l'interface `MustVerifyEmail`:

```php
class User extends Authenticatable implements MustVerifyEmail
{
    // ...
}
```

### Personnalisation des validations

Les règles de validation pour l'inscription et la connexion se trouvent dans:

- `App\Http\Requests\Auth\LoginRequest` pour la connexion
- `App\Http\Controllers\Auth\RegisteredUserController` pour l'inscription

## Fonctionnalités d'authentification <a name="fonctionnalités"></a>

### Connexion et inscription

Le système de connexion/inscription inclut:
- Validation des formulaires
- Protection contre les attaques par force brute (limitation de tentatives)
- Mémorisation de session (remember me)

### Réinitialisation de mot de passe

Processus en deux étapes:
1. L'utilisateur demande un lien de réinitialisation (envoyé par email)
2. L'utilisateur définit un nouveau mot de passe avec un token valide

Configuration email dans `.env`:
```
MAIL_MAILER=smtp
MAIL_HOST=mailhog
MAIL_PORT=1025
MAIL_USERNAME=null
MAIL_PASSWORD=null
MAIL_ENCRYPTION=null
MAIL_FROM_ADDRESS="hello@example.com"
MAIL_FROM_NAME="${APP_NAME}"
```

### Vérification d'email

Pour activer la vérification d'email:

1. Assurez-vous que votre modèle User implémente `MustVerifyEmail`
2. Ajoutez le middleware `verified` aux routes qui nécessitent une vérification:

```php
Route::middleware(['auth', 'verified'])->group(function () {
    Route::get('/dashboard', function () {
        return view('dashboard');
    })->name('dashboard');
});
```

### Mise à jour du profil

Breeze inclut une page de profil permettant aux utilisateurs de:
- Mettre à jour leur nom et email
- Mettre à jour leur mot de passe

## Intégration d'API <a name="intégration-api"></a>

Si vous avez choisi l'option API, Breeze configure Laravel Sanctum pour l'authentification API:

### Endpoints API disponibles

- `POST /login` - Connexion et génération de token
- `POST /register` - Inscription
- `POST /forgot-password` - Demande de réinitialisation
- `POST /reset-password` - Réinitialisation avec token
- `GET /user` - Informations utilisateur (authentifié)
- `POST /logout` - Déconnexion

### Utilisation avec clients API

Exemple d'authentification avec axios:

```javascript
// Connexion et récupération du token
const response = await axios.post('/login', {
    email: 'user@example.com',
    password: 'password',
});

// Définir le token pour les futures requêtes
axios.defaults.headers.common['Authorization'] = `Bearer ${response.data.token}`;

// Récupérer les données utilisateur
const user = await axios.get('/api/user');
```

## Authentification avec SPA <a name="authentification-spa"></a>

Pour les stacks React et Vue (avec Inertia.js), Breeze configure automatiquement:

1. **Inertia.js** pour la communication client-serveur
2. **Laravel Sanctum** pour l'authentification de session SPA
3. **Middleware d'authentification** pour protéger les routes

### CSRF Protection

La protection CSRF est automatiquement configurée:

```javascript
// resources/js/app.js
axios.defaults.headers.common['X-CSRF-TOKEN'] = document.querySelector('meta[name="csrf-token"]').content;
```

### État utilisateur dans SPA

L'état utilisateur est partagé avec le frontend via:

```php
// routes/web.php
Route::middleware(['auth:sanctum', 'verified'])->get('/dashboard', function () {
    return Inertia::render('Dashboard');
})->name('dashboard');
```

## Dépannage <a name="dépannage"></a>

### Problèmes courants

#### Les emails ne sont pas envoyés

- Vérifiez votre configuration SMTP dans `.env`
- Utilisez Mailhog ou Mailtrap pour tester en développement
- Vérifiez les logs pour les erreurs d'envoi d'email

#### Erreurs après modification des champs utilisateur

Si vous modifiez le modèle User en ajoutant des champs:

1. Mettez à jour la méthode `create()` dans `RegisteredUserController.php`
2. Ajoutez les nouveaux champs à la propriété `$fillable` du modèle User
3. Mettez à jour les migrations si nécessaire

#### Session ne persiste pas

- Vérifiez que le domaine des cookies est correctement configuré
- Assurez-vous que `SESSION_DRIVER` est correctement configuré dans `.env`

### Bonnes pratiques

- Personnalisez toujours les emails envoyés aux utilisateurs
- Implémentez une politique de mots de passe robuste
- Testez le flux d'authentification complet avant la mise en production
- Activez HTTPS en production pour sécuriser les cookies de session

---

Ce guide couvre les aspects essentiels de Laravel Breeze pour l'authentification. Pour plus d'informations, consultez la [documentation officielle de Laravel](https://laravel.com/docs/authentication) et les [sources de Laravel Breeze](https://github.com/laravel/breeze). 