# __manifest__.py mechanics — SaaS-19.4 candidates

**Scope note:** Deep git-log/code investigation was done in `~/src/194/odoo/` (that's
where `odoo/modules/module.py`, `odoo/modules/loading.py` and the manifest linter live). The
other three repos (`enterprise/`, `industry/`, `design-themes/`) were used for cross-checking via
grep over many `__manifest__.py` files (counts, patterns) rather than their own git history, since
the mechanics that matter are enforced by the core loader/linter, which all four repos share. I
stopped at 8 candidates as instructed; there are more small default-manifest keys (`bootstrap`,
`configurator_snippets*`, `new_page_templates`, `images_preview_theme`, `kpi_providers`) that are
website/theme-specific and lower blast-radius — I did not write those up.

---

## 1. `Manifest` is now a class/Mapping object, not a plain dict — values are deep-copied on every access

**Claim:** Since commit `8e496b3df0df` ("[IMP] modules: odoo.modules.Manifest class", 2025-05-27),
module manifests are represented by `odoo.modules.module.Manifest`, a `@typing.final` class
implementing `Mapping[str, Any]` with a `functools.cached_property` holding the parsed/validated
dict. `__getitem__` returns `copy.deepcopy(self.__manifest_cached[key])` on every access. A later
commit `c72cc70ded77` ("[FIX] core: prevent edit manifest", 2025-08-26) explicitly hardened this so
manifest content "cannot be modified" — code that used to read `terp.manifest_cached` and mutate
the dict in place was changed to use `terp.raw_value('icon')` and the cached property was made
private (`_Manifest__manifest_content`).

**Evidence:**
- `~/src/194/odoo/odoo/modules/module.py:179-200` (class `Manifest`, `__manifest_cached` cached_property)
- `~/src/194/odoo/odoo/modules/module.py:234-237`:
  ```python
  def __getitem__(self, key: str):
      if key in ('description', 'icon', 'addons_path', 'version', 'static_path'):
          return getattr(self, key)
      return copy.deepcopy(self.__manifest_cached[key])
  ```
- Commit `c72cc70ded77`, changing `addons/base_import_module/models/ir_module.py`:
  `terp.manifest_cached.get('icon')` → `terp.raw_value('icon')`.

**Why it matters:** A stale LLM that remembers manifests as an ordinary dict (as returned by
`get_module_resource`/`load_information_from_description_file` in 16/17) might "fix" a bug by
mutating a manifest value obtained through `manifest['data']` / `Manifest.for_addon(x)['assets']`
(e.g. appending a file at runtime, or patching `manifest['depends']` from a `post_load` hook to
work around a Sentry crash caused by a missing dependency). In 19.4 that mutation is silently
thrown away — it operates on a deep copy, not the cached original — so the "fix" does nothing and
the same crash recurs after deploy, looking like the patch didn't ship at all.

**Confidence:** high

---

## 2. `update_xml` and `demo_xml` are completely dead (silently ignored, no warning); only `data`/`demo` (and deprecated `init_xml`) are ever loaded

**Claim:** As of commit `91a2e864e1e1` ("[IMP] core, *: dead load_data stuff", 2025-06-24),
`load_data()` only reads `('init_xml', 'data')` for data-kind loading and `('demo',)` for
demo-kind loading. `update_xml` and `demo_xml` remain declared in `_DEFAULT_MANIFEST` (so no error
if present) but their contents are never iterated/loaded at all — not even with a warning. Only
`init_xml` gets an explicit deprecation warning (still loaded).

**Evidence:**
- `~/src/194/odoo/odoo/modules/loading.py:49-58`:
  ```python
  keys = ('init_xml', 'data') if kind == 'data' else ('demo',)
  ...
  if k == 'init_xml' and package.manifest[k]:
      _logger.warning("module %s: key 'init_xml' is deprecated in Odoo 19.", package.name)
  ```
- `_DEFAULT_MANIFEST` still lists `'demo_xml': []` (line 74) and `'update_xml': []` (line 94) in
  `~/src/194/odoo/odoo/modules/module.py`, but neither key appears anywhere else in
  `loading.py`.
- Cross-check: zero `__manifest__.py` files across all four repos use these keys anymore
  (`grep -rl "'update_xml'\|'demo_xml'\|'init_xml'" --include="__manifest__.py"` → no hits), so
  there's no live example to imitate — the risk is a stale LLM introducing one from memory.

**Why it matters:** If the agent, working from pre-v10-era manifest conventions still floating
around in training data, adds a new data/demo XML file to `update_xml`/`demo_xml` (or copies a
snippet from an old blog/tutorial), the file will silently never load — no error, no log line, the
module just installs "successfully" without the fix. This is worse than the `init_xml` case
(which at least warns): a Sentry bug report about a missing view/record/security rule could get a
patch that looks correct in the diff but has zero effect at runtime.

**Confidence:** high

---

## 3. `external_dependencies['python']` entries are parsed as PyPI requirement specifiers (versions + environment markers), not plain import names

**Claim:** `check_python_external_dependency()` now parses each Python external dependency string
with `packaging.requirements.Requirement`, so an entry like `"requests>=2.31; sys_platform!='win32'"`
is valid and enforced (version range checked via `importlib.metadata.version()` +
`requirement.specifier.contains(version)`; markers evaluated and can skip the dependency
entirely). Plain import-name-only strings (the old `external_dependencies: {'python': ['requests']}`
style) still work but go through the requirement parser first, and are only treated as a bare
importable-module fallback (with a warning to move to a PyPI name) if PyPI metadata lookup fails.

**Evidence:**
- `~/src/194/odoo/odoo/modules/module.py:569-595` (`check_python_external_dependency`):
  ```python
  requirement = Requirement(pydep)
  if requirement.marker and not requirement.marker.evaluate():
      ...
      return
  version = importlib.metadata.version(requirement.name)
  ...
  if requirement.specifier and not requirement.specifier.contains(version):
      msg = f"External dependency version mismatch: {{dependency}} (installed: {version})"
  ```
- Fallback path in same function (lines ~584-590) still does `importlib.import_module(pydep)` with
  a warning when the string isn't a valid PyPI name.

**Why it matters:** A stale LLM fixing a Sentry `ImportError`/`ModuleNotFoundError` crash by adding
`'external_dependencies': {'python': ['some_module']}` (old convention: any importable module name)
may not realize it can (and, per current codebase conventions, should) pin a version range, and
more importantly may not realize that if it guesses a name that happens to *look* like a package
but isn't (`Requirement()` throws `InvalidRequirement` for non-PyPI-safe strings, e.g. names with
dots or slashes) the manifest will raise `ValueError` at module-load time — turning a runtime
ImportError into a hard module-load failure at server boot.

**Confidence:** high

---

## 4. `iap_paid_service` is a new manifest key (Sep 2025) replacing the old dependency-based `has_iap` heuristic

**Claim:** Commit `39620b47ffa2` ("[IMP] *: new manifest field `iap_paid_service`", 2025-09-10)
introduces `'iap_paid_service': False` as a default manifest key. Previously, whether a module
"contains IAP" was computed purely from whether it (transitively) depended on the `iap` module,
which the commit describes as "too simplistic" (e.g. many free EDI modules depend on `iap` without
consuming credits). The key is read directly into the `ir.module.module` model's boolean field
`iap_paid_service` (`fields.Boolean("Contains In-App Purchases")`).

**Evidence:**
- `~/src/194/odoo/odoo/modules/module.py:78`: `'iap_paid_service': False,` in `_DEFAULT_MANIFEST`
- `~/src/194/odoo/odoo/addons/base/models/ir_module.py:328`: `iap_paid_service = fields.Boolean("Contains In-App Purchases")`
- `~/src/194/odoo/odoo/addons/base/models/ir_module.py:779`: `'iap_paid_service': terp.get('iap_paid_service', False),` inside `get_values_from_terp`
- The manifest linter (`~/src/194/odoo/odoo/addons/test_lint/tests/test_manifests.py:88`)
  includes `iap_paid_service` in its list of `verified_keys` (must not equal the default when set).
- Real usage example from the commit diff: `addons/crm_iap_enrich/__manifest__.py` gained
  `'iap_paid_service': True,` alongside pre-existing `'auto_install': True`.

**Why it matters:** A stale LLM investigating a Sentry crash in an IAP-consuming module (e.g. a
credit-related error) might assume — based on 16/17 knowledge — that IAP status is purely derived
from the `depends` list and try to "fix" IAP-flagging bugs by adding/removing a dependency on
`iap`, when the actual mechanism it should touch is this explicit manifest boolean.

**Confidence:** high

---

## 5. `auto_install` accepts `bool` or a `list`/iterable of dependency names — there is no dict-form, and passing a dict degrades silently to "use its keys as the trigger set"

**Claim:** `_load_manifest()` treats `auto_install` as follows: if it's any `Iterable` (this
includes `list`, `set`, **and `dict`**, since Python's `Iterable` check doesn't distinguish them),
it's converted to `set(manifest['auto_install'])` and validated against `depends`; only a bare
truthy/falsy value takes the "auto-install whenever all `depends` are installed" bool path. A
plain `dict` value is never rejected at parse time — it just silently becomes the set of its keys,
because `set({'a': 1, 'b': 2})` == `{'a', 'b'}`. The manifest linter only flags non-list/non-bool
types with a warning (not an error), and that check doesn't run at server boot — only in the
`test_lint` test suite.

**Evidence:**
- `~/src/194/odoo/odoo/modules/module.py:432-445`:
  ```python
  # auto_install is either `False` (by default) in which case the module
  # ...
  if isinstance(manifest['auto_install'], Iterable):
      manifest['auto_install'] = auto_install_set = set(manifest['auto_install'])
      non_dependencies = auto_install_set.difference(depends)
      ...
  elif manifest['auto_install']:
      manifest['auto_install'] = set(depends)
  ```
- Type check in the linter, `~/src/194/odoo/odoo/addons/test_lint/tests/test_manifests.py:140-143`:
  ```python
  elif not isinstance(value, list):
      _logger.warning(
          "Wrong type for manifest value %s in module %s, expected bool or list", key, module)
  ```
- Real-codebase confirmation of the supported (non-dict) forms via grep across all four repos:
  `'auto_install': True` (687 hits) and `'auto_install': [<dep>, ...]` conditional-trigger form
  (e.g. `~/src/194/odoo/addons/l10n_in/__manifest__.py:28`:
  `'auto_install': ['account'],`). Zero hits for a dict literal anywhere as `'auto_install': {`.

**Why it matters:** If the agent — perhaps pattern-matching against other manifest keys that *are*
dicts (like `assets` or `external_dependencies`) — writes `'auto_install': {'sale': True}` intending
a conditional trigger, it will not error anywhere in production. It will parse as
`auto_install = {'sale'}` (the dict's keys), which happens to look correct for a single-key dict
but breaks silently/confusingly for any dict with more than one key or non-True values (e.g.
`{'sale': True, 'purchase': False}` would still trigger on `purchase` too, since only keys are
kept). This is exactly the kind of "looks like it works, doesn't" bug a Sentry-triage agent could
introduce while believing it's using a documented dict form (there isn't one).

**Confidence:** high

---

## 6. `author` has no more implicit default; missing it now logs a warning and falls back to `contributors`/`maintainer`

**Claim:** Until commit `59b9e308365a` ("[IMP] core: make author field mandatory in manifests",
2025-02-04), the default manifest baked in `'author': 'Odoo S.A.'`. Now there is no default value
for `author` at all (`_DEFAULT_MANIFEST` just has a comment `#author, mandatory`); if missing,
`_load_manifest` falls back to `contributors` or `maintainer` (undocumented keys) and logs a
warning, defaulting to `''` if none of those exist either.

**Evidence:**
- `~/src/194/odoo/odoo/modules/module.py:57-61` (comment: mandatory fields with no
  defaults: author, license, name)
- `~/src/194/odoo/odoo/modules/module.py:411-417`:
  ```python
  if not manifest.get('author'):
      author = manifest.get('contributors') or manifest.get('maintainer') or ''
      manifest['author'] = str(author)
      _logger.warning("Missing `author` key in manifest for %r, defaulting to %r", module, str(author))
  ```
- Commit diff for `59b9e308365a` shows the removal of `'author': 'Odoo S.A.'` from
  `_DEFAULT_MANIFEST` and the addition of this fallback block.

**Why it matters:** A stale LLM copying an old-style manifest skeleton (or omitting `author`
entirely, assuming Odoo will silently default it to `'Odoo S.A.'` as in 16/17) will now get either
an empty author string or an unexpected value pulled from a `contributors`/`maintainer` key it may
not have intended as a stand-in — plus a logged warning that a picky Sentry-triage reviewer might
misread as evidence of an unrelated bug in the patched module.

**Confidence:** medium

---

## 7. A CI-enforced manifest linter constrains keys, license strings, and data-file/manifest consistency — a "correct-looking" patch can still fail CI

**Claim:** `odoo/addons/test_lint/tests/test_manifests.py` (`ManifestLinter.test_manifests`) is a
real, running test (tagged `at_install`) that: (a) rejects any manifest key not in a fixed
whitelist `MANIFEST_KEYS` (`_DEFAULT_MANIFEST` keys + `name/icon/addons_path/author/license` +
a few "informative"/app-store keys) — see the "theme_customizations" case below; (b) asserts
`license` is exactly `'OEEL-1'` for any module whose path contains `"enterprise"` and exactly
`'LGPL-3'` otherwise (cross-checked: 752/756 enterprise manifests use `'OEEL-1'` verbatim, 543/554
community manifests use `'LGPL-3'` verbatim — the handful of mismatches are just quoting-style
variants); (c) checks that every `.xml`/`.csv` file physically present under `data/`, `demo/`,
`report(s)/`, `security/`, `template(s)/`, `views/`, `wizard(s)/` is referenced by one of
`data`/`demo`/`other_files`/`*_xml` keys in the manifest, and vice versa (dead references fail
too) — with a special carve-out for `data/template/*` (e.g. `account` chart-of-account templates
loaded dynamically, not via the manifest).

Concretely, the linter itself has needed patches to keep up: `theme_customizations` had to be
explicitly added to `MANIFEST_KEYS` in commit `5ea81f3dd1e4` (2025-09-11) because otherwise-valid
theme manifests were failing `test_manifests` in CI.

**Evidence:**
- `~/src/194/odoo/odoo/addons/test_lint/tests/test_manifests.py:13-22` (`MANIFEST_KEYS`)
- `~/src/194/odoo/odoo/addons/test_lint/tests/test_manifests.py:186-199` (`_test_manifest_license`, exact-match `assertEqual` on `OEEL-1`/`LGPL-3`)
- `~/src/194/odoo/odoo/addons/test_lint/tests/test_manifests.py:201-254` (`_test_data_files`, `DATA_DIRS`/`DATA_EXTS`, the `data/template` exclusion comment `# account @template files, not loaded via the manifest`)
- Commit `5ea81f3dd1e4`, diff to `~/src/194/odoo/odoo/modules/module.py` adding
  `'theme_customizations': {},  # themes` to `_DEFAULT_MANIFEST` specifically to fix this lint failure.

**Why it matters:** A patch that adds a new view/security-rule XML file (very common Sentry-bug
fix shape: "add missing ir.rule", "add missing view inheritance") but forgets to add it to the
manifest's `data` list — or adds a brand-new manifest key the agent invents to record some
metadata — will look completely fine in a manual review/runtime smoke test (Odoo just won't load
the file, no crash) but will fail this CI lint outright. An agent unaware of this lint might
consider the patch "done" when it is not mergeable as-is.

**Confidence:** high

---

## 8. Manifest module is now cache-first and case-strict: only `__manifest__.py` (no legacy `__openerp__.py`), module names must match `\w{1,256}`, and `Manifest.for_addon()` results are `lru_cache`d for the process lifetime

**Claim:** `MANIFEST_NAMES = ['__manifest__.py']` — the old `__openerp__.py` fallback is fully gone
(not even attempted). Module name validity is enforced by `MODULE_NAME_RE = re.compile(r'^\w{1,256}$')`
before a manifest is even looked up — invalid names raise/`return None` rather than being read
loosely. Critically, `Manifest._get_manifest_from_addons` is decorated
`@functools.lru_cache(10_000)`, so once a module's manifest has been resolved for a given name in
the running process, it is never re-read from disk again during that process's lifetime (this
underlies why `Manifest.__getitem__` returning deep copies matters, item #1 above — the whole
`Manifest` object itself is long-lived and cached).

**Evidence:**
- `~/src/194/odoo/odoo/modules/module.py:53-54`: `MODULE_NAME_RE` and `MANIFEST_NAMES = ['__manifest__.py']`
- `~/src/194/odoo/odoo/modules/module.py:280-287`:
  ```python
  # limit cache size because this may get called from any module with any input
  @staticmethod
  @functools.lru_cache(10_000)
  def _get_manifest_from_addons(module: str) -> Manifest | None:
  ```

**Why it matters:** A stale LLM debugging a Sentry crash that looks like "my manifest edit isn't
taking effect after a code change" might not realize the running Odoo worker process caches the
parsed `Manifest` object per module name for its whole lifetime (not just per-request/per-registry
like ORM caches usually are) — so the standard "just clear ir.module / update the module" fix path
is insufficient; a full process/worker restart is required for a manifest content edit (e.g. a
changed `assets` bundle list or `depends`) to be picked up. Also worth noting for anyone tempted to
add a legacy `__openerp__.py` file as a compatibility shim — it will be silently ignored entirely.

**Confidence:** medium
