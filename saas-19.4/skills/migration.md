# Migration script mechanics — candidates (Odoo SaaS-19.4)

Scope note: this pass focused on `~/src/194/odoo/` (core framework: `odoo/modules/migration.py`,
`odoo/modules/loading.py`, `odoo/modules/module.py`, `odoo/cli/upgrade_code.py`, `odoo/upgrade_code/*.py`)
plus spot-checks of real `migrations/`/`upgrades/` scripts in `addons/`, `enterprise/`, `industry/`.
`design-themes/` was not separately checked (its modules are mostly SCSS/assets, unlikely to carry
Python migration scripts) — flagging this as a gap rather than silently skipping it. Within
`enterprise/` and `industry/` only a sample of scripts was opened (not all 12+ migrations dirs).
8 candidates below, ordered roughly by triage impact.

---

## 1. `ir.model.access` + `ir.rule` have been merged into a single `ir.access` model

**Claim:** Odoo 16/17 security is split across two independent models: `ir.model.access` (CRUD
booleans, loaded from `ir.model.access.csv`) and `ir.rule` (domain-based record rules, loaded from
XML `<record model="ir.rule">`). In 19.4 these are unified into one model, `ir.access`, with a
single `operation` selection field encoding any subset of `crud` and an optional `domain`. There is
no `ir_model_access.py` or `ir_rule.py` left in core.

**Evidence:**
- `~/src/194/odoo/odoo/addons/base/models/ir_access.py:64-81` — model declaration:
  ```python
  class IrAccess(models.Model):
      _name = 'ir.access'
      _description = "Access"
      ...
      operation = fields.Selection(list(CRUD_SELECTION.items()), required=True, ...)
      domain = fields.Char(help="The operations will only be allowed for records in this domain")
  ```
- `CRUD_SELECTION` dict at `ir_access.py:16-32` (e.g. `'crud'`, `'cru'`, `'r'`, ...) — one record can
  now grant/restrict several operations at once.
- Confirmed absence: `find . -iname "ir_model_access.py" -o -iname "ir_rule.py"` in
  `~/src/194/odoo` returns nothing.
- Data files are now named `security/ir.access.csv` (not `ir.model.access.csv`), e.g.
  `~/src/194/odoo/addons/hr/security/ir.access.csv:1-2`:
  ```
  id,name,model_id,group_id/id,operation,domain
  access_hr_employee_category_user,hr.employee.category.user,hr.employee.category,hr.group_hr_user,crud,
  ```
  (header is `operation`+`domain`, not the old `perm_read,perm_write,perm_create,perm_unlink`).
- There is an entire codemod for this rename: `~/src/194/odoo/odoo/upgrade_code/19.4-00-ir-access.py`
  (large file, rewrites old CSV/XML records into the new model on upgrade).
- Legacy per-version migration scripts written before this merge still reference the old tables raw,
  e.g. `~/src/194/odoo/addons/project/upgrades/1.3/pre-migrate.py:1-9` does
  `UPDATE ir_model_access a ... WHERE d.model = 'ir.model.access'` — this is historical (targets a
  module version predating the merge), not a sign that `ir.model.access` still exists today.

**Why it matters:** A stale-trained LLM fixing a Sentry `AccessError`/permission bug will almost
certainly try to add a row to `ir.model.access.csv` or write an `<record model="ir.rule">` XML
record — both patterns are gone. It needs to add a row to `security/ir.access.csv` (or an
`ir.access` XML record) with an `operation` code and an optional `domain`. Getting this wrong means
the patch either silently does nothing (wrong model name → module fails to load / noupdate record
never created) or fails migration entirely.

**Confidence:** high

---

## 2. `check_access_rights()` / `check_access_rule()` merged into a single `check_access(operation)`

**Claim:** The classic two-step ORM access check (`records.check_access_rights('write')` then
`records.check_access_rule('write')`) no longer exists as two methods. There is one final method,
`check_access(operation)`, plus a boolean variant `has_access(operation)`.

**Evidence:**
- `~/src/194/odoo/odoo/orm/models.py:3377-3396`:
  ```python
  @api.private  # use has_access
  @typing.final
  def check_access(self, operation: str) -> None:
      """ Verify that the current user is allowed to perform ``operation`` on
      all the records in ``self``. ...
      """
      if self.has_access(operation):
          return
      inaccessible = self - self._filtered_access(operation)
      domain = self._access_domain(operation)
      raise inaccessible._make_access_error_message(operation, domain)
  ```
- `grep -rn "check_access_rights\|check_access_rule" odoo/orm/models.py addons/base/models/*.py`
  returns **no matches** — the old method names are gone, not just renamed-with-shim.
- The method is decorated `@typing.final` and `@api.private`, i.e. it's the one sanctioned entry
  point now (`has_access` is the "public"/non-raising counterpart).

**Why it matters:** Sentry crashes are commonly `AttributeError: 'x.y' object has no attribute
'check_access_rights'` if an old snippet (or a stale-LLM-authored patch) calls the 16/17-era method
names on a recordset in 19.4. A correct fix/patch must call `check_access('read'|'write'|'create'|'unlink')`
(or `has_access(...)` for a non-raising check), not the two old separate calls.

**Confidence:** high

---

## 3. Migration scripts now have three phases: `pre-`, `post-`, and a new `end-` stage

**Claim:** Odoo 16/17 knowledge of "migrations/<version>/pre-*.py and post-*.py" is incomplete for
19.4: there is now a third, later phase, `end-*.py`, run after **all** modules in the current
transaction have finished updating (not just after the owning module).

**Evidence:**
- Docstring in `~/src/194/odoo/odoo/modules/migration.py:63-68`:
  > "Python file names must start by `pre-` or `post-` and will be executed, respectively, before
  > and after the module initialisation. `end-` scripts are run after all modules have been updated."
- `migrate_module(self, pkg, stage)` type-hints `stage: typing.Literal['pre', 'post', 'end']` —
  `odoo/modules/migration.py:150`.
- Call sites in `~/src/194/odoo/odoo/modules/loading.py`:
  - `line 172`: `migrations.migrate_module(package, 'pre')` — inside the per-module loop, before load.
  - `line 228`: `migrations.migrate_module(package, 'post')` — inside the per-module loop, after data load.
  - `line 483-487`: a **separate loop over the whole graph**, run only once after `STEP 3` (schema/data
    for every module in this run) has completed:
    ```python
    # STEP 3.5: execute migration end-scripts
    if update_module:
        migrations = MigrationManager(cr, graph)
        for package in graph:
            migrations.migrate_module(package, 'end')
    ```
- Real examples exist in local modules, e.g.
  `~/src/194/odoo/addons/l10n_es/migrations/5.4/end-migrate.py` and
  `~/src/194/odoo/addons/l10n_in/migrations/2.2/end-migrate.py`.
- `exec_script()` even has a distinct log-format marker per stage —
  `odoo/modules/migration.py:240-244`: `'pre': f'[>{version}]'`, `'post': f'[{version}>]'`,
  `'end': f'[${version}]'`.

**Why it matters:** If a Sentry crash is caused by a migration ordering issue (e.g. data written by
one module's `post-` script is read by another module's `post-` script before it's committed, or a
cross-module invariant/cleanup needs *every* module to have finished upgrading first), a stale LLM
that only knows pre/post will misplace the fix in `post-*.py` when it should go in `end-*.py`, or
will assume module-local `post-` order equals global order across the whole upgrade graph — it
doesn't; `end-` is the only phase guaranteed to run after everything else.

**Confidence:** high

---

## 4. Migration scripts can live outside the module tree via the `odoo.upgrade` namespace package (`--upgrade-path`)

**Claim:** Beyond a module's own `migrations/` (and `upgrades/`, see #5) folder, Odoo 19.4 has a
first-class external location for migration scripts: the `odoo.upgrade` namespace package, populated
from CLI/config option `upgrade_path` (formerly `--upgrades-paths`), with the old on-disk location
`<addons>/base/maintenance/migrations` kept only as a deprecated compatibility alias.

**Evidence:**
- `~/src/194/odoo/odoo/modules/migration.py:98-103`:
  ```python
  def _get_upgrade_path(pkg: str) -> Iterator[str]:
      for path in odoo.upgrade.__path__:
          upgrade_path = opj(path, pkg)
          if os.path.exists(upgrade_path):
              yield upgrade_path
  ```
  and this is merged into `self.migrations[pkg.name]["upgrade"]` (`migration.py:144-148`), alongside
  the module-local `migrations/` and `upgrades/` folders — all three sources are scanned and unioned.
- `~/src/194/odoo/odoo/modules/module.py:144-176` (`initialize_sys_path`): registers
  `tools.config['upgrade_path']` entries onto `odoo.upgrade.__path__`, and if none configured, falls
  back to `legacy_upgrade_path = <addons_base_dir>/base/maintenance/migrations` (line 159-162).
  It then builds a compatibility shim so old imports still resolve (lines 164-169):
  ```python
  sys.modules["odoo.addons.base.maintenance"] = maintenance_pkg
  sys.modules["odoo.addons.base.maintenance.migrations"] = odoo.upgrade
  ```
- `UpgradeHook.find_spec` in `odoo/modules/module.py:118-127` explicitly comments that this old
  dotted path is legacy but can't be flagged with a `DeprecationWarning` because "0.0.0 scripts, the
  tests, and the common files (utility functions) still need to import from the legacy name."
- Git history confirms recency/intent: `git log --oneline -- odoo/modules/migration.py` (run in
  `~/src/194/odoo`) shows `9c26d4d8707f [IMP] core: rename upgrades-paths to upgrade-path`
  and `bbb1a8f151b0 [IMP] ORM: Add new --upgrades-paths CLI option`.

**Why it matters:** For SaaS-hosted Odoo, the actual migration script for a given bug may not live
in the module's own `migrations/` folder at all — Odoo SA runs a separate, version-spanning
"upgrade scripts" tree (mounted via `--upgrade-path`) for SaaS-specific data fixes, similar in spirit
to the OCA `openupgradelib`/OpenUpgrade convention but now natively supported. A stale LLM asked to
"add a migration script to fix this data" will default to creating a `migrations/<version>/post-*.py`
inside the module, which may be the wrong place if the actual fix needs to run against the SaaS
upgrade-path scripts, or if the module in question has no matching module-version bump to trigger a
module-local script at all.

**Confidence:** medium — the mechanism itself is solidly evidenced; how it maps to the specific
SaaS-19.4 deployment/CI (does this codebase's CI actually populate `--upgrade-path`?) was not
verified, hence not "high".

---

## 5. `upgrades/` is a second, equally-valid folder name alongside `migrations/` — same scripts, same rules

**Claim:** It's not just `<module>/migrations/<version>/{pre,post,end}-*.py`. Some modules use a
sibling folder literally named `upgrades/` instead of `migrations/`, and the loader treats both as
the same script pool (merged before phase/version sorting) — this is not a fallback/legacy-only path,
it's actively used in current addons/enterprise code, including for versions like `1.0.2`, `1.1`
that look like plain module versions, not "0.0.0" special-cases.

**Evidence:**
- `~/src/194/odoo/odoo/modules/migration.py:139-142`:
  ```python
  self.migrations[pkg.name] = {
      'module': get_scripts(check_path(pkg.name + '/migrations')),
      'module_upgrades': get_scripts(check_path(pkg.name + '/upgrades')),
  }
  ```
  Both dicts feed into the same `_get_migration_versions`/`_get_migration_files` logic
  (`migration.py:162-191`), so a `pre-*.py` in `upgrades/1.2/` runs exactly like one in
  `migrations/1.2/` — same ordering, same version-compare rules.
- Real, current examples (not stale/orphaned): `~/src/194/odoo/addons/point_of_sale/upgrades/1.0.2/post-deduplicate-uuids.py`
  (dedupes POS UUIDs after the 18.0 upgrade), and in enterprise:
  `~/src/194/enterprise/l10n_mx_edi_stock/migrations/1.2/post-upgrade.py` /
  `~/src/194/enterprise/approvals/upgrades/`, `~/src/194/enterprise/hr_appraisal/upgrades/`.

**Why it matters:** A stale LLM (or a grep-based agent) searching only for `migrations/` folders to
find "where do I add my fix" could miss that a given module already keeps its upgrade scripts under
`upgrades/` — leading it to create a duplicate/conflicting `migrations/` folder, or to conclude
(wrongly) that the module has no prior migration scripts to pattern-match against.

**Confidence:** medium

---

## 6. `migrate(cr, version)` signature is now strictly validated — extra/kwarg-only params raise `TypeError` at load time

**Claim:** Every migration script's `migrate` function is introspected before being called; only a
fixed small set of parameter-name/kind combinations is accepted. Getting the signature "close enough"
(e.g. adding a default, a `*args`, or a differently-named second parameter) breaks module
loading/upgrade with a hard error, not a silent skip.

**Evidence:** `~/src/194/odoo/odoo/modules/migration.py:233-268`:
```python
VALID_MIGRATE_PARAMS = list(itertools.product(['cr', '_cr'], ['version', '_version']))
...
if not (
        tuple(sig.parameters.keys()) in VALID_MIGRATE_PARAMS
    and all(p.kind in (p.POSITIONAL_ONLY, p.POSITIONAL_OR_KEYWORD) for p in sig.parameters.values())
):
    raise TypeError("module %(addon)s: `migrate`'s signature should be `(cr, version)`, %(func)s is %(sig)s" ...)
```
Only `(cr, version)`, `(cr, _version)`, `(_cr, version)`, `(_cr, _version)` are accepted (the
underscore-prefixed forms are presumably for "intentionally unused arg" linting), any keyword-only
parameter or extra parameter fails.

**Why it matters:** If an LLM writes a migration helper that takes an extra parameter (e.g. an env,
a logger, or `**kwargs` for a shared helper pattern it remembers from a customization), the module
will fail to load with a `TypeError` raised from core loading code — a fairly opaque failure to
trace back to "my migrate() signature is wrong" if you don't know this validation exists.

**Confidence:** medium

---

## 7. `registry.clear_cache()` no longer exists — cache invalidation moved to the transaction: `transaction.invalidate_ormcache()`

**Claim:** Cache invalidation, commonly called from data/migration scripts and business code as
`self.env.registry.clear_cache()` (or `self.pool.clear_cache()` / `.clear_caches()` patterns from
16/17-era code), has been replaced by a transaction-scoped method.

**Evidence:**
- There is a dedicated core-authored codemod for exactly this rename:
  `~/src/194/odoo/odoo/upgrade_code/19.4-00-ormcache-on-transaction.py`:
  ```python
  def upgrade(file_manager: FileManager):
      clear_cache_re = re.compile(r"\bregistry\.clear_cache")
      for file in file_manager:
          if file.path.suffix != '.py':
              continue
          content = file.content
          content = clear_cache_re.sub(r'transaction.invalidate_ormcache', content)
          file.content = content
  ```
- The new method actually exists at `~/src/194/odoo/odoo/orm/environments.py:833`:
  `def invalidate_ormcache(self, cache_name: str = 'default') -> None:` on the `Transaction` class.
- `grep -rn "clear_cache\b" odoo/orm/*.py odoo/api.py` in `~/src/194/odoo` returns nothing
  — confirming `registry.clear_cache` is gone from core, not just discouraged.

**Why it matters:** This is a very plausible Sentry signature: `AttributeError: 'Registry' object has
no attribute 'clear_cache'`. A stale LLM patching a migration/data script that needs to bust an
ormcache after raw SQL writes will reflexively write `self.env.registry.clear_cache()`, which will
now crash. The fix is `self.env.transaction.invalidate_ormcache()` (or the equivalent accessor on
whatever transaction object is in scope).

**Confidence:** high

---

## 8. `type="base64"` field values in XML are renamed to `type="bytes"`

**Claim:** Binary field values embedded in XML data files used `<field name="..." type="base64">` in
older Odoo; 19.4 renames this attribute value to `type="bytes"`. There is a core codemod that does a
blind regex rename across all `.xml` files.

**Evidence:** `~/src/194/odoo/odoo/upgrade_code/19.3-00-base64-in-xml.py`:
```python
def upgrade(file_manager: FileManager):
    b_re = re.compile(r'type="base64"(.*file=|\n.*file=)')
    for file in file_manager:
        if file.path.suffix != '.xml':
            continue
        content = file.content
        content = b_re.sub(r'type="bytes"\1', content)
        file.content = content
```

**Why it matters:** If a Sentry crash traces to a data/migration XML file that sets a binary field
(e.g. an image or attachment payload loaded `file="..."`) using the old `type="base64"` attribute, a
stale-LLM patch that copies that old attribute value into a new XML record risks producing data that
19.4's loader no longer parses the same way. Low-frequency but concrete and easy to get backwards
(an LLM could "fix" a `type="bytes"` file by reverting it to the more-familiar `type="base64"`).

**Confidence:** medium — only verified the codemod's regex intent, did not trace the XML-loader-side
parsing code that consumes `type="bytes"` to confirm the full runtime behavior change.

---

## Non-findings worth noting

- No `openupgradelib` import or vendored copy was found anywhere under `~/src/194`
  (`grep -rln "openupgrade_util\|openupgradelib"` — empty). The "upgrade-util" referenced in this
  repo's own git history (`a8000428618d [IMP] upgrade-util: new hack to force the execution of
  upgrade scripts`, in `~/src/194/odoo` git log) is Odoo SA's own internal/private tool,
  not present in these local repos — its effects show up in core as the `_force_upgrade_scripts`
  mechanism (`~/src/194/odoo/odoo/orm/registry.py:287`,
  `odoo/modules/loading.py:169`, `odoo/modules/migration.py:135,152`), which lets specific module
  names have their upgrade scripts re-run even when the module itself isn't marked `'to upgrade'`.
  Flagging this as a possible 9th candidate the human reviewer may want to expand on, but it was
  deprioritized here since its practical triage impact (vs. the 8 above) is less direct.
- `design-themes/` was not searched at all in this pass (time-boxed); worth a follow-up grep for
  `migrations/` dirs there specifically if theme-related Sentry issues come up.
