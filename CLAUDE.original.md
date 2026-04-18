# CLAUDE.md — SOPROMER Custom Addons (Odoo 18)

## Projet

Client **SOPROMER** (distribution produits de la mer, Madagascar). Odoo 18 en production sur `odoo.sopromer.mg`. Migré depuis Sage 100 Cloud par import Excel manuel.

- **Prod** : serveur `192.73.0.43`, container `odoo-dev`, base `SOPROMER`
- **Test** : serveur `192.73.0.45`, container `odoo-dev`, copies de base
- **GitHub** : tous les modules sous `Lalaina710/<module-name>`
- **Déploiement** : `/opt/odoo18/custom_addons/dev/` sur les serveurs

## Conventions de code

### Structure module Odoo 18

```
module_name/
├── __init__.py
├── __manifest__.py
├── models/
│   ├── __init__.py
│   └── *.py
├── views/
│   └── *.xml
├── security/
│   └── ir.model.access.csv
├── static/
│   └── src/
│       ├── js/
│       ├── xml/
│       └── css/
├── data/
├── README.md
└── .gitignore
```

### Python

- Odoo 18 API : `@api.depends`, `@api.onchange`, `@api.constrains`, `@api.model`
- Héritage : `_inherit = 'model.name'` (extension) ou `_name + _inherit` (nouveau modèle)
- Pas de `sudo()` sauf nécessité absolue documentée
- Toujours définir `ir.model.access.csv` pour les nouveaux modèles
- Fields : utiliser `fields.Selection` plutôt que `fields.Char` quand les valeurs sont finies

### XML Views

- `noupdate="1"` pour les données qui ne doivent pas être écrasées à la mise à jour
- IDs : `<module>.<model>_<description>` (ex: `pos_dashboard.action_pos_dashboard`)
- Héritage : `<xpath expr="..." position="after|before|replace|inside">`

### JavaScript / OWL 2 (POS)

- Composants OWL 2 : `patch()` pour étendre les composants existants
- Imports : `import { ... } from "@point_of_sale/..."` ou `@web/...`
- Templates : fichiers `.xml` dans `static/src/xml/`
- Registrer dans `__manifest__.py` → `assets` → `point_of_sale._assets_pos`

### Manifeste

```python
{
    'name': 'Module Name',
    'version': '18.0.1.0.0',
    'category': 'Category',
    'summary': 'One-line summary',
    'author': 'SOPROMER',
    'website': 'https://github.com/Lalaina710/<module>',
    'license': 'LGPL-3',
    'depends': ['base', ...],
    'data': [...],
    'assets': {...},
    'installable': True,
    'auto_install': False,
}
```

- Version : `18.0.X.Y.Z` — incrémenter Y pour features, Z pour fixes
- License : `LGPL-3` sauf indication contraire

## Workflow de travail

### Serveur PROD (43) — CRITIQUE

- **JAMAIS** redémarrer pendant les heures de travail (115 utilisateurs + 40 sessions PdV)
- Tester d'abord sur serveur 45 (test)
- Copier les fichiers sur 43 mais reporter le restart au soir
- Commande restart : `docker restart odoo-dev`

### Serveur TEST (45)

- Ne **jamais** mettre `db_name` dans la config — ça bloque le database manager
- Utiliser `dbfilter` ou `-d <base>` en CLI à la place
- Safe pour expérimenter

### Déploiement module

1. Développer et tester en local
2. Push sur GitHub `Lalaina710/<module>`
3. Déployer sur serveur 45, tester
4. Copier sur serveur 43, restart le soir
5. Vérifier en prod le lendemain

### Git

- Chaque module = un repo GitHub séparé sous `Lalaina710/`
- Toujours un `README.md` et `.gitignore` (`__pycache__/`, `*.pyc`)
- Conventional commits : `feat:`, `fix:`, `refactor:`, `docs:`
- Nouveau repo : `gh repo create Lalaina710/<nom> --public`

## Agents et Skills spécialisés

### Repo odoo-skills

Tous les agents et skills Odoo sont centralisés dans **`Lalaina710/odoo-skills`** :
- Clone local : `C:\Users\User\AppData\Local\Temp\odoo-skills`
- Agents actifs : `~/.claude/agents/` (copier depuis le repo pour activer)

**Quand un agent spécialisé est lancé, il DOIT utiliser le skill correspondant** via l'outil `Skill` avant de commencer le travail.

### Agents disponibles

| Agent | Skill associé | Usage |
|-------|---------------|-------|
| `odoo-backend` | `odoo-business-logic` | Logique métier Python, ORM, workflows |
| `odoo-frontend` | `odoo-owl-frontend` / `odoo-views-and-ui` | OWL 2, JS, XML views, POS |
| `odoo-architect` | `odoo-module-scaffold` / `odoo-model-design` | Conception module, modèles |
| `odoo-qa` | `odoo-functional-testing` | Tests, validation, debugging |
| `odoo-devops` | `odoo-devops-infra` | Docker, Nginx, déploiement |
| `odoo-data` | `odoo-data-migration` | Import/migration données |
| `odoo-sage-migrator` | `odoo-sage-migration` | Migration Sage 100 → Odoo |
| `odoo-ops-diagnostic` | `odoo-ops-diagnostic` | Diagnostic incidents POS/stock/factures |
| `odoo-simplifier` | `odoo-code-simplifier` | Nettoyage code Odoo |
| `odoo-publisher` | `odoo-apps-store` / `odoo-quality-and-publish` | Publication Odoo Apps Store |
| `odoo-upgrader` | `odoo-version-upgrade` | Migration de version Odoo |
| `odoo-tester` | `odoo-functional-testing` | Tests fonctionnels bout en bout |
| `n8n-automator` | `n8n-automation` | Automatisations n8n/Metabase/Grafana |
| `code-reviewer` | — | Revue après implémentation |
| **`sopromer-pilot`** | **orchestrateur** | **Pilotage projet, coordination agents, planification déploiements** |

### Skills disponibles (17)

`odoo-module-scaffold`, `odoo-model-design`, `odoo-views-and-ui`, `odoo-business-logic`, `odoo-data-migration`, `odoo-quality-and-publish`, `odoo-owl-frontend`, `odoo-api-integrations`, `odoo-performance-debug`, `odoo-apps-store`, `odoo-devops-infra`, `odoo-version-upgrade`, `odoo-sage-migration`, `odoo-functional-testing`, `odoo-code-simplifier`, `odoo-ops-diagnostic`, `n8n-automation`

### Règles

- Toujours étiqueter l'agent dans les commits (`Co-Authored-By`)
- Lancer en parallèle quand la tâche est mixte (backend + frontend)
- Nouveau skill/agent → le créer dans le repo `odoo-skills`, commit + push, puis copier dans `~/.claude/agents/`

## Modules installés (contexte)

### Custom (ce dossier)

Dashboards (6), POS (5), MRP (4), Stock (2), Sécurité (1), Compta/Ventes (3), Technique (3)

### Tiers (`third-party-addons/`)

- `stock_no_negative` — bloque stock négatif
- `pos_lot_selection` — sélection lot PdV (OCA)
- `low_stocks_product_alert` — alerte stock bas visuelle
- `sensible_pos_access_rights_employee` — contrôle accès PdV par employé
- `access_roles` — groupes/rôles utilisateurs
- `om_account_followup` — relances factures

## Rappels importants

- Les problèmes stock/facturation viennent de la migration Sage bâclée, pas de la config Odoo
- Suivi par lots activé (conventions : `RO26M03A01`, `SR-24M00-RC26000`)
- Multi-entrepôts : CFMP23, CFPF23, CFC, MG04, E01/AKA, E06/PO, etc.
- Screenshots bugs : `C:\Users\User\Desktop\bugs\`
- Toujours auditer avant toute manipulation destructive
