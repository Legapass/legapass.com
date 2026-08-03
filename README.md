# Deploy WordPress on CleverCloud, the immutable way

[Bedrock](https://roots.io/bedrock/) ([GitHub Project](https://github.com/roots/bedrock)) is a modern WordPress stack that allows to maintain your installation clean from any code change during runtime. [CleverCloud](https://www.clever-cloud.com/) is a rock solid IT automation platform. Now you can take advantages of both to have your [WordPress](https://wordpress.org) installed on it. While you can follow basic steps to install BedRock yourself on CleverCloud, here is a nice shortcut that you can just fork and deploy.

## Requirements

None. Except a CleverCloud account ;-)

## Initial deployment

It will assume your GitHub account is linked to your CleverCloud account. If not, you'll just have to do the same steps but cloning and pushing the project yourself to CleverCloud.

1. Fork this magnificent repository
1. Log in to your CleverCloud console
1. Create a new application, by selecting this project fork, and obviously as a *PHP* one
1. Add one *MySQL database* add-on
1. On next page, edit the environment variables in expert mode and paste one env salts generated [here](https://cdn.roots.io/salts.html). Don't forget to save the changes.
1. Add 3 more variables : `WP_ENV` with value `production` ; `WP_HOME` with value `https://your-domain.tld` ; `WP_SITEURL` with value `https://your-domain.tld/wp`
1. While your app start, create a *Cellar S3 storage* add-on, and link it to your application
1. On the add-on configuration page, create one bucket
1. Go back in your application configuration and add the environment variable `CELLAR_ADDON_BUCKET` with the name of your bucket
1. Apply changes by restarting your application
1. Don't forget to set up your domain name as configured for `WP_HOME` (or one `*.cleverapps.io` for testing purpose)
1. You'll then have access to the installation page of WordPress
1. After installed, go to your plugins home page and active `S3 Uploads`

**Important note :** At this time, your WordPress installation is not capable of sending any emails. Follow  [CleverCloud's documentation](https://www.clever-cloud.com/doc/php/php-apps/#sending-emails) to configure your SMTP server, of activate and configure the `Mailgun` plugin installed by default.

## Gestion des dépendances (legapass.com)

WordPress, les thèmes et les extensions sont entièrement pilotés par _composer_. **Aucune
mise à jour ne passe par l'admin WordPress** : `DISALLOW_FILE_MODS` est à `true` dans
[config/application.php](config/application.php).

### Le composer.lock fait foi

Toutes les versions sont épinglées de façon exacte dans `composer.json`, et `composer.lock`
est versionné. Clever Cloud exécute `composer install`, qui lit **uniquement** le lock :
un déploiement réinstalle donc exactement les mêmes versions qu'au déploiement précédent.

C'est volontaire. Avant, les contraintes étaient ouvertes (`>=6.5`, `*`) et sans lock :
chaque rebuild réinstallait silencieusement les dernières versions disponibles de
WordPress et de toutes les extensions, sans que personne ne l'ait demandé ni testé.

**Ne jamais lancer `composer update` sans intention explicite** : cela remonte tout d'un
coup. Utilisez la procédure ci-dessous.

### Mettre à jour une extension (ou WordPress)

Une seule à la fois, pour que chaque montée de version reste identifiable et réversible
dans l'historique git :

```bash
# 1. Modifier la version voulue dans composer.json, par exemple :
#    "wpackagist-plugin/elementor": "4.1.4"  ->  "4.2.1"

# 2. Recalculer le lock pour ce seul paquet et ses dépendances
composer update wpackagist-plugin/elementor --with-dependencies

# 3. Vérifier qu'aucune vulnérabilité connue n'est introduite
composer audit

# 4. Commiter composer.json ET composer.lock ensemble
git add composer.json composer.lock
git commit -m "Elementor 4.1.4 -> 4.2.1"
```

Pour WordPress, pensez à monter `roots/wordpress` **et** les deux paquets de langue
`koodimonni-language/fr_fr` et `koodimonni-language/core-fr_fr` sur le même numéro de
version. Après déploiement, connectez-vous en administrateur : WordPress proposera la
mise à jour de la base de données si nécessaire.

### Ajouter une extension

Ajoutez-la à `composer.json` avec une version exacte, puis `composer update <paquet>`.
Les sources disponibles sont déclarées dans la section `repositories` : [wpackagist](https://wpackagist.org)
pour le répertoire officiel, packagist pour le reste.

### Version de PHP

L'application Clever Cloud tourne en **PHP 8.4**, et `config.platform.php` du
`composer.json` reflète cette valeur. Composer résout donc les dépendances pour la
plateforme réelle du serveur, quelle que soit la version de PHP installée sur la machine
qui lance la commande.

Si la version PHP est changée depuis la console Clever Cloud, il faut ajuster
`config.platform.php` **et** la contrainte `require.php` (actuellement `>=8.4`), puis
régénérer le lock :

```bash
composer update --lock
```

## Differences with Bedrock

For those who want or need to go deeper regarding Bedrock, here are the small differences between this fork (based on version __1.12.8__) and a standard Bedrock install.
- You don't need any `.env` file for your environment variables, it can be useful if you want to run your WordPress locally
- `config/application.php` has been modified to directly use MySQL and Cellar environment variables shared by CleverCloud
- Plugin `humanmade/s3-uploads` added by default to use S3 storage for media files
- `web/app/mu-plugins/s3-uploads-filter.php` have been added to use a Cellar endpoint in place of an AWS one
- `.htaccess` have been included by default

Enjoy !
