# Odoo SaaS-19.3 Manifest Guide

Reference for `__manifest__.py` in SaaS-19.3 modules.
Source of truth: `/home/achraf/src/193/odoo/odoo/modules/module.py`

> **SaaS-19.3 differences vs generic Odoo 19**
> - `post_load` hook: runs after module load, before any model/data init (server-wide, non-registry)
> - `cloc_exclude`: exclude file patterns from line-of-code counting
> - `auto_install` list: specify a *subset* of dependencies that trigger auto-install
> - `kpi_providers`: register KPI summary functions for dashboard modules
> - `bootstrap`: used only by `web` module to load login-screen translations
> - `init_xml`, `demo_xml`, `update_xml`: deprecated, use `data` and `demo`
> - Version strings use `saas~19.3.x.x` format for SaaS-specific migrations

---

## Table of Contents

1. [Minimal manifest](#minimal-manifest)
2. [Module information fields](#module-information-fields)
3. [Dependencies](#dependencies)
4. [Data files](#data-files)
5. [Assets](#assets)
6. [Hooks](#hooks)
7. [External dependencies](#external-dependencies)
8. [auto_install](#auto_install)
9. [SaaS-19.3 specific keys](#saas-193-specific-keys)
10. [Asset bundles reference](#asset-bundles-reference)
11. [Complete example](#complete-example)

---

## Minimal manifest

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

---

## Module information fields

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `name` | str | — | **Required.** Human-readable module name |
| `version` | str | `'1.0'` | Semantic version. SaaS migrations use `'saas~19.3.1.0'` |
| `description` | str | `''` | Long description in reStructuredText |
| `summary` | str | `''` | One-line description (shown in App Store) |
| `author` | str | `''` | Author name |
| `website` | str | `''` | Author/project URL |
| `license` | str | `'LGPL-3'` | See license values below |
| `category` | str | `'Uncategorized'` | App category; use `/` for hierarchy |
| `sequence` | int | `100` | Order in app list |
| `application` | bool | `False` | Appears as top-level App |
| `installable` | bool | `True` | Visible in Apps menu |
| `maintainer` | str | `''` | Maintenance contact |

### License values

`'GPL-2'`, `'GPL-2 or any later version'`, `'GPL-3'`, `'GPL-3 or any later version'`,
`'AGPL-3'`, `'LGPL-3'`, `'OEEL-1'` (Enterprise), `'OPL-1'` (Proprietary), `'Other OSI approved licence'`, `'Other proprietary'`

### Version format

```python
'version': '1.0.0',          # Community/Enterprise module
'version': 'saas~19.3.1.0',  # SaaS-specific migration-aware version
```

The version drives migration script discovery: a script at
`migrations/saas~19.3.1.1/post-migration.py` runs when upgrading to `saas~19.3.1.1`.

---

## Dependencies

```python
'depends': ['base', 'sale', 'account'],
```

All listed modules are installed/updated before this one. `base` is always present but
should still be listed to ensure upgrade ordering.

---

## Data files

```python
'data': [
    # Loaded on install AND update — order matters
    'security/my_module_groups.xml',       # groups first
    'security/ir.model.access.csv',        # then ACLs
    'data/ir_sequence_data.xml',           # sequences
    'views/my_model_views.xml',
    'data/my_module_data.xml',
],
'demo': [
    # Only loaded in demo/test databases
    'demo/demo_data.xml',
],
```

> Files are loaded in order. Put security files before views, and views before data that
> references those views (e.g., `ir.actions.act_window` with `view_id`).

---

## Assets

```python
'assets': {
    'web.assets_backend': [
        'my_module/static/src/js/my_widget.js',
        'my_module/static/src/scss/my_style.scss',
        ('include', 'web._assets_helpers'),          # include another bundle
        ('before', 'web/static/src/some.js',         # insert before a specific file
            'my_module/static/src/my_patch.js'),
        ('remove', 'web/static/src/unwanted.js'),    # remove a file from bundle
        ('replace', 'web/static/src/old.js',         # replace a file
            'my_module/static/src/new.js'),
    ],
    'web.assets_frontend': [
        'my_module/static/src/js/frontend.js',
    ],
},
```

Asset operations: plain path (append), `('include', bundle)`, `('before', target, path)`,
`('after', target, path)`, `('remove', path)`, `('replace', target, path)`.

---

## Hooks

Hooks are Python functions defined in the module's `__init__.py` (or a sub-module it imports).

```python
# In __init__.py or a helper file:

def pre_init_hook(env):
    """Runs before module installation. env is the Odoo Environment."""
    pass

def post_init_hook(env):
    """Runs after module installation."""
    pass

def uninstall_hook(env):
    """Runs after module uninstallation."""
    pass
```

```python
# In __manifest__.py:
'pre_init_hook': 'pre_init_hook',
'post_init_hook': 'post_init_hook',
'uninstall_hook': 'uninstall_hook',
```

Use hooks only for operations that are impossible via data files or ORM methods
(e.g., raw SQL migrations, external service cleanup).

### post_load (SaaS-19.3 — server-wide)

```python
# __manifest__.py
'post_load': 'post_load',

# __init__.py
def post_load():
    """Called after the module's Python code is imported, before any DB/registry work.
    Has NO access to env or cursor. Used for monkey-patching third-party libraries."""
    import some_library
    some_library.some_function = patched_version
```

`post_load` runs once per server process start, not per database. No `env` argument.
Use only for server-wide Python-level setup.

---

## External dependencies

```python
'external_dependencies': {
    'python': ['requests', 'openpyxl', 'lxml'],
    'bin': ['wkhtmltopdf', 'unzip'],
},
```

The module won't install if listed packages/binaries are missing.

---

## auto_install

Controls automatic installation when dependencies are met.

```python
# Install when ALL entries in 'depends' are installed:
'auto_install': True,

# Install when the listed subset of depends is installed (link modules):
'auto_install': ['account', 'sale'],

# Always install (regardless of depends):
'auto_install': [],

# Never auto-install:
'auto_install': False,
```

The list form (`['account', 'sale']`) means: "auto-install only when both `account` and
`sale` are installed." This is used for *bridge/glue* modules (e.g.,
`account_sale_timesheet` that bridges `account` + `sale` + `timesheet`).

> `auto_install: ['module_x']` where `module_x` is a subset of `depends` is the standard
> pattern for integration modules in SaaS. The list must be a subset of `depends`.

---

## SaaS-19.3 specific keys

### `cloc_exclude`

Exclude file paths from line-of-code counting (used by Odoo's internal tooling):

```python
'cloc_exclude': [
    'static/lib/**',
    'tests/tours/**',
],
```

### `kpi_providers`

Register functions that contribute KPI data to the dashboard module:

```python
'kpi_providers': [
    'models.kpi_provider:get_kpi_summary',
],
```

Format: `'python.module.path:function_name'`. The function receives `env` and returns
a dict of KPI values.

### `bootstrap`

Used only by the `web` module — loads translations for the login screen before a database
is selected. Do not use in custom modules.

### Website/theme keys (rarely needed outside OCA/theme modules)

```python
'images': ['static/description/screenshot1.png'],
'live_test_url': 'https://demo.example.com',
'countries': ['fr', 'be'],  # ISO 3166-1 alpha-2 country codes
'configurator_snippets': {'homepage': ['s_banner', 's_features']},
'new_page_templates': {'about': {'key': 'website.about_us', ...}},
```

---

## Asset bundles reference

Common bundles to target in `assets`:

| Bundle | Usage |
|--------|-------|
| `web.assets_backend` | Main backend JS/SCSS |
| `web.assets_backend_lazy` | Lazy-loaded backend assets |
| `web.assets_frontend` | Public website JS/SCSS |
| `web.assets_frontend_minimal` | Minimal assets for unauthenticated pages |
| `web.assets_tests` | Test utilities (tours, unit tests) |
| `web.report_assets_common` | Assets loaded in PDF reports |
| `web.report_assets_pdf` | PDF-specific assets (wkhtmltopdf) |
| `web.assets_web_dark` | Dark mode overrides |
| `web._assets_primary_variables` | SCSS primary variables (override early) |
| `web._assets_secondary_variables` | SCSS secondary variables |
| `website.assets_editor` | Website editor assets |

For icon/font libraries: `web.chartjs_lib`, `web.fullcalendar_lib`, `web.fontawesome`.

---

## Complete example

```python
{
    'name': "My Module",
    'version': '1.0.0',
    'category': 'Accounting/Accounting',
    'summary': "Extends account module with custom features",
    'description': """
My Module
=========
Adds custom accounting features.
    """,
    'author': "Odoo",
    'website': "https://odoo.com",
    'license': 'LGPL-3',

    'depends': ['account', 'sale'],

    'data': [
        'security/my_module_groups.xml',
        'security/ir.model.access.csv',
        'views/my_model_views.xml',
        'data/ir_sequence_data.xml',
    ],
    'demo': [
        'demo/demo_data.xml',
    ],

    'assets': {
        'web.assets_backend': [
            'my_module/static/src/js/my_widget.js',
            'my_module/static/src/scss/my_style.scss',
        ],
    },

    'external_dependencies': {
        'python': ['requests'],
    },

    'post_init_hook': 'post_init_hook',

    'application': False,
    'installable': True,
    'auto_install': False,
}
```
