# CLAUDE.md — SOPROMER Custom Addons (Odoo 18)

## Projet

**SOPROMER** — distribution produits mer, Madagascar. Odoo 18 prod `odoo.sopromer.mg`. Migré Sage 100 Cloud via import Excel.

- **Prod** : `192.73.0.43`, container `odoo-dev`, base `SOPROMER`
- **Test** : `192.73.0.45`, container `odoo-dev`, copies
- **GitHub** : `Lalaina710/<module-name>`
- **Deploy path** : `/opt/odoo18/custom_addons/dev/`

## Conventions code

### Structure module

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

- API : `@api.depends`, `@api.onchange`, `@api.constrains`, `@api.model`
- Héritage : `_inherit = 'model.name'` (extension) ou `_name + _inherit` (nouveau)
- Pas `sudo()` sauf nécessité documentée
- Toujours `ir.model.access.csv` pour nouveaux modèles
- `fields.Selection` > `fields.Char` quand valeurs finies

### XML Views

- `noupdate="1"` pour données non écrasables
- IDs : `<module>.<model>_<description>`
- Héritage : `<xpath expr="..." position="after|before|replace|inside">`

### JavaScript / OWL 2 (POS)

- `patch()` pour étendre composants existants
- Imports : `@point_of_sale/...` ou `@web/...`
- Templates `.xml` dans `static/src/xml/`
- Enregistrer via `__manifest__.py` → `assets` → `point_of_sale._assets_pos`

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

- Version `18.0.X.Y.Z` — Y=features, Z=fixes
- License `LGPL-3` par défaut

## Workflow

### PROD (43) — CRITIQUE

- **JAMAIS** restart pendant heures travail (115 users + 40 PdV)
- Tester sur 45 d'abord
- Copier sur 43, restart au soir : `docker restart odoo-dev`

### TEST (45)

- **Jamais** `db_name` dans config — bloque database manager
- Utiliser `dbfilter` ou `-d <base>` en CLI
- Safe pour expérimenter

### Déploiement

1. Dev + test local
2. Push GitHub `Lalaina710/<module>`
3. Deploy 45, tester
4. Copier 43, restart soir
5. Vérifier prod lendemain

Script rapide : `./scripts/deploy.sh <module> <45|43>`

### Git

- 1 module = 1 repo GitHub `Lalaina710/`
- README + `.gitignore` (`__pycache__/`, `*.pyc`) obligatoires
- Conventional commits : `feat:`, `fix:`, `refactor:`, `docs:`
- Nouveau repo : `gh repo create Lalaina710/<nom> --public`

## Agents et Skills

### Repo odoo-skills

Centralisé dans **`Lalaina710/odoo-skills`** :
- Local : `C:\Users\User\AppData\Local\Temp\odoo-skills`
- Actifs : `~/.claude/agents/`

**Règle : chaque agent DOIT invoquer son skill via `Skill` avant de coder.**

### Agents

| Agent | Skill | Usage |
|-------|-------|-------|
| `odoo-backend` | `odoo-business-logic` | Python, ORM, workflows |
| `odoo-frontend` | `odoo-owl-frontend` / `odoo-views-and-ui` | OWL 2, JS, XML, POS |
| `odoo-architect` | `odoo-module-scaffold` / `odoo-model-design` | Conception module |
| `odoo-qa` | `odoo-functional-testing` | Tests, validation |
| `odoo-devops` | `odoo-devops-infra` | Docker, Nginx, deploy |
| `odoo-data` | `odoo-data-migration` | Import/migration données |
| `odoo-sage-migrator` | `odoo-sage-migration` | Migration Sage → Odoo |
| `odoo-ops-diagnostic` | `odoo-ops-diagnostic` | Incidents POS/stock/factures |
| `odoo-simplifier` | `odoo-code-simplifier` | Nettoyage code |
| `odoo-publisher` | `odoo-apps-store` / `odoo-quality-and-publish` | Publication Apps Store |
| `odoo-upgrader` | `odoo-version-upgrade` | Migration version |
| `odoo-tester` | `odoo-functional-testing` | Tests fonctionnels E2E |
| `n8n-automator` | `n8n-automation` | n8n/Metabase/Grafana |
| `odoo-doc-writer` | `odoo-documentation` | Manuels, docs techniques, slides formation |
| `code-reviewer` | — | Revue code |
| **`sopromer-pilot`** | **orchestrateur** | **Pilotage projet, coordination** |

### Skills (18)

`odoo-module-scaffold`, `odoo-model-design`, `odoo-views-and-ui`, `odoo-business-logic`, `odoo-data-migration`, `odoo-quality-and-publish`, `odoo-owl-frontend`, `odoo-api-integrations`, `odoo-performance-debug`, `odoo-apps-store`, `odoo-devops-infra`, `odoo-version-upgrade`, `odoo-sage-migration`, `odoo-functional-testing`, `odoo-code-simplifier`, `odoo-ops-diagnostic`, `n8n-automation`, `odoo-documentation`

### Règles

- Étiqueter agent dans commits (`Co-Authored-By`)
- Paralléliser agents indépendants (backend + frontend)
- Nouveau skill/agent → repo `odoo-skills` → push → copier `~/.claude/agents/`

## Modules installés

**Custom (ce dossier)** : Dashboards (6), POS (5), MRP (4), Stock (2), Sécurité (1), Compta/Ventes (3), Technique (3)

**Tiers** (`third-party-addons/`) : `stock_no_negative`, `pos_lot_selection`, `low_stocks_product_alert`, `sensible_pos_access_rights_employee`, `access_roles`, `om_account_followup`

## Rappels

- Problèmes stock/facturation = migration Sage bâclée, pas config Odoo
- Lots activés : `RO26M03A01`, `SR-24M00-RC26000`
- Multi-entrepôts : CFMP23, CFPF23, CFC, MG04, E01..E55
- Screenshots bugs : `C:\Users\User\Desktop\bugs\`
- Auditer avant toute manipulation destructive
