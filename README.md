# Plugin GLPI 11 — Helpline WebService

Plugin GLPI 11 permettant à **Genesys Cloud** d'interagir avec GLPI via des appels REST simples, sans orchestration côté Genesys.

## Objectif

L'API native GLPI v11 ne peut pas être exploitée directement par Genesys dans les conditions définies par le contrat UDJ v3 :

- Certaines actions nécessitent plusieurs appels API en séquence (ex. : créer un ticket fils = créer le ticket + créer le lien père/fils), ce que Genesys ne peut pas orchestrer.
- Certaines actions métier n'existent pas nativement dans GLPI v11 (ex. : notion d'incident général avec catégorie spécifique).

Le plugin résout ces deux problèmes en appliquant un principe fondamental :

> **1 action Genesys = 1 requête HTTP = 1 réponse JSON.**
> Toute complexité interne est absorbée par le plugin et invisible pour Genesys.

## Compatibilité

| Élément | Version |
|---|---|
| GLPI | 11.0.0 → 11.x |
| PHP | 8.1+ |
| Plugin | 2.0.1 |

## Installation

1. Déposer le dossier `helpline/` dans le répertoire `plugins/` de l'instance GLPI
2. Dans GLPI : **Configuration > Plugins** → Installer puis Activer **Helpline WebService**
3. Configurer le plugin depuis **Configuration > Générale > onglet "Helpline WebService"**

> Aucun accès SSH ni modification de fichier n'est requis après l'installation.

## Configuration

La configuration s'effectue entièrement depuis l'interface GLPI, sans accès au serveur :

**Configuration > Générale > onglet "Helpline WebService"**

| Paramètre | Description |
|---|---|
| Catégorie ITIL — Incident Majeur | Catégorie affectée automatiquement lors de la création d'un incident majeur via Genesys. Si non définie, le plugin recherche une catégorie nommée "Incident Majeur". |
| Gabarit de solution — Clôture Genesys | Gabarit de solution (`glpi_solutiontemplates`) utilisé pour pré-remplir la solution lors de la clôture d'un ticket via Genesys. Si aucun gabarit n'est sélectionné, un contenu libre générique est utilisé. |

> Les valeurs sont stockées dans `glpi_configs` (context `plugin:helpline`) et ne nécessitent aucun accès SSH.

## Authentification OAuth2

Les requêtes sont authentifiées via **OAuth2 Password Grant**. Un client OAuth2 doit être configuré dans GLPI avec le scope `api` et le grant `password`.

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

> Le compte GLPI utilisé doit disposer du droit **UPDATE sur les tickets**. Tout appel avec un compte sans ce droit retourne HTTP 403.

## Endpoints

Les endpoints sont accessibles sous : `(https://<url_instance_glpi>/api.php/v2/Helpline/<action>)`

| Endpoint | Méthode | Rôle | Paramètres |
|---|---|---|---|
| `get_user_infos` | GET | Identification de l'appelant + liste de ses tickets actifs | `user_name` ou `email` ou `phone` |
| `create_parent_incident` | POST | Création d'un Incident Majeur | `user_name`, `title`, `content` |
| `create_child_incident` | POST | Rattachement d'un ticket fils à un Incident Majeur | `user_name`, `ticket_number` |
| `relaunch_inbound` | POST | Enregistrement d'une relance entrante (utilisateur → Service Desk) | `ticket_number` |
| `relaunch_outbound` | POST | Enregistrement d'une relance sortante (Service Desk → utilisateur) | `ticket_number` |
| `add_note` | POST | Ajout d'une note libre sur un ticket | `ticket_number`, `ticket_note` |
| `incident_status` | GET | Consultation du statut actif/inactif d'un ticket | `ticket_number` |
| `close_incident` | POST | Clôture d'un ticket via gabarit de solution | `ticket_number`, `user_name` |

### Format des réponses

Toutes les routes retournent un JSON avec au minimum :

```json
{
  "response_status": "SUCCESS" | "ERROR",
  "response_note": "CODE_RETOUR"
}
```

| Code retour | Signification |
|---|---|
| `USER_FOUND` | Utilisateur trouvé |
| `USER_NOT_FOUND` | Utilisateur introuvable |
| `PARENT_CREATED` | Incident Majeur créé |
| `CHILD_CREATED` | Ticket fils créé et lié |
| `FOLLOWUP_ADDED` | Suivi ajouté au ticket |
| `TICKET_ACTIVE` | Ticket en cours |
| `TICKET_INACTIVE` | Ticket résolu ou clôturé |
| `TICKET_CLOSED` | Ticket clôturé avec succès |
| `FORBIDDEN` | Droits insuffisants (HTTP 403) |
| `TICKET_NOT_FOUND` | Ticket introuvable |
| `MISSING_PARAM` | Paramètre obligatoire manquant |

## Structure du plugin

```
helpline/
├── src/
│   ├── ApiController.php   # Toutes les routes #[Route] + logique métier
│   └── Config.php          # Interface d'administration GLPI
├── templates/
│   └── config.html.twig    # Réservé — rendu HTML géré directement dans Config.php
├── hook.php                # Callbacks install / uninstall
├── setup.php               # Déclaration du plugin + hooks GLPI
└── README.md               # Ce fichier
```

### `setup.php`

Déclare le plugin auprès de GLPI et enregistre `ApiController` dans le Router High-Level via `Hooks::API_CONTROLLERS`. Enregistre également `Config` pour l'onglet d'administration. Les routes sont disponibles sous `/api.php/v2/` avec l'authentification OAuth2 native GLPI.

### `src/ApiController.php`

Contient l'ensemble des routes et de la logique métier. Chaque méthode publique porte les attributs `#[Route]` et `#[RouteVersion(introduced: '2.0')]`. Chaque route vérifie `Session::haveRight('ticket', UPDATE)` avant toute action.

### `src/Config.php`

Classe d'administration enregistrée sur l'objet `Config` de GLPI. Affiche l'onglet de configuration dans **Configuration > Générale**. Les valeurs sont stockées dans `glpi_configs` (context `plugin:helpline`) via `Config::setConfigurationValues`. Le rendu est fait en HTML PHP natif sans `TemplateRenderer`.

### `hook.php`

Callbacks d'installation et de désinstallation. À l'installation, initialise les valeurs de configuration en base. À la désinstallation, supprime proprement toutes les entrées du contexte `plugin:helpline`. Le plugin ne crée aucune table SQL — il s'appuie exclusivement sur les objets GLPI natifs (`Ticket`, `ITILFollowup`, `ITILSolution`, `Ticket_Ticket`, `User`).

## Conformité TECLIB

| Exigence | Statut |
|---|---|
| Configuration sans accès SSH |  Via onglet GLPI natif |
| Aucune requête SQL brute |  `$DB->request()` exclusivement |
| Contrôle des droits sur toutes les routes |  `Session::haveRight('ticket', UPDATE)` |
| Pas de fichiers front/ajax non protégés |  Plugin 100% API, aucun fichier front |
| Twig obligatoire |  Non applicable — aucun rendu HTML front |
| Compatible GLPI 11.x |  Testé sur GLPI 11.0.2 |
| Désinstallation propre |  Suppression complète de `glpi_configs` |

## Blocages techniques historisés

| ID | Description | Solution appliquée |
|---|---|---|
| B-11 | `state=1` en base malgré message "erreur" UI | Faux positif GLPI 11 — ignoré |
| B-12 | Redirection vers login sans `Accept: application/json` | Header obligatoire côté Genesys |
| B-13 | `SessionExpiredException` avec `front/api.php` legacy | Architecture Router HL via `API_CONTROLLERS` |
| B-18 | HTTP 500 sur toute l'API sans `#[RouteVersion]` | Attribut ajouté sur chaque méthode publique |
| B-19 | `is_deleted` absent de `glpi_itilcategories` en GLPI 11 | Filtre supprimé |
| B-20 | `glpi_solutiontypes` ≠ gabarits de solution | Remplacement par `glpi_solutiontemplates` |
| B-21 | Scope OAuth2 non transmis au token | Ajout `&scope=api` dans la requête de token |
| B-22 | `TemplateRenderer` namespace `@helpline` non résolu | Rendu HTML PHP natif dans `Config.php` |

## Références

| Document | Contenu |
|---|---|
| `Documentation_API_ITSM_GLPI_V1_2_UDJ.docx` | Contrats d'interface complets (UDJ v3 adapté GLPI v11) |
| [Documentation développeur GLPI](https://glpi-developer-documentation.readthedocs.io/) | Standards de développement GLPI |
