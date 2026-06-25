# Odoo SaaS-19.3 Translation Guide

Reference for i18n / translation in SaaS-19.3.
Source of truth: `/home/achraf/src/193/odoo/odoo/tools/translate.py` and `odoo/orm/environments.py`

> **SaaS-19.3 key points**
> - Translatable fields stored as **JSONB** `{'en_US': '...', 'fr_FR': '...'}` — no `ir.translation` table
> - `env._()` — environment-aware translation using `env.lang`
> - `_lt()` / `LazyTranslate` — lazy translation for module-level constants
> - `StoredTranslations` with automatic language fallback
> - `ir.translation` table is gone — queries against it will fail

---

## Table of Contents

1. [Python: _() vs _lt()](#python--vs-lt)
2. [env._() — environment-aware translation](#env---environment-aware-translation)
3. [Field translation](#field-translation)
4. [Selection field translation](#selection-field-translation)
5. [XML view translation](#xml-view-translation)
6. [JavaScript translation](#javascript-translation)
7. [.po file structure](#po-file-structure)
8. [Import / export](#import--export)
9. [Common patterns](#common-patterns)

---

## Python: _() vs _lt()

### `_()` — immediate translation

Use inside methods, where the translation language is known at call time.

```python
from odoo.tools.translate import _

class MyModel(models.Model):
    _name = 'my.model'

    def action_done(self):
        raise UserError(_("This record is already done."))

    def get_label(self, name):
        return _("Hello %s", name)          # positional
        return _("Hello %(name)s", name=name)  # keyword
```

`_()` auto-detects the calling module from the stack frame and uses `self.env.lang` (or
the active language context) at call time.

### `_lt()` — lazy translation (LazyTranslate)

Use at **module level** for constants and class-level strings. The translation is deferred
until the string is actually rendered — avoiding the wrong-language-at-import-time problem.

```python
from odoo.tools.translate import LazyTranslate

_lt = LazyTranslate(__name__)   # bind to this module once

# Module-level constants — OK with lazy
ERROR_MSG = _lt("Record not found.")
STATES = [
    ('draft', _lt("Draft")),
    ('done',  _lt("Done")),
]

class MyModel(models.Model):
    _name = 'my.model'
    _description = _lt("My Model")   # class attribute — must be lazy
```

> **Rule**: `_()` inside functions/methods. `_lt()` at module level, in class bodies, or
> in any context where the language is not yet known.

---

## env._() — environment-aware translation

Use `env._()` when you need to translate in a specific language context (e.g., the
partner's language, not the current user's language):

```python
# Translate in the partner's language
partner_lang = self.partner_id.lang or 'en_US'
msg = self.with_context(lang=partner_lang).env._("Invoice confirmed.")

# Or equivalently
env = self.env(context=dict(self.env.context, lang=partner_lang))
msg = env._("Invoice confirmed.")

# Also accepts LazyTranslate objects
from odoo.tools.translate import LazyTranslate
_lt = LazyTranslate(__name__)
LABEL = _lt("Status")

msg = env._(LABEL)   # re-evaluates in env.lang
```

---

## Field translation

### `translate=True` — simple JSONB translation

The field value is stored per language in a JSONB column.

```python
class MyModel(models.Model):
    _name = 'my.model'

    name        = fields.Char(translate=True)
    description = fields.Text(translate=True)
    # Html fields with sanitize=True auto-enable html_translate
    body        = fields.Html(translate=True)
```

DB storage: `{"en_US": "Hello", "fr_FR": "Bonjour"}`

Read with `env.lang = 'fr_FR'` → returns `"Bonjour"`.
Fallback: `fr_FR` → `en_US`.

### `translate=html_translate` / `translate=xml_translate` — markup translation

For fields with HTML/XML content where only the *text nodes* should be translated while
the markup structure is preserved:

```python
from odoo.tools.translate import html_translate, xml_translate

class MyModel(models.Model):
    _name = 'my.model'

    instructions = fields.Text(translate=xml_translate)
    rich_content = fields.Html(translate=html_translate)
```

These translators extract individual terms from the markup, translate each, and rebuild
the HTML/XML structure. This allows partial updates without re-translating the entire field.

### Writing a translated field

```python
# Write in the active language
record.with_context(lang='fr_FR').name = 'Bonjour'
record.with_context(lang='en_US').name = 'Hello'

# Write all languages at once
record.write({'name': {'en_US': 'Hello', 'fr_FR': 'Bonjour'}})
```

---

## Selection field translation

Selection labels are translated via `ir.model.fields` — they are NOT stored in the field
definition at runtime. The translation is loaded from `.po` files during import.

```python
state = fields.Selection([
    ('draft',  'Draft'),
    ('done',   'Done'),
    ('cancel', 'Cancelled'),
])
```

When rendered in the UI with `lang='fr_FR'`, Odoo looks up the translated labels in
`ir.model.fields.selection`. Add translations in your `.po` file:

```po
#: model:ir.model.fields.selection,name:my_module.field_my_model__state_draft
msgid "Draft"
msgstr "Brouillon"
```

---

## XML view translation

### Automatically translated attributes

These XML attributes are extracted and translated automatically via `.po` files:

```
string, help, sum, avg, confirm, placeholder, alt, title,
aria-label, aria-placeholder, data-tooltip, label,
confirm-label, cancel-label, confirm-title
```

```xml
<field name="partner_id" string="Customer" placeholder="Select a customer..."/>
<button string="Confirm" confirm="Are you sure?"/>
<field name="note" help="Internal note — not visible to customers"/>
```

### Disabling translation on an element

```xml
<span t-translation="off">Technical ID: <t t-out="record.id"/></span>
```

### Report translation

In QWeb report templates, use `t-lang` to render in the document's partner language:

```xml
<t t-call="my_module.report_document" t-lang="doc.partner_id.lang"/>
```

---

## JavaScript translation

Use `_t()` for strings in `.js` files:

```javascript
import { _t } from "@web/core/l10n/translation";

// Simple
const msg = _t("Save");

// With placeholder (uses sprintf-style)
const msg2 = _t("Hello %(name)s", { name: user.name });
```

OWL template attributes that are automatically extracted:
`alt`, `aria-label`, `aria-placeholder`, `data-tooltip`, `label`, `placeholder`, `title`

```xml
<!-- These are extracted and translated automatically -->
<button title="Delete record"/>
<input placeholder="Search..."/>
```

---

## .po file structure

```po
# Translation for my_module in French
# Copyright (C) 2025 Odoo SA
msgid ""
msgstr ""
"Project-Id-Version: Odoo Server saas~19.3\n"
"Language: fr\n"
"Content-Type: text/plain; charset=UTF-8\n"

# Python code string
#. odoo-python
#: my_module/models/my_model.py:0
msgid "This record is already done."
msgstr "Cet enregistrement est déjà terminé."

# View attribute
#: model:ir.ui.view,arch_db:my_module.view_my_model_form
msgid "Customer"
msgstr "Client"

# Selection value
#: model:ir.model.fields.selection,name:my_module.field_my_model__state_draft
msgid "Draft"
msgstr "Brouillon"

# Field value (translate=True)
#: model:my.model,name:my_module.my_record_ref
msgid "Default Name"
msgstr "Nom par défaut"

# JavaScript string
#. odoo-javascript
#: my_module/static/src/js/my_widget.js:0
msgid "Save"
msgstr "Enregistrer"
```

### Occurrence prefixes

| Prefix | Source |
|--------|--------|
| `#: code:...` | Python `_()` call |
| `#: model:ir.ui.view,arch_db:...` | XML view attribute |
| `#: model:ir.model.fields.selection,...` | Selection label |
| `#: model:model.name,field:module.id` | `translate=True` field value |
| `#. odoo-python` | Python code translation |
| `#. odoo-javascript` | JS translation |

---

## Import / export

### Export via CLI

```bash
# Export .pot template (all translatable strings)
odoo-bin --i18n-export=/tmp/my_module.pot --modules=my_module --language=en_US -d mydb

# Export existing translations
odoo-bin --i18n-export=/tmp/my_module_fr.po --modules=my_module --language=fr -d mydb
```

### Import via CLI

```bash
odoo-bin --i18n-import=/path/to/fr.po --language=fr -d mydb
```

### Import via UI

Settings → Translations → Import Translation → upload `.po` file.

### Where to put .po files in a module

```
my_module/
└── i18n/
    ├── my_module.pot    # template (extracted from source)
    ├── fr.po            # French
    ├── es.po            # Spanish
    └── de.po            # German
```

At module install/update, Odoo auto-loads all `.po` files from `i18n/` that match active
languages.

---

## Common patterns

### Translate a field value in a specific language

```python
# Get the French name of a record
french_name = record.with_context(lang='fr_FR').name

# Translate a computed string in partner's language
def _get_confirmation_text(self):
    lang = self.partner_id.lang or 'en_US'
    return self.with_context(lang=lang).env._(
        "Your order %(name)s has been confirmed.", name=self.name
    )
```

### Mark a field as translatable

```python
# Simple text
name = fields.Char(translate=True)

# Rich HTML (structure preserved)
body = fields.Html(translate=html_translate)

# Plain text with markup (XML-aware)
template = fields.Text(translate=xml_translate)
```

### Avoid translating at module import time

```python
# WRONG — _() evaluated at import, wrong language
LABEL = _("Draft")

# CORRECT — lazy, evaluated when displayed
from odoo.tools.translate import LazyTranslate
_lt = LazyTranslate(__name__)
LABEL = _lt("Draft")
```

### Format a translated string safely with Markup

```python
from markupsafe import Markup, escape

# Safe: escape user data, wrap in Markup
msg = Markup(_("Hello <b>%(name)s</b>")) % {'name': escape(partner.name)}
```
