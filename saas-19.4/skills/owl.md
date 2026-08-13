# OWL / JS Frontend — Candidate Nuances for Odoo SaaS-19.4

Scope note: investigation focused on `odoo/addons/web/static/src` (core JS framework) as the
canonical source, cross-checked against samples in `enterprise/`, `industry/`, and
`design-themes/`. Coverage is not exhaustive of every OWL feature — it is focused on the
highest-impact, best-evidenced divergences from Odoo 16/17-era OWL 2 knowledge. The single
dominant finding is that **SaaS-19.4 ships OWL 3 (alpha)**, and the community `web` addon is
mid-migration to it while most of `enterprise`/`industry`/`design-themes` still use OWL
2‑era syntax. This coexistence is itself the most important thing for the agent to internalize:
it must check the actual target file's style before patching, not assume one global convention.

---

## 1. Core ships OWL 3 (alpha), not OWL 2 — reactivity is signal-based, `useState`/`reactive` are gone

**Claim**: The bundled Owl library is `3.0.0-alpha.45`, a pre-release of OWL 3 built on a
signals-based reactivity model (`signal()`, `proxy()`, `computed()`), not OWL 2's
`useState`/`reactive` Proxy-based system. `useState` has **zero live call sites** anywhere in
`odoo/addons` (community) — the only "hits" for the string are a stale JSDoc comment and
unrelated identifier substrings, not real imports/usages. `odoo/addons/web/static/src/owl2/utils.js`
re-exports several OWL-2-named hooks as compatibility shims over the new primitives, and does not
re-export `useState` at all.

**Evidence**:
- `~/src/194/odoo/addons/web/static/lib/owl/owl.js:1739` — `var version = "3.0.0-alpha.45";`
- `grep -rln "useState" ~/src/194/odoo/addons --include=*.js` → only 3 hits, none of which
  is a real `useState(...)` call (`html_editor/static/src/others/embedded_component_utils.js:6` is a
  JSDoc reference to a legacy concept; `html_editor/static/src/main/table/table_plugin.js` hits are
  `_currentMouseState`, a substring match).
- `~/src/194/odoo/addons/web/static/src/owl2/utils.js:1-25` — explicit compat shim file;
  `useState` is conspicuously absent from its re-exports, while `useRef`, `useComponent`,
  `useEnv`, `useSubEnv`, etc. are shimmed over `globalThis.owl`.
- Real usage of the new reactive primitive instead of `useState`:
  `~/src/194/odoo/addons/web/static/src/core/checkbox/checkbox.js:33` —
  `rootRef = signal(null);` (instance field, not a hook call assigned in `setup()`).
  `~/src/194/enterprise/obox/static/src/widgets/obox_status.js:20` —
  `this.state = proxy({ ... })` used in `setup()` instead of `useState({...})`, even in a file that
  otherwise still uses OWL-2-style `static props`.
- `~/src/194/odoo/addons/web/static/src/owl2/utils.js:7` — deprecation pointer:
  `@deprecated use Owl reactivity {@link https://github.com/odoo/owl/blob/master/doc/v3/owl/reference/reactivity.md}`

**Why it matters**: A stale LLM will reflexively reach for `useState(this, {...})` to add local
reactive state when fixing a Sentry crash in a component's `setup()`. In this codebase that hook
does not exist for newly-written/migrated core code; the fix should use `signal(...)` /
`proxy(...)` (or, in an unmigrated file, follow whatever that file already does). Blindly adding
`useState` will produce an import error or `undefined is not a function` at runtime — a
completely wrong patch for what looked like a one-line fix.

**Confidence**: high

---

## 2. Props declaration changed shape: instance-field `props = props({...})` + `t.`/`types.` schema builder, replacing plain `static props = {...}` — but the two styles coexist file-by-file

**Claim**: Newly-migrated OWL 3 components declare props as an **instance field** built by calling
an imported `props()` function with a schema object whose values are built via a `t.`/`types.`
builder (`t.boolean().optional()`, `t.string().optional()`, `t.function().optional(fn)`,
`t.any()`, `t.object()`, etc.), rather than the OWL-2-era `static props = { foo: { type: Boolean,
optional: true } }` plain-object convention. Both `t` (aliased from `types`) and `props` are
imported directly from `"@odoo/owl"`. Crucially, this is **not yet universal**: `static props =
{...}` still appears in 287 files under `odoo/addons/web` alone (45 in `core/`, 93 in `views/`,
33 in `webclient/`, plus ~105 test files) and in 567 files under `enterprise/` (169 of which also
match `props(`, so there is real per-file variance, not a clean split). A patch must match the
convention already used in the target file/component, not impose one style globally.

**Evidence**:
- `~/src/194/odoo/addons/web/static/src/core/checkbox/checkbox.js:3` —
  `import { Component, props, signal, t } from "@odoo/owl";`
- `~/src/194/odoo/addons/web/static/src/core/checkbox/checkbox.js:22-31` —
  ```js
  props = props({
      id: t.any().optional(),
      disabled: t.boolean().optional(),
      value: t.boolean().optional(),
      slots: t.object().optional(),
      onChange: t.function().optional(() => () => {}),
      className: t.string().optional(),
      name: t.string().optional(),
      indeterminate: t.boolean().optional(),
  });
  ```
  (note: this is an **instance field**, not `static props`.)
- `~/src/194/odoo/addons/web/static/src/core/notification_alert_dialog/notification_alert_dialog.js:8-14`
  — schema built as a standalone exported const (`t.any().optional(true)`) then wired via
  `props = props(notificationAlertDialogProps);` on line 14.
- Raw builder definitions in the bundled lib:
  `~/src/194/odoo/addons/web/static/lib/owl/owl.js:1334-1355` — `var types = { and,
  any, array, boolean, constructor, customValidator, function, instanceOf, literal, number,
  object, or, promise, record, ref, selection, signal, strictObject, string, tuple };`
- Legacy style still very much alive: `~/src/194/enterprise/obox/static/src/widgets/obox_status.js:11-13`
  — `static props = { ...standardWidgetProps };`
- Coexistence within the same test file, both styles imported together:
  `~/src/194/enterprise/obox/static/tests/widgets/json_tags_field.test.js:3` —
  `import { Component, xml, props, types } from "@odoo/owl";` then line 13 uses
  `props = props({ fakeServices: types.array(types.string()) });`.

**Why it matters**: If a Sentry crash is a props-validation error in a core `web` component that
already uses the new schema style, a stale-knowledge patch that "fixes" it by rewriting to
`static props = { foo: Boolean }` will silently drop the runtime validation/optionality semantics
(and may not even be read, since the class no longer treats `props` as a plain contract in the
new convention) — or vice versa, injecting `t.boolean().optional()` schema syntax into an
unmigrated `enterprise` file will throw `t is not defined` / `props is not a function` if that
file never imported those symbols. The agent must grep the target file's own existing
`@odoo/owl` import line before deciding which prop style to write.

**Confidence**: high

---

## 3. Event-handler template expressions now require an explicit `this.` prefix (`t-on-click="this.onClick"`), not bare `t-on-click="onClick"`

**Claim**: In current core templates, inline event/handler expressions in `t-on-*` (and general JS
expression) attributes are written with an explicit `this.` prefix pointing at the component
instance. Searching all XML templates under the core `web` addon, 229 occurrences of
`t-on-xxx="this.foo"`-shaped expressions were found vs. essentially 0 (1, likely a false match)
without the `this.` prefix — a near-total, consistent convention in current templates, unlike
OWL-2-era Odoo templates which commonly wrote `t-on-click="onClick"` (bare method name, resolved
implicitly against the rendering context).

**Evidence**:
- `~/src/194/odoo/addons/web/static/src/core/checkbox/checkbox.xml:5` —
  `<div ... t-on-click="this.onClick" t-ref="this.rootRef">`
- Aggregate check: `grep -roh 't-on-[a-z]*="this\.[^"]*"' addons/web/static/src --include=*.xml | wc -l`
  → 229, vs. `grep -roh 't-on-[a-z]*="[a-zA-Z][^".]*"' ... ` (bare-identifier form) → 1.
- Representative sample of the current style: `t-on-click="this.props.close"`,
  `t-on-click="() => this.props.close()"`, `t-on-click="this._confirm"`,
  `t-on-click="this.toggleStacked"` (all found via the same grep, in
  `odoo/addons/web/static/src/**/*.xml`).

**Why it matters**: A stale LLM patching a crashing click handler in a template is likely to write
the handler reference the OWL-2 way (bare method name) by habit. In a file that has migrated to
this convention, that either fails to resolve at all or resolves to the wrong lexical scope,
producing a template compile/runtime error instead of the intended fix. Conversely, blindly
adding `this.` into a template file that hasn't migrated could also be wrong if that file relies
on an older expression-resolution mode — so this must be verified per-file, but the strong
skew (229:1) means "no `this.`" should be treated as suspicious in any file under active OWL3
migration.

**Confidence**: high

---

## 4. The slot-calling directive was renamed: `t-call-slot="name"` replaces `t-slot="name"` (call-side); `t-set-slot="name"` (define-side) is unchanged

**Claim**: To *render* a slot inside a component's own template, the directive is now
`t-call-slot="name"` (46 files under `odoo/addons/web/static/src`, 31 files under `enterprise/`).
The **old** call-side directive `t-slot="name"` has **zero** remaining occurrences in either
`odoo/addons/web` (community) or all of `enterprise` — this looks like a hard, already-completed
rename at the OWL 3 core level (not a soft, still-supported alias), unlike the props/reactivity
migration above which is genuinely mid-flight. The *define*-side directive used by a caller to
supply slot content, `t-set-slot="name"`, is unchanged and still in wide use.

**Evidence**:
- `~/src/194/odoo/addons/web/static/src/core/checkbox/checkbox.xml:15` —
  `<t t-call-slot="default"/>`
- `~/src/194/odoo/addons/web/static/src/core/dropdown/dropdown_item.xml:15` and
  `~/src/194/odoo/addons/web/static/src/core/dropdown/accordion_item.xml:15` — same
  `t-call-slot="default"` pattern.
- Verified absence of the old call-side directive:
  `grep -rEl '[[:space:]]t-slot="' odoo/addons/web/static/src --include=*.xml` → 0 files.
  `grep -rEl '[[:space:]]t-slot="' enterprise --include=*.xml` → 0 files.
  (An earlier naive `grep t-slot=` matched only `t-set-slot=` substrings, e.g.
  `~/src/194/odoo/addons/web/static/src/core/select_menu/select_menu.xml:68`, which is
  the still-valid define-side directive, not a counterexample.)

**Why it matters**: A stale LLM fixing a "slot content not rendering" Sentry-adjacent bug (e.g. a
`Cannot read property of undefined` from a slot function call) that adds/edits
`<t t-slot="default"/>` based on OWL-2 memory will write a directive that this OWL version
doesn't recognize as the render-slot call at all, likely leaving the slot uncalled/blank rather
than fixing the crash — a silent behavioral regression rather than a loud error.

**Confidence**: high

---

## 5. Services now come in two coexisting shapes: legacy `{dependencies, start(env)}` factory objects vs. new OWL 3 `Plugin` classes registered through a `Resource`

**Claim**: Alongside the long-standing `registry.category("services").add(name, { dependencies,
start(env) {...} })` pattern (still fully functional and still how most services register), core
JS now also defines some services as `Plugin` subclasses (from `@odoo/owl`) added to a
`Resource` named `"services"` exported from `@web/core/services`. The ORM service is the clearest
example: it is implemented as `class ORM extends Plugin`, registered via `services.add(ORM)`, and
then bridged back into the classic registry (`registry.category("services").add("orm", {...})`)
purely as a "todo owl3 migration" temporary shim so old `useService('orm')` call sites keep
working. This dual system is explicitly flagged as in-progress in the source.

**Evidence**:
- `~/src/194/odoo/addons/web/static/src/core/services.js:1-10` —
  ```js
  import { Plugin, Resource, types as t } from "@odoo/owl";
  export const services = new Resource({ name: "services", validation: t.constructor(Plugin) });
  ```
- `~/src/194/odoo/addons/web/static/src/core/orm_plugin.js:1,79` —
  `import { assertType, plugin, Plugin, types as t } from "@odoo/owl";` and
  `export class ORM extends Plugin { ... }`
- `~/src/194/odoo/addons/web/static/src/core/orm_plugin.js` (end of file) —
  `services.add(ORM);` immediately followed by a block comment:
  `@todo owl3 migration` / `temporary - to remove when all use of the orm service are removed`,
  then `registry.category("services").add("orm", { async: [...], start() { const orm =
  Object.create(plugin(ORM)); orm.toString = () => "orm"; return orm; } });`
- Same `@todo owl3 migration` marker recurs in 11 files, e.g.
  `~/src/194/odoo/addons/web/static/src/core/l10n/localization_plugin.js`,
  `~/src/194/odoo/addons/web/static/src/core/browser/title_plugin.js`,
  `~/src/194/odoo/addons/web/static/src/views/view_hook.js`.
- Classic-style service registration is still the norm elsewhere, e.g.
  `~/src/194/odoo/addons/web/static/src/webclient/actions/action_service.js:1941` —
  `registry.category("services").add("action", actionService);` where `actionService` is a plain
  `{ dependencies, start(env) {...} }` object (lines ~1935-1940).

**Why it matters**: If a Sentry crash points into `this.orm.xxx` (or another plugin-backed
service) and the actual defect is in the underlying `Plugin` class, patching the object returned
by `registry.category("services").get("orm")` (the classic pattern) misses the real
implementation, which lives in the `ORM` class body in `orm_plugin.js` and must be patched (or
fixed) as a class, potentially via `patch(ORM.prototype, {...})`, not as a service-factory object.
A stale-knowledge agent that only knows the classic services registry will look in the wrong
place entirely for such bugs.

**Confidence**: medium (the ORM case is concretely verified; how many other services have already
been converted to `Plugin` beyond ORM was not exhaustively enumerated — only cross-referenced via
the 11 files carrying the `@todo owl3 migration` marker).

---

## 6. `useComponent`/`useRef`/env hooks are OWL-2-compatibility shims, not native OWL 3 primitives — the real primitive is `useScope()`

**Claim**: Common hooks that a stale LLM would expect to import straight from `"@odoo/owl"`
(`useComponent`, `useRef`, `useEnv`, `useSubEnv`, `useChildEnv`, `useExternalListener`,
`useLayoutEffect`, `onWillRender`, `onRendered`) are not part of native OWL 3's public API in this
build — they are monkey-patched onto `globalThis.owl` by an explicit "Owl 2 → Owl 3 compatibility
layer" module, built on top of the real OWL 3 primitive `useScope()` (and `signal()`). The web
framework's own `useService` implementation (a hook virtually every component uses) still calls
the shimmed `useComponent()`, meaning even "current" core code routes through the compatibility
layer rather than native OWL 3 APIs.

**Evidence**:
- `~/src/194/odoo/addons/web/static/src/owl2/owl3_compatibility_layer.js:1-35` — file
  banner: "Owl 2 → Owl 3 compatibility layer. This file patches Owl 3 so that existing Owl 2 code
  can continue to run with minimal changes... temporary bridge to ease incremental migration."
- Same file, lines 113-117:
  ```js
  owl.useComponent = function useComponent() {
      return owl.useScope().component;
  };
  ```
- Same file, lines 94-111 — `useRef` reimplemented as a shim wrapping a `signal(null)` under the
  hood rather than OWL 2's ref-registry-by-string-name mechanism.
- `~/src/194/odoo/addons/web/static/src/core/utils/hooks.js:156-158` —
  `useService` itself: `export function useService(serviceName) { const component =
  useComponent(); const { services } = component.env; ... }` — i.e. even the central `useService`
  hook depends on the shimmed `useComponent`, not a native OWL 3 call.
- `~/src/194/odoo/addons/web/static/src/owl2/utils.js:13-25` — a second, narrower
  compat re-export module (`export const useComponent = owl.useComponent;` etc.) used by files
  still importing OWL-2-named hooks explicitly from `@web/owl2/utils` instead of `@odoo/owl`,
  e.g. `~/src/194/odoo/addons/html_editor/static/src/others/embedded_component_utils.js:1`
  — `import { onRendered, useComponent, useRef } from "@web/owl2/utils";`

**Why it matters**: A stale LLM debugging a hook-ordering or ref-timing bug may reason from OWL
2's actual reactive-Proxy/ref-registry internals (e.g. assuming `useRef("foo")` looks up a
DOM-registered string key at render time). Here `useRef` is a shim over a `signal()`, with
different timing/identity semantics (e.g. it exposes `.el` via `untrack(signal)`— an untracked
read). A patch that "fixes" a race condition by changing render/subscribe ordering based on OWL
2 internals may not apply, or may paper over a symptom while the actual signal-timing bug remains.
Also relevant: some files import hooks from `@web/owl2/utils` (Odoo-specific shim) rather than
`@odoo/owl` directly — an agent should preserve whichever import source the surrounding file
already uses rather than "correcting" it to `@odoo/owl`.

**Confidence**: medium (the shim mechanics are concretely verified; the practical runtime timing
differences vs. OWL 2 were not independently exercised/tested, only read from source).

---

## 7. `t-ref` now binds directly to a `signal` instance property (`t-ref="this.rootRef"`), not a string name paired with `useRef("name")`

**Claim**: Newly-migrated components declare a ref as a plain instance field created with
`signal(null)` and reference it in the template as `t-ref="this.<field>"` (a JS expression
evaluating to the signal), rather than OWL 2's pattern of `useRef("someName")` in `setup()` plus
a matching string literal `t-ref="someName"` in the template. The compatibility layer explicitly
tells migrators to rename `t-ref` to `t-custom-ref` when they still want the *old* string-based
`useRef` behavior while running on Owl 3 with the shim loaded — i.e. the bare `t-ref` attribute
name has different semantics depending on which convention the surrounding code follows.

**Evidence**:
- `~/src/194/odoo/addons/web/static/src/core/checkbox/checkbox.js:33` —
  `rootRef = signal(null);` (instance field).
- `~/src/194/odoo/addons/web/static/src/core/checkbox/checkbox.xml:5` —
  `t-ref="this.rootRef"` (expression referring to that field, not a string constant).
- `~/src/194/odoo/addons/web/static/src/core/checkbox/checkbox.js:42-47` — the ref is
  read by calling it as a function: `area: () => this.rootRef()`, and in `onClick`,
  `this.rootRef().querySelector("input")` — i.e. `this.rootRef` is callable (a signal getter),
  not a `{ el }` object as OWL 2's `useRef` return value was.
- `~/src/194/odoo/addons/web/static/src/owl2/owl3_compatibility_layer.js:12-15` —
  explicit migration instruction: "replace `t-ref` → `t-custom-ref`" when bringing OWL-2 code onto
  this compatibility layer, confirming `t-ref` itself is repurposed/ambiguous across the two
  conventions in this codebase.

**Why it matters**: A stale LLM fixing a null-ref crash (`Cannot read properties of null (reading
'...')` from `this.someRef.el`) will likely reproduce the OWL 2 idiom: `this.someRef =
useRef("someName")` in `setup()` + `t-ref="someName"` string in the template, then access
`.el`. In a migrated file this is a structural mismatch (no `useRef` call site pattern to hook
into, and reading `.el` off a bare signal is wrong — the checkbox example calls the ref as a
function `this.rootRef()` to get the element directly, no `.el` property). The wrong patch either
throws immediately or silently returns `undefined`, and in an unmigrated file using
`t-custom-ref`, "correcting" it back to plain `t-ref` under old assumptions could break the
compatibility-layer wiring for that ref instead of fixing anything.

**Confidence**: high

---

## 8. Migration is explicitly partial and file-granular — grep the target file's own imports/markers before choosing a convention

**Claim**: This is a meta-finding rather than a single API fact, but it is the highest-leverage
takeaway: SaaS-19.4's JS frontend is a genuine two-Owl-version codebase in transition, not a
uniformly-upgraded one. The proportions found make this concrete: `static props` still appears in
287 non-test files under `odoo/addons/web` alone (out of ~530 component-ish files there) and in
567 files under `enterprise/`, while the new `props(...)+t.` idiom is already the norm in several
core `core/` subfolders (checkbox, dialog, autocomplete, dropdown, datetime, domain_selector,
etc.). The literal string `@todo owl3 migration` appears in 11 separate core files as a marker
that a given file/mechanism is a known temporary bridge. This means there is no single "current
Odoo 19.4 OWL convention" to memorize — the correct convention is a per-file/per-module fact that
must be checked, not assumed, before writing a patch.

**Evidence**:
- `grep -rln "static props" odoo/addons/web --include=*.js` → 287 files (93 in `views/`, 45 in
  `core/`, 33 in `webclient/`, 9 in `search/`, remainder tests) vs. many `core/` files already on
  `props(...)+t.` (see Candidate 2's evidence).
- `grep -rln "static props" enterprise --include=*.js` → 567 files.
- `grep -rln "owl3 migration" odoo/addons/web/static/src -r` → 11 files, including
  `~/src/194/odoo/addons/web/static/src/core/position/position_hook.js`,
  `~/src/194/odoo/addons/web/static/src/search/action_hook.js`,
  `~/src/194/odoo/addons/web/static/src/core/dropzone/dropzone_hook.js`,
  `~/src/194/odoo/addons/web/static/src/views/view_button/view_button_hook.js`.
- Direct coexistence in a single enterprise test file:
  `~/src/194/enterprise/obox/static/tests/widgets/json_tags_field.test.js:3,13` —
  imports both `props` and `types` from `@odoo/owl` and uses the new schema-builder idiom, while
  the sibling production file `~/src/194/enterprise/obox/static/src/widgets/json_tags_field.js:10`
  still uses `static props = {...}` (old idiom) for the same component family.

**Why it matters**: This directly guards against the single most likely failure mode for a
stale-knowledge agent: picking *a* plausible-looking OWL convention (old or new) and applying it
uniformly to whatever file the Sentry stack trace points at, instead of first checking that
specific file's existing imports (`static props` vs `props()`+`t.`, `useState`/`useRef` vs
`signal()`, `t-slot` vs `t-call-slot`, bare handler names vs `this.`-prefixed) and matching it.
Any patch that "modernizes" or "reverts" a file's convention as a side effect of a bug fix is very
likely an over-reach and a red flag in review.

**Confidence**: high
