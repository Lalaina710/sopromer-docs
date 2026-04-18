# SOPROMER Report Layout Module — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create a custom Odoo 18 module `sopromer_report_layout` that provides a unified, configurable report layout (header SOPROMER, client block "DOIT:", footer, signatures) for all SOPROMER documents: factures, BL, BC achat, BC vente, ordres de fabrication.

**Architecture:** Approche B — Un layout de base QWeb partagé (`sopromer_external_layout`) remplace le layout standard Odoo. Chaque rapport métier hérite de ce layout et définit uniquement ses colonnes spécifiques. Des champs configurables sur `res.company` permettent d'activer/désactiver des blocs (signatures, conditions, mentions légales) sans toucher au code.

**Tech Stack:** Odoo 18, QWeb templates, Python (models.Model), XML (views/reports), SCSS (styling)

---

## File Structure

```
sopromer_report_layout/
├── __manifest__.py                          # Module manifest
├── __init__.py                              # Root init
├── models/
│   ├── __init__.py
│   └── res_company.py                       # Champs config layout sur res.company
├── views/
│   ├── res_company_views.xml                # Onglet "Rapport SOPROMER" dans Company form
│   └── res_config_settings_views.xml        # Expose config dans Settings (optionnel)
├── report/
│   ├── sopromer_layout.xml                  # Layout de base: header, footer, signatures
│   ├── report_invoice.xml                   # Override facture client + avoir
│   ├── report_sale_order.xml                # Override devis / BC vente
│   ├── report_purchase_order.xml            # Override BC achat / demande de prix
│   ├── report_delivery_slip.xml             # Override bon de livraison
│   └── report_mrp_production.xml            # Override ordre de fabrication
├── static/
│   └── src/
│       └── scss/
│           └── sopromer_report.scss          # Styles SOPROMER (polices, couleurs, marges)
├── data/
│   └── paperformat.xml                      # Format papier SOPROMER A4
└── security/
    └── ir.model.access.csv                  # Accès (si nouveaux modèles)
```

---

## Task 1: Scaffold du module — manifest et structure

**Files:**
- Create: `sopromer_report_layout/__manifest__.py`
- Create: `sopromer_report_layout/__init__.py`
- Create: `sopromer_report_layout/models/__init__.py`

- [ ] **Step 1: Créer `__manifest__.py`**

```python
{
    "name": "SOPROMER - Report Layout",
    "version": "18.0.1.0.0",
    "category": "Reporting",
    "summary": "Mise en page personnalisée des rapports SOPROMER (factures, BL, BC, OF)",
    "description": """
        Layout de rapports unifié pour SOPROMER :
        - En-tête avec infos société (NIF, STAT, RC, CIF)
        - Bloc client "DOIT:" avec infos fiscales
        - Colonnes métier (Nb/Colis, nombre_sacs)
        - Zones de signatures configurables
        - Pied de page avec mentions légales et conditions
    """,
    "author": "MadaWebZone MWZ",
    "website": "https://github.com/Lalaina710",
    "license": "OPL-1",
    "depends": [
        "account",
        "sale",
        "purchase",
        "stock",
        "mrp",
    ],
    "data": [
        "data/paperformat.xml",
        "views/res_company_views.xml",
        "report/sopromer_layout.xml",
        "report/report_invoice.xml",
        "report/report_sale_order.xml",
        "report/report_purchase_order.xml",
        "report/report_delivery_slip.xml",
        "report/report_mrp_production.xml",
    ],
    "assets": {
        "web.report_assets_common": [
            "sopromer_report_layout/static/src/scss/sopromer_report.scss",
        ],
    },
    "installable": True,
    "application": False,
    "auto_install": False,
}
```

- [ ] **Step 2: Créer `__init__.py`**

```python
from . import models
```

- [ ] **Step 3: Créer `models/__init__.py`**

```python
from . import res_company
```

- [ ] **Step 4: Commit**

```bash
cd C:/odoo18/custom-addons/sopromer_report_layout
git init
git add __manifest__.py __init__.py models/__init__.py
git commit -m "init: scaffold sopromer_report_layout module"
```

---

## Task 2: Champs de configuration sur res.company

**Files:**
- Create: `sopromer_report_layout/models/res_company.py`
- Create: `sopromer_report_layout/views/res_company_views.xml`

- [ ] **Step 1: Créer le modèle res_company.py avec les champs**

```python
from odoo import fields, models


class ResCompany(models.Model):
    _inherit = "res.company"

    # --- Infos fiscales SOPROMER (en-tête) ---
    sopromer_nif = fields.Char(string="NIF")
    sopromer_stat = fields.Char(string="N° Stat")
    sopromer_rc = fields.Char(string="RC")
    sopromer_cif = fields.Char(string="CIF")

    # --- Options d'affichage rapport ---
    sopromer_show_signatures = fields.Boolean(
        string="Afficher les zones de signature",
        default=True,
    )
    sopromer_signature_labels = fields.Char(
        string="Libellés signatures (séparés par ;)",
        default="Le Client;Le Service Commercial;Le Magasinier",
        help="Libellés des zones de signature, séparés par des point-virgules. Ex: Le Client;Le Commercial;Le Magasinier",
    )
    sopromer_show_payment_conditions = fields.Boolean(
        string="Afficher conditions de règlement",
        default=True,
    )
    sopromer_legal_mention = fields.Text(
        string="Mention légale",
        default="Dans le cas où le paiement intégral n'interviendrait pas à la date prévue par les parties, le vendeur se réserve le droit de reprendre la chose livrée et de résoudre le contrat.",
    )
    sopromer_show_amount_in_words = fields.Boolean(
        string="Afficher montant en lettres",
        default=True,
    )
    sopromer_report_title_invoice = fields.Char(
        string="Titre rapport facture",
        default="Facture Vente Gros",
    )
    sopromer_report_title_sale = fields.Char(
        string="Titre rapport vente",
        default="Bon de Commande",
    )
    sopromer_report_title_purchase = fields.Char(
        string="Titre rapport achat",
        default="Bon de Commande Fournisseur",
    )
    sopromer_report_title_delivery = fields.Char(
        string="Titre rapport livraison",
        default="Bon de Livraison",
    )
    sopromer_report_title_production = fields.Char(
        string="Titre rapport fabrication",
        default="Ordre de Fabrication",
    )
```

- [ ] **Step 2: Créer la vue res_company_views.xml**

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <record id="view_company_form_sopromer_report" model="ir.ui.view">
        <field name="name">res.company.form.sopromer.report</field>
        <field name="model">res.company</field>
        <field name="inherit_id" ref="base.view_company_form"/>
        <field name="arch" type="xml">
            <xpath expr="//page[@name='general_info']" position="after">
                <page string="Rapport SOPROMER" name="sopromer_report">
                    <group string="Informations fiscales">
                        <group>
                            <field name="sopromer_nif"/>
                            <field name="sopromer_stat"/>
                        </group>
                        <group>
                            <field name="sopromer_rc"/>
                            <field name="sopromer_cif"/>
                        </group>
                    </group>
                    <group string="Options d'affichage">
                        <group>
                            <field name="sopromer_show_signatures"/>
                            <field name="sopromer_signature_labels"
                                   invisible="not sopromer_show_signatures"/>
                            <field name="sopromer_show_payment_conditions"/>
                            <field name="sopromer_show_amount_in_words"/>
                        </group>
                    </group>
                    <group string="Titres des rapports">
                        <group>
                            <field name="sopromer_report_title_invoice"/>
                            <field name="sopromer_report_title_sale"/>
                            <field name="sopromer_report_title_purchase"/>
                        </group>
                        <group>
                            <field name="sopromer_report_title_delivery"/>
                            <field name="sopromer_report_title_production"/>
                        </group>
                    </group>
                    <group string="Mention légale">
                        <field name="sopromer_legal_mention" nolabel="1" colspan="2"/>
                    </group>
                </page>
            </xpath>
        </field>
    </record>
</odoo>
```

- [ ] **Step 3: Commit**

```bash
git add models/res_company.py views/res_company_views.xml
git commit -m "feat: add SOPROMER report config fields on res.company"
```

---

## Task 3: Format papier et styles SCSS

**Files:**
- Create: `sopromer_report_layout/data/paperformat.xml`
- Create: `sopromer_report_layout/static/src/scss/sopromer_report.scss`

- [ ] **Step 1: Créer paperformat.xml**

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <record id="paperformat_sopromer_a4" model="report.paperformat">
        <field name="name">SOPROMER A4</field>
        <field name="default" eval="True"/>
        <field name="format">A4</field>
        <field name="orientation">Portrait</field>
        <field name="margin_top">10</field>
        <field name="margin_bottom">15</field>
        <field name="margin_left">7</field>
        <field name="margin_right">7</field>
        <field name="header_line" eval="False"/>
        <field name="header_spacing">5</field>
        <field name="dpi">90</field>
    </record>
</odoo>
```

- [ ] **Step 2: Créer sopromer_report.scss**

```scss
/* === SOPROMER Report Layout Styles === */

.sopromer-report {
    font-family: 'Arial', 'Helvetica', sans-serif;
    font-size: 11px;
    color: #000;

    /* En-tête */
    .sopromer-header {
        display: flex;
        justify-content: space-between;
        margin-bottom: 15px;

        .sopromer-company-info {
            width: 45%;
            h2 {
                font-size: 18px;
                font-weight: bold;
                text-decoration: underline;
                margin-bottom: 5px;
            }
            .company-details {
                font-size: 10px;
                line-height: 1.4;
            }
        }

        .sopromer-client-info {
            width: 45%;
            border: 1px solid #000;
            padding: 10px;

            .doit-label {
                font-size: 20px;
                font-weight: bold;
                margin-bottom: 5px;
            }
        }
    }

    /* Titre du document */
    .sopromer-doc-title {
        font-size: 22px;
        font-weight: bold;
        margin: 15px 0;
    }

    /* Bloc méta (numéro, date, ref) */
    .sopromer-meta-table {
        width: auto;
        margin-bottom: 15px;
        border: 1px solid #000;

        th, td {
            padding: 4px 10px;
            border: 1px solid #000;
        }
        th {
            font-weight: bold;
            background-color: #f0f0f0;
        }
    }

    /* Tableau principal des lignes */
    .sopromer-lines-table {
        width: 100%;
        border-collapse: collapse;
        margin-bottom: 15px;

        th, td {
            padding: 4px 6px;
            border: 1px solid #000;
        }
        th {
            font-weight: bold;
            background-color: #f0f0f0;
            text-align: center;
        }
        td.text-right {
            text-align: right;
        }
        td.text-center {
            text-align: center;
        }
    }

    /* Totaux */
    .sopromer-totals-table {
        margin-left: auto;
        border: 1px solid #000;

        th, td {
            padding: 4px 10px;
            border: 1px solid #000;
        }
        th {
            background-color: #f0f0f0;
            font-weight: bold;
        }
    }

    /* Mention légale */
    .sopromer-legal {
        font-size: 9px;
        font-style: italic;
        margin: 10px 0;
    }

    /* Conditions de règlement */
    .sopromer-conditions {
        border: 1px solid #000;
        padding: 5px 10px;
        margin: 10px 0;
        font-size: 10px;
    }

    /* Zones de signatures */
    .sopromer-signatures {
        display: flex;
        justify-content: space-between;
        margin-top: 30px;

        .signature-zone {
            text-align: center;
            width: 30%;
            min-height: 60px;
            border-top: 1px dotted #999;
            padding-top: 5px;
            font-weight: bold;
            font-size: 10px;
        }
    }

    /* Montant en lettres */
    .sopromer-amount-words {
        font-style: italic;
        margin: 5px 0;
    }
}
```

- [ ] **Step 3: Commit**

```bash
git add data/paperformat.xml static/src/scss/sopromer_report.scss
git commit -m "feat: add SOPROMER paper format and SCSS styles"
```

---

## Task 4: Layout de base SOPROMER (template QWeb partagé)

**Files:**
- Create: `sopromer_report_layout/report/sopromer_layout.xml`

C'est le coeur du module — le layout réutilisable par tous les rapports.

- [ ] **Step 1: Créer sopromer_layout.xml**

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>

    <!-- ============================================================
         SOPROMER External Layout — remplace le layout standard Odoo
         Usage dans chaque rapport:
           <t t-call="sopromer_report_layout.sopromer_external_layout">
               <t t-set="o" t-value="doc_object"/>
               <t t-set="doc_title" t-value="'Facture Vente Gros'"/>
               <t t-set="doc_number" t-value="object.name"/>
               <t t-set="doc_date" t-value="object.date"/>
               <t t-set="doc_ref" t-value="object.ref"/>
               <t t-set="partner" t-value="object.partner_id"/>
               ... contenu spécifique ...
           </t>
         ============================================================ -->

    <template id="sopromer_external_layout">
        <div class="sopromer-report">

            <!-- ===== EN-TÊTE ===== -->
            <div class="sopromer-header">

                <!-- Colonne gauche: infos SOPROMER -->
                <div class="sopromer-company-info">
                    <t t-set="company" t-value="o.company_id if o else res_company"/>
                    <t t-if="company.logo">
                        <img t-att-src="image_data_uri(company.logo)"
                             style="max-height:60px; margin-bottom:5px;"
                             alt="Logo"/>
                    </t>
                    <h2 t-field="company.name"/>
                    <div class="company-details">
                        <t t-if="company.report_header">
                            <div t-field="company.report_header"/>
                        </t>
                        <div t-field="company.partner_id" t-options='{"widget": "contact", "fields": ["address"], "no_marker": true}'/>
                        <div>
                            <t t-if="company.email">
                                E-mail: <span t-field="company.email"/>
                            </t>
                        </div>
                        <div>
                            <t t-if="company.phone">
                                Tél: <span t-field="company.phone"/>
                            </t>
                        </div>
                        <div>
                            <t t-if="company.sopromer_stat">
                                N° Stat: <span t-field="company.sopromer_stat"/>
                            </t>
                        </div>
                        <div>
                            <t t-if="company.sopromer_rc">
                                R.C: <span t-field="company.sopromer_rc"/>
                            </t>
                        </div>
                        <div>
                            <t t-if="company.sopromer_nif">
                                N.I.F: <span t-field="company.sopromer_nif"/>
                            </t>
                        </div>
                        <div>
                            <t t-if="company.sopromer_cif">
                                CIF: <span t-field="company.sopromer_cif"/>
                            </t>
                        </div>
                    </div>
                </div>

                <!-- Colonne droite: infos client "DOIT:" -->
                <div class="sopromer-client-info">
                    <t t-if="partner">
                        <div class="doit-label">DOIT :</div>
                        <div style="font-weight:bold; font-size:14px;">
                            <span t-field="partner.name"/>
                        </div>
                        <div t-field="partner" t-options='{"widget": "contact", "fields": ["address"], "no_marker": true}'/>
                        <t t-if="partner.phone">
                            <div>Tél : <span t-field="partner.phone"/></div>
                        </t>
                        <!-- Champs fiscaux client si renseignés -->
                        <t t-if="partner.vat">
                            <div>NIF: <span t-field="partner.vat"/></div>
                        </t>
                    </t>
                </div>
            </div>

            <!-- ===== TITRE DU DOCUMENT ===== -->
            <div class="sopromer-doc-title">
                <span t-esc="doc_title"/>
            </div>

            <!-- ===== BLOC MÉTA (Numéro, Date, Référence) ===== -->
            <table class="sopromer-meta-table">
                <thead>
                    <tr>
                        <th>NUMERO</th>
                        <th>DATE</th>
                        <t t-if="doc_ref">
                            <th>REFERENCE</th>
                        </t>
                    </tr>
                </thead>
                <tbody>
                    <tr>
                        <td><span t-esc="doc_number"/></td>
                        <td><span t-esc="doc_date" t-options='{"widget": "date"}'/></td>
                        <t t-if="doc_ref">
                            <td><span t-esc="doc_ref"/></td>
                        </t>
                    </tr>
                </tbody>
            </table>

            <!-- ===== CONTENU SPÉCIFIQUE (injecté par chaque rapport) ===== -->
            <t t-raw="0"/>

            <!-- ===== MENTION LÉGALE ===== -->
            <t t-set="company" t-value="o.company_id if o else res_company"/>
            <t t-if="company.sopromer_legal_mention">
                <div class="sopromer-legal">
                    <span t-field="company.sopromer_legal_mention"/>
                </div>
            </t>

            <!-- ===== CONDITIONS DE RÈGLEMENT ===== -->
            <t t-if="company.sopromer_show_payment_conditions and payment_term">
                <div class="sopromer-conditions">
                    <strong>Conditions de règlement :</strong>
                    <span t-esc="payment_term"/>
                    <t t-if="due_date">
                        <span style="float:right;">
                            <strong>Echéance :</strong> <span t-esc="due_date"/>
                        </span>
                    </t>
                </div>
            </t>

            <!-- ===== MONTANT EN LETTRES ===== -->
            <t t-if="company.sopromer_show_amount_in_words and amount_in_words">
                <div class="sopromer-amount-words">
                    <span t-esc="amount_in_words"/>
                </div>
            </t>

            <!-- ===== ZONES DE SIGNATURES ===== -->
            <t t-if="company.sopromer_show_signatures">
                <div class="sopromer-signatures">
                    <t t-set="sig_labels" t-value="(company.sopromer_signature_labels or '').split(';')"/>
                    <t t-foreach="sig_labels" t-as="label">
                        <div class="signature-zone">
                            <span t-esc="label.strip()"/>
                        </div>
                    </t>
                </div>
            </t>

        </div>
    </template>

</odoo>
```

- [ ] **Step 2: Vérifier la syntaxe XML**

```bash
python -c "import xml.etree.ElementTree as ET; ET.parse('report/sopromer_layout.xml'); print('XML OK')"
```

- [ ] **Step 3: Commit**

```bash
git add report/sopromer_layout.xml
git commit -m "feat: add SOPROMER shared external layout template"
```

---

## Task 5: Override rapport Facture client

**Files:**
- Create: `sopromer_report_layout/report/report_invoice.xml`

- [ ] **Step 1: Créer report_invoice.xml**

Ce template hérite de `account.report_invoice_document` et remplace le contenu par le layout SOPROMER avec les colonnes: Référence | Désignation | Qté | Unité | Nb/Colis | PU HT | Montant HT + totaux HT/TVA/TTC.

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>

    <!-- Override du document facture pour utiliser le layout SOPROMER -->
    <template id="report_invoice_document_sopromer"
              inherit_id="account.report_invoice_document"
              priority="99">

        <!-- Remplacer tout le contenu du document -->
        <xpath expr="//div[@class='page']" position="replace">
            <div class="page">
                <t t-set="doc_title" t-value="o.company_id.sopromer_report_title_invoice or 'Facture'"/>
                <t t-set="doc_number" t-value="o.name"/>
                <t t-set="doc_date" t-value="o.invoice_date"/>
                <t t-set="doc_ref" t-value="o.ref or o.payment_reference"/>
                <t t-set="partner" t-value="o.partner_id"/>
                <t t-set="payment_term" t-value="o.invoice_payment_term_id.name if o.invoice_payment_term_id else ''"/>
                <t t-set="due_date" t-value="o.invoice_date_due"/>
                <t t-set="amount_in_words" t-value="o.currency_id.amount_to_text(o.amount_total)"/>

                <t t-call="sopromer_report_layout.sopromer_external_layout">
                    <t t-set="o" t-value="o"/>

                    <!-- Tableau des lignes facture -->
                    <table class="sopromer-lines-table">
                        <thead>
                            <tr>
                                <th>Référence</th>
                                <th>Désignation</th>
                                <th>Qté</th>
                                <th>Unité</th>
                                <th>Nb/Colis</th>
                                <th>PU HT</th>
                                <th>Montant HT</th>
                            </tr>
                        </thead>
                        <tbody>
                            <t t-foreach="o.invoice_line_ids.filtered(lambda l: not l.display_type)" t-as="line">
                                <tr>
                                    <td><span t-esc="line.product_id.default_code or ''"/></td>
                                    <td><span t-field="line.name"/></td>
                                    <td class="text-right"><span t-field="line.quantity"/></td>
                                    <td class="text-center"><span t-field="line.product_uom_id" t-options='{"widget": "name"}'/></td>
                                    <td class="text-center">
                                        <t t-if="'nombre_sacs' in line._fields">
                                            <span t-esc="line.nombre_sacs or ''"/>
                                        </t>
                                    </td>
                                    <td class="text-right"><span t-field="line.price_unit"/></td>
                                    <td class="text-right"><span t-field="line.price_subtotal"/></td>
                                </tr>
                            </t>
                        </tbody>
                    </table>

                    <!-- Totaux -->
                    <table class="sopromer-totals-table">
                        <tr>
                            <th>Total HT</th>
                            <th>TVA</th>
                            <th>Total TTC</th>
                            <th>NET A PAYER</th>
                        </tr>
                        <tr>
                            <td class="text-right"><span t-field="o.amount_untaxed"/></td>
                            <td class="text-right"><span t-field="o.amount_tax"/></td>
                            <td class="text-right"><span t-field="o.amount_total"/></td>
                            <td class="text-right"><strong><span t-field="o.amount_residual"/></strong></td>
                        </tr>
                    </table>

                </t>
            </div>
        </xpath>

    </template>

</odoo>
```

- [ ] **Step 2: Commit**

```bash
git add report/report_invoice.xml
git commit -m "feat: add SOPROMER invoice report override"
```

---

## Task 6: Override rapport Bon de Commande Vente / Devis

**Files:**
- Create: `sopromer_report_layout/report/report_sale_order.xml`

- [ ] **Step 1: Créer report_sale_order.xml**

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>

    <template id="report_saleorder_document_sopromer"
              inherit_id="sale.report_saleorder_document"
              priority="99">

        <xpath expr="//div[@class='page']" position="replace">
            <div class="page">
                <t t-set="doc_title" t-value="doc.company_id.sopromer_report_title_sale or 'Bon de Commande'"/>
                <t t-set="doc_number" t-value="doc.name"/>
                <t t-set="doc_date" t-value="doc.date_order"/>
                <t t-set="doc_ref" t-value="doc.client_order_ref"/>
                <t t-set="partner" t-value="doc.partner_id"/>
                <t t-set="payment_term" t-value="doc.payment_term_id.name if doc.payment_term_id else ''"/>
                <t t-set="due_date" t-value="''"/>
                <t t-set="amount_in_words" t-value="doc.currency_id.amount_to_text(doc.amount_total)"/>

                <t t-call="sopromer_report_layout.sopromer_external_layout">
                    <t t-set="o" t-value="doc"/>

                    <table class="sopromer-lines-table">
                        <thead>
                            <tr>
                                <th>Référence</th>
                                <th>Désignation</th>
                                <th>Qté</th>
                                <th>Unité</th>
                                <th>Nb/Colis</th>
                                <th>PU HT</th>
                                <th>Montant HT</th>
                            </tr>
                        </thead>
                        <tbody>
                            <t t-foreach="doc.order_line.filtered(lambda l: not l.display_type)" t-as="line">
                                <tr>
                                    <td><span t-esc="line.product_id.default_code or ''"/></td>
                                    <td><span t-field="line.name"/></td>
                                    <td class="text-right"><span t-field="line.product_uom_qty"/></td>
                                    <td class="text-center"><span t-field="line.product_uom"/></td>
                                    <td class="text-center">
                                        <t t-if="'nombre_sacs' in line._fields">
                                            <span t-esc="line.nombre_sacs or ''"/>
                                        </t>
                                    </td>
                                    <td class="text-right"><span t-field="line.price_unit"/></td>
                                    <td class="text-right"><span t-field="line.price_subtotal"/></td>
                                </tr>
                            </t>
                        </tbody>
                    </table>

                    <table class="sopromer-totals-table">
                        <tr>
                            <th>Total HT</th>
                            <th>TVA</th>
                            <th>Total TTC</th>
                            <th>NET A PAYER</th>
                        </tr>
                        <tr>
                            <td class="text-right"><span t-field="doc.amount_untaxed"/></td>
                            <td class="text-right"><span t-field="doc.amount_tax"/></td>
                            <td class="text-right"><span t-field="doc.amount_total"/></td>
                            <td class="text-right"><strong><span t-field="doc.amount_total"/></strong></td>
                        </tr>
                    </table>

                </t>
            </div>
        </xpath>

    </template>

</odoo>
```

- [ ] **Step 2: Commit**

```bash
git add report/report_sale_order.xml
git commit -m "feat: add SOPROMER sale order report override"
```

---

## Task 7: Override rapport Bon de Commande Achat

**Files:**
- Create: `sopromer_report_layout/report/report_purchase_order.xml`

- [ ] **Step 1: Créer report_purchase_order.xml**

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>

    <template id="report_purchaseorder_document_sopromer"
              inherit_id="purchase.report_purchaseorder_document"
              priority="99">

        <xpath expr="//div[@class='page']" position="replace">
            <div class="page">
                <t t-set="doc_title" t-value="o.company_id.sopromer_report_title_purchase or 'Bon de Commande Fournisseur'"/>
                <t t-set="doc_number" t-value="o.name"/>
                <t t-set="doc_date" t-value="o.date_order"/>
                <t t-set="doc_ref" t-value="o.partner_ref"/>
                <t t-set="partner" t-value="o.partner_id"/>
                <t t-set="payment_term" t-value="o.payment_term_id.name if o.payment_term_id else ''"/>
                <t t-set="due_date" t-value="''"/>
                <t t-set="amount_in_words" t-value="o.currency_id.amount_to_text(o.amount_total)"/>

                <t t-call="sopromer_report_layout.sopromer_external_layout">
                    <t t-set="o" t-value="o"/>

                    <table class="sopromer-lines-table">
                        <thead>
                            <tr>
                                <th>Référence</th>
                                <th>Désignation</th>
                                <th>Qté</th>
                                <th>Unité</th>
                                <th>Nb/Colis</th>
                                <th>PU HT</th>
                                <th>Montant HT</th>
                            </tr>
                        </thead>
                        <tbody>
                            <t t-foreach="o.order_line.filtered(lambda l: not l.display_type)" t-as="line">
                                <tr>
                                    <td><span t-esc="line.product_id.default_code or ''"/></td>
                                    <td><span t-field="line.name"/></td>
                                    <td class="text-right"><span t-field="line.product_qty"/></td>
                                    <td class="text-center"><span t-field="line.product_uom"/></td>
                                    <td class="text-center">
                                        <t t-if="'nombre_sacs' in line._fields">
                                            <span t-esc="line.nombre_sacs or ''"/>
                                        </t>
                                    </td>
                                    <td class="text-right"><span t-field="line.price_unit"/></td>
                                    <td class="text-right"><span t-field="line.price_subtotal"/></td>
                                </tr>
                            </t>
                        </tbody>
                    </table>

                    <table class="sopromer-totals-table">
                        <tr>
                            <th>Total HT</th>
                            <th>TVA</th>
                            <th>Total TTC</th>
                        </tr>
                        <tr>
                            <td class="text-right"><span t-field="o.amount_untaxed"/></td>
                            <td class="text-right"><span t-field="o.amount_tax"/></td>
                            <td class="text-right"><strong><span t-field="o.amount_total"/></strong></td>
                        </tr>
                    </table>

                </t>
            </div>
        </xpath>

    </template>

</odoo>
```

- [ ] **Step 2: Commit**

```bash
git add report/report_purchase_order.xml
git commit -m "feat: add SOPROMER purchase order report override"
```

---

## Task 8: Override rapport Bon de Livraison

**Files:**
- Create: `sopromer_report_layout/report/report_delivery_slip.xml`

- [ ] **Step 1: Créer report_delivery_slip.xml**

Colonnes adaptées BL : Référence | Désignation | Qté | Nb/Colis | Observations. Pas de prix.

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>

    <template id="report_delivery_document_sopromer"
              inherit_id="stock.report_delivery_document"
              priority="99">

        <xpath expr="//div[@class='page']" position="replace">
            <div class="page">
                <t t-set="doc_title" t-value="o.company_id.sopromer_report_title_delivery or 'Bon de Livraison'"/>
                <t t-set="doc_number" t-value="o.name"/>
                <t t-set="doc_date" t-value="o.scheduled_date"/>
                <t t-set="doc_ref" t-value="o.origin"/>
                <t t-set="partner" t-value="o.partner_id"/>
                <t t-set="payment_term" t-value="''"/>
                <t t-set="due_date" t-value="''"/>
                <t t-set="amount_in_words" t-value="''"/>

                <t t-call="sopromer_report_layout.sopromer_external_layout">
                    <t t-set="o" t-value="o"/>

                    <table class="sopromer-lines-table">
                        <thead>
                            <tr>
                                <th>Référence</th>
                                <th>Désignation</th>
                                <th>Qté</th>
                                <th>Unité</th>
                                <th>Nb/Colis</th>
                                <th>Observations</th>
                            </tr>
                        </thead>
                        <tbody>
                            <t t-foreach="o.move_ids.filtered(lambda m: m.state != 'cancel')" t-as="move">
                                <tr>
                                    <td><span t-esc="move.product_id.default_code or ''"/></td>
                                    <td><span t-field="move.product_id.display_name"/></td>
                                    <td class="text-right"><span t-field="move.quantity"/></td>
                                    <td class="text-center"><span t-field="move.product_uom"/></td>
                                    <td class="text-center">
                                        <t t-if="'nombre_sacs' in move._fields">
                                            <span t-esc="move.nombre_sacs or ''"/>
                                        </t>
                                    </td>
                                    <td/>
                                </tr>
                            </t>
                        </tbody>
                        <tfoot>
                            <tr>
                                <td colspan="2"><strong>Total</strong></td>
                                <td class="text-right">
                                    <strong><span t-esc="sum(o.move_ids.filtered(lambda m: m.state != 'cancel').mapped('quantity'))"/></strong>
                                </td>
                                <td/>
                                <td class="text-center">
                                    <t t-if="'nombre_sacs' in o.move_ids[:1]._fields if o.move_ids else False">
                                        <strong><span t-esc="sum(o.move_ids.filtered(lambda m: m.state != 'cancel').mapped('nombre_sacs'))"/></strong>
                                    </t>
                                </td>
                                <td/>
                            </tr>
                        </tfoot>
                    </table>

                </t>
            </div>
        </xpath>

    </template>

</odoo>
```

- [ ] **Step 2: Commit**

```bash
git add report/report_delivery_slip.xml
git commit -m "feat: add SOPROMER delivery slip report override"
```

---

## Task 9: Override rapport Ordre de Fabrication

**Files:**
- Create: `sopromer_report_layout/report/report_mrp_production.xml`

- [ ] **Step 1: Créer report_mrp_production.xml**

Colonnes MRP : Composant/Produit | Qté Prévue | Qté Réalisée | UdM | Lot

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>

    <template id="report_mrporder_sopromer"
              inherit_id="mrp.report_mrporder"
              priority="99">

        <xpath expr="//div[@class='page']" position="replace">
            <div class="page">
                <t t-set="doc_title" t-value="o.company_id.sopromer_report_title_production or 'Ordre de Fabrication'"/>
                <t t-set="doc_number" t-value="o.name"/>
                <t t-set="doc_date" t-value="o.date_start or o.create_date"/>
                <t t-set="doc_ref" t-value="o.origin"/>
                <t t-set="partner" t-value="''"/>
                <t t-set="payment_term" t-value="''"/>
                <t t-set="due_date" t-value="''"/>
                <t t-set="amount_in_words" t-value="''"/>

                <t t-call="sopromer_report_layout.sopromer_external_layout">
                    <t t-set="o" t-value="o"/>

                    <!-- Produit fini -->
                    <div style="margin-bottom:10px;">
                        <strong>Produit :</strong> <span t-field="o.product_id.display_name"/>
                        <br/>
                        <strong>Qté prévue :</strong> <span t-field="o.product_qty"/>
                        <span t-field="o.product_uom_id"/>
                        <t t-if="o.lot_producing_id">
                            | <strong>Lot :</strong> <span t-field="o.lot_producing_id.name"/>
                        </t>
                    </div>

                    <!-- Composants consommés -->
                    <h4>Composants</h4>
                    <table class="sopromer-lines-table">
                        <thead>
                            <tr>
                                <th>Référence</th>
                                <th>Composant</th>
                                <th>Qté Prévue</th>
                                <th>Qté Consommée</th>
                                <th>UdM</th>
                            </tr>
                        </thead>
                        <tbody>
                            <t t-foreach="o.move_raw_ids.filtered(lambda m: m.state != 'cancel')" t-as="raw">
                                <tr>
                                    <td><span t-esc="raw.product_id.default_code or ''"/></td>
                                    <td><span t-field="raw.product_id.display_name"/></td>
                                    <td class="text-right"><span t-field="raw.product_uom_qty"/></td>
                                    <td class="text-right"><span t-field="raw.quantity"/></td>
                                    <td class="text-center"><span t-field="raw.product_uom"/></td>
                                </tr>
                            </t>
                        </tbody>
                    </table>

                    <!-- Sous-produits si présents -->
                    <t t-if="o.move_byproduct_ids">
                        <h4 style="margin-top:10px;">Sous-produits</h4>
                        <table class="sopromer-lines-table">
                            <thead>
                                <tr>
                                    <th>Référence</th>
                                    <th>Sous-produit</th>
                                    <th>Qté</th>
                                    <th>UdM</th>
                                    <th>Lot</th>
                                </tr>
                            </thead>
                            <tbody>
                                <t t-foreach="o.move_byproduct_ids.filtered(lambda m: m.state != 'cancel')" t-as="bp">
                                    <tr>
                                        <td><span t-esc="bp.product_id.default_code or ''"/></td>
                                        <td><span t-field="bp.product_id.display_name"/></td>
                                        <td class="text-right"><span t-field="bp.quantity"/></td>
                                        <td class="text-center"><span t-field="bp.product_uom"/></td>
                                        <td>
                                            <t t-foreach="bp.move_line_ids" t-as="ml">
                                                <span t-field="ml.lot_id.name"/>
                                            </t>
                                        </td>
                                    </tr>
                                </t>
                            </tbody>
                        </table>
                    </t>

                </t>
            </div>
        </xpath>

    </template>

</odoo>
```

- [ ] **Step 2: Commit**

```bash
git add report/report_mrp_production.xml
git commit -m "feat: add SOPROMER MRP production order report override"
```

---

## Task 10: Fichier de sécurité + Installation et test

**Files:**
- Create: `sopromer_report_layout/security/ir.model.access.csv`

- [ ] **Step 1: Créer ir.model.access.csv**

```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
```

(Fichier vide avec header — pas de nouveau modèle, mais le fichier doit exister pour convention)

- [ ] **Step 2: Installer le module dans Odoo**

```bash
cd "C:/Program Files/Odoo 18/server"
./odoo-bin -c odoo.conf -d sopromer --update=sopromer_report_layout --stop-after-init
```

- [ ] **Step 3: Tester manuellement**

1. Aller dans Configuration > Sociétés > SOPROMER > onglet "Rapport SOPROMER"
2. Remplir NIF, STAT, RC, CIF
3. Imprimer une facture → vérifier le layout SOPROMER
4. Imprimer un bon de commande vente → vérifier
5. Imprimer un BC achat → vérifier
6. Imprimer un BL → vérifier
7. Imprimer un OF → vérifier

- [ ] **Step 4: Ajuster les styles si nécessaire**

Modifier `sopromer_report.scss` pour aligner le rendu avec les modèles PDF de référence.

- [ ] **Step 5: Commit final**

```bash
git add .
git commit -m "feat: complete sopromer_report_layout module v18.0.1.0.0"
```

---

## Résumé des dépendances entre tâches

```
Task 1 (scaffold) ──→ Task 2 (config) ──→ Task 3 (styles)
                                              │
Task 4 (layout base) ←───────────────────────┘
    │
    ├──→ Task 5 (facture)
    ├──→ Task 6 (vente)
    ├──→ Task 7 (achat)
    ├──→ Task 8 (BL)
    └──→ Task 9 (OF)
              │
              └──→ Task 10 (install + test)
```
