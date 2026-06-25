# Odoo SaaS-19.3 Reports Guide

Reference for QWeb PDF and HTML reports in SaaS-19.3.
Source of truth: `/home/achraf/src/193/odoo/odoo/addons/base/models/ir_actions_report.py`

> **SaaS-19.3 key addition**
> - `css_margins` field on `report.paperformat` — use CSS margins instead of wkhtmltopdf arguments (more flexible, responsive)

---

## Table of Contents

1. [Defining a report action](#defining-a-report-action)
2. [Paperformat](#paperformat)
3. [QWeb template structure](#qweb-template-structure)
4. [Available layout templates](#available-layout-templates)
5. [Translation](#translation)
6. [Attachment caching](#attachment-caching)
7. [Custom rendering context](#custom-rendering-context)
8. [Multi-record PDF splitting](#multi-record-pdf-splitting)
9. [Triggering reports from code](#triggering-reports-from-code)
10. [Complete example](#complete-example)

---

## Defining a report action

```xml
<!-- views/reports.xml -->
<record id="action_report_my_document" model="ir.actions.report">
    <field name="name">My Document</field>
    <field name="model">my.model</field>
    <field name="report_type">qweb-pdf</field>      <!-- qweb-pdf | qweb-html | qweb-text -->
    <field name="report_name">my_module.report_my_document</field>
    <field name="report_file">my_module.report_my_document</field>

    <!-- Dynamic filename — Python expression, `object` = current record -->
    <field name="print_report_name">'MyDoc-%s' % object.name</field>

    <!-- Bind to a model so it appears in the Print menu -->
    <field name="binding_model_id" ref="model_my_model"/>
    <field name="binding_type">report</field>

    <!-- Restrict to specific records -->
    <field name="domain" eval="[('state', '=', 'done')]"/>

    <!-- Access control — only show for these groups -->
    <field name="group_ids" eval="[(4, ref('my_module.group_my_manager'))]"/>
</record>
```

### Key `ir.actions.report` fields

| Field | Type | Description |
|-------|------|-------------|
| `report_name` | Char | QWeb template XML ID (`module.template_id`) |
| `report_type` | Selection | `qweb-pdf`, `qweb-html`, or `qweb-text` |
| `model` | Char | Model the report is for |
| `print_report_name` | Char | Python expression for the filename |
| `paperformat_id` | Many2one | Page format (`report.paperformat`) |
| `attachment` | Char | Python expression for attachment filename (saves PDF) |
| `attachment_use` | Boolean | Reuse existing attachment if present |
| `binding_model_id` | Many2one | Model to bind to (appears in Print menu) |
| `binding_type` | Selection | `'report'` for Print menu |
| `domain` | Char | Filter for when to show the action |
| `group_ids` | Many2many | Groups that can see this action |
| `multi` | Boolean | `True` → not shown on single-record form toolbar |

---

## Paperformat

```xml
<record id="paperformat_a4_custom" model="report.paperformat">
    <field name="name">A4 Custom</field>
    <field name="format">A4</field>              <!-- A4, Letter, Legal, custom, … -->
    <field name="orientation">Portrait</field>   <!-- Portrait | Landscape -->
    <field name="margin_top">20</field>          <!-- mm -->
    <field name="margin_bottom">10</field>
    <field name="margin_left">10</field>
    <field name="margin_right">10</field>
    <field name="header_spacing">10</field>      <!-- mm between header and content -->
    <field name="dpi">90</field>
    <field name="css_margins" eval="True"/>      <!-- SaaS-19.3: use CSS margins -->
</record>
```

### `css_margins` (SaaS-19.3)

When `css_margins=True`, Odoo passes margins via CSS (the body gets class `o_css_margins`)
instead of wkhtmltopdf command-line arguments. Use this for:
- More predictable cross-platform margins
- CSS-based responsive layouts
- When wkhtmltopdf margin handling is unreliable

### Per-document overrides

Override paperformat settings on the HTML root tag using `data-report-*` attributes:

```xml
<t t-call="web.html_container">
    <t t-att-data-report-margin-top="40"/>
    <t t-att-data-report-dpi="120"/>
    <t t-att-data-report-landscape="1"/>
    ...
</t>
```

---

## QWeb template structure

Every report has **two templates**:
1. **Outer wrapper** (`report_xxx`) — calls `web.html_container`, iterates over `docs`
2. **Document template** (`report_xxx_document`) — renders a single record with the layout

```xml
<!-- Outer wrapper: iterates over all records -->
<template id="report_my_document">
    <t t-call="web.html_container">
        <t t-foreach="docs" t-as="doc">
            <!-- t-lang renders this template in the partner's language -->
            <t t-call="my_module.report_my_document_document"
               t-lang="doc.partner_id.lang"/>
        </t>
    </t>
</template>

<!-- Per-document template -->
<template id="report_my_document_document">
    <!-- Address block for the recipient -->
    <t t-set="address">
        <address t-field="doc.partner_id"
            t-options='{"widget": "contact", "fields": ["address", "name"], "no_marker": True}'/>
    </t>

    <!-- Document title shown in header -->
    <t t-set="layout_document_title">
        <span>Order </span><span t-field="doc.name"/>
    </t>

    <!-- Main content wrapped in the company layout -->
    <t t-call="web.external_layout"
       address="address"
       layout_document_title="layout_document_title">
        <div class="page">
            <h2 t-field="doc.name"/>
            <!-- report body -->
        </div>
    </t>
</template>
```

### Template context variables

| Variable | Type | Description |
|----------|------|-------------|
| `docs` | recordset | All records being rendered |
| `doc` / `o` | record | Current record (inside `t-foreach`) |
| `report_type` | str | `'html'` or `'pdf'` |
| `time` | module | Python `time` module |
| `user` | record | Current user |
| `res_company` | record | Current company |
| `web_base_url` | str | Base URL of the Odoo instance |
| `is_html_empty(val)` | func | Returns True if HTML value is empty |
| `context_timestamp(dt)` | func | Convert datetime to user's timezone |

---

## Available layout templates

| Template | Description |
|----------|-------------|
| `web.html_container` | Outer HTML document wrapper — always use this as the root |
| `web.external_layout` | Company-branded layout (auto-selects from company settings) |
| `web.external_layout_standard` | Standard layout (logo + company info) |
| `web.external_layout_wave` | Wave decorative header |
| `web.external_layout_bubble` | Bubble decoration |
| `web.external_layout_folder` | Tab-like header |
| `web.external_layout_center` | 3-column header |
| `web.external_layout_dual` | Split colour design |
| `web.external_layout_lines` | Curved lines decoration |
| `web.internal_layout` | Internal report (no company branding, has page counter) |
| `web.basic_layout` | Minimal layout with no header/footer |

`web.external_layout` automatically uses the layout selected in **Settings → Companies →
Report Layout**. Custom modules should always call `web.external_layout` rather than a
specific variant.

### `web.external_layout` parameters

```xml
<t t-call="web.external_layout"
   address="address_node"
   layout_document_title="title_node"
   information_block="info_node"
   forced_vat="o.fiscal_position_id.foreign_vat"
   lang="o.partner_id.lang">
    <div class="page">
        <!-- report body -->
    </div>
</t>
```

---

## Translation

Reports render in the **server locale** by default. Use `t-lang` to render in the
document's partner language:

```xml
<!-- On the t-call of the document template: -->
<t t-call="my_module.report_document" t-lang="doc.partner_id.lang"/>

<!-- Or inside the template, switch the record's context: -->
<t t-set="doc" t-value="doc.with_context(lang=doc.partner_id.lang)"/>
```

For translatable strings inside the template:

```xml
<span>Invoice</span>        <!-- Will be translated via QWeb i18n -->
<t t-out="doc.name"/>       <!-- Field value — no translation needed -->
```

---

## Attachment caching

Store the rendered PDF as an `ir.attachment` so it is not regenerated on every print:

```xml
<record id="action_report_invoice" model="ir.actions.report">
    <field name="name">Invoice PDF</field>
    <field name="model">account.move</field>
    <field name="report_type">qweb-pdf</field>
    <field name="report_name">account.report_invoice</field>

    <!-- Filename expression — `object` = the record being rendered -->
    <field name="attachment">'invoice-%s.pdf' % (object.name.replace('/', '-'),)</field>

    <!-- Reuse existing attachment if already generated -->
    <field name="attachment_use">True</field>
</record>
```

When `attachment_use=True`:
1. On first print: PDF rendered, saved as `ir.attachment` with the given filename
2. On subsequent prints: existing attachment returned (no re-render)

To force regeneration: delete the attachment or set `attachment_use=False`.

---

## Custom rendering context

For reports that need extra data (joins, aggregates, external API calls), create an
`AbstractModel` named `report.<module>.<template_id>`:

```python
# my_module/models/report_my_document.py

from odoo import models, api

class ReportMyDocument(models.AbstractModel):
    _name = 'report.my_module.report_my_document'
    _description = 'My Document Report'

    @api.model
    def _get_report_values(self, docids, data=None):
        docs = self.env['my.model'].browse(docids)
        return {
            'docs': docs,
            'extra_data': self._compute_extra(docs),
            'company': self.env.company,
        }

    def _compute_extra(self, docs):
        return {doc.id: doc.order_line._read_group([], ['amount:sum']) for doc in docs}
```

The dict returned by `_get_report_values` is merged into the template rendering context,
so `extra_data` becomes available as a template variable.

---

## Multi-record PDF splitting

When printing multiple records, Odoo splits the single generated PDF back into per-record
PDFs (for attachment storage). This relies on `data-oe-*` attributes on the article div:

```xml
<template id="report_my_document_document">
    <!-- These attributes allow Odoo to split the PDF per record -->
    <div class="article"
         t-att-data-oe-model="doc and doc._name"
         t-att-data-oe-id="doc and doc.id"
         t-att-data-oe-lang="doc and doc.env.context.get('lang')">
        <t t-call="web.external_layout">
            <div class="page">
                <!-- content -->
            </div>
        </t>
    </div>
</template>
```

---

## Triggering reports from code

```python
# Generate PDF bytes
pdf_content, content_type = self.env['ir.actions.report']._render_qweb_pdf(
    'my_module.action_report_my_document',
    res_ids=self.ids,
)
# pdf_content is bytes; content_type is 'pdf'

# Generate HTML bytes
html_content, content_type = self.env['ir.actions.report']._render_qweb_html(
    'my_module.action_report_my_document',
    docids=self.ids,
)

# Return as HTTP response (in a controller)
from odoo.http import request, Response
return Response(
    pdf_content,
    headers=[
        ('Content-Type', 'application/pdf'),
        ('Content-Disposition', 'attachment; filename="report.pdf"'),
    ],
)

# Return as a download action (from a button method)
def action_print(self):
    return self.env.ref('my_module.action_report_my_document').report_action(self)
```

---

## Complete example

**`my_module/views/report_order.xml`:**

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <!-- 1. Action -->
    <record id="action_report_order" model="ir.actions.report">
        <field name="name">Order PDF</field>
        <field name="model">my.order</field>
        <field name="report_type">qweb-pdf</field>
        <field name="report_name">my_module.report_order</field>
        <field name="print_report_name">
            (object.state == 'draft' and 'Quotation-%s' or 'Order-%s') % object.name
        </field>
        <field name="binding_model_id" ref="model_my_order"/>
        <field name="binding_type">report</field>
        <field name="attachment">'order-%s.pdf' % object.name.replace('/', '-')</field>
        <field name="attachment_use">True</field>
    </record>

    <!-- 2. Outer wrapper -->
    <template id="report_order">
        <t t-call="web.html_container">
            <t t-foreach="docs" t-as="doc">
                <t t-call="my_module.report_order_document"
                   t-lang="doc.partner_id.lang"/>
            </t>
        </t>
    </template>

    <!-- 3. Per-document template -->
    <template id="report_order_document">
        <t t-set="address">
            <address t-field="doc.partner_id"
                t-options='{"widget": "contact", "fields": ["address", "name"], "no_marker": True}'/>
        </t>
        <t t-set="layout_document_title">
            <span t-if="doc.state == 'draft'">Quotation</span>
            <span t-else="">Order</span>
            <span> — </span>
            <span t-field="doc.name"/>
        </t>
        <t t-call="web.external_layout"
           address="address"
           layout_document_title="layout_document_title">
            <div class="page">
                <table class="table table-sm">
                    <thead>
                        <tr>
                            <th>Description</th>
                            <th class="text-end">Qty</th>
                            <th class="text-end">Unit Price</th>
                            <th class="text-end">Subtotal</th>
                        </tr>
                    </thead>
                    <tbody>
                        <tr t-foreach="doc.order_line" t-as="line" t-key="line.id">
                            <td t-field="line.name"/>
                            <td class="text-end" t-field="line.product_uom_qty"/>
                            <td class="text-end" t-field="line.price_unit"/>
                            <td class="text-end" t-field="line.price_subtotal"/>
                        </tr>
                    </tbody>
                    <tfoot>
                        <tr>
                            <td colspan="3" class="text-end"><strong>Total</strong></td>
                            <td class="text-end"><strong t-field="doc.amount_total"/></td>
                        </tr>
                    </tfoot>
                </table>
            </div>
        </t>
    </template>
</odoo>
```
