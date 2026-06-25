# Odoo SaaS-19.3 Development Guide

Reference for module structure, creation patterns, security, and wizards in SaaS-19.3.
Source of truth: `/home/achraf/src/193/odoo/` and `/home/achraf/src/193/enterprise/`

> **SaaS-19.3 notes vs generic Odoo 19**
> - View tag `<list>` replaces `<tree>` (since Odoo 17) — never use `<tree>` in SaaS-19.3
> - `tracking=True` on a field replaces the deprecated `track_visibility='always'`
> - `_inherit` alone (no `_name`) = extension; both `_name` + `_inherit` = prototype copy

---

## Table of Contents

1. [Module structure](#module-structure)
2. [Creating a module](#creating-a-module)
3. [Models](#models)
4. [Views](#views)
5. [Security](#security)
6. [Wizards](#wizards)
7. [Common patterns](#common-patterns)

---

## Module structure

```
my_module/
├── __init__.py
├── __manifest__.py
├── models/
│   ├── __init__.py
│   └── my_model.py
├── views/
│   └── my_model_views.xml
├── security/
│   ├── ir.model.access.csv
│   └── my_module_security.xml
├── data/
│   └── my_module_data.xml
├── demo/
│   └── demo_data.xml
├── migrations/            ← also accepted: upgrades/
│   └── saas~19.3.1.0/
│       └── post-migration.py
├── tests/
│   ├── __init__.py
│   └── test_my_model.py
├── wizard/
│   ├── __init__.py
│   └── my_wizard.py
├── controllers/
│   ├── __init__.py
│   └── my_controller.py
├── report/
│   └── my_report.xml
└── static/
    ├── description/
    │   └── icon.png
    └── src/
        ├── js/
        ├── xml/
        └── scss/
```

> SaaS version strings use `saas~19.3` format — migration scripts must match.

---

## Creating a module

### `__init__.py`

```python
from . import models
from . import controllers  # only if controllers/ exists
```

```python
# models/__init__.py
from . import my_model
```

### Minimal `__manifest__.py`

```python
{
    'name': 'My Module',
    'version': '1.0.0',
    'license': 'LGPL-3',
    'depends': ['base'],
    'data': [
        'security/ir.model.access.csv',
        'views/my_model_views.xml',
    ],
}
```

See the **manifest** skill for all manifest keys.

### Model skeleton

```python
from odoo import models, fields, api

class MyModel(models.Model):
    _name = 'my.model'
    _description = 'My Model'
    _order = 'name'

    name = fields.Char(required=True)
    active = fields.Boolean(default=True)
```

---

## Models

### Extension (add fields/methods to existing model)

```python
class ResPartner(models.Model):
    _inherit = 'res.partner'   # _name is NOT redefined

    my_field = fields.Char(string="My Field")
```

### Prototype (new model that copies all fields/methods from another)

```python
class MyOrder(models.Model):
    _name = 'my.order'         # new _name
    _inherit = 'sale.order'    # inherits everything from sale.order

    extra_field = fields.Char()
```

### TransientModel (wizard)

```python
class MyWizard(models.TransientModel):
    _name = 'my.wizard'
    _description = 'My Wizard'
```

### AbstractModel (mixin)

```python
class MyMixin(models.AbstractModel):
    _name = 'my.mixin'
    _description = 'My Mixin'

    def shared_method(self):
        pass
```

### Important dunder attributes

| Attribute | Description |
|-----------|-------------|
| `_name` | Technical model name (dotted, lowercase) |
| `_description` | Human-readable name (required) |
| `_inherit` | Parent model(s) |
| `_order` | Default sort: `'name'`, `'date desc'`, `'sequence, id'` |
| `_rec_name` | Field used as display_name (default: `name`) |
| `_table` | Override DB table name |
| `_log_access` | Add create/write date+uid fields (default True) |
| `_sql_constraints` | List of `(name, sql, message)` tuples |

---

## Views

> **SaaS-19.3**: Always use `<list>` — never `<tree>`.

### List + Form + Search + Action + Menu

```xml
<?xml version="1.0" encoding="UTF-8"?>
<odoo>
    <record id="view_my_model_list" model="ir.ui.view">
        <field name="name">my.model.list</field>
        <field name="model">my.model</field>
        <field name="arch" type="xml">
            <list string="My Models">
                <field name="name"/>
                <field name="active"/>
            </list>
        </field>
    </record>

    <record id="view_my_model_form" model="ir.ui.view">
        <field name="name">my.model.form</field>
        <field name="model">my.model</field>
        <field name="arch" type="xml">
            <form string="My Model">
                <sheet>
                    <group>
                        <group>
                            <field name="name"/>
                        </group>
                        <group>
                            <field name="active"/>
                        </group>
                    </group>
                    <notebook>
                        <page string="Notes">
                            <field name="description"/>
                        </page>
                    </notebook>
                </sheet>
            </form>
        </field>
    </record>

    <record id="view_my_model_search" model="ir.ui.view">
        <field name="name">my.model.search</field>
        <field name="model">my.model</field>
        <field name="arch" type="xml">
            <search>
                <field name="name"/>
                <filter string="Active" name="active" domain="[('active','=',True)]"/>
                <group expand="0" string="Group By">
                    <filter string="Status" name="group_active" context="{'group_by': 'active'}"/>
                </group>
            </search>
        </field>
    </record>

    <record id="action_my_model" model="ir.actions.act_window">
        <field name="name">My Models</field>
        <field name="res_model">my.model</field>
        <field name="view_mode">list,form</field>
        <field name="help" type="html">
            <p class="o_view_nocontent_smiling_face">Create your first record</p>
        </field>
    </record>

    <menuitem id="menu_my_module_root" name="My Module"
        web_icon="my_module,static/description/icon.png"/>
    <menuitem id="menu_my_model" name="My Models"
        parent="menu_my_module_root" action="action_my_model"/>
</odoo>
```

### View inheritance (XPath)

```xml
<record id="view_partner_form_inherit_my_module" model="ir.ui.view">
    <field name="name">res.partner.form.my_module</field>
    <field name="model">res.partner</field>
    <field name="inherit_id" ref="base.view_partner_form"/>
    <field name="arch" type="xml">
        <!-- Add after a field -->
        <xpath expr="//field[@name='email']" position="after">
            <field name="my_field"/>
        </xpath>
        <!-- Add inside a group -->
        <xpath expr="//group[@name='contact_info']" position="inside">
            <field name="my_other_field"/>
        </xpath>
        <!-- Replace an attribute -->
        <xpath expr="//field[@name='phone']" position="attributes">
            <attribute name="readonly">True</attribute>
        </xpath>
    </field>
</record>
```

---

## Security

### `security/ir.model.access.csv`

```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_my_model_user,my.model user,model_my_model,base.group_user,1,1,1,0
access_my_model_manager,my.model manager,model_my_model,my_module.group_my_module_manager,1,1,1,1
```

The `model_id:id` column is always `model_` + model name with dots replaced by underscores.

### Groups

```xml
<odoo>
    <data noupdate="0">
        <record id="group_my_module_user" model="res.groups">
            <field name="name">My Module / User</field>
            <field name="category_id" ref="base.module_category_hidden"/>
            <field name="implied_ids" eval="[(4, ref('base.group_user'))]"/>
        </record>

        <record id="group_my_module_manager" model="res.groups">
            <field name="name">My Module / Manager</field>
            <field name="category_id" ref="base.module_category_hidden"/>
            <field name="implied_ids" eval="[(4, ref('group_my_module_user'))]"/>
        </record>
    </data>
</odoo>
```

### Record rules

```xml
<odoo>
    <data noupdate="1">
        <record id="rule_my_model_own" model="ir.rule">
            <field name="name">My Model: see own records</field>
            <field name="model_id" ref="model_my_model"/>
            <field name="domain_force">[('create_uid', '=', user.id)]</field>
            <field name="groups" eval="[(4, ref('base.group_user'))]"/>
        </record>
        <record id="rule_my_model_manager_all" model="ir.rule">
            <field name="name">My Model: manager sees all</field>
            <field name="model_id" ref="model_my_model"/>
            <field name="domain_force">[(1, '=', 1)]</field>
            <field name="groups" eval="[(4, ref('group_my_module_manager'))]"/>
        </record>
    </data>
</odoo>
```

---

## Wizards

```python
from odoo import models, fields

class MyWizard(models.TransientModel):
    _name = 'my.wizard'
    _description = 'My Wizard'

    date = fields.Date(required=True, default=fields.Date.context_today)
    note = fields.Text()

    def action_confirm(self):
        self.ensure_one()
        records = self.env['my.model'].browse(self.env.context.get('active_ids', []))
        records.write({'state': 'done'})
        return {'type': 'ir.actions.act_window_close'}
```

```xml
<record id="view_my_wizard_form" model="ir.ui.view">
    <field name="name">my.wizard.form</field>
    <field name="model">my.wizard</field>
    <field name="arch" type="xml">
        <form string="My Wizard">
            <group>
                <field name="date"/>
                <field name="note"/>
            </group>
            <footer>
                <button name="action_confirm" string="Confirm" type="object" class="btn-primary"/>
                <button string="Cancel" class="btn-secondary" special="cancel"/>
            </footer>
        </form>
    </field>
</record>
```

Open wizard from a method:

```python
def action_open_wizard(self):
    return {
        'type': 'ir.actions.act_window',
        'name': 'My Wizard',
        'res_model': 'my.wizard',
        'view_mode': 'form',
        'target': 'new',
        'context': {'default_date': fields.Date.context_today(self)},
    }
```

---

## Common patterns

### State field with status bar

```python
state = fields.Selection([
    ('draft', 'Draft'),
    ('confirmed', 'Confirmed'),
    ('done', 'Done'),
    ('cancel', 'Cancelled'),
], default='draft', tracking=True)
```

```xml
<header>
    <button name="action_confirm" string="Confirm" type="object"
            invisible="state != 'draft'" class="btn-primary"/>
    <button name="action_cancel" string="Cancel" type="object"
            invisible="state not in ('draft', 'confirmed')"/>
    <field name="state" widget="statusbar"
           statusbar_visible="draft,confirmed,done"/>
</header>
```

### `default_get` from context

```python
def default_get(self, fields_list):
    defaults = super().default_get(fields_list)
    if 'partner_id' in fields_list:
        defaults['partner_id'] = self.env.context.get('default_partner_id')
    return defaults
```

### `name_search` override

```python
@api.model
def _name_search(self, name, domain=None, operator='ilike', limit=100, order=None):
    domain = domain or []
    if name:
        domain = [('name', operator, name)] + domain
    return self._search(domain, limit=limit, order=order)
```

### SQL constraints

```python
_sql_constraints = [
    ('unique_code', 'UNIQUE(code)', 'Code must be unique.'),
    ('positive_amount', 'CHECK(amount >= 0)', 'Amount must be positive.'),
]
```

### Migrations

```python
# migrations/saas~19.3.1.1/post-migration.py
def migrate(cr, version):
    cr.execute("""
        UPDATE my_model
        SET state = 'confirmed'
        WHERE state = 'open'
    """)
```
