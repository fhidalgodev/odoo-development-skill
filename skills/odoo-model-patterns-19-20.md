# Odoo Model Patterns Migration Guide: 19.0 → 20.0

```
╔══════════════════════════════════════════════════════════════════════════════╗
║  MODEL MIGRATION GUIDE: Odoo 19.0 → 20.0                                     ║
║  Focus: removed ORM aliases, read_group, BinaryValue, zoneinfo, caches       ║
║  Every "before" snippet is valid 19.0 code, every "after" is valid 20.0 code ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

## Summary

| Feature | 19.0 | 20.0 | Action |
|---------|------|------|--------|
| `check_access_rights` / `check_access_rule` | Deprecated aliases | **Removed** | `check_access` / `has_access` |
| `_filter_access_rules(_python)` | Deprecated | **Removed** | `_filtered_access` |
| `check_field_access_rights` | Deprecated | **Removed** | `check_field_access` / `has_field_access` |
| `_check_recursion` / `_check_m2m_recursion` | Deprecated | **Removed** | `_has_cycle` |
| `toggle_active` | Deprecated | **Removed** | `action_archive` / `action_unarchive` |
| `read_group(domain, fields, groupby, ..., lazy)` | Deprecated | **New signature** | `_read_group` / `formatted_read_group` |
| `registry.clear_cache()` | Available | **Removed** | `env.transaction.invalidate_ormcache()` |
| `tools.ormcache` / `ormcache_context` | Available | Deprecated / **Removed** | `api.ormcache` |
| `odoo.osv.expression` | Available | **Removed** | `odoo.fields.Domain` |
| Binary values | base64 bytes | `BinaryValue` | `BinaryBytes` |
| `ir.attachment.datas` | Field | **Removed** | `raw` |
| `env.tz` | pytz tz | `zoneinfo.ZoneInfo` | stdlib API |
| `_track_subtype` / `_track_template` | Hooks | **Renamed** | `_track_log_get_default_subtype` / `_track_template_parameters` |
| `SQL(...).code` / `.params` | Properties | **Removed** | pass `SQL` objects |
| `_order_to_sql`, `_read_group_groupby`, `_read_group_select` | `(alias, ..., query)` | `(table: TableSQL, ...)` | update overrides |

## Access Checks

### Before (v19)
```python
def action_lend(self):
    self.check_access_rights('write')
    self.check_access_rule('write')
    lendable = self._filter_access_rules('write')
    allowed = self.check_field_access_rights('read', ['price', 'isbn'])
```

### After (v20)
```python
def action_lend(self):
    self.check_access('write')                      # model + record level
    lendable = self._filtered_access('write')
    price_field = self._fields['price']
    if self.has_field_access(price_field, 'read'):
        ...
```

For a pure model-level question use `self.env['library.book'].has_access('create')` (empty recordset).

## `read_group`

### Before (v19)
```python
data = self.env['library.loan'].read_group(
    [('state', '=', 'open')], ['duration:sum'], ['book_id'], lazy=False,
)
totals = {row['book_id'][0]: row['duration'] for row in data}
```

### After (v20)
```python
totals = {
    book.id: duration
    for book, duration in self.env['library.loan']._read_group(
        [('state', '=', 'open')], ['book_id'], ['duration:sum'],
    )
}
```

Need the dict format for a client? `formatted_read_group(domain, groupby, aggregates)` (web module).
Calling `read_group` from JS/RPC in v20 returns tuples: `[[book_id, total], ...]`.

## Hierarchies and Archiving

### Before (v19)
```python
@api.constrains('parent_id')
def _check_parent_id(self):
    if not self._check_recursion():
        raise ValidationError(_("Recursive categories are not allowed."))

def action_toggle(self):
    self.toggle_active()
```

### After (v20)
```python
@api.constrains('parent_id')
def _check_parent_id(self):
    if self._has_cycle():
        raise ValidationError(self.env._("Recursive categories are not allowed."))

def action_toggle(self):
    active = self.filtered('active')
    active.action_archive()
    (self - active).action_unarchive()
```

## Domains

### Before (v19)
```python
from odoo.osv import expression

domain = expression.AND([
    [('state', '=', 'available')],
    expression.OR([[('author_id', '=', author.id)], [('user_id', '=', self.env.uid)]]),
])
```

### After (v20)
```python
from odoo.fields import Domain

domain = Domain('state', '=', 'available') & (
    Domain('author_id', '=', author.id) | Domain('user_id', '=', self.env.uid)
)
```

Boolean fields must be compared with booleans (`('active', '=', True)`, not `'True'` or `1`): a
`DeprecationWarning` is emitted otherwise.

## Caches

### Before (v19)
```python
from odoo import tools

@tools.ormcache('code')
def _get_rate(self, code):
    ...

def write(self, vals):
    res = super().write(vals)
    self.env.registry.clear_cache()
    return res
```

### After (v20)
```python
from odoo import api

@api.ormcache('code')
def _get_rate(self, code):
    ...

def write(self, vals):
    res = super().write(vals)
    self.env.transaction.invalidate_ormcache()          # 'default' cache
    return res
```

`ormcache_context('key', keys=('lang',))` → `@api.ormcache('key', 'self.env.lang')`.

## Binary Fields and Attachments

### Before (v19)
```python
record.document = base64.b64encode(pdf_bytes)
attachment = self.env['ir.attachment'].create({'name': 'a.pdf', 'datas': base64.b64encode(pdf_bytes)})
content = base64.b64decode(record.document)
```

### After (v20)
```python
from odoo.tools import BinaryBytes

record.document = BinaryBytes(pdf_bytes, filename='a.pdf')
attachment = self.env['ir.attachment'].create({'name': 'a.pdf', 'raw': pdf_bytes})
content = record.document.content          # bytes; .to_base64() for a base64 str
```

`datas` in `ir.attachment.create()` is **dropped with a warning** in 20.0: search for it before migrating data.

## Time Zones

### Before (v19)
```python
import pytz

user_tz = pytz.timezone(self.env.user.tz or 'UTC')
local = pytz.utc.localize(record.date_due).astimezone(user_tz)
start_utc = user_tz.localize(naive_start).astimezone(pytz.utc).replace(tzinfo=None)
```

### After (v20)
```python
from datetime import UTC
from zoneinfo import ZoneInfo

user_tz = ZoneInfo(self.env.user.tz or 'UTC')          # or self.env.tz
local = record.date_due.replace(tzinfo=UTC).astimezone(user_tz)
start_utc = naive_start.replace(tzinfo=user_tz).astimezone(UTC).replace(tzinfo=None)
```

## Mail Tracking

### Before (v19)
```python
def _track_subtype(self, init_values):
    self.ensure_one()
    if 'state' in init_values and self.state == 'lost':
        return self.env.ref('library.mt_book_lost')
    return super()._track_subtype(init_values)

def _track_template(self, changes):
    res = super()._track_template(changes)
    ...
```

### After (v20)
```python
def _track_log_get_default_subtype(self, track_init_values):
    self.ensure_one()
    if 'state' in track_init_values and self.state == 'lost':
        return self.env.ref('library.mt_book_lost')
    return super()._track_log_get_default_subtype(track_init_values)

def _track_template_parameters(self, tracked_fields):
    res = super()._track_template_parameters(tracked_fields)
    ...
```

The old names are not called anymore: a forgotten override silently stops working.

## SQL Helpers and Query Internals

| v19 | v20 |
|-----|-----|
| `from odoo.tools.query import Query` | `from odoo.models import Query, TableSQL` |
| `query.join(alias, col, table, col2, link)` | `table._join(field_name)` / `query.add_join(...)` |
| `query.add_where("state = %s", ['done'])` | `query.add_where(SQL("%s = %s", query.table.state, 'done'))` |
| `query.order = "id desc"` | `query.order = SQL("%s DESC", query.table.id)` |
| `self._order_to_sql(order, query, alias)` | `self._order_to_sql(query.table, order)` |
| `self._field_to_sql(alias, fname, query)` | still available, or `TableSQL(alias, self, query)[fname]` |
| `sql.code`, `sql.params` | `SQL` objects are composed directly; `_sql_tuple` is internal |
| `tools.sql.escape_psql(value)` | `tools.sql.escape_like_value(value)` |

## Base Model Renames Seen in Business Code

```python
# v19
partner.company_type == 'company'
partner.company_registry
bank_account.acc_number, bank_account.acc_holder_name, bank_account.bank_id.name
currency.date
country.zip_required

# v20
partner.is_company
partner.additional_identifiers            # Json {key: value}, keys declared by _get_all_additional_identifiers_metadata()
bank_account.account_number, bank_account.holder_name, bank_account.bank_name
currency.rate_date
country.zip_applicability
```

## Tests

```python
# v19
from odoo.tests.common import SingleTransactionCase
class TestLibrary(SingleTransactionCase): ...

# v20
from odoo.addons.base.tests.common import BaseCommon
class TestLibrary(BaseCommon): ...
```

## Migration Checklist

- [ ] Replace removed access helpers, recursion helpers, `toggle_active`
- [ ] Rewrite every `read_group` call
- [ ] `registry.clear_cache` → `transaction.invalidate_ormcache`; `ormcache` from `odoo.api`
- [ ] `odoo.osv.expression` → `Domain`
- [ ] Binary writes with `BinaryBytes`; attachments with `raw`
- [ ] `pytz` → `zoneinfo`
- [ ] Rename mail tracking overrides
- [ ] Update `_order_to_sql` / `_read_group_*` overrides to `TableSQL`
- [ ] Apply base model renames
- [ ] Replace `SingleTransactionCase`
