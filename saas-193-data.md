# Odoo SaaS-19.3 Data Files Guide

Reference for XML and CSV data files in the SaaS-19.3 codebase.
Source of truth: `/home/achraf/src/193/odoo/odoo/tools/convert.py`

> **SaaS-19.3 differences vs generic Odoo 19**
> - `<field search="...">` gained a `use` attribute to extract a specific field from search results
> - `<delete>` supports `search` attribute (domain-based deletion)
> - `forcecreate` attribute controls cross-module record creation behaviour
> - `base64` field type deprecated since 20.0 — prefer `file` + `type="base64"` or raw bytes

---

## Table of Contents

1. [XML Data Files Structure](#xml-data-files-structure)
2. [record tag](#record-tag)
3. [field tag](#field-tag)
4. [delete tag](#delete-tag)
5. [function tag](#function-tag)
6. [Shortcuts](#shortcuts)
7. [CSV Data Files](#csv-data-files)

---

## XML Data Files Structure

```xml
<?xml version="1.0" encoding="UTF-8"?>
<odoo>
    <operation/>
    ...
</odoo>
```

### noupdate

```xml
<odoo>
    <data noupdate="1">
        <!-- Only loaded on module install, never re-applied on update -->
        <record .../>
    </data>

    <!-- Loaded on install AND update -->
    <record .../>
</odoo>
```

Use `noupdate="1"` for records that users are expected to customise (e.g., email templates, demo data).

---

## record tag

Creates or updates a database record.

### Attributes

| Attribute     | Type   | Required | Description |
|---------------|--------|----------|-|
| `model`       | string | Yes      | Model to create/update |
| `id`          | string | Recommended | External ID (required for updates) |
| `context`     | dict   | No       | Context passed to `create`/`write` |
| `forcecreate` | bool   | No       | In update mode, create record if it doesn't exist (default `True`) |

```xml
<record id="partner_odoo" model="res.partner">
    <field name="name">Odoo</field>
    <field name="is_company" eval="True"/>
    <field name="customer_rank" eval="1"/>
</record>
```

---

## field tag

### Attributes

| Attribute | Type   | Description |
|-----------|--------|-------------|
| `name`    | string | **Required** — field to set |
| `ref`     | string | External ID to resolve and assign |
| `search`  | domain | Python domain; first match used for Many2one, full set for x2many |
| `use`     | string | **SaaS-19.3** — field to extract from search results (default: `id`) |
| `eval`    | string | Python expression |
| `type`    | string | Content type (see below) |

### Setting a value by external ID (`ref`)

```xml
<field name="country_id" ref="base.vn"/>
<field name="user_id" ref="base.user_admin"/>
```

### Setting a value by domain search (`search` + `use`)

```xml
<!-- Many2one: first match -->
<field name="partner_id" search="[('name', '=', 'Odoo')]"/>

<!-- SaaS-19.3: extract a specific field from the search result instead of id -->
<field name="partner_id" search="[('name', '=', 'Odoo')]" use="id"/>
<field name="currency_id" search="[('name', '=', 'EUR')]" use="id"/>
```

The `use` attribute tells the loader which field of the matched record to use as the value.
Defaults to `id`, which is the standard behaviour.

### Setting a value with `eval`

```xml
<field name="active" eval="True"/>
<field name="date" eval="datetime.date.today()"/>
<field name="partner_id" eval="ref('base.main_partner')"/>
<field name="sequence" eval="10"/>
```

Evaluation context available in `eval`:
- `time`, `datetime`, `timedelta`, `relativedelta`
- `ref(external_id)` — resolves an external ID to a database ID
- `obj` — current record's model

### `type` attribute

| type | Description |
|------|-------------|
| `xml`, `html` | Parse child nodes as document content |
| `file` | Path relative to module; stores as `module,path` |
| `base64` | **Deprecated since 20.0** — base64-encode content (still works in SaaS-19.3) |
| `char` | Raw string, no transformation |
| `int`, `float` | Convert content to number |
| `list`, `tuple` | Wrap `<value>` child elements into a Python list/tuple |

```xml
<!-- Embed XML content -->
<field name="arch" type="xml">
    <form>
        <field name="name"/>
    </form>
</field>

<!-- Embed a file as base64 (deprecated, but still works) -->
<field name="image_1920" type="base64" file="module/static/img/logo.png"/>

<!-- HTML field -->
<field name="description" type="html">
    <p>Hello world</p>
</field>
```

### Many2many via `eval`

```xml
<field name="group_ids" eval="[(4, ref('base.group_user'))]"/>
<!-- ORM commands: (4, id) = link, (6, 0, [ids]) = set, (5,) = clear -->
```

---

## delete tag

Removes records.

### Attributes

| Attribute | Type   | Required | Description |
|-----------|--------|----------|-|
| `model`   | string | Yes      | Model to delete from |
| `id`      | string | One of id/search | External ID of record to remove |
| `search`  | domain | One of id/search | Domain — all matching records removed |

```xml
<!-- Delete a specific record by external ID -->
<delete model="ir.ui.view" id="my_module.old_view"/>

<!-- Delete all matching records by domain (SaaS-19.3 supported) -->
<delete model="ir.ui.menu" search="[('name', '=', 'Old Menu')]"/>
```

---

## function tag

Calls a Python method on a model during module load.

### Attributes

| Attribute | Type   | Required | Description |
|-----------|--------|----------|-|
| `model`   | string | Yes      | Model to call method on |
| `name`    | string | Yes      | Method name |

### Passing arguments

Via `eval` (must evaluate to a sequence):

```xml
<function model="res.partner" name="send_inscription_notice"
    eval="[[ref('partner_1'), ref('partner_2')]]"/>
```

Via nested `<function>` (first child's result is passed as argument):

```xml
<function model="res.users" name="send_vip_inscription_notice">
    <function eval="[[('vip','=',True)]]" model="res.partner" name="search"/>
</function>
```

---

## Shortcuts

### menuitem

Creates an `ir.ui.menu` record.

| Attribute | Description |
|-----------|-------------|
| `id`      | External ID |
| `name`    | Display name (inferred from action if omitted) |
| `parent`  | External ID of parent, or `/`-separated path |
| `action`  | External ID of action to execute |
| `groups`  | Comma-separated group external IDs; prefix `-` to remove a group |
| `sequence`| Integer ordering |
| `web_icon`| `module,path/to/icon.png` for app-level menu items |

```xml
<menuitem id="menu_root" name="My App"
    web_icon="my_module,static/description/icon.png"/>

<menuitem id="menu_items" name="Items"
    parent="menu_root" action="action_my_model"/>
```

### template

Creates a `ir.ui.view` of type `qweb` without needing a full `<record>` block.

| Attribute    | Description |
|--------------|-------------|
| `id`         | External ID |
| `name`       | View name |
| `inherit_id` | External ID of parent view |
| `priority`   | View priority (integer) |
| `primary`    | If `True` with `inherit_id`, marks as a primary view |
| `groups`     | Comma-separated group external IDs |
| `active`     | Whether the view is active |

```xml
<template id="my_page" name="My Page">
    <t t-call="website.layout">
        <div class="oe_structure">
            <h1>Hello</h1>
        </div>
    </t>
</template>
```

Inheritance:

```xml
<template id="my_extension" inherit_id="website.layout">
    <xpath expr="//div[@id='wrap']" position="before">
        <div class="my-banner">Banner</div>
    </xpath>
</template>
```

### asset

Creates or updates an `ir.asset` record.

```xml
<asset id="my_module.assets_backend" name="My module assets">
    <bundle>web.assets_backend</bundle>
    <path>my_module/static/src/js/my_widget.js</path>
</asset>

<!-- Disable an asset by default -->
<asset id="my_module.optional_style" name="Optional style" active="False">
    <bundle>web.assets_frontend</bundle>
    <path>my_module/static/src/css/optional.scss</path>
</asset>
```

---

## CSV Data Files

For bulk same-model record creation. File name must match the model: `{model_name}.csv`.

```
id,country_id:id,name,code
state_al,base.us,Alabama,AL
state_ak,base.us,Alaska,AK
```

- `id` column → external ID for the record (allows update on reinstall)
- Relational fields: use `:id` suffix and provide the external ID of the related record
- All columns after `id` map directly to model field names

---

## Common patterns

### Create security groups with implied access

```xml
<record id="group_my_user" model="res.groups">
    <field name="name">My Module / User</field>
    <field name="category_id" ref="base.module_category_hidden"/>
    <field name="implied_ids" eval="[(4, ref('base.group_user'))]"/>
</record>
```

### Demo data that won't interfere with real data

```xml
<odoo>
    <data noupdate="1">
        <record id="demo_partner" model="res.partner">
            <field name="name">Demo Customer</field>
        </record>
    </data>
</odoo>
```

### Update a record from another module

```xml
<!-- forcecreate="0" means raise if record doesn't exist -->
<record id="base.res_partner_rule" model="ir.rule" forcecreate="0">
    <field name="active" eval="False"/>
</record>
```
