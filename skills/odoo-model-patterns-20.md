# Odoo 20.0 Model Patterns

```
╔══════════════════════════════════════════════════════════════════════════════╗
║  ODOO 20.0 ORM PATTERNS                                                      ║
║  Domain objects, models.Constraint/Index, BinaryValue, zoneinfo, ormcache    ║
║  Verified against odoo/orm/*.py of the 20.0 branch                           ║
║  VERIFY: https://github.com/odoo/odoo/tree/20.0/odoo/orm                     ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

## Key Characteristics

| Feature | Odoo 20.0 pattern |
|---------|-------------------|
| Python | 3.12 - 3.14 (PEP 695 generics and `typing.Self` available) |
| Type hints | Optional, not enforced (core annotates signatures, not field declarations) |
| Create | `@api.model_create_multi` with a list of vals |
| SQL constraints / indexes | `models.Constraint`, `models.Index`, `models.UniqueIndex` attributes (`_sql_constraints` ignored) |
| Domains | `odoo.fields.Domain` (`&`, `\|`, `~`, `Domain.AND`, `Domain.OR`); `odoo.osv` removed |
| Aggregations | `_read_group` (backend), `read_group` (new tuple API), `formatted_read_group` (web dicts) |
| Access checks | `check_access`, `has_access`, `_filtered_access`, `_access_domain`, `check_field_access` |
| Binary | `BinaryValue` / `BinaryBytes`; `ir.attachment.raw` (no `datas`) |
| Time zones | `zoneinfo.ZoneInfo`, `datetime.UTC`; `env.tz` is a `ZoneInfo` |
| Cache | `@api.ormcache(..., cache=...)`, `env.transaction.invalidate_ormcache(name)` |
| Translations | `self.env._("Text %s", value)` |

## Imports (ruff/isort order)

```python
import logging
from datetime import UTC, timedelta
from zoneinfo import ZoneInfo

from odoo import Command, api, fields, models
from odoo.exceptions import AccessError, UserError, ValidationError
from odoo.fields import Domain
from odoo.tools import SQL, BinaryBytes, float_compare

_logger = logging.getLogger(__name__)
```

Four groups, alphabetically sorted: stdlib, third-party, `odoo`, `odoo.addons.*`.

## Model Definition

```python
class LibraryBook(models.Model):
    _name = 'library.book'
    _description = "Library Book"
    _explanation = "A physical book of the library that members can borrow."   # new in 20.0 (optional)
    _inherit = ['mail.thread', 'mail.activity.mixin']
    _order = 'sequence, id desc'
    _check_company_auto = True
    _rec_names_search = ('name', 'isbn')    # tuple (upgrade script 19.5 converts lists)

    # SQL constraints and indexes: attribute name must start with '_'
    _isbn_company_uniq = models.Constraint(
        'UNIQUE(isbn, company_id)',
        "The ISBN must be unique per company.",
    )
    _copies_positive = models.Constraint(
        'CHECK(copies >= 0)',
        "The number of copies cannot be negative.",
    )
    _state_date_idx = models.Index('(state, date_published DESC)')
    _active_code_uniq = models.UniqueIndex('(code) WHERE active IS TRUE')

    name = fields.Char(required=True, index='trigram', tracking=True, translate=True)
    code = fields.Char(copy=False, index=True)
    isbn = fields.Char(string="ISBN", copy=False)
    sequence = fields.Integer(default=10)
    active = fields.Boolean(default=True)
    state = fields.Selection([
        ('draft', "Draft"),
        ('available', "Available"),
        ('borrowed', "Borrowed"),
        ('lost', "Lost"),
    ], default='draft', required=True, tracking=True)
    date_published = fields.Date()
    date_due = fields.Datetime()
    copies = fields.Integer(default=1)
    price = fields.Monetary(currency_field='currency_id')
    currency_id = fields.Many2one(related='company_id.currency_id')
    company_id = fields.Many2one(
        'res.company', required=True, index=True, default=lambda self: self.env.company,
    )
    author_id = fields.Many2one('res.partner', check_company=True, index='btree_not_null')
    user_id = fields.Many2one('res.users', default=lambda self: self.env.user, check_company=True)
    tag_ids = fields.Many2many('library.tag')
    loan_ids = fields.One2many('library.loan', 'book_id')
    loan_count = fields.Integer(compute='_compute_loan_count')
    cover = fields.Image(max_width=1024, max_height=1024)
    document = fields.Binary(attachment=True)
    metadata = fields.Json()
    is_overdue = fields.Boolean(compute='_compute_is_overdue', search='_search_is_overdue')
```

Model body order (official guideline): private attributes → default methods → fields →
`models.Constraint`/`models.Index` (core also puts them right after private attributes) →
compute/inverse/search → selection methods → `@api.constrains`/`@api.onchange` → CRUD → actions → business methods.

## Computes, Search Methods and Constraints

```python
    @api.depends('loan_ids')
    def _compute_loan_count(self):
        counts = {
            book.id: count
            for book, count in self.env['library.loan']._read_group(
                [('book_id', 'in', self.ids)], ['book_id'], ['__count'],
            )
        }
        for book in self:
            book.loan_count = counts.get(book.id, 0)

    @api.depends('date_due', 'state')
    def _compute_is_overdue(self):
        now = fields.Datetime.now()
        for book in self:
            book.is_overdue = book.state == 'borrowed' and book.date_due and book.date_due < now

    def _search_is_overdue(self, operator, value):
        if operator not in ('in', 'not in'):
            return NotImplemented
        overdue = Domain('state', '=', 'borrowed') & Domain('date_due', '<', fields.Datetime.now())
        return overdue if operator == 'in' else ~overdue

    @api.constrains('date_published')
    def _check_date_published(self):
        today = fields.Date.context_today(self)
        for book in self:
            if book.date_published and book.date_published > today:
                raise ValidationError(self.env._("Book %s cannot be published in the future.", book.display_name))
```

- `@api.depends` must list every field the compute reads.
- Search methods receive the optimized operator (`in`/`not in` for booleans since 19.0) and may return a `Domain`.
- Put conditions in the domain, not in `filtered()`, and batch ORM calls (one `_read_group` instead of a `search_count` per record).

## CRUD

```python
    @api.model_create_multi
    def create(self, vals_list):
        for vals in vals_list:
            if not vals.get('code'):
                vals['code'] = self.env['ir.sequence'].next_by_code('library.book')
        return super().create(vals_list)

    def write(self, vals):
        if 'state' in vals and vals['state'] == 'lost' and not self.env.user.has_group('library.group_library_manager'):
            raise AccessError(self.env._("Only librarians can mark a book as lost."))
        return super().write(vals)

    @api.ondelete(at_uninstall=False)
    def _unlink_except_borrowed(self):
        if any(book.state == 'borrowed' for book in self):
            raise UserError(self.env._("You cannot delete a borrowed book."))

    def copy_data(self, default=None):
        vals_list = super().copy_data(default=default)
        return [dict(vals, name=self.env._("%s (copy)", book.name)) for book, vals in zip(self, vals_list)]
```

## Actions and Public vs Private Methods

```python
    def action_mark_available(self):
        self.ensure_one()
        self.state = 'available'

    def action_view_loans(self):
        self.ensure_one()
        return {
            'type': 'ir.actions.act_window',
            'name': self.env._("Loans"),
            'res_model': 'library.loan',
            'view_mode': 'list,form',
            'domain': [('book_id', '=', self.id)],
            'context': {'default_book_id': self.id},
        }

    @api.private
    def compute_fine(self, days_late):
        """Public name kept for callers, but not callable through RPC."""
        return days_late * 0.5

    def _get_overdue_domain(self):
        # small overridable extension point, private by default
        return Domain('is_overdue', '=', True)
```

Every public method is an RPC entry point (`get_public_method`): `_` prefix by default,
`@api.private` for public names that must not be exposed, validate inputs otherwise.

## Domains

```python
domain = Domain('state', '=', 'available') & Domain('company_id', 'in', self.env.companies.ids)
if partner:
    domain &= Domain('author_id', '=', partner.id)
books = self.search(domain)

either = Domain.OR([Domain('state', '=', 'draft'), Domain('copies', '=', 0)])
not_lost = ~Domain('state', '=', 'lost')
related = Domain('loan_ids', 'any', Domain('partner_id', '=', partner.id))
```

Never hand-build `'&'`/`'|'` prefix lists and never concatenate a user-provided list onto a security domain.

## Aggregations

```python
# Backend: recordsets and aggregated values
for author, total, count in self._read_group(
    [('state', '!=', 'lost')], ['author_id'], ['price:sum', '__count'],
):
    ...

# RPC-friendly (NEW contract in 20.0): same arguments, ids instead of recordsets
rows = self.read_group([('state', '!=', 'lost')], ['author_id'], ['price:sum'])   # [(author_id, total), ...]

# Web-formatted dicts (old read_group output) from the web module
groups = self.formatted_read_group([('state', '!=', 'lost')], ['author_id'], ['price:sum'])
```

## Access Checks

```python
self.check_access('write')                  # raises AccessError (model + record level)
if self.browse().has_access('create'):      # model-level boolean check
    ...
allowed = self._filtered_access('read')     # subset the user may read
domain = self._access_domain('read')        # Domain of readable records
field = self._fields['price']
self.check_field_access(field, 'read')      # field `groups`
```

## SQL (only when the ORM cannot do it)

```python
def _get_top_authors(self, limit=10):
    query = self._search([('state', '!=', 'lost')])   # access rules and active_test applied
    query.groupby = SQL("%s", query.table.author_id)
    query.order = SQL("COUNT(*) DESC")
    query.limit = limit
    return self.env.execute_query(query.select(query.table.author_id, SQL("COUNT(*)")))

def _bulk_mark_lost(self, book_ids):
    self.env.cr.execute(SQL(
        "UPDATE %s SET state = %s WHERE id = ANY(%s)",
        SQL.identifier(self._table), 'lost', list(book_ids),
    ))
    self.browse(book_ids).invalidate_recordset(['state'])
```

- `SQL.identifier()` for table/column names; values always as parameters.
- `env.execute_query(sql)` returns tuples, `env.execute_query_dict(sql)` returns dicts.
- `SQL(...).code`/`.params` no longer exist in 20.0.

## Binary Data and Attachments

```python
def _attach_pdf(self, pdf_bytes):
    self.ensure_one()
    return self.env['ir.attachment'].create({
        'name': f"{self.code}.pdf",
        'raw': pdf_bytes,              # 'datas' is removed and ignored in 20.0
        'res_model': self._name,
        'res_id': self.id,
    })

def _set_document(self, content):
    self.document = BinaryBytes(content, filename='book.pdf')   # raw bytes raise TypeError

def _document_size(self):
    return self.document.size if self.document else 0          # BinaryValue API
```

## Dates and Time Zones

```python
def _due_date_in_user_tz(self):
    self.ensure_one()
    if not self.date_due:
        return False
    return self.date_due.replace(tzinfo=UTC).astimezone(self.env.tz)   # env.tz is a ZoneInfo

def _to_utc_naive(self, local_naive, tz_name):
    return local_naive.replace(tzinfo=ZoneInfo(tz_name or 'UTC')).astimezone(UTC).replace(tzinfo=None)
```

## Caching

```python
    @api.ormcache('self.env.company.id', cache='stable')
    def _get_default_loan_days(self):
        return int(self.env['ir.config_parameter'].sudo().get_param('library.loan_days', 14))

    def _clear_library_cache(self):
        self.env.transaction.invalidate_ormcache('stable')
```

Import `ormcache` from `odoo.api`; never return recordsets from a cached method.

## Mail Tracking Overrides

```python
    def _track_log_get_default_subtype(self, track_init_values):
        self.ensure_one()
        if 'state' in track_init_values and self.state == 'lost':
            return self.env.ref('library.mt_book_lost')
        return super()._track_log_get_default_subtype(track_init_values)

    def _track_template_parameters(self, tracked_fields):
        res = super()._track_template_parameters(tracked_fields)
        book = self[0]
        if 'state' in tracked_fields and book.state == 'available':
            res['state'] = (self.env.ref('library.mail_template_book_available'), {
                'auto_delete_keep_log': False,
                'subtype_id': self.env['ir.model.data']._xmlid_to_res_id('mail.mt_note'),
            })
        return res
```

## Archiving and Hierarchies

```python
books.action_archive()        # toggle_active() was removed in 20.0
books.action_unarchive()

@api.constrains('parent_id')
def _check_parent_id(self):
    if self._has_cycle():     # _check_recursion() was removed in 20.0
        raise ValidationError(self.env._("You cannot create recursive categories."))
```

## Crons

```python
@api.model
def _cron_send_reminders(self):
    books = self.search(self._get_overdue_domain(), limit=500)
    self.env['ir.cron']._commit_progress(remaining=len(books))
    for book in books:
        book._send_reminder()
        if not self.env['ir.cron']._commit_progress(1):   # returns remaining time; 0 = stop
            break
```

## New Field Capabilities in 20.0

```python
    # searchable/groupable non-stored compute through SQL
    loan_state = fields.Selection(
        [('free', "Free"), ('loaned', "Loaned")],
        compute='_compute_loan_state', compute_sql='_compute_sql_loan_state', compute_sudo=True,
    )

    @api.depends('state')
    def _compute_loan_state(self):
        for book in self:
            book.loan_state = 'loaned' if book.state == 'borrowed' else 'free'

    def _compute_sql_loan_state(self, table):
        return SQL("CASE WHEN %s = 'borrowed' THEN 'loaned' ELSE 'free' END", table.state)

    # initialize a new column for existing rows (called on install/upgrade)
    barcode = fields.Char(init_storage='_init_barcode')

    def _init_barcode(self):
        self.env.cr.execute(SQL("UPDATE %s SET barcode = code WHERE barcode IS NULL", SQL.identifier(self._table)))

    # computed copy value
    reference = fields.Char(copy=lambda book: f"{book.reference}-COPY" if book.reference else False)
```

## Stable Read-Mostly Data: `CachedModel`

```python
class LibraryShelf(models.CachedModel):
    _name = 'library.shelf'
    _description = "Library Shelf"
    _cached_data_domain = [('active', '=', True)]
    _cached_data_fields = ('name', 'code')

    name = fields.Char(required=True)
    code = fields.Char(required=True)
    active = fields.Boolean(default=True)

# self.env['library.shelf'].get_all() -> all cached shelves, served from the 'stable' cache
```

## v20 Best Practices

1. `self.env._()` with static literals and arguments; never `_()` at module level (use `LazyTranslate`).
2. `Domain` objects for every domain combination.
3. `_read_group` in backend code; review every `read_group` call (new contract).
4. `check_access`/`has_access`/`_filtered_access` instead of the removed v18-v19 aliases.
5. Batch ORM calls; indexes only on selective searched fields.
6. `@api.ondelete` for deletion rules instead of overriding `unlink`.
7. `BinaryBytes`/`raw` for binary data; `zoneinfo` for time zones.
8. Never `cr.commit()` outside your own cursor (except `_commit_progress` in crons).

## AI Agent Instructions (v20 models)

1. **USE** `models.Constraint`/`models.Index`, `Domain`, `@api.model_create_multi`, `@api.ondelete`.
2. **USE** `_read_group`; only use `read_group` with the 20.0 tuple contract.
3. **DO NOT** use `check_access_rights`, `check_access_rule`, `_filter_access_rules`, `toggle_active`,
   `_check_recursion`, `registry.clear_cache`, `odoo.osv.expression`, `pytz`, `ir.attachment.datas`.
4. **DO NOT** override `_track_subtype`/`_track_template` (renamed).
5. **VERIFY** any API you are unsure about with `git grep` in the 20.0 source.
