# HTTP Controllers — SaaS-19.4 candidate nuances

Scope note: I read the full `odoo/odoo/http/` package (routing_map.py, router.py,
dispatcher.py, requestlib.py, session.py, _facade.py), `odoo/odoo/addons/base/models/ir_http.py`,
and the entire `odoo/odoo/addons/test_http/tests/` suite (test_bearer_scope.py, test_captcha.py,
test_webjson2.py, test_device.py, test_common.py, etc.), then cross-checked real usage with grep
across `odoo/addons/`, `enterprise/`, `industry/`, `design-themes/`. `git log` on the http module
was scanned but didn't add anything beyond what the code/tests already show. I did **not**
open industry/design-themes controller code in depth (grepped only) — coverage there is shallow.
8 candidates below, capped as instructed.

---

## 1. `readonly=` routing — requests can run on a read-only DB cursor, with automatic RW retry

**Claim:** `@route(..., readonly=...)` is a real, load-bearing option in 19.4, not a docstring
curiosity. Odoo now opens a **read-only replica cursor** by default logic depends on `auth`:
if a route doesn't set `readonly` explicitly, it defaults to `readonly=True` only when
`auth='none'`; otherwise it defaults to `False` (read/write). If the endpoint is marked
`readonly=True`/callable-True and it actually attempts a write, Odoo **catches
`ReadOnlySqlTransaction` and transparently retries the same request on a fresh RW cursor** — it
does not just crash.

**Evidence:**
- `odoo/odoo/http/routing_map.py:172-174` — docstring: `readonly: Whether this endpoint should
  open a cursor on a read-only replica instead of (by default) the primary read/write database.`
- `odoo/odoo/http/routing_map.py:351-362` — default computation:
  `default_mode = submethod.original_routing.get('readonly', default_auth == 'none')` and the
  override-mismatch guard that forces `readonly=False` with a logged warning when a child route
  disagrees with its parent's readonly mode.
- `odoo/odoo/http/router.py:396-438` (`serve_db`) — the actual cursor lifecycle:
  ```python
  cr = registry.cursor(readonly=True)
  ...
  readonly = endpoint.routing['readonly']
  if callable(readonly):
      readonly = readonly(endpoint.func.__self__, rule, args)
  if readonly and cr.readonly:
      threading.current_thread().cursor_mode = 'ro'
      try:
          return retrying(serve_func, env=request.env)
      except ReadOnlySqlTransaction as exc:
          _logger.warning("%s, retrying with a read/write cursor", ...)
          threading.current_thread().cursor_mode = 'ro->rw'
      ...
  # falls through to open a RW cursor and retry
  ```
- Real usage confirming `readonly=` is mainstream, not exotic:
  `odoo/addons/portal/controllers/portal.py:245`, `odoo/addons/website/controllers/webclient.py:9`,
  `odoo/addons/rpc/controllers/__init__.py:24`, and a **callable** form at
  `odoo/addons/im_livechat/controllers/cors/webclient.py:11`
  (`readonly=lambda self, *_: self._is_mail_fetch_readonly()`).

**Why it matters:** A stale LLM triaging a Sentry `psycopg2.errors.ReadOnlySqlTransaction` /
"cannot execute UPDATE in a read-only transaction" crash would likely assume a DB permissions
misconfiguration or propose disabling something at the infra level. The real, idiomatic 19.4 fix
is almost always one of: (a) the route is legitimately read/write and just needs
`readonly=False` explicitly (or its parent controller's readonly default needs to stop leaking
down), or (b) the write is unintentional/a symptom of a real bug reachable even from a supposedly
read-only page. Also, since Odoo auto-retries once, a Sentry error here is often the SECOND
failure (after the ro->rw retry also failed) — a stale-knowledge agent won't know the request was
already retried once at the transaction layer, and may write a duplicate manual-retry patch.

**Confidence:** high

---

## 2. `auth='bearer'` requires `bearer_scope=` and falls back to session auth only behind `Sec-Fetch-*` header checks

**Claim:** `auth='bearer'` is not "just check an Authorization header, else 401." It (a) is
**mandatory to pair with `bearer_scope=`** (asserted at route-registration time), (b) scopes API
keys so a key generated for scope `'A'` is rejected on a route requiring scope `'B'`, and (c) if
no bearer token is present, it silently falls back to normal session-cookie auth **but only
accepts that fallback if the request carries browser `Sec-Fetch-Dest/Mode/Site/User` headers**
proving it's an interactive top-level navigation — otherwise it 401s even with a valid session
cookie. This is a CSRF defense specific to bearer routes and didn't exist pre-19.

**Evidence:**
- `odoo/odoo/http/routing_map.py:204-206`:
  ```python
  if routing.get('auth') == 'bearer':
      routing.setdefault('save_session', False)  # stateless
      assert 'bearer_scope' in routing, "bearer_scope must be set for auth='bearer'"
  ```
- `odoo/odoo/addons/base/models/ir_http.py:311-352` (`_auth_method_bearer`):
  ```python
  if token := get_http_authorization_bearer_token():
      uid = request.env['res.users.apikeys']._check_credentials(scope=routing['bearer_scope'], key=token)
      ...
  elif not request.env.uid:
      raise Unauthorized(...)
  elif not check_sec_headers():
      e = 'Missing "Authorization" or Sec-headers for interactive usage.'
      raise werkzeug.exceptions.Unauthorized(e, www_authenticate=WWWAuthenticate('bearer'))
  cls._authenticate_explicit(dict(routing, auth='user'))
  ```
- Test proving scope enforcement: `odoo/odoo/addons/test_http/tests/test_bearer_scope.py:27-31`
  (`test_rpc_key_rejected_on_non_rpc_endpoint`, `test_non_rpc_key_rejected_on_rpc_endpoint`).
- Real usage: `odoo/addons/web/controllers/json.py:39` (`bearer_scope='rpc'`),
  `enterprise/ai_mcp/controllers/mcp_controller.py:17` (`bearer_scope='mcp'`),
  `odoo/addons/api_doc/controllers/api_doc.py:49,167`.

**Why it matters:** A Sentry 401 on a bearer-auth endpoint could get mis-triaged as "the API key
is wrong" or "auth='bearer' is broken," when the actual cause is a legitimate browser session hit
without the fetch metadata headers (e.g. an XHR/fetch call missing `Sec-Fetch-*`, or a proxy
stripping them) — or a key generated with the wrong `bearer_scope`. A stale LLM might "fix" this
by weakening auth (e.g. removing the scope check or switching to `auth='public'`), reintroducing
a security hole, instead of fixing the client to either send a real bearer token or the missing
navigation headers.

**Confidence:** high

---

## 3. `type='json'` is a deprecated alias; `'jsonrpc'` and a new enveloping-free `'json2'` type both exist

**Claim:** `@route(type='json')` now emits a `DeprecationWarning` and is silently rewritten to
`type='jsonrpc'` (the JSON-RPC 2.0 envelope: `{"jsonrpc":"2.0","result"/"error":...,"id":...}`).
Separately, there is a **third, newer dispatcher type `'json2'`** that has no JSON-RPC envelope at
all — it reads/writes bare JSON, is used for the generic REST-ish model API under
`/json/1/<path>` and `/json/2/<model>/<method>`, and is the type paired with `bearer`+scoped
routes. Also, the whole per-type `HttpRequest`/`JsonRequest` subclass split from older Odoo is
gone: there is now a single `Request` class, and per-type behavior lives in swappable
`Dispatcher` subclasses (`HttpDispatcher`, `JsonRPCDispatcher`, `Json2Dispatcher`) selected via
`request.dispatcher`.

**Evidence:**
- Deprecation: `odoo/odoo/http/routing_map.py:189-195`:
  ```python
  if routing.get('type') == 'json':
      warnings.warn("Since 19.0, @route(type='json') is a deprecated alias to @route(type='jsonrpc')",
                     DeprecationWarning, stacklevel=3)
      routing['type'] = 'jsonrpc'
  ```
- `grep -rl "type='json'"` across `odoo/addons`, `enterprise`, `industry`, `design-themes` →
  **0 hits**: the whole 19.4 codebase has already migrated off the old spelling.
- Three dispatcher classes: `odoo/odoo/http/dispatcher.py:266` (`class JsonRPCDispatcher`,
  `routing_type = 'jsonrpc'`) vs `odoo/odoo/http/dispatcher.py:363` (`class Json2Dispatcher`,
  `routing_type = 'json2'`) — the latter's `dispatch()` (lines ~380-395) returns
  `self.request.make_json_response(result)` directly, no `{jsonrpc, id, result}` wrapper.
- Real `json2` routes: `odoo/addons/rpc/controllers/json2.py:40,53`,
  `odoo/addons/api_doc/controllers/api_doc.py:49,167`; test coverage confirming bare-JSON shape
  and header requirements in `odoo/odoo/addons/test_http/tests/test_webjson2.py:39-77`
  (asserts response body is e.g. `"[]"` with no envelope, and requires `X-odoo-database` header
  in multi-db setups).
- No more subclassed request types: `grep -rn "class HttpRequest\|class JsonRequest"` in
  `odoo/odoo/http/` → no matches; single `class Request` in
  `odoo/odoo/http/requestlib.py:116`, with `HTTPRequest` (the werkzeug wrapper, unrelated name) in
  `odoo/odoo/http/_facade.py:14`.

**Why it matters:** A stale LLM fixing a Sentry error on a JSON endpoint may assume every JSON
route returns/expects a JSON-RPC envelope (`params`/`result`/`error` with `id`), and either
misread a `json2` route's already-correct bare-JSON behavior as "the envelope is missing —
bug," or patch a `json2` controller to wrap its return value in a fake JSON-RPC envelope,
breaking every real caller. It may also try to import/subclass `JsonRequest`/`HttpRequest`
(muscle memory from 16/17), which no longer exist.

**Confidence:** high

---

## 4. `captcha=` only guards unsafe HTTP methods, and failure is a plain `UserError` → 422

**Claim:** `@route(..., captcha='action_name')` is checked in `ir_http._dispatch`, but **only when
the HTTP method is unsafe** (not GET/HEAD/OPTIONS/TRACE) — a GET to a captcha-protected route is
never verified. The verification hook `_verify_request_recaptcha_token` is a no-op stub in base
(returns `None` unconditionally) — actual verification is added by an iap/recaptcha-integration
module override — and on failure it's expected to raise a plain `UserError`, which the http layer
turns into HTTP 422 (Unprocessable Entity), not 403.

**Evidence:**
- `odoo/odoo/addons/base/models/ir_http.py:463-469`:
  ```python
  @classmethod
  def _dispatch(cls, endpoint):
      captcha = endpoint.routing.get('captcha')
      if captcha and request.httprequest.method not in SAFE_HTTP_METHODS:
          request.env['ir.http']._verify_request_recaptcha_token(captcha)
  ```
- Base stub: `odoo/odoo/addons/base/models/ir_http.py:573-575`
  (`def _verify_request_recaptcha_token(self, action): return`).
- Test confirms GET bypasses verification entirely and POST failure yields 422:
  `odoo/odoo/addons/test_http/tests/test_captcha.py:33-40`
  (`test_get_invalid` calls `res.raise_for_status()` — i.e. GET succeeds even when
  `patch_captcha_valid(False)` is active) vs `test_post_invalid`:
  `self.assertEqual(res.status_code, HTTPStatus.UNPROCESSABLE_ENTITY)`.
- Real usage: `odoo/addons/auth_signup/controllers/main.py:40,96` (`captcha='signup'`,
  `captcha='password_reset'`), `odoo/addons/website/controllers/form.py:30`
  (`captcha='website_form'`).

**Why it matters:** Given a Sentry report of a captcha-bypass or of unexpected 422s on a
signup/contact-form endpoint, a stale LLM might assume captcha is validated on every request (and
"fix" a bypass report by trying to add checks on GET, which is a functional regression for normal
page loads), or might treat the 422 as a generic validation bug rather than recognizing it as the
captcha subsystem's expected failure mode and looking at the actual recaptcha-provider override
instead of the stub in base.

**Confidence:** high

---

## 5. Identity re-confirmation flow (`CheckIdentityException`) for untrusted devices on `auth='user'` routes

**Claim:** Successful session auth on `auth='user'` routes is not the end of the authorization
check anymore. Odoo now tracks a notion of a "trusted device" per session and can force an
**identity re-confirmation** (redirect to `/web/session/identity`, MFA-ish) even for an
already-logged-in user, via `CheckIdentityException` (a subclass of
`SessionExpiredException`). This is gated by a system parameter and only applies to internal
users on unrecognized devices.

**Evidence:**
- `odoo/odoo/http/session.py:88` — `class CheckIdentityException(SessionExpiredException):`.
- `odoo/odoo/addons/base/models/ir_http.py:400-408` (`_authenticate_explicit`):
  ```python
  if (routing['auth'] == 'user' and request.session.uid is not None
          and (must_check_identity := cls._must_check_identity())):
      if must_check_identity.get('logout'):
          raise SessionExpiredException(...)
      if must_check_identity.get('check_identity') and routing.get('check_identity', True):
          raise CheckIdentityException(...)
  ```
- Trust-gating logic: `odoo/odoo/addons/base/models/ir_http.py:577-594` (`_must_check_identity`) —
  checks `base.session_check_device` config param and `_is_internal()`; untrusted device gets
  `{'check_identity': True, 'mfa': True, 'fingerprint_check': True}`.
- Dedicated redirect handling: `odoo/odoo/http/dispatcher.py:232-238` (`HttpDispatcher.handle_error`)
  special-cases `CheckIdentityException` to redirect to `/web/session/identity` instead of the
  normal `/web/login` session-expired redirect.
- New DB models backing device trust: `odoo/odoo/addons/base/models/res_device.py:250`
  (`_name = 'res.session'`), tests in
  `odoo/odoo/addons/test_http/tests/test_device.py` (device-trust/session tracking, e.g.
  `test_detection_device_readonly` at line ~92).

**Why it matters:** A Sentry "SessionExpiredException" or repeated-login-redirect report on an
otherwise-valid session would, under 16/17-era knowledge, point only at cookie/session-store
expiry (timeout, cleared filestore, load-balancer sticky-session issues). In 19.4 it can equally
be the identity-recheck flow kicking in for a device Odoo doesn't yet trust, which is a different
subsystem (res.device/res.session + `base.session_check_device`) and a different fix path (e.g.
device-trust bug or overly aggressive fingerprinting) than classic session-timeout tuning. Fixing
it as a plain "increase session timeout" patch would miss the real cause.

**Confidence:** medium (mechanism and citations are solid; how *often* this fires in practice
depends on config default for `base.session_check_device`, which I did not chase down further)

---

## 6. `cors=` auto-handles the OPTIONS preflight and forces `auth='none'` for it

**Claim:** Setting `cors=` on a route doesn't just add an `Access-Control-Allow-Origin` response
header — the framework also **auto-answers the CORS preflight `OPTIONS` request itself**
(aborting with an empty `204` plus `Access-Control-Max-Age`/`Access-Control-Allow-Headers`) before
the controller method ever runs, and it forces the *auth* for that preflight call to `'none'`
regardless of the route's declared `auth=`, since a preflight can't carry credentials/cookies
correctly.

**Evidence:**
- `odoo/odoo/http/dispatcher.py:132-145` (`Dispatcher.pre_dispatch`):
  ```python
  cors = routing.get('cors')
  if cors:
      set_header('Access-Control-Allow-Origin', cors)
      set_header('Access-Control-Allow-Methods', ('POST' if routing['type'] == JsonRPCDispatcher.routing_type
                  else ', '.join(routing['methods'] or ['GET', 'POST'])))
  if cors and self.request.httprequest.method == 'OPTIONS':
      set_header('Access-Control-Max-Age', str(CORS_MAX_AGE))
      set_header('Access-Control-Allow-Headers', 'Origin, X-Requested-With, Content-Type, Accept, Authorization, Range')
      abort(Response(status=204))
  ```
- Forced `auth='none'` on preflight: `odoo/odoo/addons/base/models/ir_http.py:372-373`
  (`_authenticate`): `routing = dict(endpoint.routing, auth='none') if is_cors_preflight(request, endpoint) else endpoint.routing`.
- `is_cors_preflight` helper: `odoo/odoo/http/requestlib.py:109-114`.
- Real usage of `cors=` on a stateful-looking route:
  `odoo/addons/im_livechat/controllers/cors/webclient.py:11`
  (`cors="*"` combined with `auth="public"` and a callable `readonly=`).

**Why it matters:** A stale LLM debugging a Sentry 401/403 that only appears on `OPTIONS`
requests to a CORS-enabled route might try to "fix" it by loosening the route's `auth=`
(e.g. changing `auth='user'` to `auth='public'`) thinking normal auth is somehow blocking
preflight — when the preflight is already forced to `auth='none'` and unrelated to the real
(non-OPTIONS) request's auth failure. Conversely, if `cors=` is set but the *actual* GET/POST
request still fails CORS in the browser, the fix is almost never in the OPTIONS-handling code
path at all (that part is fully automatic), so patches should focus elsewhere (missing/wrong
`cors=` value, or `Access-Control-Allow-Methods` mismatch from the route's `methods=`).

**Confidence:** high

---

## 7. `save_session=` controls session-cookie persistence and defaults to `False` for `auth='bearer'`

**Claim:** `save_session=False` on a route means: no `session_id` cookie is set and the session is
not persisted to disk for that request, even if the session was mutated. It is **automatically
defaulted to `False`** whenever `auth='bearer'` is used (bearer routes are meant to be stateless),
so a bearer-auth route silently does not participate in normal session persistence unless the
route author explicitly overrides it back to `True`.

**Evidence:**
- Doc: `odoo/odoo/http/routing_map.py:181-183` — "Whether it should set a session_id cookie on
  the http response and save dirty session on disk. `False` by default for `auth='bearer'`. `True`
  by default otherwise."
- Default-setting code: `odoo/odoo/http/routing_map.py:204-205`
  (`routing.setdefault('save_session', False)  # stateless` inside the `auth == 'bearer'` branch).
- Enforcement point: `odoo/odoo/http/dispatcher.py:129-130` (`Dispatcher.pre_dispatch`):
  `self.request.session.can_save &= routing.get('save_session', True)`.
- Additional explicit stateless marking in the bearer-token-present path:
  `odoo/odoo/addons/base/models/ir_http.py:344-345`
  (`request.update_env(user=uid); request.session.can_save = False  # stateless`).

**Why it matters:** A Sentry report of "user session/cookie not updated after calling this API
endpoint" (e.g. a device-trust flag, a `lang`/`tz` context change, or a cart/website-session value
that should have persisted) could be mis-triaged as a session-store bug, when the actual cause is
that the route is `auth='bearer'` and therefore never saves the session at all unless
`save_session=True` is added explicitly. A stale-knowledge fix that pokes at the session store or
cookie code directly would miss the one-line routing fix.

**Confidence:** high

---

## 8. Controller inheritance: overriding methods must re-decorate with `@route()`, and mismatched `type`/`readonly` in overrides get silently corrected (with a warning), not honored

**Claim:** In the current controller-merge algorithm, if a subclass overrides a routed method
without adding a (possibly-empty) `@route()` decorator, Odoo auto-wraps it and logs a warning
rather than erroring — but more importantly, if an override *does* re-decorate and tries to change
`type=` or `readonly=` relative to what an ancestor controller declared, the framework does **not**
honor the override: it logs a warning and forces the route back to the ancestor's `type`, and
forces `readonly=False` on any readonly/read-write disagreement between parent and child.

**Evidence:**
- Missing re-decoration handling: `odoo/odoo/http/routing_map.py:305-307`:
  ```python
  if not hasattr(submethod, 'original_routing'):
      _logger.warning("The endpoint %s is not decorated by @route(), decorating it myself.", ...)
      submethod = route()(submethod)
  ```
- Type-mismatch correction: `odoo/odoo/http/routing_map.py:342-349` — a child's differing `type=`
  triggers `_logger.warning("The endpoint %s changes the route type, using the original type: %r.")`
  and the child's `original_routing['type']` is force-set back to the parent's.
- Readonly-mismatch correction: `odoo/odoo/http/routing_map.py:354-362` — logs
  `"The endpoint %s made the route %s altough its parent was defined as %s. Setting the route
  read/write."` and forces `submethod.original_routing['readonly'] = False`.
- Documented expectation in the `Controller` class docstring itself:
  `odoo/odoo/http/routing_map.py:88-105` — "It is mandatory to re-decorate any method that is
  overridden in controller extensions but the arguments can be omitted... any provided argument
  will override previously defined ones" (with the worked `@route(auth='user')` example).

**Why it matters:** When patching a Sentry bug by overriding a controller method in module B to
extend one from module A, a stale LLM might assume any `@route(...)` arguments on the override
"win" outright (true for most options like `auth=`), and might specifically try to flip a parent's
`readonly=True` route to `readonly=False` (or change `type=`) purely via the override decorator to
fix a write-attempt bug — not realizing the merge algorithm actively overrides that specific
override back to the ancestor's `readonly` semantics (only forcing to non-readonly on conflict,
never the reverse) and logs a warning instead of applying the intended value. The actual fix
needs to touch the ancestor route or use the callable-`readonly=` form instead.

**Confidence:** medium (the mechanism and citations are exact; I did not find a real addon example
of this specific conflict firing in practice — the evidence is from the framework code and
docstring/tests-adjacent logic in `routing_map.py`, not a live triage case)
