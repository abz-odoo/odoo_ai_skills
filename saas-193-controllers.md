# Odoo SaaS-19.3 HTTP Controllers Guide

Reference for writing HTTP controllers in the SaaS-19.3 codebase.
Source of truth: `/home/achraf/src/193/odoo/odoo/http.py`

> **SaaS-19.3 differences vs generic Odoo 18/19**
> - `auth='bearer'` is a new auth type for machine-to-machine API endpoints
> - `readonly` param added to `@http.route` (replaces old `website_published` pattern)
> - `save_session` param added to `@http.route` — prevents session writes for read-only routes
> - `cors` param added directly on `@http.route`
> - `captcha` param added to `@http.route` for CAPTCHA validation
> - `Controller.env` property — shortcut for `request.env`
> - `request.geoip` — GeoIP lookup on the request (available in SaaS-19.3)
> - `request.future_response` — for deferred / streaming responses
> - `handle_params_access_error(error)` override in Controller class

---

## Table of Contents

1. [Basic structure](#basic-structure)
2. [Route decorator parameters](#route-decorator-parameters)
3. [Auth types](#auth-types)
4. [Request object](#request-object)
5. [Returning responses](#returning-responses)
6. [Error handling](#error-handling)
7. [Controller inheritance](#controller-inheritance)
8. [Common patterns](#common-patterns)

---

## Basic structure

```python
from odoo import http
from odoo.http import request


class MyController(http.Controller):

    @http.route('/my/endpoint', methods=['GET'], auth='user', type='http')
    def my_endpoint(self, **kwargs):
        records = request.env['my.model'].search([])
        return request.render('my_module.my_template', {'records': records})
```

Controllers **must** inherit from `http.Controller`.  
Each method decorated with `@http.route` becomes an HTTP endpoint.

---

## Route decorator parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `route` | str \| list[str] | URL pattern(s), e.g. `'/api/v1/items'` or `['/a', '/b']` |
| `methods` | list[str] | HTTP methods: `['GET']`, `['POST']`, `['GET', 'POST']` |
| `auth` | str | Auth type (see below) |
| `type` | str | `'http'` (default) or `'json'` |
| `readonly` | bool \| Callable | **SaaS-19.3** Don't acquire write lock on DB cursor |
| `save_session` | bool | **SaaS-19.3** If `False`, don't write the session after the request |
| `cors` | str | **SaaS-19.3** CORS origin header value, e.g. `'*'` or `'https://example.com'` |
| `captcha` | bool | **SaaS-19.3** Validate reCAPTCHA token before handling request |
| `website` | bool | Enable website-specific behaviour (set `request.website`) |
| `sitemap` | bool | Include in sitemap generation |
| `csrf` | bool | Enforce CSRF check (default `True` for POST) |

### `readonly` (SaaS-19.3)

When `True`, the cursor is opened in `readonly` mode — no writes allowed, but queries are faster (no row lock contention). Pass a callable to make it conditional:

```python
@http.route('/api/items', auth='user', readonly=True)
def list_items(self):
    ...

@http.route('/api/items', auth='user', readonly=lambda: request.httprequest.method == 'GET')
def items(self):
    ...
```

### `save_session` (SaaS-19.3)

Setting `save_session=False` prevents the session from being written to the store after the request. Use on pure read endpoints to avoid session contention.

```python
@http.route('/api/public-data', auth='public', save_session=False, readonly=True)
def public_data(self):
    ...
```

### `cors` (SaaS-19.3)

```python
@http.route('/api/v1/data', auth='bearer', cors='*', type='json')
def api_data(self):
    ...
```

### `captcha` (SaaS-19.3)

```python
@http.route('/web/contact', methods=['POST'], auth='public', captcha=True)
def contact_form(self, **kwargs):
    ...
```

---

## Auth types

| Value | Who can call it | Description |
|-------|-----------------|-------------|
| `'user'` | Authenticated Odoo users | Standard protected endpoint; 401 if not logged in |
| `'public'` | Anyone (logged in or not) | `request.env.user` is the public user record |
| `'none'` | Anyone, no env setup | No `request.env`; use for very low-level endpoints |
| `'bearer'` | **SaaS-19.3** Machine-to-machine API calls | Validates `Authorization: Bearer <token>` header; used for REST APIs |

### `auth='bearer'` example

```python
@http.route('/api/v1/resource', auth='bearer', type='json', methods=['GET', 'POST'])
def resource_endpoint(self):
    # request.env is set up from the bearer token's associated user
    data = request.env['res.partner'].search_read([], ['name', 'email'])
    return data
```

---

## Request object

`request` is a thread-local proxy to the current `HttpRequest` / `JsonRequest`.

### Common attributes

| Attribute | Description |
|-----------|-------------|
| `request.env` | Odoo Environment for the current user |
| `request.httprequest` | The underlying Werkzeug `Request` object |
| `request.session` | Session dict (persisted across requests) |
| `request.params` | Merged GET + POST parameters |
| `request.uid` | Current user's ID (`int`) |
| `request.geoip` | **SaaS-19.3** GeoIP result object for the request IP |
| `request.future_response` | **SaaS-19.3** Set to a `Future` for streaming responses |
| `request.website` | Current `website` record (when `website=True` on route) |

### `request.env` vs `self.env` (SaaS-19.3)

Both work inside a controller method. `self.env` is a `Controller` property that delegates to `request.env`:

```python
class MyController(http.Controller):

    @http.route('/my/route', auth='user')
    def my_route(self):
        # These are equivalent:
        partners = request.env['res.partner'].search([])
        partners = self.env['res.partner'].search([])
```

### `request.geoip` (SaaS-19.3)

```python
@http.route('/shop', auth='public', website=True)
def shop(self):
    country_code = request.geoip.country_code  # e.g. 'FR'
    ...
```

---

## Returning responses

### HTML template

```python
return request.render('my_module.my_template', {'values': data})
```

### JSON (type='json' route)

Return a plain Python object — the framework serialises it automatically:

```python
@http.route('/api/items', type='json', auth='user')
def get_items(self):
    return {'items': [...]}
```

### JSON from a type='http' route

```python
import json
from odoo.http import request, Response

return Response(
    json.dumps({'ok': True}),
    content_type='application/json',
    status=200,
)
```

### Redirect

```python
return request.redirect('/web#action=my_module.action_my_model')
```

### File download

```python
return request.make_response(
    file_content,
    headers=[
        ('Content-Type', 'application/pdf'),
        ('Content-Disposition', 'attachment; filename="report.pdf"'),
    ],
)
```

### Not Found / 404

```python
from werkzeug.exceptions import NotFound
raise NotFound()
```

---

## Error handling

### Raising user-facing errors

```python
from odoo.exceptions import UserError, AccessError, ValidationError

raise UserError("Something went wrong.")
raise AccessError("You don't have permission to do this.")
```

### `handle_params_access_error` (SaaS-19.3)

Override this method in your Controller to provide a custom response when an access error is raised during URL parameter processing (e.g. reading a record from a URL slug):

```python
class MyController(http.Controller):

    def handle_params_access_error(self, error):
        # error is the AccessError instance
        return request.render('my_module.access_denied', status=403)

    @http.route('/item/<model("my.model"):item>', auth='public')
    def item_page(self, item):
        return request.render('my_module.item_page', {'item': item})
```

Without the override, Odoo raises a 403 JSON error by default.

---

## Controller inheritance

Override a parent controller's route by redefining it in a child controller with the same path:

```python
from odoo.addons.web.controllers.home import Home

class MyHome(Home):

    @http.route('/web/login', auth='public', website=True)
    def web_login(self, redirect=None, **kw):
        # Custom logic before delegating
        result = super().web_login(redirect=redirect, **kw)
        return result
```

> **Important**: The child class does **not** need to re-declare `@http.route` with full params if the intent is only to override behaviour — but if you change `auth`, `type`, `methods`, etc., you must re-declare the full decorator.

---

## Common patterns

### JSON REST API with bearer auth

```python
class ApiV1(http.Controller):

    @http.route('/api/v1/orders', auth='bearer', type='json',
                methods=['GET'], cors='*', readonly=True, save_session=False)
    def list_orders(self):
        orders = request.env['sale.order'].search_read(
            [('state', '=', 'sale')],
            ['name', 'partner_id', 'amount_total'],
        )
        return {'orders': orders}

    @http.route('/api/v1/orders', auth='bearer', type='json', methods=['POST'], cors='*')
    def create_order(self):
        data = request.get_json_data()
        order = request.env['sale.order'].create(data)
        return {'id': order.id}
```

### Public website page

```python
class WebsiteShop(http.Controller):

    @http.route('/shop', auth='public', website=True, sitemap=True)
    def shop(self, **kwargs):
        products = request.env['product.template'].search([('website_published', '=', True)])
        return request.render('website_sale.shop', {'products': products})
```

### Multi-company context

```python
@http.route('/api/invoice', auth='user', type='json')
def get_invoice(self, company_id=None):
    env = request.env
    if company_id:
        env = env(context=dict(env.context, allowed_company_ids=[int(company_id)]))
    invoices = env['account.move'].search([('move_type', '=', 'out_invoice')])
    return invoices.read(['name', 'amount_total'])
```

### Sudo when needed

```python
@http.route('/api/public-info', auth='public', type='json', readonly=True)
def public_info(self):
    # Public user can't read res.partner directly — use sudo()
    partner = request.env['res.partner'].sudo().browse(int(request.params.get('id', 0)))
    if not partner.exists():
        return {}
    return {'name': partner.name}
```

### File upload

```python
@http.route('/upload', methods=['POST'], auth='user', csrf=True)
def upload_file(self, attachment=None, **kwargs):
    if attachment is None:
        return {'error': 'no file'}
    content = attachment.read()
    record = request.env['ir.attachment'].create({
        'name': attachment.filename,
        'datas': content,
        'res_model': 'my.model',
    })
    return request.redirect('/my/page')
```

---

## Route URL patterns

| Pattern | Example URL | Python parameter |
|---------|-------------|------------------|
| `'/item/<int:item_id>'` | `/item/42` | `item_id: int` |
| `'/item/<string:name>'` | `/item/hello` | `name: str` |
| `'/item/<model("my.model"):item>'` | `/item/42` | `item: my.model recordset` |
| `'/item/<path:subpath>'` | `/item/a/b/c` | `subpath: str` |
| `'/item/<slug:item>'` | `/item/my-name-42` | `item: my.model recordset` |

`<model("my.model"):item>` looks up the record by ID; raises `AccessError` if not found / not accessible (handled by `handle_params_access_error`).

`<slug:item>` looks up by the slug field (`_rec_name`-derived or custom); same access error behaviour.
