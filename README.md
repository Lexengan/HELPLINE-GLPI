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
| Plugin   | 2.6.1          |

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

## Pré-requis fonctionnels — plugins externes (VIP & Incidents de masse)

Les champs `vip`/`vip_group` et `mass_incidents_count`/`mass_incidents_list` retournés par
`get_user_infos` dépendent de **deux plugins GLPI tiers**, optionnels. S'ils sont absents, le
contrôleur les détecte via `class_exists()` ou `$DB->tableExists()` et retourne des valeurs par
défaut (`vip: false`, `mass_incidents_count: 0`, `mass_incidents_list: []`) sans erreur.

### Statut VIP — plugin InfotelGLPI VIP

1. Installer et activer le plugin **VIP** (InfotelGLPI) dans Configuration > Plugins.
2. Dans la configuration du plugin VIP, **activer l'attribut VIP sur un ou plusieurs groupes
   d'utilisateurs** (ex. groupe "Direction", "VIP Genesys", etc.).
3. Tout utilisateur **membre d'un de ces groupes** est alors considéré comme VIP :
   - côté GLPI (priorisation visuelle, règles métier du plugin VIP) ;
   - côté Genesys, via `get_user_infos` qui retourne `"vip": true` et `"vip_group": "<nom du
     groupe>"` (le contrôleur appelle `GlpiPlugin\Vip\Ticket::isUserVip()` puis
     `GlpiPlugin\Vip\Group::getVipName()`, cf. AJF-05).

Sans ce plugin (ou sans groupe VIP configuré), `get_user_infos` retourne systématiquement
`"vip": false` et `"vip_group": ""`, et `create_parent_incident` n'applique pas la priorité
4/urgence 4 réservée aux VIP (AJF-07).

### Incidents de masse — plugin Fields

1. Installer et activer le plugin **Fields** dans Configuration > Plugins.
2. Dans Configuration > Fields, créer un **bloc de champs** (« container ») nommé par exemple
   **"Incident masse"**, rattaché à l'objet **Ticket** (`itemtype = Ticket`). Ce bloc ajoute un
   nouvel onglet sur la fiche ticket.
3. Dans ce bloc, ajouter un champ :
   - **Libellé** : `Ticket père`
   - **Type** : `Oui/Non` (Yes/No)
   - **Valeur par défaut** : aucune (ne pas en définir)
   - **Obligatoire** : Non
   - **Lecture seule** : Non

   Ce champ est stocké en base par le plugin Fields dans une table dédiée au container —
   typiquement `glpi_plugin_fields_ticketincidentmasses`, colonne `ticketprefield`
   (0 = Non, 1 = Oui).
4. Un ticket marqué `ticketprefield = 1` ("Ticket père" = Oui) est considéré comme un
   **Incident Majeur / ticket père** par `getMassIncidents()`. Pour un utilisateur donné,
   le contrôleur recherche les tickets pères (`ticketprefield=1`) auxquels l'un de ses tickets
   est rattaché en tant que ticket fils (lien `link=SON_OF`, cf. B-36), et les expose dans
   `mass_incidents_count` (entier) et `mass_incidents_list` (tableau d'objets ticket, AJF-06).

Sans ce plugin (table `glpi_plugin_fields_ticketincidentmasses` absente), `get_user_infos`
retourne `"mass_incidents_count": 0` et `"mass_incidents_list": []`.

> ⚠️ Le **nom exact du container/bloc** créé dans Fields n'est pas vérifié par le code — seule
> la table générée (`glpi_plugin_fields_ticketincidentmasses`) et la colonne `ticketprefield`
> comptent. Si le bloc est créé sous un autre nom, vérifier que le plugin Fields génère bien
> cette table/colonne (le nom de table dépend du nom technique donné au container lors de sa
> création).

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
| `get_user_infos` | POST | Identification appelant + tickets actifs + incidents majeurs | `user_name` ou `email` ou `phone` |
| `create_parent_incident` | POST | Création d'un Incident Majeur | `user_name`, `title`, `content` |
| `create_child_incident` | POST | Rattachement ticket fils à un Incident Majeur | `user_name`, `ticket_number` |
| `relaunch_inbound` | POST | Relance entrante (utilisateur → Service Desk) | `ticket_number` |
| `relaunch_outbound` | POST | Relance sortante (Service Desk → utilisateur) | `ticket_number` |
| `add_note` | POST | Note libre sur un ticket | `ticket_number`, `ticket_note` |
| `incident_status` | POST | Statut actif/inactif + label textuel du statut | `ticket_number` |
| `close_incident` | POST | Clôture avec gabarit de solution | `ticket_number`, `user_name` |

> `get_user_infos` retourne au maximum **3 tickets actifs** dans `active_incidents_list` et les incidents majeurs concernant l'utilisateur dans `mass_incidents_list`.
> Chaque ticket inclut un champ `ticket_number` (int) utilisable directement dans les appels suivants.
> Le champ `vip` retourne le statut réel via le plugin VIP (toujours `false` si plugin absent). Le champ `vip_group` retourne le nom du groupe VIP.

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
| SEC-13 | Droits insuffisants sur items — `Session::haveRight()` global insuffisant | `canView()` + `canUpdate()` ajoutés après `getFromDB()` sur toutes les routes ticket (v2.5.1) |
| SEC-14 | `error_log()` de debug en production dans `Config.php` | 3 appels supprimés (v2.5.1) |

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
| AJF-05 | `vip` retournait toujours `false` | Statut réel via `GlpiPlugin\\Vip\\Ticket::isUserVip()` + champ `vip_group` (v2.5.0) |
| AJF-06 | `mass_incidents_list` toujours vide | Alimentée via tickets fils → pères avec `ticketprefield=1` dans plugin Fields (v2.5.0) |
| AJF-07 | Ticket père VIP sans priorité adaptée | `priority=4` / `urgency=4` si utilisateur VIP dans `create_parent_incident` (v2.5.0) |
| AJF-08 | Ticket fils héritait de la priorité du père | `priority=2` / `urgency=2` fixe — le père est prioritaire (v2.5.0) |
| AJF-09 | `id` et `number` dans les listes de tickets retournaient la même valeur | Fusionnés en `ticket_number` (int) — cohérent avec tous les autres endpoints (v2.5.1) |
| AJF-10 | `incident_status` ne retournait pas le statut textuel | Ajout du champ `ticket_status` avec le label GLPI en clair (v2.5.1) |
| SEC-13 | `canView()`/`canUpdate()` absents après `getFromDB()` | Ajoutés sur toutes les routes ticket — `Session::haveRight()` seul insuffisant (v2.5.1) |
| SEC-14 | 3 `error_log()` de debug dans `Config.php` en production | Supprimés (v2.5.1) |
| B-34 | Routes POST sans `#[Doc\\Route(parameters:[...])]` → body vide HTTP 500, aucune trace PHP | Ajout `use Glpi\\Api\\HL\\Doc as Doc;` + `#[Doc\\Route]` sur les 8 méthodes publiques (v2.6.1) |
| B-35 | `getMassIncidents()` — `'SELECT' => ['DISTINCT col AS alias']` génère du SQL invalide | `'SELECT' => ['col AS alias'], 'DISTINCT' => true` (clé top-level séparée) (v2.6.1) |
| B-36 | Convention `glpi_tickets_tickets` pour `link=SON_OF(3)` inversée dans `hasActiveChild()`/`getMassIncidents()` | `tickets_id_1`=ENFANT, `tickets_id_2`=PÈRE — vérifié via `CommonITILObject_CommonITILObject::add()` (v2.6.1) |
| B-37 | `createChildIncident()` — création du lien `Ticket_Ticket` avec `tickets_id_1`/`tickets_id_2` inversés (oubli lors de B-36) | `tickets_id_1 => $childId` (ENFANT), `tickets_id_2 => $parentNumber` (PÈRE) (v2.6.1, **validé en base le 12/06/2026** : ticket fils 1231 → père 298, id=515, link=3) |
