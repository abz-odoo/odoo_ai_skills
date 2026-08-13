# @api decorators — SaaS-19.4 candidate nuances

Scope note: I focused depth on the core `@api` decorator surface (`odoo/orm/decorators.py`,
re-exported via `odoo/api/__init__.py`), its interaction with the RPC/HTTP dispatch layer
(`odoo/addons/web/controllers/dataset.py`) and the ORM (`odoo/orm/models.py`,
`odoo/orm/environments.py`). I verified every commit-log-based claim with
`git merge-base --is-ancestor <sha> HEAD` against the `~/src/194/odoo` checkout to
make sure it actually landed in this SaaS-19.4 branch (not just visible via `--all` on some other
branch). One promising lead — a "deferred/lazy `@api.constrains`" refactor (commits
`75782af9e5ce`, `f41f4e4d1905`) — turned out to **not** be an ancestor of HEAD, i.e. not in
19.4, and was discarded after verification; I'm calling this out explicitly since it's exactly
the kind of false positive the method in the brief warns about. I did not deeply chase the
`@api.onchange` "is it replaced by computed writable fields" angle beyond a usage-count sanity
check (still 245 real `@api.onchange` occurrences in `addons/`, docstring unchanged) — nothing
version-specific enough to 19.4 turned up quickly, so I left it out rather than pad the list.
8 candidates below, all with file:line evidence.

---

## 1. `@api.readonly` now literally selects a read-only DB connection for RPC calls

**Claim:** `@api.readonly` is not just documentation/metadata anymore. For requests going
through `/web/dataset/call_kw` (i.e. almost all JS-client and external RPC calls), the presence
of `_readonly` on the target method directly decides whether the HTTP route opens a **read-only
Postgres connection** (`transaction_read_only = on`) or a read/write one. Base ORM methods like
`read`, `search`, `search_read`, `search_count`, `search_fetch`, `name_search` are marked
`@api.readonly`. If a custom override of one of these (or of any RPC-exposed method carrying
`@api.readonly`) performs a write (e.g. lazy cache population via SQL `INSERT`/`UPDATE`, a
`self.env.cr.execute(...)` side effect, calling `write()`/`create()` internally), it will now
raise a **Postgres `ReadOnlySqlTransaction` error** where in 16/17 it would have silently
succeeded.

**Evidence:**
- `~/src/194/odoo/odoo/addons/web/controllers/dataset.py:15-28` — `_call_kw_readonly` walks the model's MRO and returns `method._readonly` to decide the route's cursor mode; `@http.route(..., readonly=_call_kw_readonly)` on `call_kw`/`call_button`.
- `~/src/194/odoo/odoo/orm/models.py:2736-2737` — `read()` is `@api.readonly`.
- `~/src/194/odoo/odoo/orm/models.py:5152-5154` — `search_read()` is `@api.model` + `@api.readonly`.
- `~/src/194/odoo/odoo/orm/models.py:1397,1413,1433` — `search_count`, `search`, `search_fetch` all `@api.readonly` (the last also `@api.private`).
- `~/src/194/odoo/odoo/orm/models.py:1571` — `name_search` is `@api.readonly`.
- Test proving the wiring end-to-end: `~/src/194/odoo/odoo/addons/test_orm/tests/test_registry_signaling.py:327-359` (`test_call_kw_readonly`) — patches `read`/`write` to return `self.env.cr.readonly` and asserts `read` runs with `readonly=True`, `write` with `readonly=False`, over an actual `/web/dataset/call_kw` HTTP call.
- Real-world usage sampled outside core: `~/src/194/enterprise/account_reports/models/account_report.py:2249,6056,6204`, `~/src/194/enterprise/timer/models/timer.py:71`.

**Why it matters:** A Sentry crash showing `psycopg2.errors.ReadOnlySqlTransaction: cannot execute ... in a read-only transaction` inside a method that "obviously" just reads data (or inside a `read`/`search` override) is a 19.4-specific failure mode a stale-trained agent won't recognize. It might misdiagnose it as an access-rights problem and wrap the call in `sudo()` (does nothing — sudo doesn't affect cursor mode), or add a commit/rollback (masks rather than fixes it), instead of the real fix: remove the accidental write from the override, or explicitly decide the method should not be `@api.readonly` (removing the decorator only helps for custom methods, not for overrides of already-readonly base methods, where the write needs to move elsewhere, e.g. a cron/queue).

**Confidence:** high

---

## 2. `@api.private` blocks RPC on common "public-looking" ORM helper methods

**Claim:** New decorator (added Jan 2025, `40da85aab905`). It marks record-style utility methods
(that lack the usual `_` prefix for historical reasons) as **not callable over RPC/JSON-RPC/XML-RPC**,
even though they remain regular public Python methods for server-side code. `browse`/`fetch` are
not private, but `ensure_one`, `with_env`, `sudo`, `with_user`, `with_company`, `with_context`,
`with_prefetch`, `search_fetch`, and several others are now hard-blocked from RPC with an
`AccessError`.

**Evidence:**
- Decorator definition/docstring: `~/src/194/odoo/odoo/orm/decorators.py:289-303`.
- Enforcement: `~/src/194/odoo/odoo/orm/models.py:200-223` (`get_public_method`) — raises `AccessError(f"Private methods (such as '{model._name}.{name}') cannot be called remotely.")` when `getattr(cla_method, '_api_private', False)` for any class in the MRO.
- Concrete decorated methods: `sudo` at `~/src/194/odoo/odoo/orm/models.py:5352-5353`, `with_user` at `:5379-5380`, `with_context` at `:5418-5419`, `ensure_one` at `:5329-5330`, `with_env` at `:5343-5344`.
- Real addon usage: `~/src/194/enterprise/account_reports/models/account_journal_group.py:15`; core `~/src/194/odoo/odoo/orm/models_cached.py:59-60` (stacked with `@api.model`).
- Commit rationale: `git show 40da85aab905` — explicitly says these methods were "historically named without the usual underscore prefix" and this is "preferred over renaming them with an underscore prefix, as that would break a lot of existing code."

**Why it matters:** A Sentry error `AccessError: Private methods (such as 'res.partner.sudo') cannot be called remotely` (typically surfacing from an external XML-RPC/JSON-RPC integration, or a webhook that calls `model.sudo().search(...)` remotely) will look bizarre to an agent that has never seen `@api.private` — `sudo` and friends being blocked over RPC did not exist in 16/17. The wrong fix is trying to grant more ACLs or debug `ir.model.access`; the real fix is restructuring the RPC-facing method so the privileged/context-manipulating call happens server-side (in a model method), not from the RPC client.

**Confidence:** high

---

## 3. `@api.model_create_single` is fully removed; `@api.model` on `create` now always forces batch (list) semantics

**Claim:** `model_create_single` (the decorator that let a `create` override accept and return a
single dict/record) was deprecated in Aug 2024 and **removed** in Nov 2024
(`f2e6f7c31cb4`, `9820ed81b861`). It no longer exists in `odoo.api`/`odoo.orm.decorators` at all.
Moreover, `@api.model` applied directly to a method named `create` now automatically rewrites it
into `model_create_multi` behavior — i.e. **the override will always receive a list of dicts**,
never a bare dict, regardless of which decorator the code author intended.

**Evidence:**
- Confirmed absent from current source: `grep -rn "model_create_single"` across `~/src/194/odoo/odoo/` and `~/src/194/odoo/addons/` returns no hits (0 results).
- Auto-rewrite of `@api.model` on `create`: `~/src/194/odoo/odoo/orm/decorators.py:274-286` —
  ```python
  def model[C: Callable](method: C) -> C:
      if method.__name__ == 'create':
          return model_create_multi(method)  # type: ignore
      method._api_model = True
      return method
  ```
- Removal diff (`git show f2e6f7c31cb4`) shows the old `model_create_single` wrapper — which raised `DeprecationWarning("Since 18.0, \`create\` must be batched, an override in {module} is in single mode")` — deleted outright, with `model(method)` changed from calling `model_create_single` to calling `model_create_multi`.
- Canonical current pattern confirmed pervasively in real code: `~/src/194/odoo/odoo/addons/base/models/res_partner.py:931-932` (`@api.model_create_multi` / `def create(self, vals_list):`); 368 files under `addons/`+`odoo/addons/` use `@api.model_create_multi` on `create` vs. 371 total `def create(self, vals_list)` overrides (i.e. virtually all `create` overrides in this codebase are already batch-shaped).
- Both removal commits (`f2e6f7c31cb4`, `9820ed81b861`) verified as ancestors of HEAD via `git merge-base --is-ancestor`.

**Why it matters:** A stale-trained agent patching a Sentry crash in a `create()` override may
write `@api.model` (thinking it preserves old single-dict semantics, as in 16/17-era code it
had seen) with body code like `vals['partner_id']` assuming `vals` is a dict. On 19.4 this
receives a **list of dicts** and crashes with `TypeError: list indices must be integers or
slices, not str` — a wrong "fix" that introduces a fresh bug instead of resolving the original
Sentry issue. The correct pattern is `@api.model_create_multi` with `vals_list` iterated.

**Confidence:** high

---

## 4. `@api.returns` decorator removed entirely

**Claim:** The old `@api.returns(model, downgrade=..., upgrade=...)` decorator (used to declare
that a method returns records of a given model, with type-conversion hooks for RPC) has been
deleted from the codebase, along with its supporting `Meta`/`propagate` machinery.
`call_kw`'s conversion logic (turning a returned recordset into ids for RPC) is now handled
generically in `odoo.service.model.call_kw` instead of per-method via the decorator.

**Evidence:**
- Removal commit `76967686cc07` ("`[IMP] core: remove @api.returns and move call_kw to odoo.service.model`"), verified ancestor of HEAD. Diff deletes `class Meta`, `def propagate`, `def returns` and the `# method._returns: set by @returns` comment from `odoo/orm/decorators.py`.
- Confirmed absent from `odoo/orm/decorators.py` (read in full: `~/src/194/odoo/odoo/orm/decorators.py`, no `returns` decorator defined) and not exported from `~/src/194/odoo/odoo/api/__init__.py`.
- Only remaining trace in the whole codebase is a stale code comment: `~/src/194/odoo/odoo/addons/mail/models/mail_template.py:734` (`# TDE CLEANME: return mail + api.returns ?`) — i.e. even Odoo's own comment referencing it is dead.
- Generic RPC result conversion now lives in `~/src/194/odoo/odoo/service/model.py:29-64` (`call_kw`), which unconditionally does `if isinstance(result, BaseModel): result = result.ids`.

**Why it matters:** If a stale-trained agent tries to "fix" an RPC serialization bug (e.g. a
method returning a recordset object that the JS client can't handle) by adding
`@api.returns('self')` as it may have seen in older Odoo source, this will crash immediately with
`AttributeError: module 'odoo.api' has no attribute 'returns'` — a broken patch that doesn't even
import. The correct approach in 19.4 doesn't need any such decorator at all; plain recordset
returns are auto-converted to ids by `call_kw`.

**Confidence:** high

---

## 5. `@api.depends_context('uid')` combined with `compute_sudo=True` changed cache-key semantics

**Claim:** The special `'uid'` context key supported by `@api.depends_context` used to always
resolve to the pair `(self.uid, self.su)` (user id + sudo flag) for cache/dependency-key
purposes. Since `60ab6a556ed6` (Jul 2024, confirmed ancestor of HEAD), when the field being
computed has `compute_sudo=True`, the key collapses to just `self.uid` (dropping the `su` flag)
— specifically to let a `compute_sudo=True` method that reads `self.env.user`/`self.uid` under
`depends_context('uid')` correctly **assign a value back onto a non-sudo recordset** without a
cache-consistency conflict.

**Evidence:**
- Fix location: `odoo/api.py` diff in `git show 60ab6a556ed6`:
  ```python
  elif key == 'uid':
-     return (self.uid, self.su)
+     return self.uid if field.compute_sudo else (self.uid, self.su)
  ```
  (this logic now lives in `Environment._get_cache_key`/context resolution in the current `odoo/orm/environments.py`; verified the commit is an ancestor of HEAD).
- Regression test added by the same commit: `~/src/194/odoo/odoo/addons/test_new_api/tests/test_new_fields.py` — new `TestComputeSudo.test_compute_sudo_depends_context_uid` (asserts `record.with_user(self.user_demo).name_for_uid == self.user_demo.name`), model in `odoo/addons/test_new_api/models/test_new_api.py` (`ComputeSudo` model, `_compute_name_for_uid` decorated with `@api.depends_context('uid')`).
- Real-world usage of exactly this pattern: `~/src/194/odoo/odoo/addons/hr/models/res_users.py:30-31` — `@api.depends(f'employee_id.{name}')` stacked with `@api.depends_context('uid')` on a helper building per-user related fields with `compute_sudo` implied by the surrounding factory (`_add_related_field`-style helper); similar stacking also in `~/src/194/odoo/odoo/addons/im_livechat/models/res_users.py:87,103,119`.
- Documented special keys (`company`, `uid`, `active_test`) at `~/src/194/odoo/odoo/orm/decorators.py:249-256`.

**Why it matters:** This is a subtle bug-fix already baked into 19.4's core — but if a Sentry
crash/incorrect-value report involves a `compute_sudo=True` field that depends on `uid` context
and an agent "fixes" it by reverting to a mental model where `uid`+`su` are always paired (e.g.
adding a redundant `self.env.uid` check, or wrapping in extra `sudo()`/`with_user()` calls to
"work around" a caching bug that no longer exists in 19.4), it risks reintroducing the very stale
value / wrong-user assignment bug this commit fixed, or masking a different bug behind
unnecessary sudo escalation.

**Confidence:** medium

---

## 6. `odoo.api` is now a thin re-export shim; decorators physically live in `odoo.orm.decorators`, and internal marker attributes changed name

**Claim:** `odoo/api.py` (a single file in 16/17) no longer exists as a module with the actual
decorator implementations. It's now the package `odoo/orm/decorators.py`, re-exported through
`odoo/api/__init__.py` for backward-compatible `import odoo.api as api; @api.depends(...)` usage.
Additionally, the internal attribute the ORM used to stamp decorated methods with changed: it used
to be a single `method._api = 'model'` / `method._api = 'model_create'` string; it is now split
into separate boolean attributes `method._api_model` and `method._api_private`. Any code
(monkeypatches, introspection helpers, test utilities) that reads `getattr(method, '_api', None)`
to detect `@api.model` methods will silently see `None` and stop working.

**Evidence:**
- Package split commits: `de93ba06a10b` ("`[REF] core: separated odoo.orm.environments and decorators`"), `0a5b1f96b812` ("`[REF] core: odoo.orm package`") — both confirmed ancestors of HEAD.
- Current shim: `~/src/194/odoo/odoo/api/__init__.py:1-21` (`from odoo.orm.decorators import (autovacuum, constrains, depends, depends_context, model, model_create_multi, onchange, ondelete, private, readonly)`).
- Attribute rename diff, `git show 40da85aab905` on `odoo/orm/decorators.py`:
  ```python
  -    method._api = 'model'  # type: ignore
  +    method._api_model = True  # type: ignore
   ...
  -    create._api = 'model_create'  # type: ignore
  +    create._api_model = True  # type: ignore
  ```
- Current usages of the new attribute names: `~/src/194/odoo/odoo/service/model.py:38` (`getattr(method, '_api_model', False)`), `~/src/194/odoo/odoo/orm/models.py:223` (`getattr(cla_method, '_api_private', False)`).

**Why it matters:** Lower direct Sentry-crash relevance than the others, but relevant if an
agent's patch needs to introspect or replicate decorator behavior (e.g. writing a custom
decorator that should behave "like `@api.model`", or fixing a bug in test/monkeypatch code that
inspects `_api`). Code written against the old `method._api == 'model'` contract will not detect
19.4 `@api.model` methods at all, causing silent no-ops rather than a loud crash — a harder class
of bug to catch via Sentry (it manifests as "feature X doesn't behave as expected" reports rather
than exceptions).

**Confidence:** medium

---

## 7. `@api.constrains` still validates synchronously at `create`/`write` time in 19.4 — a "deferred/lazy" rewrite exists upstream but is NOT in this branch (verified negative)

**Claim:** While researching, I found real commits titled `[REF] core: deffered constraint.`
(`75782af9e5ce`) and `[FIX] core: check constrains for all fields in create` (`f41f4e4d1905`)
that rework `@api.constrains` into a deferred/lazy mechanism (checked at `flush()` time via a
dependency-trigger graph similar to `@api.depends`, supporting dotted-path triggers). This looked
like exactly the kind of "new decorator semantics" this task is hunting for. However,
`git merge-base --is-ancestor 75782af9e5ce HEAD` (and same for `f41f4e4d1905`) returns **false**
— these commits are not part of the SaaS-19.4 history checked out at
`~/src/194/odoo`. Confirmed by grepping the actual current `odoo/orm/models.py`: it
still contains the old, synchronous `_constraint_methods` / `_validate_fields` implementation,
with no trace of the `Constrain` class, `_check_constrain`, or `check_deferred_constrains`
introduced by that other-branch refactor.

**Evidence:**
- `git merge-base --is-ancestor 75782af9e5ce HEAD` in `~/src/194/odoo` → exits non-zero (not an ancestor); same for `f41f4e4d1905`.
- Current (real, in-branch) implementation: `~/src/194/odoo/odoo/orm/models.py:558-586` (`_constraint_methods` property) and `:1291-1305` (`_validate_fields`, invoked synchronously from `write`/`create`/`_compute_field_value`).
- Current docstring, unchanged and still accurate for 19.4: `~/src/194/odoo/odoo/orm/decorators.py:69-80` — dotted names are "not supported and will be ignored"; "will be triggered only if the declared fields in the decorated method are included in the create or write call."

**Why it matters:** Listing this as a negative-result entry on purpose — so whoever curates the
final skill doesn't independently find the same upstream commits (they're easy to stumble on via
`git log --all`) and wrongly assume the deferred-constraints rewrite applies to SaaS-19.4. It
doesn't, in this checkout. `@api.constrains` behaves the same way here as in 17: synchronous,
simple-field-names-only, triggered only when the field is part of the `create`/`write` vals dict.

**Confidence:** high (as a verified negative / non-candidate)

---

## 8. `@api.private` and `@api.readonly` can be stacked with `@api.model`, and ordering/interaction matters for real base methods

**Claim:** Several base ORM methods stack two or three of these decorators together, e.g.
`search_fetch` is `@api.model` + `@api.private` + `@api.readonly` simultaneously. This is new
territory (neither `@api.private` nor the RPC-cursor-routing behavior of `@api.readonly` existed
pre-19.x), so an agent patching/overriding one of these multi-decorated base methods needs to
understand that removing or reordering the stack changes both RPC-callability *and* cursor mode
at once, not just one axis.

**Evidence:**
- `~/src/194/odoo/odoo/orm/models.py:1431-1433`:
  ```python
  @api.model
  @api.private
  @api.readonly
  def search_fetch(
  ```
- `~/src/194/odoo/odoo/orm/models.py:5152-5154` — `search_read` stacks `@api.model` + `@api.readonly` (but not `@api.private`, since it *is* meant to be RPC-callable, unlike `search_fetch`).
- Confirms these are independent, composable flags read off the method object (`_api_model`, `_api_private`, `_readonly`), each checked at a different point in the stack (`get_public_method` for `_api_private`, `_call_kw_readonly` for `_readonly`, and `call_kw`'s `getattr(method, '_api_model', False)` for id/browse handling) — see decorator definitions at `~/src/194/odoo/odoo/orm/decorators.py:274-315`.

**Why it matters:** If a Sentry bug requires overriding e.g. `search_fetch` or `search_read` in a
custom model and the override needs to do a write internally (common anti-pattern: "search and
touch a `last_accessed` field"), a patch that only strips one of the stacked decorators (e.g.
removes `@api.private` thinking that's what blocks the write) will still fail on the
`@api.readonly` cursor restriction, because the two concerns are enforced by entirely separate
code paths (`get_public_method` vs. `_call_kw_readonly`) reading different attributes. A
correct patch needs to reason about both independently.

**Confidence:** medium

