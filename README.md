# Plugin GLPI 11 — Helpline WebService

Plugin GLPI 11 permettant à **Genesys Cloud** d'interagir avec GLPI via des appels REST atomiques,
sans orchestration côté Genesys.

> **1 action Genesys = 1 requête HTTP = 1 réponse JSON.**
> Toute la complexité interne est absorbée par le plugin et invisible pour Genesys.

## Compatibilité

| Élément  | Version        |
|----------|----------------|
| GLPI     | 11.0.0 → 11.x  |
| PHP      | 8.1+           |
| Plugin   | 2.3.0          |

## Installation

```bash
# 1. Déposer le contenu du dossier dans plugins/
# IMPORTANT : le point final est obligatoire — sans lui docker cp crée un double dossier helpline/helpline/
docker cp ./helpline/. glpi-app:/var/www/glpi/plugins/helpline/

# 2. Installer les dépendances PHP
cd /var/www/glpi/plugins/helpline
composer install --no-dev

# Si l'accès réseau est bloqué (SSL), générer uniquement l'autoloader :
composer dump-autoload --no-dev

# 3. Dans GLPI : Configuration > Plugins → Installer puis Activer "Helpline WebService"

# 4. Vider le cache GLPI (obligatoire après installation ou modification)
cd /var/www/glpi && php bin/console cache:clear --allow-superuser
```

## Mise à jour depuis une version antérieure

```bash
# 1. Remplacer les fichiers du plugin (ne pas supprimer le dossier)
docker cp ./helpline/. glpi-app:/var/www/glpi/plugins/helpline/

# 2. Mettre à jour les dépendances
cd /var/www/glpi/plugins/helpline
composer install --no-dev

# 3. Dans GLPI : Configuration > Plugins → cliquer sur "Mettre à jour"
#    GLPI appelle automatiquement plugin_helpline_update() qui initialise
#    les nouvelles clés de configuration (ex: trusted_proxies) sans écraser
#    les valeurs déjà en place.

# 4. Vider le cache GLPI
cd /var/www/glpi && php bin/console cache:clear --allow-superuser
```

> **Nouveau en v2.3.0 :** après mise à jour, configurer les **proxies de confiance**
> dans Configuration > Générale > Helpline WebService si un load-balancer est
> présent entre Genesys et GLPI (correction S-01).

## Configuration

**Configuration > Générale > onglet "Helpline WebService"**

| Paramètre | Description |
|-----------|-------------|
| IPs autorisées — Genesys Cloud | Liste blanche d'IPs/CIDR (une par ligne). Laisser vide = toutes IPs acceptées (déconseillé). |
| Proxies de confiance | IPs/CIDR des proxies/load-balancers entre Genesys et GLPI. Requis pour que X-Forwarded-For soit lu correctement (correction S-01). Laisser vide si connexion directe. |
| Catégorie ITIL — Incident Majeur | Catégorie affectée automatiquement lors de la création d'un incident majeur via Genesys. Si non définie, le plugin recherche une catégorie nommée "Incident Majeur". |
| Gabarit de solution — Clôture Genesys | Gabarit (`glpi_solutiontemplates`) utilisé pour pré-remplir la solution lors de la clôture d'un ticket. Si aucun gabarit, un contenu libre générique est utilisé. |

> Les valeurs sont stockées dans `glpi_configs` (contexte `plugin:helpline`). Aucun accès SSH requis.

## Authentification OAuth2

Les requêtes sont authentifiées via **OAuth2 Password Grant**. Un client OAuth2 doit être configuré
dans GLPI avec le scope `api` et le grant `password`.

**Obtenir un token :**
```
POST /api.php/token
Content-Type: application/x-www-form-urlencoded

grant_type=password
&client_id=<identifier>
&client_secret=<secret>
&username=<login>
&password=<password>
&scope=api
```

**Utiliser le token sur chaque appel :**
```
Authorization: Bearer <access_token>
Accept: application/json
```

> Le compte GLPI doit disposer du droit **UPDATE sur les tickets**. Sans ce droit → HTTP 403.

## Endpoints

Base URL : `https://<host>/api.php/v2/Helpline/<action>`

| Endpoint | Méthode | Rôle | Paramètres |
|----------|---------|------|------------|
| `get_user_infos` | POST | Identification appelant + tickets actifs | `user_name` ou `email` ou `phone` |
| `create_parent_incident` | POST | Création d'un Incident Majeur | `user_name`, `title`, `content` |
| `create_child_incident` | POST | Rattachement ticket fils à un Incident Majeur | `user_name`, `ticket_number` |
| `relaunch_inbound` | POST | Relance entrante (utilisateur → Service Desk) | `ticket_number` |
| `relaunch_outbound` | POST | Relance sortante (Service Desk → utilisateur) | `ticket_number` |
| `add_note` | POST | Note libre sur un ticket | `ticket_number`, `ticket_note` |
| `incident_status` | POST | Statut actif/inactif d'un ticket | `ticket_number` |
| `close_incident` | POST | Clôture avec gabarit de solution | `ticket_number`, `user_name` |

### Format des réponses

```json
{
  "response_status": "SUCCESS" | "ERROR",
  "response_note": "CODE_RETOUR"
}
```

| Code retour | Signification |
|-------------|---------------|
| `USER_FOUND` | Utilisateur trouvé |
| `USER_NOT_FOUND` | Utilisateur introuvable |
| `PARENT_CREATED` | Incident Majeur créé |
| `CHILD_CREATED` | Ticket fils créé et lié |
| `FOLLOWUP_ADDED` | Suivi ajouté |
| `TICKET_ACTIVE` | Ticket en cours |
| `TICKET_INACTIVE` | Ticket résolu ou clôturé |
| `TICKET_CLOSED` | Ticket clôturé avec succès |
| `FORBIDDEN` | Droits insuffisants (HTTP 403) |
| `IP_NOT_ALLOWED` | IP source non autorisée (HTTP 403) |
| `TICKET_NOT_FOUND` | Ticket introuvable |
| `MISSING_PARAM` | Paramètre obligatoire manquant (HTTP 400) |
| `INVALID_PARAM` | Paramètre invalide — format incorrect (HTTP 400) |

### Contraintes sur les paramètres

| Paramètre | Type | Contrainte | Rejet |
|-----------|------|-----------|-------|
| `ticket_number` | Entier | **Strictement positif** — chiffres uniquement, valeur > 0. Toute valeur non numérique, nulle, négative ou contenant des caractères non numériques est refusée. | HTTP 400 `INVALID_PARAM` |
| `email` | Chaîne | Format email valide selon RFC 5322 (`FILTER_VALIDATE_EMAIL`). | HTTP 400 `INVALID_PARAM` |
| `user_name` | Chaîne | 255 caractères maximum. | Tronqué silencieusement |
| `email` | Chaîne | 255 caractères maximum. | Tronqué silencieusement |
| `phone` | Chaîne | 50 caractères maximum. | Tronqué silencieusement |
| `title` | Chaîne | 255 caractères maximum. | Tronqué silencieusement |
| `content` | Chaîne | 65 535 caractères maximum. | Tronqué silencieusement |
| `ticket_note` | Chaîne | 4 096 caractères maximum. | Tronqué silencieusement |

> **Exemples de valeurs refusées pour `ticket_number` :** `0`, `-1`, `1.5`, `12abc`, `; DROP TABLE`, chaîne vide. Seul un entier strictement positif comme `42` ou `1208` est accepté.

## Sécurité (v2.3.0)

| Réf. | Protection | Mécanisme |
|------|-----------|-----------|
| S-01 | Usurpation IP via X-Forwarded-For | `getClientIp()` : XFF lu uniquement si REMOTE_ADDR ∈ trusted_proxies |
| S-02/03 | XSS stockée | Délégation Sanitizer GLPI via `CommonDBTM::add()` / `ITILFollowup::add()` |
| S-04 | Injection via ticket_number | `ctype_digit() && (int) > 0` avant tout cast |
| S-05 | Requête SQL sur email malformé | `FILTER_VALIDATE_EMAIL` → HTTP 400 si invalide |
| S-06 | DoS payload massif | Troncature : user_name/email=255, phone=50, title=255, content=65535, note=4096 |
| S-07 | PHPDoc incomplet | `@since`, `@param`, `@return` sur toutes les méthodes |
| S-08 | `$rightname` typé `string` natif (interdit sur propriété héritée) | `@var string` uniquement — type natif retiré |
| S-09 | Headers sécurité absents sur certaines réponses | `secureResponse()` wrappé sur toutes les réponses |
| S-10 | Bypass liste blanche IPv6 | `ipMatchesCidr()` gère IPv4 et IPv6 via `inet_pton()` |
| S-11 | Coercions PHP silencieuses | `declare(strict_types=1)` dans tous les fichiers |
| S-12 | `echo` dans `check_prerequisites` | `Session::addMessageAfterRedirect()` |
| SEC-11 | Droits insuffisants sur un item spécifique (TECLIB) | `canUpdateItem()` / `canViewItem()` sur chaque ticket manipulé |
| SEC-12 | Accès à une entité non autorisée (TECLIB) | `Session::haveAccessToEntity()` avant création de ticket |

## Structure du plugin

```
helpline/
├── src/
│   ├── Controller/
│   │   └── ApiController.php      # Toutes les routes #[Route] + logique métier + sécurité
│   └── Config.php                 # Interface d'administration GLPI
├── templates/
│   └── config.html.twig           # Formulaire de configuration (Twig)
├── phpunit/
│   └── functional/
│       └── ApiControllerTest.php  # Tests fonctionnels
├── phpstan-stubs/
│   └── GlpiClasses.stub           # Stubs PHPStan pour les classes GLPI
├── hook.php                       # Callbacks install / update / uninstall
├── setup.php                      # Déclaration du plugin + hooks GLPI
├── composer.json                  # Autoloading PSR-4 + dépendances
├── plugin.xml                     # Descripteur catalogue GLPI
├── phpstan.neon                   # Configuration PHPStan level 5
├── phpstan-bootstrap.php          # Bootstrap PHPStan hors contexte GLPI
├── rector.php                     # Configuration Rector
├── CHANGELOG.md                   # Journal des modifications
└── README.md                      # Ce fichier
```

## Développement

### Règle PSR-12 — ordre obligatoire dans chaque fichier PHP

```php
<?php

declare(strict_types=1);

namespace GlpiPlugin\Helpline;

use ClassA;
use ClassB;

/**
 * Docblock de classe — description, annotations @since, @author, etc.
 */
class MaClasse
{
```

Ordre absolu — ne jamais déroger :

| Position | Élément | Ligne vide après |
|----------|---------|-----------------|
| 1 | `<?php` | Oui |
| 2 | `declare(strict_types=1);` | Oui |
| 3 | `namespace` | Oui |
| 4 | blocs `use` | Oui |
| 5 | Docblock `/** ... */` | **Non** — directement attaché à `class` |
| 6 | `class` / `final class` | — |

Règles clés :
- Le docblock est sur la **classe**, pas sur le fichier
- Le docblock est **directement attaché** à `class` — aucune ligne vide entre `*/` et `class`
- `declare(strict_types=1)` précède toujours `namespace`, `use` et le docblock
- PHPCBF corrige les lignes vides et l'ordre automatiquement, mais ne rajoute pas `declare()` s'il est absent — c'est au développeur de l'écrire

### Commandes

```bash
# Installer toutes les dépendances (dont dev)
composer install

# Si accès réseau bloqué (SSL) — autoloader uniquement, sans téléchargement
composer dump-autoload --no-dev

# Linting PSR-12 — vérification
phpcs --standard=PSR12 hook.php setup.php src/

# Linting PSR-12 — correction automatique
phpcbf --standard=PSR12 hook.php setup.php src/

# Analyse statique PHPStan level 5
phpstan analyse src/ hook.php setup.php --level=5 --configuration=phpstan.neon --memory-limit=512M

# Tests fonctionnels
./vendor/bin/phpunit phpunit/functional/

# Rector — voir les modernisations proposées sans modifier
vendor/bin/rector process src/ --dry-run

# Rector — appliquer les modernisations
vendor/bin/rector process src/

# Vider le cache GLPI (obligatoire après toute modification PHP ou Twig)
cd /var/www/glpi && php bin/console cache:clear --allow-superuser
```

## Conformité GLPI 11 (MO-GLPI-PLUGIN-002 v2.0)

| Exigence | Statut |
|----------|--------|
| Configuration sans accès SSH | ✅ Via onglet GLPI natif (Twig) |
| Aucune requête SQL brute | ✅ `$DB->request()` exclusivement |
| Contrôle des droits global sur toutes les routes | ✅ `Session::haveRight('ticket', UPDATE)` |
| Contrôle des droits par item (TECLIB) | ✅ `canUpdateItem()` / `canViewItem()` sur chaque ticket |
| Contrôle d'accès à l'entité (TECLIB) | ✅ `Session::haveAccessToEntity()` avant création |
| Pas de fichiers front/ajax non protégés | ✅ Plugin 100% API HL |
| Twig obligatoire pour les formulaires | ✅ `templates/config.html.twig` |
| Contrôleur dans `src/Controller/` | ✅ `src/Controller/ApiController.php` |
| Classe contrôleur `final` | ✅ `final class ApiController` |
| `declare(strict_types=1)` partout | ✅ Tous fichiers PHP (S-11) |
| `plugin_*_update()` dans hook.php | ✅ Initialise les clés manquantes sans écraser |
| `composer.json` avec PSR-4 | ✅ Mapping `GlpiPlugin\\Helpline\\ → src/` |
| `CHANGELOG.md` présent | ✅ |
| Tests unitaires | ✅ `phpunit/functional/ApiControllerTest.php` |
| PHPDoc complet | ✅ `@since`, `@param`, `@return` sur toutes les méthodes |
| Compatibilité GLPI 11.x | ✅ MAX = `11.0.99` |

## Blocages techniques historisés

| ID | Description | Solution appliquée |
|----|-------------|---------------------|
| B-11 | `state=1` en base malgré message "erreur" UI | Faux positif GLPI 11 — ignoré |
| B-12 | Redirection vers login sans `Accept: application/json` | Header obligatoire côté Genesys |
| B-13 | `SessionExpiredException` avec `front/api.php` legacy | Architecture Router HL via `API_CONTROLLERS` |
| B-18 | HTTP 500 sur toute l'API sans `#[RouteVersion]` | Attribut ajouté sur chaque méthode publique |
| B-19 | `is_deleted` absent de `glpi_itilcategories` en GLPI 11 | Filtre supprimé |
| B-20 | `glpi_solutiontypes` ≠ gabarits de solution | Remplacement par `glpi_solutiontemplates` |
| B-21 | Scope OAuth2 non transmis au token | Ajout `&scope=api` dans la requête de token |
| B-22 | `TemplateRenderer` namespace `@helpline` non résolu | Résolu en v2.1.0 — rendu Twig natif (EC-07) |
| B-23 | `config_class` stocké avec quadruples backslashes en base après `docker cp` | UPDATE direct SQL via container MariaDB |
| B-24 | Champs Twig absents malgré template et Config.php corrects | Cache APCu web GLPI — `php bin/console cache:clear --allow-superuser` |

## Références

| Document | Contenu |
|----------|---------| 
| [PSR-12](https://www.php-fig.org/psr/psr-12/) | Standard de codage PHP appliqué |
| [Catalogue plugins GLPI](https://plugins.glpi-project.org) | Publication du plugin |
