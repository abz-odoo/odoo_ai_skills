# Odoo SaaS-19.3 Views Guide

Reference for all view types in SaaS-19.3.
Source of truth: `/home/achraf/src/193/odoo/odoo/addons/base/models/ir_ui_view.py`
and `/home/achraf/src/193/odoo/addons/sale/views/sale_order_views.xml`

> **SaaS-19.3 breaking changes vs Odoo 17/18**
> - `<tree>` is **dead** — use `<list>` (using `attrs=` or `states=` raises a `ValidationError`)
> - `attrs=` and `states=` are **removed** — raise `ValidationError` at view loading
> - Use direct expression syntax: `invisible="state != 'draft'"` (Python-like boolean)
> - `<chatter/>` replaces `<div class="oe_chatter">...</div>`
> - `column_invisible` and `optional="show|hide"` are the list column visibility attributes

---

## Table of Contents

1. [Breaking changes summary](#breaking-changes-summary)
2. [List view](#list-view)
3. [Form view](#form-view)
4. [Kanban view](#kanban-view)
5. [Search view](#search-view)
6. [Graph view](#graph-view)
7. [Pivot view](#pivot-view)
8. [Calendar view](#calendar-view)
9. [Activity view](#activity-view)
10. [View inheritance (xpath)](#view-inheritance-xpath)
11. [Common widgets](#common-widgets)

---

## Breaking changes summary

| Old (Odoo 17/18) | New (SaaS-19.3) | Notes |
|------------------|-----------------|-------|
| `<tree>` | `<list>` | ValidationError if `<tree>` used |
| `attrs="{'invisible': [...]}"` | `invisible="expr"` | ValidationError if `attrs=` present |
| `states="draft,sent"` | `invisible="state not in ['draft','sent']"` | ValidationError if `states=` present |
| `<div class="oe_chatter">` | `<chatter/>` | Old syntax still works but deprecated |
| `track_visibility='onchange'` | `tracking=True` | Field attribute, not view |
| `group_operator="sum"` | `aggregator="sum"` | Field attribute |

---

## List view

```xml
<record id="view_my_model_list" model="ir.ui.view">
    <field name="name">my.model.list</field>
    <field name="model">my.model</field>
    <field name="arch" type="xml">
        <list string="My Records"
              decoration-muted="state == 'cancel'"
              decoration-danger="overdue"
              editable="bottom"
              multi_edit="1"
              sample="1">

            <!-- Action buttons applied to selection (header in editable list) -->
            <header>
                <button string="Mark Done" name="action_mark_done" type="object"
                        class="btn-secondary"/>
            </header>

            <!-- Columns -->
            <field name="name" decoration-bf="1"/>
            <field name="partner_id" optional="show"/>
            <field name="date" optional="hide"/>
            <field name="state"
                   decoration-success="state == 'done'"
                   decoration-warning="state == 'pending'"
                   decoration-info="state == 'draft'"/>
            <field name="amount_total" sum="Total" widget="monetary"/>

            <!-- Hidden columns — needed for logic or decoration expressions -->
            <field name="overdue" column_invisible="1"/>
            <field name="currency_id" column_invisible="1"/>

            <!-- Inline creation controls (editable lists) -->
            <control>
                <create string="Add a Line"/>
                <button string="Open Catalog" name="action_open_catalog" type="object"
                        class="btn-link"/>
            </control>
        </list>
    </field>
</record>
```

### Key list attributes

| Attribute | Description |
|-----------|-------------|
| `editable="top\|bottom"` | Enable inline row editing |
| `multi_edit="1"` | Allow bulk editing of selected rows |
| `sample="1"` | Show placeholder sample data when empty |
| `decoration-*` | Row-level CSS class (`success`, `warning`, `danger`, `info`, `primary`, `muted`, `bf`) |
| `limit="N"` | Default page size |
| `create="false"` | Disable new record creation from list |

### Column attributes

| Attribute | Description |
|-----------|-------------|
| `optional="show\|hide"` | User-toggleable column (default shown/hidden) |
| `column_invisible="1"` | Always hidden — used for values needed in expressions |
| `sum="Label"` | Show sum in footer |
| `decoration-*` | Cell-level CSS class |
| `decoration-bf="1"` | Bold font for this cell |
| `width="120px"` | Fixed column width |
| `nolabel="1"` | Hide column header |

---

## Form view

```xml
<record id="view_my_model_form" model="ir.ui.view">
    <field name="name">my.model.form</field>
    <field name="model">my.model</field>
    <field name="arch" type="xml">
        <form string="My Record">
            <!-- Status bar + action buttons -->
            <header>
                <button string="Confirm" name="action_confirm" type="object"
                        class="btn-primary"
                        invisible="state != 'draft'"/>
                <button string="Cancel" name="action_cancel" type="object"
                        invisible="state in ['cancel', 'done']"/>
                <field name="state" widget="statusbar"
                       statusbar_visible="draft,confirmed,done"/>
            </header>

            <!-- Main content -->
            <sheet>
                <!-- Smart buttons (top-right area) -->
                <div class="oe_button_box" name="button_box">
                    <button name="action_view_invoices" type="object"
                            class="oe_stat_button" icon="fa-pencil-square-o"
                            invisible="invoice_count == 0">
                        <field name="invoice_count" widget="statinfo" string="Invoices"/>
                    </button>
                </div>

                <!-- Title area -->
                <div class="oe_title">
                    <h1><field name="name" placeholder="Reference..."/></h1>
                </div>

                <group>
                    <group>
                        <field name="partner_id"
                               readonly="state in ['done', 'cancel']"/>
                        <field name="date_order"/>
                        <field name="user_id"/>
                    </group>
                    <group>
                        <field name="currency_id" invisible="1"/>
                        <field name="amount_total" widget="monetary"/>
                    </group>
                </group>

                <!-- One2many tab -->
                <notebook>
                    <page string="Order Lines" name="order_lines">
                        <field name="order_line">
                            <list editable="bottom">
                                <field name="product_id"/>
                                <field name="product_uom_qty"/>
                                <field name="price_unit"/>
                                <field name="price_subtotal" column_invisible="1"/>
                            </list>
                        </field>
                    </page>
                    <page string="Notes" name="notes">
                        <field name="note" nolabel="1"/>
                    </page>
                </notebook>
            </sheet>

            <!-- Chatter — new simplified syntax -->
            <chatter/>
        </form>
    </field>
</record>
```

### Expression syntax for modifiers

```xml
<!-- Boolean field -->
<field name="discount" invisible="not show_discount"/>

<!-- String comparison -->
<field name="date" readonly="state == 'cancel'"/>

<!-- List membership -->
<field name="partner_id" readonly="state in ['done', 'cancel', 'sale']"/>

<!-- Compound expressions -->
<div invisible="state not in ['draft', 'sent'] or not warning_message">
    <field name="warning_message"/>
</div>

<!-- Required based on condition -->
<field name="delivery_address_id" required="delivery_mode == 'shipping'"/>

<!-- Parent field access (inside nested o2m) -->
<field name="tax_ids" context="{'default_country_id': parent.country_id}"/>
```

### `<chatter/>` attributes

```xml
<chatter/>                                   <!-- minimal -->
<chatter reload_on_post="1"/>                <!-- reload form after posting message -->
<chatter reload_on_attachment="1"/>          <!-- reload on attachment change -->
<chatter reload_on_follower="1"/>            <!-- reload on follower change -->
```

---

## Kanban view

```xml
<record id="view_my_model_kanban" model="ir.ui.view">
    <field name="name">my.model.kanban</field>
    <field name="model">my.model</field>
    <field name="arch" type="xml">
        <kanban sample="1" quick_create="false" default_group_by="stage_id">
            <!-- Fields available in card template expressions -->
            <field name="currency_id"/>
            <field name="state"/>

            <!-- Progress bar on column header -->
            <progressbar field="activity_state"
                colors='{"planned": "success", "today": "warning", "overdue": "danger"}'/>

            <templates>
                <t t-name="card">
                    <!-- Card title row -->
                    <div class="d-flex mb-1">
                        <field name="name" class="fw-bold me-auto"/>
                        <field name="amount_total" widget="monetary"/>
                    </div>
                    <!-- Card body -->
                    <field name="partner_id" class="text-muted"/>
                    <!-- Card footer -->
                    <footer>
                        <field name="user_id" widget="many2one_avatar_user"/>
                        <field name="state" widget="badge"
                               decoration-info="state == 'draft'"
                               decoration-success="state == 'done'"
                               class="ms-auto"/>
                    </footer>
                </t>
            </templates>
        </kanban>
    </field>
</record>
```

### Key kanban attributes

| Attribute | Description |
|-----------|-------------|
| `default_group_by="field"` | Initial grouping field |
| `quick_create="false"` | Disable the quick-create card |
| `sample="1"` | Show sample cards when empty |
| `group_create="false"` | Disable new column creation |
| `group_delete="false"` | Disable column deletion |

---

## Search view

```xml
<record id="view_my_model_search" model="ir.ui.view">
    <field name="name">my.model.search</field>
    <field name="model">my.model</field>
    <field name="arch" type="xml">
        <search string="Search My Model">
            <!-- Searchable fields -->
            <field name="name"/>
            <field name="partner_id"/>
            <field name="name" string="Reference or Customer"
                   filter_domain="['|', ('name', 'ilike', self), ('partner_id', 'child_of', self)]"/>

            <!-- Quick filters -->
            <filter name="my_records" string="My Records"
                    domain="[('user_id', '=', uid)]"/>
            <filter name="draft" string="Draft"
                    domain="[('state', '=', 'draft')]"/>
            <filter name="overdue" string="Overdue"
                    domain="[('date_deadline', '&lt;', context_today().strftime('%Y-%m-%d'))]"/>

            <separator/>

            <!-- Group-by -->
            <group string="Group By">
                <filter name="by_partner" string="Customer"
                        context="{'group_by': 'partner_id'}"/>
                <filter name="by_month" string="Month"
                        context="{'group_by': 'date_order:month'}"/>
                <filter name="by_state" string="Status"
                        context="{'group_by': 'state'}"/>
            </group>
        </search>
    </field>
</record>
```

### Search filter with date comparison

```xml
<!-- Use context_today() for dynamic date filters -->
<filter name="today" string="Due Today"
        domain="[('date_deadline', '=', context_today().strftime('%Y-%m-%d'))]"/>

<!-- Date range -->
<filter name="this_month" string="This Month"
        domain="[('date_order', '&gt;=', (context_today() + relativedelta(day=1)).strftime('%Y-%m-%d'))]"/>
```

---

## Graph view

```xml
<graph string="My Model" sample="1" type="bar">
    <field name="partner_id"/>                    <!-- dimension (X axis) -->
    <field name="amount_total" type="measure"/>   <!-- measure (Y axis) -->
    <field name="state" type="col"/>              <!-- column grouping (stacked bar) -->
</graph>
```

`type` attribute on `<graph>`: `bar` (default), `line`, `pie`.

---

## Pivot view

```xml
<pivot string="My Model" sample="1">
    <field name="date_order" type="row"/>      <!-- row dimension -->
    <field name="partner_id" type="col"/>      <!-- column dimension -->
    <field name="amount_total" type="measure" string="Revenue"/>
    <field name="order_count" type="measure"/>
</pivot>
```

---

## Calendar view

```xml
<calendar string="My Model"
          date_start="date_deadline"
          date_stop="date_end"
          color="user_id"
          mode="month"
          event_limit="5"
          quick_create="0"
          create="0">
    <field name="partner_id" avatar_field="avatar_128"/>
    <field name="amount_total" widget="monetary"/>
    <field name="state" filters="1" invisible="1"/>
</calendar>
```

| Attribute | Description |
|-----------|-------------|
| `date_start` | Mandatory — event start field |
| `date_stop` | Optional — event end field |
| `color` | Field used to colour-code events |
| `mode` | `month`, `week`, `day`, `year` |
| `event_limit` | Max events shown per day cell |

---

## Activity view

```xml
<activity string="My Model">
    <field name="currency_id"/>
    <templates>
        <div t-name="activity-box" class="d-flex">
            <div class="flex-grow-1">
                <field name="name" display="full" class="o_text_bold"/>
                <field name="partner_id" muted="1"/>
            </div>
            <div class="ms-auto">
                <field name="amount_total" widget="monetary"/>
                <field name="state" widget="badge"
                       decoration-success="state == 'done'"
                       decoration-info="state == 'draft'"/>
            </div>
        </div>
    </templates>
</activity>
```

---

## View inheritance (xpath)

```xml
<record id="view_my_model_form_inherit" model="ir.ui.view">
    <field name="name">my.model.form.inherit</field>
    <field name="model">my.model</field>
    <field name="inherit_id" ref="base_module.view_my_model_form"/>
    <field name="arch" type="xml">

        <!-- Insert before a specific element -->
        <xpath expr="//field[@name='partner_id']" position="before">
            <field name="custom_field"/>
        </xpath>

        <!-- Insert after -->
        <xpath expr="//field[@name='amount_total']" position="after">
            <field name="discount_total"/>
        </xpath>

        <!-- Replace element -->
        <xpath expr="//button[@name='action_confirm']" position="replace">
            <button string="Custom Confirm" name="action_custom_confirm" type="object"
                    class="btn-primary"/>
        </xpath>

        <!-- Modify attributes only -->
        <xpath expr="//field[@name='name']" position="attributes">
            <attribute name="readonly">1</attribute>
            <attribute name="string">Reference</attribute>
        </xpath>

        <!-- Shorter: use field/button selectors directly -->
        <field name="date_order" position="attributes">
            <attribute name="invisible">state == 'cancel'</attribute>
        </field>

        <!-- Add inside a group -->
        <xpath expr="//group[1]" position="inside">
            <field name="extra_info"/>
        </xpath>

        <!-- Move an existing element -->
        <field name="note" position="move"/>
        <xpath expr="//page[@name='notes']" position="inside">
            <field name="note" position="move"/>
        </xpath>
    </field>
</record>
```

### XPath position values

| Position | Effect |
|----------|--------|
| `before` | Insert sibling before matched node |
| `after` | Insert sibling after matched node |
| `inside` | Append as last child of matched node |
| `replace` | Replace matched node entirely |
| `attributes` | Modify XML attributes of matched node |
| `move` | Move matched node to current position |

---

## Common widgets

| Widget | Field type | Description |
|--------|-----------|-------------|
| `monetary` | Float/Monetary | Shows currency symbol |
| `statusbar` | Selection | Workflow progress bar |
| `many2one_avatar_user` | Many2one res.users | User with avatar |
| `many2many_tags` | Many2many | Coloured tags |
| `badge` | Selection | Coloured badge |
| `label_selection` | Selection | Selection with colour classes |
| `list_activity` | — | Activity indicator in list |
| `statinfo` | Integer | Smart button counter |
| `progressbar` | Float | Progress bar 0–100 |
| `html` | Html | Rich text editor |
| `image` | Binary | Image display |
| `char_emojis` | Char | Char with emoji picker |
| `priority` | Selection | Star priority widget |
| `handle` | Integer | Drag-to-reorder handle |
| `color_picker` | Integer | Colour picker (0–11) |
