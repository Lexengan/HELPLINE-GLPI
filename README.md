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
| Plugin   | 2.4.0          |

## Installation

```bash
# 1. Déposer le dossier dans plugins/
cp -r helpline/ /var/www/glpi/plugins/

# 2. Installer les dépendances PHP
cd /var/www/glpi/plugins/helpline
composer install --no-dev

# 3. Dans GLPI : Configuration > Plugins → Installer puis Activer "Helpline WebService"
```

## Mise à jour depuis une version antérieure

```bash
# 1. Remplacer les fichiers du plugin (ne pas supprimer le dossier)
cp -r helpline/ /var/www/glpi/plugins/

# 2. Mettre à jour les dépendances
cd /var/www/glpi/plugins/helpline
composer install --no-dev

# 3. Dans GLPI : Configuration > Plugins → cliquer sur "Mettre à jour"
#    GLPI appelle automatiquement plugin_helpline_update() qui initialise
#    les nouvelles clés de configuration (ex: trusted_proxies) sans écraser
#    les valeurs déjà en place.
```

> **Nouveau en v2.3.0 :** après mise à jour, configurer les **proxies de confiance**
> dans Configuration > Générale > Helpline WebService si un load-balancer est
> présent entre Genesys et GLPI (correction S-01).

## Configuration

**Configuration > Générale > onglet "Helpline WebService"**

| Paramètre | Description |
|-----------|-------------|
| Catégorie ITIL — Incident Majeur | Catégorie affectée automatiquement lors de la création d'un incident majeur via Genesys. Si non définie, le plugin recherche une catégorie nommée "Incident Majeur". |
| Gabarit de solution — Clôture Genesys | Gabarit (`glpi_solutiontemplates`) utilisé pour pré-remplir la solution lors de la clôture d'un ticket. Si aucun gabarit, un contenu libre générique est utilisé. |
| IPs autorisées — Genesys Cloud | Liste blanche d'IPs/CIDR (une par ligne). Laisser vide = toutes IPs acceptées (déconseillé). |
| Proxies de confiance | IPs/CIDR des proxies/load-balancers entre Genesys et GLPI. Requis pour que X-Forwarded-For soit lu correctement (correction S-01). Laisser vide si connexion directe. |

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
| `get_user_infos` | POST | Identification appelant + tickets actifs (3 max) | `user_name` ou `email` ou `phone` |
| `create_parent_incident` | POST | Création d'un Incident Majeur | `user_name`, `title`, `content` |
| `create_child_incident` | POST | Rattachement ticket fils à un Incident Majeur | `user_name`, `ticket_number` |
| `relaunch_inbound` | POST | Relance entrante (utilisateur → Service Desk) | `ticket_number` |
| `relaunch_outbound` | POST | Relance sortante (Service Desk → utilisateur) | `ticket_number` |
| `add_note` | POST | Note libre sur un ticket | `ticket_number`, `ticket_note` |
| `incident_status` | POST | Statut actif/inactif d'un ticket | `ticket_number` |
| `close_incident` | POST | Clôture avec gabarit de solution | `ticket_number`, `user_name` |

> `get_user_infos` retourne au maximum **3 tickets actifs** dans `active_incidents_list`.
> Chaque ticket inclut un champ `id` (int, clé primaire `glpi_tickets`) nécessaire aux appels suivants.

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
| `MULTIPLE_USERS` | Plusieurs utilisateurs trouvés — critère ambigu (ERROR) |
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

## Sécurité (v2.3.0)

| Réf. | Protection | Mécanisme |
|------|-----------|-----------|
| S-01 | Usurpation IP via X-Forwarded-For | `getClientIp()` : XFF lu uniquement si REMOTE_ADDR ∈ trusted_proxies |
| S-02/03 | XSS stockée | Délégation Sanitizer GLPI via `CommonDBTM::add()` / `ITILFollowup::add()` |
| S-04 | Injection via ticket_number | `ctype_digit() && (int) > 0` avant tout cast |
| S-05 | Requête SQL sur email malformé | `FILTER_VALIDATE_EMAIL` → HTTP 400 si invalide |
| S-06 | DoS payload massif | Troncature : user_name/email=255, phone=50, title=255, content=65535, note=4096 |
| S-09 | Headers sécurité absents | `secureResponse()` wrappé sur toutes les réponses |
| S-10 | Bypass liste blanche IPv6 | `ipMatchesCidr()` gère IPv4 et IPv6 via `inet_pton()` |
| S-11 | Coercions PHP silencieuses | `declare(strict_types=1)` dans tous les fichiers |

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

```bash
# Installer toutes les dépendances (dont dev)
composer install

# Linting PSR-12
composer run lint

# Corriger automatiquement les erreurs de style
composer run lint-fix

# Analyse statique PHPStan level 5
composer run analyse

# Tests fonctionnels
composer run test

# Rector (modernisation automatique du code)
composer run rector
```

## Conformité GLPI 11 (MO-GLPI-PLUGIN-002 v2.0)

| Exigence | Statut |
|----------|--------|
| Configuration sans accès SSH | ✅ Via onglet GLPI natif (Twig) |
| Aucune requête SQL brute | ✅ `$DB->request()` exclusivement |
| Contrôle des droits sur toutes les routes | ✅ `Session::haveRight('ticket', UPDATE)` |
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
| AJF-01 | Doublon utilisateur retournait le 1er résultat silencieusement | `findUserIdsByEmail/Phone` sans LIMIT → `MULTIPLE_USERS` / ERROR si count > 1 (v2.4.0) |
| AJF-02 | `sys_id` (nomenclature ServiceNow) incohérent avec la doc V4 | Renommé `glpi_id` (int, clé primaire `glpi_users.id`) (v2.4.0) |
| AJF-03 | `id` du ticket absent de `active_incidents_list` | Champ `id` (int, clé primaire `glpi_tickets`) ajouté dans chaque objet (v2.4.0) |
| AJF-04 | `active_incidents_list` limitée à 10 tickets | Limite ramenée à 3 sur demande expert Genesys (v2.4.0) |

