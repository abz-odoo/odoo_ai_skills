# Translation mechanics — candidates (Odoo SaaS-19.4)

Scope note: this pass focused on `odoo/odoo/tools/translate.py`, `odoo/odoo/orm/environments.py`,
`odoo/odoo/orm/fields_textual.py`, `odoo/odoo/orm/models.py`, and the tests in
`odoo/odoo/addons/test_tools/tests/test_translate.py`, `odoo/odoo/addons/test_orm/tests/test_related_translation.py`
and `test_indexed_translation.py`. Usage-frequency greps were run across all four repos
(`odoo`, `enterprise`, `industry`, `design-themes`). This is not exhaustive — `test_related_translation.py`
(626 lines) was located but not read in full, and JS-side translation (`web/static/src/core/l10n`) was not
investigated at all. Repos are plain directories, not git repos, so `git log` could not be used.

---

## 1. `_lt = LazyTranslate(__name__)` is the canonical pattern, not a bare `_lt` import

**Claim:** In 19.4 the idiomatic way to get a module-level lazy translator is to instantiate
`LazyTranslate(__name__)` and bind it to `_lt` yourself, not to import a ready-made `_lt` symbol
from `odoo.tools.translate`. The module does still expose a module-level `_lt = LazyGettext` (the class
itself, translate.py line 1020), which technically works if called directly, but it is **not** in
`__all__` (translate.py lines 47-51 export only `"_"`, `"LazyTranslate"`, `"html_translate"`, `"xml_translate"`)
and is used directly in only 2 files in the entire 4-repo corpus, versus 149 files using the
`_lt = LazyTranslate(__name__)` pattern.

**Evidence:**
- `odoo/odoo/tools/translate.py:996-1020` — `class LazyTranslate` (the factory) and the trailing
  `_ = get_text_alias` / `_lt = LazyGettext` module-level aliases.
- `odoo/odoo/tools/translate.py:47-51` — `__all__` list omits `_lt`.
- Canonical usage docstring at `odoo/odoo/tools/translate.py:996-1016`:
  ```python
  class LazyTranslate:
      """ Lazy translation template.
      Usage:
      _lt = LazyTranslate(__name__)
      MYSTR = _lt('Translate X')
      """
      def __init__(self, module: str, *, default_lang: str = '') -> None:
          self.module = module = get_translated_module(module or 2)
          ...
      def __call__(self, source: str, *args, **kwargs) -> LazyGettext:
          return LazyGettext(source, *args, **kwargs, _module=self.module, _default_lang=self.default_lang)
  ```
- Grep counts (this session): `grep -rl "_lt = LazyTranslate(" odoo/ enterprise/ industry/ design-themes/` → 149 files
  (101 in odoo core+addons, 48 across enterprise/industry/design-themes) vs. only
  `odoo/addons/account_edi_ubl_cii/models/account_edi_common.py:16` and
  `odoo/odoo/addons/base/models/ir_access.py:12` doing `from odoo.tools.translate import _lt` directly.
- Core framework files bind a module-wide `_lt` this way too, e.g. `odoo/odoo/orm/models.py:85`
  (`_lt = LazyTranslate('base')`), `odoo/odoo/tools/image.py:25`, `odoo/odoo/tools/template_inheritance.py:15`.

**Why it matters:** A stale-trained LLM patching a bug that touches a module-level lazy string
(e.g. adding a new `_lt("some error")` constant) would most likely write
`from odoo.tools.translate import _lt` at the top and call `_lt("text")` directly — copying the pre-19.4
`lazy()`-wrapper idiom. This still runs (since `_lt` still resolves to `LazyGettext`), but it silently
relies on frame-based module detection (`get_translated_module(_module or 2)`, stack level 2) instead of
the explicit `__name__` passed by `LazyTranslate`. If the LLM's fix wraps the call in a helper function or
decorator, the frame-walk can attribute the string to the wrong module, breaking `.po` extraction/lookup
for that string. The correct/expected fix is the `_lt = LazyTranslate(__name__)` module-level singleton.

**Confidence:** high


## 2. `env._()` resolves language directly; global `_()` / lazy `str(_lt(...))` resolve it by walking the call stack

**Claim:** `self.env._(...)` (the `Environment._` method) reads `self.lang` directly — a cached property
already anchored to a known environment. The module-level `_()` (`get_text_alias`) and the lazy value
returned by `_lt(...)` (via `LazyGettext.__str__` → `_translate()`) instead have to *guess* the current
language by walking the Python call stack looking for `cr`/`self.env`/`request.env`/etc. This is
fundamentally different machinery, not just two spellings of the same call.

**Evidence:**
- `odoo/odoo/orm/environments.py:344-369` — `Environment._`:
  ```python
  def _(self, source: str | LazyGettext, *args, **kwargs) -> str:
      lang = self.lang or 'en_US'
      if isinstance(source, str):
          ...
      elif isinstance(source, LazyGettext):
          return source._translate(lang)      # <-- direct, no frame-walking
  ```
- `odoo/odoo/tools/translate.py:838-915` — `_get_cr`, `_get_uid`, `_get_lang`, `_get_translation_source`:
  these walk `frame.f_locals` looking for `cr`/`cursor`/`self.env`/`request`, and log a
  `WARNING`/`DEBUG` (line 903: `_logger.log(log_level, 'no translation language detected, skipping
  translation %s', frame, stack_info=True)`) if nothing is found — i.e. silent/degraded translation
  is a designed fallback path, not just a bug.
- `odoo/odoo/tools/translate.py:918-922` — `get_text_alias` (`_()`) calls
  `_get_translation_source(1)` which triggers this frame walk on every call.
- `odoo/odoo/tools/translate.py:926-946` — the `LazyGettext` docstring explicitly contrasts the two:
  > "Similar to get_text_alias but the translation lookup will be done only at `__str__` execution...
  > A code using translated global variables... works as expected (unlike the classic get_text_alias
  > implementation)." — with the recommended pattern being `env._(LABEL)`, not bare `str(LABEL)`.

**Why it matters:** If a Sentry crash shows a translation appearing in the wrong language (or English
leaking through in a non-English tenant), a stale-knowledge LLM will likely "fix" it by adding more
`_()`/`_lt()` calls or fiddling with `with_context(lang=...)elsewhere, assuming `_()` behaves like a pure
string-formatting function. The actual fix in 19.4 code is usually to route the translation through
`env._(...)` (or `env._(LAZY_CONST)`) so it reads `self.lang` directly instead of relying on stack
introspection succeeding at the exact right frame — e.g. inside threads, generators, or lazily-evaluated
values passed across call boundaries where the "self.env" frame is no longer on the stack when `str()`
is finally invoked.

**Confidence:** high


## 3. `ir.translation` model no longer exists — translations are stored per-field (jsonb) and per-module (.po files read at runtime)

**Claim:** There is no `ir.translation` model anywhere in the 19.4 codebase. Any Sentry traceback or
patch idea that references `env['ir.translation']` (searching/creating/writing translation rows) is
targeting a model that does not exist in this version.

**Evidence:**
- `grep -rln "'ir\.translation'" odoo/ enterprise/ industry/ design-themes/` (excluding `.po`/`.pot` files)
  → zero hits across all four repos.
- Confirmed replacement storage: translated field values are jsonb dicts, `{lang_code: value}`, handled by
  `StoredTranslations` (`odoo/odoo/tools/translate.py:468-598`) and read/written via
  `Field._get_stored_translations` (`odoo/odoo/orm/fields_textual.py:217`, referenced in
  `test_translate.py` `test_get_stored_translations`, ~line 716-758).
- Code/UI-string translations (from `.po` files) are read live from disk per request via
  `CodeTranslations` (`odoo/odoo/tools/translate.py:2172-2209`, `_get_code_translations` reading
  `get_po_paths(module_name, lang)`), not queried from a DB table.
- `odoo/odoo/orm/fields.py:825` — `column_type` returns `('jsonb', 'jsonb')` for
  `self.company_dependent or self.translate` fields, confirming the storage shape.

**Why it matters:** A stale LLM asked to fix "translation not saving" or "wrong translation displayed"
bugs may write a patch that does `self.env['ir.translation'].search([...])` or creates/updates rows in
that model — this will raise `KeyError`/`ValueError: Invalid model name` outright. The real fix path is
through the field's stored jsonb value (`record._fields['x']._get_stored_translations(record)`,
`record.update_field_translations(...)`, see candidate 5) or through `.po` file / `TranslationImporter`
mechanics for module-level and code-string translations.

**Confidence:** high


## 4. Technical `_`-prefixed language keys (`_en_US`, `_fr_FR`, …) inside stored jsonb translations for HTML/QWeb (`model_terms`) fields

**Claim:** For fields whose `translate` parameter is a callable (`html_translate` / `xml_translate` —
i.e. "model_terms" translated fields), the stored jsonb dict is not simply `{lang: rendered_value}`.
It also contains "technical" keys prefixed with an underscore (e.g. `'_en_US'`, `'_fr_FR'`) holding the
**raw/original** value (with the un-substituted term structure) used to keep term extraction/re-translation
consistent across edits. These technical languages are invalid as an actual `context['lang']` value and
raise `UserError` if used directly.

**Evidence:**
- `odoo/odoo/tools/translate.py:479-484` — `StoredTranslations.fallback_langs`:
  ```python
  if lang == '_en_US':
      return '_en_US', 'en_US'
  if lang.startswith('_'):
      return lang, lang[1:], '_en_US', 'en_US'
  ```
- `odoo/odoo/tools/translate.py:568-579` — `_validate` checks "technical languages which start with `_`"
  and raises `ValidationError` if a technical key exists without its corresponding real-language key.
- `odoo/odoo/tools/translate.py:600-657` — `StoredTranslations.written()` docstring: "store recomputed
  values in technical keys (`_lang`)" (line 613), and the body reads/writes `valid_self['_' + k]` throughout.
- `odoo/odoo/orm/environments.py:334-341` — `Environment._lang` cached property:
  ```python
  def _lang(self) -> str:
      """... technical language code ... for **model_terms** translated field"""
      lang = self.lang or 'en_US'
      if context.get('edit_translations') or context.get('check_translations'):
          lang = '_' + lang
  ```
- `odoo/odoo/addons/test_tools/tests/test_translate.py:560-561` — explicit test:
  ```python
  with self.assertRaises(UserError):
      # technical langauge starts with '_'
      category.with_context(lang='_en_US').name
  ```

**Why it matters:** If a Sentry crash dump shows a jsonb translation dict with keys like `_en_US` or
`_fr_FR`, a stale LLM is likely to assume this is corrupted/malformed data (duplicate/mistyped language
codes) and "fix" it by stripping those keys or normalizing them away — which would break re-translation
term-matching for HTML/QWeb `model_terms` fields (`arch_db`, `description`, etc. on `ir.ui.view`,
`mail.template`, etics.). Conversely, if asked to construct a translations dict manually (e.g. in a data
migration script) for such a field, omitting the `_lang` technical keys would produce a state the ORM's
own validator (`StoredTranslations._validate`) treats as inconsistent.

**Confidence:** high


## 5. `record.update_field_translations(field_name, translations, source_lang='')` — public bulk-translation-write API

**Claim:** There is a first-class model method for atomically writing translations for *multiple*
languages of one field in one call, with the accepted shape of `translations` depending on whether the
field is a plain translatable field (`{lang: new_value | False}`) or a `model_terms` field
(`{lang: {old_term: new_term}}` — nested dict). Passing the wrong shape for the field kind raises
`UserError`.

**Evidence:**
- `odoo/odoo/orm/models.py:2760-2799` (`update_field_translations` / `_update_field_translations`):
  ```python
  def update_field_translations(
      self, field_name: str,
      translations: dict[str, str | typing.Literal[False] | dict[str, str]],
      source_lang: str = '',
  ) -> bool:
      """ ... See 'self._update_field_translations' docstring for details. """
  ```
  Docstring: if `field.translate is True`, values are `str` or `False` (removes translation, falls back to
  latest `en_US`); if `field.translate` is callable, values must be `{old_source_lang_term: new_term}` dicts.
- Test coverage in `odoo/odoo/addons/test_tools/tests/test_translate.py:1021-1056`
  (`test_update_field_translations`), including:
  ```python
  with self.assertRaises(UserError):
      # raise error when the translations are in the form for model_terms translated fields
      self.category.update_field_translations('name', {'fr_FR': {'English Name': 'French Name'}})
  ```
  and `test_update_field_translations_for_empty` (line ~1058) showing `False` removing a translation and
  falling back to another lang's value as the new default.

**Why it matters:** A stale LLM fixing a "translation didn't save for language X" bug will likely reach
for `record.with_context(lang='fr_FR').write({'name': 'val'})` in a loop over languages — which works for
plain fields but is easy to get wrong for `model_terms` fields (HTML/QWeb), and doesn't match the
atomic/validated code path the ORM itself uses. Not knowing `update_field_translations` exists means
reinventing per-language write loops that can leave `StoredTranslations` in a state that fails
`_validate()` (see candidate 4).

**Confidence:** medium-high


## 6. `translate=` field parameter is still `bool | callable`, but `ir.model.fields.translate` stores a string identifier via `FIELD_TRANSLATE`

**Claim:** At the Python field-definition level, `translate=` only ever takes `True`/`False` or a callable
(`html_translate`/`xml_translate`) — this part is unchanged from recent Odoo. But the *database-backed*
reflection of that field parameter, `ir.model.fields.translate` (used for custom/manual fields), stores a
plain string (`None`, `'standard'`, `'html_translate'`, `'xml_translate'`) that is mapped back and forth via
the `FIELD_TRANSLATE` dict — a different vocabulary than the Python-level parameter.

**Evidence:**
- `odoo/odoo/orm/fields_textual.py:33-40` — `BaseString.__init__`:
  ```python
  translate: bool | Callable[[Callable[[str], str | None], str], str] = False
  if 'translate' in kwargs and not callable(kwargs['translate']):
      kwargs['translate'] = bool(kwargs['translate'])
  ```
- `odoo/odoo/tools/translate.py:90-93` — `FIELD_TRANSLATE = {None: False, 'standard': True}`, extended at
  lines 451-452: `FIELD_TRANSLATE['html_translate'] = html_translate`,
  `FIELD_TRANSLATE['xml_translate'] = xml_translate`.
- `odoo/odoo/addons/base/models/ir_model.py:1243` — reflecting a field back to DB:
  `translate = next(k for k, v in FIELD_TRANSLATE.items() if v == field.translate)`.
- `odoo/odoo/addons/base/models/ir_model.py:1378-1380` — reconstructing a field from DB:
  `attrs['translate'] = FIELD_TRANSLATE.get(field_data['translate'], True)`.
- Test cross-check: `odoo/odoo/addons/test_orm/tests/test_schema.py:62`:
  `self.assertEqual(FIELD_TRANSLATE.get(ir_field.translate or None, True), field.translate)`.

**Why it matters:** An LLM writing a fix that touches `ir.model.fields` rows directly (e.g. a data
migration toggling translation on a custom field) might set `translate = True` (Python-level convention)
directly on the `ir.model.fields` record, which is wrong — the DB column expects the string token
(`'standard'`, `'html_translate'`, `'xml_translate'`, or `None`/falsy), not a boolean.

**Confidence:** medium


## 7. `prefetch_langs` context key — an explicit perf lever for translated-field reads that batches all languages in one query

**Claim:** Reading a translated field for one language normally issues one query per record batch, but
setting `context['prefetch_langs'] = True` changes the fetch to also aggregate/prefetch all languages'
values together (avoiding N further queries when the same field is later read in a different `lang`
context on the same recordset). This is a concrete, documented performance mechanism, verified by an
explicit query-count test.

**Evidence:**
- `odoo/odoo/orm/fields_textual.py` — `prefetch_langs` referenced at lines 237, 249, 255, 257, 261, 263,
  274, 291, 363, 405, 413, 427, 430, 457, 458, 485 (pervasive throughout the translated-field read/write path).
- `odoo/odoo/addons/test_tools/tests/test_translate.py:691-714` — `test_111_prefetch_langs`:
  ```python
  category_fr.with_context(prefetch_langs=True).name
  ...
  with self.assertQueryCount(1):
      self.assertEqual(category_fr.with_context(prefetch_langs=True).name, 'Clients')
  with self.assertQueryCount(0):
      self.assertEqual(category_nl.name, 'Klanten')
      self.assertEqual(category_en.name, 'Customers')
  ```

**Why it matters:** If a Sentry issue is actually a performance problem (timeout, too many queries) tied
to reading a translated field across many languages/records, a stale LLM won't know `prefetch_langs`
exists and may instead propose ad hoc caching or bulk `read()` workarounds, missing the ORM-native lever
that's already wired into `_get_stored_translations`/`to_sql`/`_to_prefetch` for exactly this case.

**Confidence:** medium


## 8. `get_translation()` auto-escapes `Markup` args and auto-localizes list-like args via `format_list` before `%`-formatting

**Claim:** The core translation-formatting function (used by both `_()` and `env._()`) does more than a
`translation % args` substitution: if any positional/keyword arg is a `Markup` instance, the translated
string itself is HTML-escaped first (to avoid double-escaping/broken markup); if any arg is an
`Iterable` (and not `str`/`bytes`), it is automatically passed through `format_list(...)` to render it as
a localized, language-aware list (e.g. "a, b and c" vs "a, b et c") before substitution.

**Evidence:**
- `odoo/odoo/tools/translate.py:764-801` — `get_translation`:
  ```python
  if any(isinstance(a, Markup) for a in (...)):
      translation = escape(translation)
  if any(isinstance(a, LazyGettext) for a in (...)):
      args = {... v._translate(lang) ...}  # or tuple(...)
  if any(isinstance(a, Iterable) and not isinstance(a, (str, bytes)) for a in (...)):
      def process_translation_arg(v):
          return format_list(env=None, lst=v, lang_code=lang) if isinstance(v, Iterable) and not isinstance(v, (str, bytes)) else v
      ...
  return translation % args
  ```

**Why it matters:** A stale LLM debugging a Sentry error like a `TypeError`/`ValueError` in `%`-formatting
of a translated string, or a report of HTML being double-escaped / a list rendering as a raw Python
`repr` instead of a natural-language list, would not expect the translation layer itself to be silently
pre-processing `Markup` and iterable arguments — it would look purely at the call site's `%s`/`%(name)s`
placeholders and args, missing that passing a plain `list`/`tuple`/`set` as a translation arg is meant to
auto-format as a localized list, and that passing `Markup` values changes escaping behavior for the whole
string, not just that argument.

**Confidence:** medium (mechanism is clearly evidenced; could not confirm in the time available whether
this auto-list-formatting existed already in 17, so it may be a partial rather than fully new behavior)
