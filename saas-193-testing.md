# Odoo SaaS-19.3 Testing Guide

Reference for writing tests in SaaS-19.3.
Source of truth: `/home/achraf/src/193/odoo/odoo/tests/common.py`

> **SaaS-19.3 key additions**
> - `MockHTTPClient` — built-in HTTP request mocking (new)
> - `freeze_time` class-level decorator via Odoo wrapper
> - `Like` / `Approx` assertion helpers
> - `RecordCapturer` context manager
> - `assertQueries` — exact SQL query matching with wildcards

---

## Table of Contents

1. [Test class hierarchy](#test-class-hierarchy)
2. [@tagged decorator](#tagged-decorator)
3. [setUp patterns](#setup-patterns)
4. [Form helper](#form-helper)
5. [Assertion methods](#assertion-methods)
6. [User helpers](#user-helpers)
7. [HttpCase — HTTP and tour testing](#httpcase--http-and-tour-testing)
8. [Utility helpers](#utility-helpers)
9. [Complete example](#complete-example)

---

## Test class hierarchy

```
unittest.TestCase
  └── BaseCase                  # Root Odoo class
        ├── TransactionCase     # Most tests — one transaction per class, savepoint per method
        └── HttpCase            # HTTP/UI tests with headless browser
```

### TransactionCase

The standard class for all model/ORM tests.

```python
from odoo.tests import TransactionCase, tagged

@tagged('post_install', '-at_install')
class TestMyModel(TransactionCase):

    @classmethod
    def setUpClass(cls):
        super().setUpClass()
        # Shared setup — runs once per class, inside the class transaction
        cls.partner = cls.env['res.partner'].create({'name': 'Test Partner'})

    def test_something(self):
        # Each test runs in a savepoint that is rolled back automatically
        record = self.env['my.model'].create({'name': 'Test'})
        self.assertEqual(record.name, 'Test')
```

Key behaviour:
- One DB transaction for the entire class (never committed)
- Each test method runs in a savepoint → automatically rolled back
- `self.env` is SUPERUSER by default
- `setUpClass` data is shared across all test methods (not rolled back between them)

### HttpCase

For tests that need an HTTP session or a headless browser.

```python
from odoo.tests import HttpCase, tagged

@tagged('post_install', '-at_install')
class TestMyUI(HttpCase):

    def test_tour(self):
        self.start_tour('/odoo/my-model', 'my_module.tour_name', login='admin')
```

---

## @tagged decorator

Controls when and whether a test runs.

```python
@tagged('post_install', '-at_install')  # run after install, not during
@tagged('standard')                      # run in standard test suite
@tagged('-standard', 'my_tag')          # remove 'standard', add 'my_tag'
```

| Tag | Meaning |
|-----|---------|
| `standard` | Run in the default test suite |
| `at_install` | Run during module installation |
| `post_install` | Run after all modules are installed (default) |
| `-tag` | Remove an inherited tag |

Run specific tags: `--test-tags my_tag` or `--test-tags post_install`

---

## setUp patterns

```python
@classmethod
def setUpClass(cls):
    super().setUpClass()
    # Runs once. Creates records shared by all tests in the class.
    cls.pricelist = cls.env.ref('product.list0')
    cls.product = cls.env['product.product'].create({
        'name': 'Widget',
        'list_price': 100.0,
    })

def setUp(self):
    super().setUp()
    # Runs before each test. Use for per-test overrides or cleanups.
    self.env = self.env(context=dict(self.env.context, lang='fr_FR'))
```

---

## Form helper

Simulates the web client form — applies `onchange` hooks automatically.

```python
from odoo.tests import Form

# Create a new record
with Form(self.env['sale.order']) as f:
    f.partner_id = self.partner       # triggers partner onchange
    with f.order_line.new() as line:
        line.product_id = self.product
        line.product_uom_qty = 3
    order = f.save()

# Edit an existing record
with Form(order) as f:
    f.note = 'Updated note'
    # auto-saved on context exit

# Many2many manipulation
with Form(self.env.user) as u:
    u.groups_id.add(self.env.ref('account.group_account_manager'))
    u.groups_id.remove(id=self.env.ref('base.group_portal').id)
```

> `Form` respects `readonly`, `required`, and `invisible` rules — it raises if you try to
> set a readonly field, matching what the UI would do.

---

## Assertion methods

### assertRecordValues

Compare a recordset against a list of expected dicts. Fails with a clear diff.

```python
self.assertRecordValues(
    self.env['sale.order'].search([('partner_id', '=', self.partner.id)]),
    [
        {'name': 'S00001', 'state': 'sale',  'amount_total': 300.0},
        {'name': 'S00002', 'state': 'draft', 'amount_total': 0.0},
    ],
)
```

### assertQueryCount

Assert the number of SQL queries executed in a block.

```python
with self.assertQueryCount(3):
    order.action_confirm()

# With multiple users (used with @users decorator)
with self.assertQueryCount(admin=3, demo=4):
    order.read(['name', 'state'])
```

Add `@warmup` to the test to pre-warm the ORM cache before counting:

```python
from odoo.tests.common import warmup

@warmup
def test_query_count(self):
    with self.assertQueryCount(5):
        self.env['sale.order'].search([])
```

### assertQueries

Match exact SQL queries with `...` as a wildcard:

```python
with self.assertQueries([
    'SELECT ... FROM "sale_order" WHERE ...',
    'UPDATE "sale_order" SET ...',
]):
    order.action_confirm()
```

### assertXMLEqual / assertHTMLEqual

```python
self.assertXMLEqual(view.arch_db, '<form><field name="name"/></form>')
self.assertHTMLEqual(response.text, '<div class="container"><p>Hello</p></div>')
```

### Like — pattern matching

```python
from odoo.tests.common import Like

self.assertEqual(order.name, Like('S0%'))
self.assertIn(Like('Company ... (SF)'), partner_names)
```

### Approx — float comparison with precision

```python
from odoo.tests.common import Approx

self.assertEqual(order.amount_total, Approx(99.99, 2))          # 2 decimal places
self.assertEqual(order.amount_total, Approx(100.0, self.currency))  # currency rounding
```

---

## User helpers

### new_test_user

```python
from odoo.tests.common import new_test_user

user = new_test_user(
    self.env,
    login='test_user',
    groups='base.group_user,sale.group_sale_manager',
    name='Test User',
    email='test@example.com',
)
```

### @users — run a test as multiple users

```python
from odoo.tests.common import users

@users('admin', 'demo')
def test_access(self):
    # Runs twice: once as admin, once as demo
    # self.env.user is the current test user
    record = self.env['my.model'].search([])
    self.assertTrue(len(record) >= 0)
```

### with_user context

```python
def test_permission(self):
    with self.with_user('demo'):
        with self.assertRaises(AccessError):
            self.env['my.model'].create({'name': 'Forbidden'})
```

---

## HttpCase — HTTP and tour testing

### url_open

```python
def test_api(self):
    self.authenticate('admin', 'admin')
    response = self.url_open('/api/my-endpoint', data={'key': 'value'})
    self.assertEqual(response.status_code, 200)
```

### start_tour

Runs a JavaScript tour defined in `static/src/tours/`:

```python
def test_flow(self):
    order = self.env['sale.order'].create({
        'partner_id': self.env.ref('base.partner_demo').id,
    })
    self.start_tour(
        f'/odoo/sales/{order.id}',
        'sale_order_my_tour',
        login='admin',
        timeout=60,
    )
```

### browser_js

Run arbitrary JavaScript in the headless browser:

```python
def test_js(self):
    self.browser_js(
        '/odoo/sales',
        "console.log('test passed')",
        ready="document.querySelector('.o_list_view')",
        login='admin',
    )
```

---

## Utility helpers

### freeze_time — freeze date/time

```python
from freezegun import freeze_time

@freeze_time('2025-06-01')
class TestDateLogic(TransactionCase):
    def test_expiry(self):
        from odoo import fields
        self.assertEqual(fields.Date.today(), date(2025, 6, 1))
```

### RecordCapturer — capture created records

```python
from odoo.tests.common import RecordCapturer

with RecordCapturer(self.env['res.partner'], []) as cap:
    self.env['sale.order'].create({'partner_id': self.partner.id})
    # ... code that may create partners

new_partners = cap.records  # Only partners created inside the block
```

### MockHTTPClient — mock external HTTP calls (new in SaaS-19.3)

```python
from odoo.tests.common import MockHTTPClient

with MockHTTPClient(
    url='https://api.example.com/v1/data',
    return_json={'status': 'ok', 'id': 42},
) as mock:
    result = self.env['my.model']._call_external_api()
    mock.assert_called_once()

self.assertEqual(result['id'], 42)
```

### ref / browse_ref

```python
partner = self.env.ref('base.partner_demo')       # browse
partner_id = self.ref('base.partner_demo')        # integer ID
```

### debug_mode context

```python
with self.debug_mode():
    # Activates base.group_no_one for the duration
    view = self.env['ir.ui.view'].browse(view_id)
```

### allow_pdf_render

```python
with self.allow_pdf_render():
    pdf, _ = self.env['ir.actions.report']._render_qweb_pdf(
        'my_module.action_report_my_doc', self.record.ids
    )
self.assertGreater(len(pdf), 0)
```

---

## Complete example

```python
from odoo.tests import Form, TransactionCase, tagged
from odoo.tests.common import new_test_user, warmup, users


@tagged('post_install', '-at_install')
class TestSaleOrder(TransactionCase):

    @classmethod
    def setUpClass(cls):
        super().setUpClass()
        cls.partner = cls.env['res.partner'].create({'name': 'Test Customer'})
        cls.product = cls.env['product.product'].create({
            'name': 'Widget',
            'list_price': 50.0,
        })
        cls.manager = new_test_user(
            cls.env,
            login='test_manager',
            groups='base.group_user,sale.group_sale_manager',
        )

    def test_order_creation(self):
        with Form(self.env['sale.order']) as f:
            f.partner_id = self.partner
            with f.order_line.new() as line:
                line.product_id = self.product
                line.product_uom_qty = 2
            order = f.save()

        self.assertRecordValues(order, [{'amount_total': 100.0, 'state': 'draft'}])

    @warmup
    def test_confirm_query_count(self):
        order = self.env['sale.order'].create({
            'partner_id': self.partner.id,
        })
        with self.assertQueryCount(10):
            order.action_confirm()

    @users('admin', 'test_manager')
    def test_multi_user_access(self):
        order = self.env['sale.order'].search([], limit=1)
        self.assertTrue(order.name)
```
