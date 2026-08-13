# Candidates: Module Structure / General Development (SaaS-19.4)

Discovery pass only — curated candidates for human review, not a finished skill.
Coverage note: I did not exhaustively scan all 4 repos; I sampled recently-touched/newly-added
modules (via `git log --diff-filter=A -- '**/__manifest__.py'`) in `odoo/addons` and `enterprise`,
plus core ORM source in `odoo/odoo/orm/`, and one deep dive into `enterprise/web_studio`. I did not
get to `industry/` or `design-themes/` in depth, and did not check `__init__.py` ordering conventions
beyond a couple of samples. The 8 candidates below are the highest-confidence, best-evidenced finds
from that sampling, not a survey of the whole topic.

---

## 1. `ir.access` replaces both `ir.model.access` AND `ir.rule` — merged model, new CSV format

**Title**: Unified access-control model `ir.access` (CRUD + record-rule domain in one row)

**Claim**: In stale (16/17-era) knowledge, access control is two separate mechanisms: `ir.model.access`
(defined in `security/ir.model.access.csv`, columns `id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink`)
and `ir.rule` (record rules with a `domain_force` field, defined in XML). As of SaaS-19.4 these are
**fully merged** into a single model `ir.access`, and `ir.rule`/`ir_rule.py` no longer exists anywhere
in the codebase. The CSV file is now named `security/ir.access.csv` with columns
`id,name,model_id,group_id/id,operation,domain` where `operation` is a short code combining CRUD letters
(`c`,`r`,`u`,`d` in any combination, e.g. `crud`, `r`, `cu`) and `domain` holds the record-rule-style
domain string directly on the same row.

**Evidence**:
- Model definition: `~/src/194/odoo/odoo/addons/base/models/ir_access.py:16-32` (the `CRUD_SELECTION` map) and `:64-81` (the `IrAccess` model: `_name = 'ir.access'`, fields `model_id`, `group_id`, `operation` (Selection over CRUD_SELECTION), `domain` (Char)).
- Commit that introduced it: `77c17e7111c1` — `[ADD] core: new model ir.access that merges ir.model.access and ir.rule` (dated 2026-06-17 in this checkout's history — i.e. very recent, saas-19.4 timeframe).
- Dedicated upgrade script confirms the migration and old naming: `~/src/194/odoo/odoo/upgrade_code/19.4-00-ir-access.py` (e.g. lines 482-522 read old `ir.model.access.csv` files and old `<record model="ir.rule">` XML and convert them into `ir.access` records).
- Confirmed empty repo-wide search: `find ~/src/194 -iname "ir.model.access.csv"` → 0 results. `find ~/src/194 -iname "ir.access.csv"` → 500 results.
- Real example, new format: `~/src/194/odoo/addons/populate/security/ir.access.csv`:
  ```
  id,name,model_id,group_id/id,operation,domain
  access_populate_job,access_populate_job,populate.job,base.group_system,r,
  ```
- `ir_rule.py` no longer exists: `find ~/src/194/odoo -iname "ir_rule.py"` → no results.

**Why it matters**: A stale-trained agent fixing a "permission denied" / `AccessError` Sentry crash would very likely (a) write or edit a `security/ir.model.access.csv` file with the old 4-boolean-column format, which will simply not be loaded/recognized by this codebase's module structure, or (b) look for an `ir.rule` record to patch a domain restriction and not find the model at all, or misunderstand where the effective domain lives now (it's on the same `ir.access` row as the CRUD permission, not a separate rule row). This would produce a patch that silently does nothing (wrong file name, `ir.model.access.csv` being ignored) or errors out (`ir.rule` no longer registered).

**Confidence**: high

---

## 2. Model class names must mechanically match `_name` (script-enforced, not free-form)

**Title**: Class names are derived from `_name`, including embedded underscores for `l10n_XX`/prefix segments

**Claim**: 16/17-era convention: class names are freeform CamelCase chosen by the developer (e.g. `class ResPartner(models.Model):`, `class ArPartnerTax(models.Model):`). Current codebase: class names are **mechanically derived** from the `_name` string by a script, including preserving underscores after certain segments (notably localization prefixes), so `_name = 'l10n_ar.partner.tax'` becomes `class L10n_ArPartnerTax(models.Model):`, and `_name = 'l10n_es_edi_tbai.document'`-style names become `class L10n_Es_Edi_TbaiDocument(models.Model):`. This looks unusual/wrong to eyes trained on older conventions, but it is the current, applied-everywhere standard.

**Evidence**:
- `~/src/194/odoo/addons/l10n_ar_withholding/models/l10n_ar_partner_tax.py:8` — `class L10n_ArPartnerTax(models.Model):` with `_name = 'l10n_ar.partner.tax'` at line 9.
- Also: `~/src/194/odoo/addons/l10n_br/models/l10n_br_zip_range.py:8` (`class L10n_BrZipRange`), `~/src/194/odoo/addons/l10n_es_edi_tbai/models/l10n_es_edi_tbai_document.py:48` (`class L10n_Es_Edi_TbaiDocument`), `~/src/194/odoo/addons/l10n_eg_edi_eta/models/eta_activity_type.py:8` (`class L10n_Eg_EdiActivityType`).
- Commit that mechanically applied this repo-wide: `f10bd8ddcf99` — `[IMP] *: adapt model class names to correspond to model names (apply script)`, dated 2024-10-15, referencing `Part-of: odoo/odoo#178200` and companion PRs across `odoo/enterprise#69762`, `odoo/design-themes#988`, confirming this was applied across all four repos, not just core.

**Why it matters**: If a stale agent is asked to add a new model or a new class to an existing localization/technical module, it will likely invent a "clean" CamelCase class name (e.g. `class ArPartnerTax` instead of `class L10n_ArPartnerTax`), which is inconsistent with the codebase's actual (script-enforced, likely lint-checked) convention and would look like an out-of-place diff in review, or could clash if the agent renames an *existing* class during a refactor, breaking the invariant other tooling may rely on. Lower functional risk than #1, but a very visible tell of stale-knowledge patches.

**Confidence**: high

---

## 3. `Domain` is a first-class Python object, not just a prefix-notation list

**Title**: `from odoo.fields import Domain` — domains built with `&`, `|`, `~`, `Domain.OR(...)` operators

**Claim**: Stale knowledge: domains in Python code are plain lists in Polish/prefix notation, e.g. `['&', ('state', '=', 'done'), ('amount', '>', 0)]`. Current codebase makes heavy use of a new `Domain` class (`odoo/orm/domains.py`) that lets you build the same thing as `Domain('state', '=', 'done') & Domain('amount', '>', 0)`, with `Domain.OR([...])`/`Domain.AND([...])` helpers, and combinators `&`, `|`, `~`. Old-style list domains still exist/are still accepted in many places, but new code (258+ files found using `Domain(`) is written in the new object style.

**Evidence**:
- Class definition: `~/src/194/odoo/odoo/orm/domains.py:193` (`class Domain:`), with docstring at lines 1-49 describing the dual-notation-vs-object approach and explicitly calling the list/prefix notation "legacy" (line ~29: "For legacy reasons, a domain uses an inconsistent two-levels abstract syntax...").
- Real usage: `~/src/194/odoo/addons/account/models/account_account_tag.py:101-104`:
  ```python
  return self.env['account.report.expression'].search(Domain('engine', '=', 'tax_tags') & Domain.OR(
      ...
      Domain('report_line_id.report_id.country_id', '=', record.country_id.id)
      & Domain('formula', 'in', (record.name, '-' + record.name))
  ```
- `~/src/194/odoo/addons/account/models/product.py:541,543,546` — `Domain('barcode', '=', barcode)`, `Domain('default_code', '=', default_code)`, `Domain('name', '=ilike', name)`.
- Scale of adoption: `grep -rl "from odoo.fields import Domain\|from odoo import.*Domain\b" addons/*/models/*.py` → 258 files.

**Why it matters**: A stale agent patching a search/domain-related bug will default to old-style bracket/prefix-notation lists even inside a file that already uses `Domain(...)` objects nearby, producing an inconsistent mix, or may not know that `&`/`|`/`~` work directly on domain values (potentially "fixing" a bug by manually re-deriving prefix notation instead of just combining two `Domain` objects, increasing the chance of getting `'&'`/`'|'` arity wrong).

**Confidence**: high

---

## 4. `compute_sql` — a new field kwarg for SQL-computed (non-stored) fields

**Title**: Computed fields can now be computed directly in SQL via `compute_sql=`

**Claim**: Beyond the familiar `compute='_compute_x'` (Python method, needs `@api.depends`), there is now a `compute_sql` kwarg where the "compute" is a method that returns a `SQL` fragment (`compute_sql(model, table) -> SQL`), used for fields that must be computed inside `search`/`read_group`/SQL queries without ever loading Python-side compute logic. This is a distinct, additional mechanism, not a renaming of the old one.

**Evidence**:
- Attribute definition and docs: `~/src/194/odoo/odoo/orm/fields.py:241` (`:param str compute_sql: name of a method that produces SQL for the field`) and `:290` (`compute_sql: str | Callable[[BaseModel, TableSQL], SQL] | None = None`).
- Validation logic requiring `compute_sudo` alongside it: `~/src/194/odoo/odoo/orm/fields.py:447-451`.
- Real usage example: `~/src/194/enterprise/../` not needed — core example in `~/src/194/odoo/odoo/addons/base/models/ir_access.py:83-91` (the `kind` field: `compute='_compute_kind', compute_sql='_compute_sql_kind'`) and `:94-101` (`for_read` field: `compute=lambda self: self._compute_for('read')`, `compute_sql=lambda self, table: self._compute_sql_for(table, 'read')`).
- Usage count: `grep -rl "compute_sql=" addons/*/models/*.py` → 22 files (plus core `odoo/orm/*.py`).

**Why it matters**: If a Sentry crash involves a computed field behaving inconsistently between ORM read and `search()`/`read_group()`, a stale agent may only look at (and only patch) the Python `compute` method, missing that the actual value used during search/grouping comes from the separate `compute_sql` method — patching one without the other reproduces the bug in half the code paths.

**Confidence**: medium (mechanism is clearly new and real; I did not find many worked examples outside `ir_access.py`, so I can't yet characterize all its usage rules/edge cases)

---

## 5. Core ORM code physically moved out of `odoo/models.py` / `odoo/fields.py` into `odoo/orm/`

**Title**: `odoo/models.py` and `odoo/fields.py` are now thin re-export shims; real code lives in `odoo/orm/*.py`

**Claim**: In 16/17, `odoo/models.py` and `odoo/fields.py` were large monolithic files containing the actual `BaseModel`, `Model`, `TransientModel`, `Field` classes etc. In this codebase, `odoo/models/__init__.py` and `odoo/fields/__init__.py` are now just import-re-export shims (docstring literally says this split exists "to avoid merge conflicts on `odoo/models.py`"), and the real implementation is split across many files in `odoo/orm/` (`models.py`, `models_transient.py`, `models_cached.py`, `fields.py`, `fields_relational.py`, `fields_temporal.py`, `fields_properties.py`, `domains.py`, `registry.py`, `query.py`, etc). `from odoo import models, fields` still works the same for consumers, so this mostly matters when reading stack traces / Sentry frames, which will point at `odoo/orm/...` file paths, not `odoo/models.py`/`odoo/fields.py`.

**Evidence**:
- Shim file, full contents: `~/src/194/odoo/odoo/models/__init__.py:1-34`, notably line 3 (`# This is a \`__init__.py\` file to avoid merge conflicts on \`odoo/models.py\`.`) and line 25 (`from odoo.orm.models_transient import TransientModel`).
- Directory listing showing the split: `~/src/194/odoo/odoo/orm/` contains `cache.py, commands.py, decorators.py, domains.py, environments.py, fields.py, fields_binary.py, fields_misc.py, fields_numeric.py, fields_properties.py, fields_reference.py, fields_relational.py, fields_selection.py, fields_temporal.py, fields_textual.py, model_classes.py, models.py, models_cached.py, models_transient.py, query.py, registry.py, table_objects.py, types.py, utils.py`.
- `TransientModel` (wizard base class) actual definition: `~/src/194/odoo/odoo/orm/models_transient.py:10` (`class TransientModel(Model):`), not in `models.py` at the top level.

**Why it matters**: When triaging a Sentry traceback that shows a frame in `odoo/orm/fields_relational.py` or `odoo/orm/domains.py`, a stale agent may not recognize this as "core ORM field/domain code" (expecting `odoo/fields.py:NNNN` instead) and could mistakenly treat the frame as belonging to some unfamiliar third-party or addon code, misdirecting the whole triage.

**Confidence**: high

---

## 6. JS unit tests migrated to Hoot (`*.test.js` + `@odoo/hoot`), loaded via `web.assets_unit_tests`

**Title**: New JS test framework "Hoot" runs alongside/replacing QUnit-style tests

**Claim**: Stale (16/17) convention: JS tests live under `static/tests/`, use `QUnit.module(...)`/`QUnit.test(...)`, and are wired into the manifest via a `web.qunit_suite_tests` (or similar) assets bundle. Current codebase has a large, actively-used second convention: tests named `*.test.js` (not `*_tests.js`), written with `import { describe, expect, test } from "@odoo/hoot"`, wired via a `web.assets_unit_tests` manifest bundle. Both conventions currently coexist in the codebase (roughly comparable counts), so an agent needs to recognize which style a given module uses before adding/fixing a test.

**Evidence**:
- Manifest bundle usage counts: `grep -rl "web.assets_unit_tests" addons/*/__manifest__.py` → 102 modules; `grep -rl "web.qunit_suite_tests\|web.assets_tests" addons/*/__manifest__.py` → 121 modules (both conventions actively present).
- Real manifest example: `~/src/194/odoo/addons/pos_cashmatic/__manifest__.py` — `'assets': {'web.assets_unit_tests': ['pos_cashmatic/static/tests/**/*', 'pos_cashmatic/static/src/cashmatic_service.js']}`.
- Real test file using Hoot: `~/src/194/odoo/addons/web/static/tests/reactivity.test.js:1` — `import { describe, expect, test } from "@odoo/hoot";`.
- File naming convention: `find addons -path "*static/tests*" -name "*.test.js"` returns many hits, e.g. `addons/test_mail/static/tests/activity.test.js`, `addons/web/static/tests/env.test.js`.

**Why it matters**: If a Sentry crash is a JS error and the fix requires adding/adjusting a regression test, a stale agent may write a `QUnit.module()`-style test and/or wire it through the wrong manifest bundle key for a module that has already migrated to Hoot (or vice versa), producing a test that is never actually executed by CI/runbot.

**Confidence**: medium (bundle-name and file-naming evidence is solid; I did not verify whether QUnit is fully sunset vs. still the default for older untouched modules)

---

## 7. Web Studio generated models/fields have specific, non-obvious naming quirks

**Title**: Studio model `_name`/table gets `x_` prefix, but the "name" field is `x_name` (not `x_studio_name`), and `x_active` cannot be `x_studio_active`

**Claim**: Web Studio (the enterprise "build your own model/field via UI" feature) generates models and fields following specific naming rules that are easy to get wrong if re-derived from first principles: the model's technical name becomes `x_<sanitized>` (e.g. `studio_model_create`), but the special "display name" field on a Studio-created model is `x_name` (matching core's magic `name` field convention, not the `x_studio_` prefix used for all other Studio-added fields), and the "active" field is hardcoded as `x_active` with an explicit code comment that `x_studio_active` is "not supported by ORM". Every other Studio-added convenience field (`x_studio_sequence`, `x_studio_user_id`, `x_studio_partner_id`, `x_studio_company_id`, `x_studio_notes`, `x_studio_date`, `x_studio_currency_id`, `x_studio_value`, ...) does use the `x_studio_` prefix.

**Evidence**:
- Model/table + `x_name` field creation: `~/src/194/enterprise/web_studio/models/ir_model.py:98-113` (`'model': 'x_' + sanitize_for_xmlid(name)`, and field `'name': 'x_name'` with `'translate': 'standard'`).
- The `x_active` exception, with the explanatory comment: `~/src/194/enterprise/web_studio/models/ir_model.py:225-233`:
  ```python
  def _create_option_use_active(self, model_vals):
      model_vals['field_id'].append(
          Command.create({
              'name': 'x_active',  # can't use x_studio_active as not supported by ORM
              ...
  ```
- Contrast with normal `x_studio_`-prefixed fields on the same model, e.g. `:255` (`'name': 'x_studio_user_id'`), `:268` (`'name': 'x_studio_partner_id'`), `:296` (`'name': 'x_studio_company_id'`).
- Existing-model retrofits (adding Studio fields to a *non*-Studio model) also special-case the active field: `~/src/194/enterprise/web_studio/models/res_company.py:17,28` checks for both `'x_studio_company_id'` and `"x_company_id"` and `'x_studio_currency_id'` names.

**Why it matters**: A Sentry crash touching a Studio-created model (recognizable by an `x_`-prefixed model/table name) that a stale agent tries to "fix" by, say, adding a missing active/name field by hand would very plausibly name it `x_studio_active` or `x_model_name` instead of the actual expected `x_active`/`x_name`, silently breaking Studio's own assumptions about these fields (e.g. `ir.default` value-setting code elsewhere in the same file explicitly sets defaults on `'x_active'` and `'x_studio_company_id'` by exact string name).

**Confidence**: high

---

## 8. `data/neutralize.sql` — an un-declared, convention-only data file for SaaS DB neutralization

**Title**: Any module can ship `data/neutralize.sql`, auto-discovered by filename convention (not listed in the manifest `data` key)

**Claim**: Most files under a module's `data/` directory must be explicitly listed in the manifest's `data` (or `demo`) key to be loaded. `data/neutralize.sql` is an exception: the framework looks it up by a hardcoded path pattern (`f'{module}/data/neutralize.sql'`) for every *installed* module when neutralizing a database (e.g., turning a production/SaaS DB dump into a safe staging copy — disabling outgoing mail, payment providers, cron jobs, etc.), regardless of whether it's mentioned in the manifest at all.

**Evidence**:
- Discovery mechanism: `~/src/194/odoo/odoo/modules/neutralize.py:27-33` (`get_neutralization_queries`):
  ```python
  def get_neutralization_queries(modules: Iterable[str]) -> Iterator[str]:
      for module in modules:
          filename = f'{module}/data/neutralize.sql'
          with suppress(FileNotFoundError):
              with file_open(filename) as file:
                  yield file.read().strip()
  ```
- Real examples present but never referenced in their module's manifest `data`/`demo` keys: `~/src/194/odoo/addons/pos_cashmatic/data/neutralize.sql`, `~/src/194/enterprise/delivery_shipstation/data/neutralize.sql` — confirmed by inspecting `pos_cashmatic/__manifest__.py` and `delivery_shipstation/__manifest__.py` (neither lists `data/neutralize.sql` under `'data'`).

**Why it matters**: This is specifically relevant given the target is *SaaS*-19.4 Sentry triage: crashes reproduced against a neutralized/staging copy of a customer database can look different from production (e.g. disabled payment provider, rewritten cron/mail settings) because of exactly this file, and it won't show up if the agent only reads the manifest's declared `data` files to understand "what gets loaded for this module" — it's invisible unless you know the naming convention.

**Confidence**: high
