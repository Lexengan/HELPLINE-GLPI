# Plugin GLPI v11 — Helpline WebService

Plugin GLPI 11 permettant à **Genesys Cloud** d'interagir avec GLPI via des appels REST simples, sans orchestration côté Genesys.

## Objectif

L'API native GLPI v11 ne peut pas être exploitée directement par Genesys dans les conditions définies par le contrat UDJ v3 :

- Certaines actions nécessitent plusieurs appels API en séquence (ex. : créer un ticket fils = créer le ticket + créer le lien père/fils), ce que Genesys ne peut pas orchestrer.
- Certaines actions métier n'existent pas nativement dans GLPI v11 (ex. : notion d'incident général avec catégorie spécifique).

Le plugin résout ces deux problèmes en appliquant un principe fondamental :

> **1 action Genesys = 1 requête HTTP = 1 réponse JSON.**
> Toute complexité interne est absorbée par le plugin et invisible pour Genesys.

## Utilisation

Le plugin est appelé par **Genesys Cloud** lors des parcours d'appels utilisateurs. Il n'expose pas d'interface graphique dans GLPI.

Les requêtes sont authentifiées via **OAuth2 Password Grant** (compte technique GLPI). Le token est obtenu sur `/api.php/token` et transmis en header `Authorization: Bearer <token>` sur chaque appel.

Les endpoints sont accessibles sous : `http://<host>/api.php/v2/Helpline/<action>`

| Endpoint | Méthode | Rôle |
|---|---|---|
| `get_user_infos` | GET | Identification de l'appelant + liste de ses tickets actifs |
| `create_parent_incident` | POST | Création d'un incident général (Incident Majeur) |
| `create_child_incident` | POST | Rattachement d'un ticket fils à un incident général |
| `relaunch_inbound` | POST | Enregistrement d'une relance entrante (utilisateur → Service Desk) |
| `relaunch_outbound` | POST | Enregistrement d'une relance sortante (Service Desk → utilisateur) |
| `add_note` | POST | Ajout d'une note libre sur un ticket |
| `incident_status` | GET | Consultation du statut actif/inactif d'un ticket |
| `close_incident` | POST | Clôture d'un ticket (création d'une Solution ITIL) |

La documentation complète des contrats d'interface (paramètres, réponses, codes d'erreur) est dans `Documentation_API_ITSM_GLPI_V1_2_UDJ.docx`.

## Structure

```
helpline/
├── config/
│   └── config.php        # Paramètres d'instance : MAJOR_CATEGORY_ID, SOLUTION_TYPE_ID
├── src/
│   └── ApiController.php # Toutes les routes #[Route] + logique métier
├── hook.php              # Callbacks install / uninstall (pas de table SQL)
└── setup.php             # Déclaration du plugin + enregistrement hook API_CONTROLLERS
```

### `setup.php`

Déclare le plugin auprès de GLPI et enregistre `ApiController` dans le Router High-Level via le hook `Hooks::API_CONTROLLERS`. C'est ce hook qui rend les routes disponibles sous `/api.php/v2/` avec l'authentification OAuth2 native GLPI, sans gestion de session manuelle.

### `src/ApiController.php`

Contient l'ensemble des routes et de la logique métier. Chaque méthode publique porte les attributs `#[Route]` et `#[RouteVersion(introduced: '2.0')]` — ce second attribut est obligatoire sous GLPI 11, son absence provoque un HTTP 500 sur toute l'API.

### `config/config.php`

Trois constantes à adapter pour chaque instance GLPI avant mise en production :

```php
define('PLUGIN_HELPLINE_GENESYS_TOKEN',     '');  // token Bearer partagé avec Genesys
define('PLUGIN_HELPLINE_MAJOR_CATEGORY_ID', 0);   // ID catégorie "Incident Majeur" (0 = résolution auto par nom)
define('PLUGIN_HELPLINE_SOLUTION_TYPE_ID',  0);   // ID type de solution (0 = premier type is_incident=1)
```

### `hook.php`

Callbacks d'installation et de désinstallation. Le plugin ne crée pas de table SQL — il s'appuie exclusivement sur les objets GLPI natifs (`Ticket`, `ITILFollowup`, `ITILSolution`, `User`).

## Références

| Document | Contenu |
|---|---|
| `Documentation_API_ITSM_GLPI_V1_2_UDJ.docx` | Contrats d'interface complets (UDJ v3 adapté GLPI v11) |
| `Helpline_WebService_Documentation_Technique_v2.0.docx` | Architecture, blocages historisés, résultats de tests |
