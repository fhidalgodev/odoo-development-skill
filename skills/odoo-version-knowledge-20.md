# Odoo 20.0 Version Knowledge

```
╔══════════════════════════════════════════════════════════════════════════════╗
║  ODOO 20.0 KNOWLEDGE BASE                                                    ║
║  ir.access security, OWL 3, BinaryValue, zoneinfo, Material Symbols icons    ║
║  Verified against the odoo/odoo 20.0 and odoo/enterprise 20.0 source trees   ║
║  VERIFY: https://github.com/odoo/odoo/tree/20.0                              ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

## Version Overview

| Aspect | Details |
|--------|---------|
| Branch | `20.0` (`odoo/release.py`: `version_info = (20, 0, 0, FINAL, 0, '')`) |
| Python | 3.12 minimum, 3.14 maximum (`MIN_PY_VERSION`, `MAX_PY_VERSION`). Running with `python -O` raises at startup |
| PostgreSQL | 16 minimum (`MIN_PG_VERSION = 16`) |
| Frontend | OWL 3 (bundled `3.0.0-alpha.49`) plus a temporary Owl 2 compatibility layer (`@web/owl2/*`) |
| CSS / icons | Bootstrap 5.3.3, Material Symbols + `odoo_ui_icons` fonts. FontAwesome CSS is no longer loaded |
| JS tests | Hoot only (QUnit removed) |
| Security model | `ir.access` (replaces `ir.model.access` and `ir.rule`) |
| Official AI skills | `<odoo_src>/skills/`: `odoo-guidelines`, `odoo-web-guidelines`, `odoo-security`, `odoo-review` |

> Odoo 20 ships its own agent skills in the source tree (`<odoo_src>/skills/`). When writing or
> reviewing v20 code, read the matching section there as the house rules; several rules in this
> file come from them.

## Breaking Changes at a Glance (19.0 → 20.0)

| # | Area | Change | Symptom if ignored |
|---|------|--------|--------------------|
| 1 | Security | `ir.model.access` and `ir.rule` models removed, replaced by `ir.access` (`security/ir.access.csv`) | `KeyError: 'ir.model.access'` at install |
| 2 | ORM | `read_group()` has a NEW signature and returns tuples | Wrong results or `TypeError` |
| 3 | ORM | Removed: `check_access_rights`, `check_access_rule`, `_filter_access_rules(_python)`, `check_field_access_rights`, `_check_recursion`, `_check_m2m_recursion`, `toggle_active`, `copy_translations` | `AttributeError` |
| 4 | Caching | `registry.clear_cache()`, `registry.clear_all_caches()`, `tools.ormcache_context` removed | `AttributeError` / `ImportError` |
| 5 | Imports | `odoo.osv`, `odoo.tools.query`, `odoo.tools.pycompat`, `odoo.tools.populate`, `odoo.tools.test_reports`, `odoo.service.db`, `odoo.service.security` removed | `ModuleNotFoundError` |
| 6 | HTTP | `odoo.http` only exposes `request`, `route`, `Controller`, `Response` (and `request_var`); helpers live in submodules | `ImportError` |
| 7 | HTTP | `auth='bearer'` requires `bearer_scope` | `AssertionError` when the module is imported |
| 8 | Binary | Binary values are `BinaryValue` objects; writing raw `bytes` raises; `ir.attachment.datas` removed and silently ignored | `TypeError` / empty attachments |
| 9 | XML data | `type="base64"` deprecated in favour of `type="bytes"` | `DeprecationWarning` |
| 10 | Time zones | Core dropped `pytz`; `env.tz` returns `zoneinfo.ZoneInfo` | `AttributeError: ... has no attribute 'localize'` |
| 11 | Server QWeb | `t-esc` / `t-raw` no longer compiled; `t-set` children of `t-call` no longer passed to the callee | Blank values in reports, mails, website |
| 12 | Reports | `ir.actions.report.report_file` field removed | `ValueError: Invalid field 'report_file' in 'ir.actions.report'` at install |
| 13 | Views | Button `icon="fa-..."` not rendered; widgets `remaining_days`, `selection_badge` removed | Missing icons, unknown widget |
| 14 | Mail | `_track_subtype` → `_track_log_get_default_subtype`; `_track_template` → `_track_template_parameters` | Override silently never called |
| 15 | JS | OWL 3: `static props`/`defaultProps` throw; `useState`, `useRef`, `useExternalListener`, `reactive` gone; templates need `this.` | Runtime errors |
| 16 | JS | jQuery, legacy `publicWidget`, QUnit removed | Frontend errors |
| 17 | Tests | `SingleTransactionCase` removed | `ImportError` |
| 18 | Base data | `res.bank` removed, `res.partner.bank` fields renamed, `res.partner.company_type` / `company_registry` removed; `l10n_latam_base` removed, `base_vat`/`base_iban`/`stock_picking_batch` merged | Invalid field, missing dependency |

> **Code rewriting helper:** `./odoo-bin upgrade_code --from 19.0 --addons-path=<your_addons>`
> runs every official rewrite script between 19.0 and 20.0 (see [Tooling](#tooling-odoo-bin-upgrade_code)).
> The OWL 3 script must be run explicitly with `--script owl3-migration`.

---

## 1. Security: `ir.access` Replaces ACLs and Record Rules

One model, one CSV. Details and patterns: `odoo-security-guide-20.md`.

```csv
id,name,model_id,group_id/id,operation,domain
access_library_book_user,library.book user,library.book,library.group_library_user,r,
access_library_book_manager,library.book manager,library.book,library.group_library_manager,crud,
library_book_rule_own,library.book own books,library.book,library.group_library_user,ru,"[('user_id', '=', user.id)]"
library_book_rule_company,library.book multi-company,library.book,,crud,"[('company_id', 'in', company_ids + [False])]"
```

| Rule | Detail |
|------|--------|
| File | `security/ir.access.csv`, header `id,name,model_id,group_id/id,operation,domain` |
| `model_id` | Model **technical name** (`library.book`), not the `model_library_book` XML id |
| `operation` | Required. Letters in `c`,`r`,`u`,`d` order: `r`, `ru`, `cru`, `crud`... (`ur` is invalid) |
| Row with group | *Permission*: rows are OR-ed; the domain limits only what that row grants |
| Row without group | *Restriction*: AND-ed onto **every** user; never grants anything |
| Default | Deny. A model without a permission row is inaccessible |
| Everyone | `base.group_everyone` (implied by internal, portal and public groups) |
| Domain context | `user`, `time`, `company_ids`, `company_id` |
| New operator | `('order_id', 'access', 'read')`: record accessible when the related record is |

## 2. ORM API Changes

| Removed / changed (v19) | Use in v20 |
|-------------------------|------------|
| `check_access_rights(op)` | `check_access(op)` / `has_access(op)` (on `self.browse()` for model level) |
| `check_access_rule(op)` | `check_access(op)` |
| `_filter_access_rules(op)`, `_filter_access_rules_python(op)` | `_filtered_access(op)` |
| `check_field_access_rights(op, fnames)` | `check_field_access(field, op)` / `has_field_access(field, op)`; list allowed fields with `fields_get()` |
| `_check_field_access(field, op)` | `check_field_access(field, op)` (old name deprecated) |
| `_check_recursion()` / `_check_m2m_recursion(f)` | `not self._has_cycle()` / `not self._has_cycle(f)` |
| `toggle_active()` | `action_archive()` / `action_unarchive()` |
| `copy_translations()` | Removed (translations are copied by `copy()`) |
| `read_group(domain, fields, groupby, ..., lazy)` | See [read_group](#3-read_group-new-signature) |
| `self.env.registry.clear_cache(name)` | `self.env.transaction.invalidate_ormcache(name)` |
| `self.env.registry.clear_all_caches()` | `self.env.transaction.invalidate_ormcache(...)` per cache name |
| `from odoo.tools import ormcache` | `from odoo import api` → `@api.ormcache(...)` (old import deprecated) |
| `tools.ormcache_context(...)` | `@api.ormcache('key', 'self.env.context.get("x")')` |
| `from odoo.osv import expression` | `from odoo.fields import Domain` (`&`, `\|`, `~`, `Domain.AND`, `Domain.OR`) |
| `from odoo.tools.query import Query` | `from odoo.models import Query, TableSQL` |
| `self.env.clear()` | `self.env.transaction.clear()` / `.reset()` (old deprecated) |
| `self.env.cache` | Field methods (old property deprecated) |
| `transaction.clear_access_cache()` | `transaction.invalidate_access_cache()` |
| `Query.join()` / `left_join()`, `add_where(str)`, `query.order = str` | `TableSQL._join()`, `add_join()`, `SQL` values only (old forms deprecated) |
| `_order_to_sql(order, query, alias)` | `_order_to_sql(table: TableSQL, order, reverse=False)` |
| `_read_group_groupby(alias, spec, query)` / `_read_group_select(spec, query)` | `_read_group_groupby(table, spec)` / `_read_group_select(table, spec)` |
| `SQL(...).code` / `.params` / `.to_flush` | `sql._sql_tuple` (internal; prefer passing `SQL` objects around) |
| `@api.deprecated("msg")` | `from odoo.tools.func import deprecated` → `@deprecated("msg")` |
| `odoo.api.Self` | `typing.Self` |

Still valid and unchanged: `@api.model_create_multi`, `@api.depends`, `@api.constrains`, `@api.onchange`,
`@api.ondelete`, `@api.private`, `@api.readonly`, `Command`, `models.Constraint`, `models.Index`,
`models.UniqueIndex`, `_search_display_name`, `name_search`, `search_fetch`, `_read_group`,
`self.env._(...)`, and raw `cr.execute("... %s", params)` (still accepted, but prefer `SQL(...)`).

## 3. `read_group` New Signature

The name survived, the contract did not. `read_group` is now a final, RPC-friendly wrapper of `_read_group`.

```python
# v19 (deprecated since 19.0, REMOVED in 20.0)
groups = Order.read_group([('state', '=', 'sale')], ['amount_total:sum'], ['partner_id'])
# -> [{'partner_id': (7, 'Azure'), 'amount_total': 1500.0, 'partner_id_count': 3, ...}]

# v20 backend code: _read_group returns recordsets and values
for partner, total in Order._read_group([('state', '=', 'sale')], ['partner_id'], ['amount_total:sum']):
    ...

# v20 read_group: same arguments as _read_group, ids instead of recordsets
rows = Order.read_group([('state', '=', 'sale')], ['partner_id'], ['amount_total:sum'])
# -> [(7, 1500.0), ...]

# Web-formatted dicts (what the old read_group returned) come from the web module
Order.formatted_read_group([('state', '=', 'sale')], ['partner_id'], ['amount_total:sum'])
```

> A v19 call `read_group(domain, ['amount:sum'], ['partner_id'])` still runs in v20 but interprets
> the field list as `groupby`: review every call.

## 4. Binary Fields and Attachments

```python
from odoo.tools import BinaryBytes

# v19
attachment = self.env['ir.attachment'].create({
    'name': 'report.pdf',
    'datas': base64.b64encode(pdf_bytes),   # v20: 'datas' is IGNORED (warning), attachment is empty
    'res_model': self._name,
    'res_id': self.id,
})
record.document = base64.b64encode(pdf_bytes)  # v20: TypeError (bytes are not accepted)

# v20
attachment = self.env['ir.attachment'].create({
    'name': 'report.pdf',
    'raw': pdf_bytes,                        # 'raw' accepts bytes or BinaryBytes
    'res_model': self._name,
    'res_id': self.id,
})
record.document = BinaryBytes(pdf_bytes, filename='report.pdf')
record.document = base64_str                  # a base64 *str* (RPC style) is still accepted

value = record.document                       # BinaryValue (EMPTY_BINARY when unset, falsy)
value.content      # bytes
value.to_base64()  # str
value.mimetype, value.size, value.filename, value.checksum
with value.open() as stream:
    ...
```

- `read()`/RPC return `{'content': <base64>, 'size': int, 'filename'?: str}` for binary fields.
- XML data: `<field name="image_1920" type="bytes" file="my_module/static/img/logo.png"/>`
  (`type="base64"` still works with a deprecation warning).
- `odoo.tools.image`: `base64_to_image` and `image_to_base64` are deprecated; use
  `binary_to_image` and `image_apply_opt`.

## 5. Time Zones: `pytz` → `zoneinfo`

`pytz` is no longer a core requirement and core code uses the stdlib (`zoneinfo`, `datetime.UTC`).

```python
from datetime import UTC
from zoneinfo import ZoneInfo

# v19
local_dt = pytz.timezone(self.env.user.tz or 'UTC').localize(naive_dt)
utc_dt = self.env.tz.localize(naive_dt).astimezone(pytz.utc)

# v20
tz = ZoneInfo(self.env.user.tz or 'UTC')
local_dt = naive_dt.replace(tzinfo=tz)
utc_naive = local_dt.astimezone(UTC).replace(tzinfo=None)   # Odoo stores naive UTC
user_now = fields.Datetime.now().replace(tzinfo=UTC).astimezone(self.env.tz)  # env.tz is a ZoneInfo
```

Server actions / automation code: `timezone` is now `zoneinfo.ZoneInfo` and `BinaryBytes` is available
in the evaluation context.

## 6. HTTP Controllers

```python
# v19
from odoo.http import request, route, Controller, content_disposition, Stream, SessionExpiredException

# v20
from odoo.http import Controller, Response, request, route
from odoo.http.stream import Stream, content_disposition, STATIC_CACHE, STATIC_CACHE_LONG
from odoo.http.session import SessionExpiredException, authenticate, logout
from odoo.http.dispatcher import serialize_exception
from odoo.http.router import root, dispatch_rpc
from odoo.http.requestlib import Request
```

| v19 | v20 |
|-----|-----|
| `request.session.authenticate(env, credential)` | `authenticate(request.session, request.env, credential)` |
| `request.session.logout()` | `logout(request.session)` |
| `Stream.from_binary_field(...)`, `Stream.from_attachment(...)` | `self.env['ir.binary']._record_to_stream(record, field)` / `_get_stream_from(...)` |
| `@route(auth='bearer')` | `@route(auth='bearer', bearer_scope='rpc')` (mandatory; the API key scope must match) |
| `@route(type='json')` | `@route(type='jsonrpc')` (`json` is a deprecated alias since 19.0) |

Route types: `'http'`, `'jsonrpc'`, `'json2'`. Re-decorate every overridden route with `@route()`.

## 7. Server-side QWeb (reports, mails, website)

```xml
<!-- v19 -->
<span t-esc="doc.name"/>
<t t-call="my_module.address_block">
    <t t-set="partner" t-value="doc.partner_id"/>
    <t t-set="title">Invoice address</t>
</t>

<!-- v20: t-esc/t-raw render NOTHING; t-set children are NOT passed to the callee -->
<span t-out="doc.name"/>
<t t-call="my_module.address_block" partner="doc.partner_id" title.translate="Invoice address"/>
```

| `t-call` attribute | Meaning |
|--------------------|---------|
| `name="expr"` | Python expression |
| `name.f="Text {{ expr }}"` | Format string (like `t-valuef`) |
| `name.translate="Text"` | Translatable literal |
| `t-args="dict_expr"` | Several values at once |
| Body of the `t-call` | Still rendered as the `0` slot (`t-out="0"` in the callee) |

HTML safety: `t-out` escapes unless the value is `Markup`. Build HTML with
`Markup("<b>{}</b>").format(value)`, never with f-strings inside `Markup`.

## 8. Views (XML)

| Topic | v20 behaviour |
|-------|---------------|
| New view type `card` | `<card><templates><t t-name="card">...</t><t t-name="menu">...</t></templates></card>` |
| Kanban reuse | `<kanban card_id="%(my_module.my_model_view_card)d">` inlines a card view |
| Calendar | `<popover card_id="..."><templates><t t-name="popover-footer">...</t></templates></popover>`, `schedule="1"`; `date_delay` removed |
| List | `<column name=".." string=".." width=".." column_invisible="..">` groups several fields in one column; `<list dialog_size="...">` |
| Buttons | `icon` = Material Symbols name (`icon="menu"`, `icon="edit_square"`); new `icon_class`, `confirm-title` |
| Search | `<filter date="field" end_date="field2"/>` (`end_date` requires `date`); nested filter groups with `string` |
| Icons in arch | `<i class="oi" data-icon="check" title="Done"/>`, accessibility checked on `data-icon` |
| Widgets removed | `remaining_days` → `relative_date`; `selection_badge` → `badges_selection` (selection) / `badges_many2one` (many2one) |
| Card widgets | Kanban-specific variants now use the `card.` prefix (`card.many2many_tags`, `card.many2one_avatar_user`) |
| Unchanged | `<list>`, `<chatter/>`, direct `invisible`/`readonly`/`required`/`column_invisible` expressions, `<t t-name="card">` in kanban |

## 9. Reports

- `ir.actions.report.report_file` no longer exists: remove `<field name="report_file">` from report actions.
- PDF engine is pluggable: `base_report_wkhtmltox` (auto-installed, wkhtmltopdf) or
  `base_report_paper_muncher`. `ir.actions.report.get_pdf_engine_state(engine_name)` reports its state.
- `t-esc` in report templates renders nothing: use `t-out` / `t-field`.
- `t-lang` only on a node that also has `t-call`.

## 10. Mail Tracking API

| v19 override | v20 override |
|--------------|--------------|
| `_track_subtype(self, init_values)` | `_track_log_get_default_subtype(self, track_init_values)` |
| `_track_template(self, changes)` | `_track_template_parameters(self, tracked_fields)` |
| `_track_get_default_log_message(...)` | `_track_log_get_default_body(...)` |
| `_track_set_author(author)` | `_track_set_log_author(author)` |

```python
def _track_log_get_default_subtype(self, track_init_values):
    self.ensure_one()
    if 'state' in track_init_values and self.state == 'done':
        return self.env.ref('my_module.mt_library_book_done')
    return super()._track_log_get_default_subtype(track_init_values)
```

`message_post(...)` keeps its keyword-only signature. Technical tracking views moved to the new
`mail_tracking` module.

## 11. Base Data Model Changes

| Model | v19 | v20 |
|-------|-----|-----|
| `res.partner` | `company_type` (Selection), `company_name`, `company_registry` | Removed. `is_company` is computed (stored) from `commercial_partner_id` and `has_vat`; new `has_vat`, `additional_identifiers` (Json), `address`, `address_inline`, `contact_address_inline` |
| `res.company` | `company_registry`, `layout_background*` | Removed; `additional_identifiers` (Json), `has_vat`, `report_tables_id` |
| `res.bank` | Model | **Removed** (bank data lives on `res.partner.bank`) |
| `res.partner.bank` | `acc_number`, `acc_holder_name`, `acc_type`, `sanitized_acc_number`, `bank_id`, `currency_id` | `account_number`, `holder_name`, `account_type`, `sanitized_account_number`, bank address fields (`street`, `city`, `zip`, `state_id`, `country_id`, `bank_name`), `clearing_label_id` |
| `res.currency` | `date` | `rate_date` |
| `res.country` | `zip_required` (Boolean) | `zip_applicability` (Selection) |
| `res.users` | `role`: user / admin | `role`: `light_user` / `regular_user` / `group_system`; new group `base.group_user_regular` |
| `res.groups` | `model_access`, `rule_groups` | `access_ids` (`ir.access`) |
| `website` | Defined by the `website` module | Base model in `base` (`env.website` from context `website_id`) |
| `ir.actions.*` | `views`, `params` (Binary) | Json |
| `ir.model.fields` | `index` (Boolean) | `index` (Selection of index types) |
| `ir.ui.view` | - | `technical_usage`, `notes`; view type `card` |
| `ir.actions.report` | `report_file` | Removed |
| `ir.attachment` | `datas` | Removed (use `raw`) |

Groups kept from v19: `res.users.group_ids`, `res.users.all_group_ids`, `res.groups.user_ids`,
`res.groups.privilege_id` (`res.groups.privilege`). `groups_id`/`users` were renamed in **19.0**.

## 12. Localization Impact (LATAM and others)

- `l10n_latam_base` (`l10n_latam.identification.type`) is **removed**. Identifiers are now metadata dicts:
  - Tax IDs (`TIN_METADATA` in `odoo/tools/partner_identifiers.py`, e.g. `VE_RIF`, `PA_RUC`, `PE_RUC`)
    describe the `vat` field per country (label, placeholder, validation).
  - Other identifiers (citizen/enterprise numbers, categories `CN`/`EN`) live in
    `res.partner.additional_identifiers` (Json dict `{key: value}`) and are declared by overriding
    `res.partner._get_all_additional_identifiers_metadata()`.
- `base_vat` and `base_iban` are merged into `base` (validation helpers in
  `odoo/tools/partner_identifier_validation.py` and `odoo/tools/bank_account_number.py`). Remove them from
  `depends`.
- Keep the `l10n_{country}_` prefix rule for localization fields and methods.

Declaring localization identifiers (core examples: `l10n_co`, `l10n_uy`, `l10n_do`, `l10n_es`):

```python
# l10n_xx/tools/partner_identifiers.py
from odoo.tools.translate import LazyTranslate

_lt = LazyTranslate(__name__)

XX_ADDITIONAL_IDENTIFIERS_METADATA = {
    'XX_CI': {
        'label': _lt('Cedula de identidad'),
        'help': _lt('National identity card number.'),
        'category': 'CN',          # CN = natural person, EN = legal entity
        'countries': ['XX'],
    },
}

# l10n_xx/models/res_partner.py
from odoo import api, models
from odoo.addons.l10n_xx.tools.partner_identifiers import XX_ADDITIONAL_IDENTIFIERS_METADATA


class ResPartner(models.Model):
    _inherit = 'res.partner'

    @api.model
    def _get_all_additional_identifiers_metadata(self):
        return {**super()._get_all_additional_identifiers_metadata(), **XX_ADDITIONAL_IDENTIFIERS_METADATA}
```

## 13. Frontend Summary (details: `odoo-owl-components-20.md`)

| v19 (OWL 2.8) | v20 (OWL 3) |
|---------------|-------------|
| `static props = {...}` / `static defaultProps` | `props = useProps({ name: t.string().optional("x") })` (static props **throw**) |
| `useState({...})` / `reactive()` | `proxy({...})`, `signal(value)`, `computed(fn)` |
| `useRef("input")` + `t-ref="input"` | `input = signal.ref()` + `t-ref="this.input"`, read with `this.input()` |
| `useExternalListener(target, ev, fn)` | `useListener(target, ev, fn)` |
| `useEffect(fn, () => deps)` | `useLayoutEffect(fn, () => deps)` from `@web/owl2/utils`, or OWL 3 `useEffect(fn)` (auto-tracked) / `useOnChange(deps, cb)` |
| `props.x`, `state.y` in templates | `this.props.x`, `this.state.y` |
| `t-esc` | `t-out` |
| `t-slot="default"` | `t-call-slot="default"` |
| `useService("notification")` | still works; plugins available: `usePlugin(NotificationPlugin)` |
| `@web/core/orm_service` | `@web/core/orm_plugin` |
| `@web/webclient/actions/action_service` (`standardActionServiceProps`) | `@web/webclient/actions/action_plugin` |
| jQuery, `publicWidget` | Removed; use `Interaction` (`@web/public/interaction`) |
| QUnit | Hoot (`@odoo/hoot`) |
| `<i class="fa fa-check"/>` | `<i class="oi" data-icon="check"/>` |

## 14. SCSS and Theming

- Removed: `fontawesome_overridden.scss`, `$o-touch-btn-padding`, `$o-touch-btn-with-icon-padding`,
  the custom `.user-select-none` rule.
- `$o-colors` moved to `primary_variables.scss`; new `$o-colors-border`, `$o-colors-bg-subtle`,
  `$o-colors-text-emphasis`, `$o-colors-ui` (merged into `$theme-colors` as `color-1`...`color-12`).
- New: `$o-enable-backdrop-blur`, `$o-gradient*`, `$o-border-color`, `$o-modal-sm`,
  `$o-modal-animation-duration`, `$o-border-radius-xl`, `$o-form-check-input-border-color`, `disabled-*`
  keys in button maps, Bootstrap focus-ring variables.
- Changed defaults: `$o-modal-md` 650px → 620px, `$o-breadcrumb-item-padding-x` .5rem → .25rem,
  modal backdrop opacity .25, popover/modal radius `$border-radius-lg`.
- Icons: `.oi` + `data-icon="<material_symbol>"`, custom Odoo/brand icons `data-icon="oi_<name>"`,
  helpers `oi-fw`, `oi-lg`, `oi-2x`...`oi-10x`, `oi-spin`, `oi-pulse`, `oi-filled`, `oi-rotate-90`.
  Only the icons listed in `addons/web/tooling/icons/icons_wishlist.txt` are in the shipped subset.
- New bundles: `web.icons_fonts`, `web.material_symbols_{outlined,rounded,sharp}`, `web.odoo_ui_icons`.
  Removed bundles: `web._assets_jquery`, `web.qunit_suite_tests`.

## 15. Caching

```python
from odoo import api, models

class LibraryConfig(models.Model):
    _name = 'library.config'

    @api.ormcache('key', cache='stable')      # caches: default, stable, assets, templates, routing, groups
    def _get_value(self, key):
        return self.sudo().search([('key', '=', key)], limit=1).value

    def write(self, vals):
        res = super().write(vals)
        self.env.transaction.invalidate_ormcache('stable')
        return res
```

- `models.CachedModel` (Python-inherited mixin, not an ORM model): set `_cached_data_domain` and
  `_cached_data_fields` to serve those fields from the `'stable'` cache; `get_all()` returns all cached records.
- Never return recordsets from an `ormcache` method.

## 16. New ORM Capabilities

| Feature | Example |
|---------|---------|
| `compute_sql` on non-stored computed fields (search/group/order in SQL) | `kind = fields.Selection(..., compute='_compute_kind', compute_sql='_compute_sql_kind', compute_sudo=True)` with `def _compute_sql_kind(self, table): return SQL(...)` |
| `init_storage` (initialize a new column for existing rows) | `fields.Char(init_storage='_init_code')` |
| `copy` accepts a callable | `fields.Char(copy=lambda rec: rec.code and f"{rec.code}-COPY")` |
| `TableSQL` field access | `query = self._search(domain)`; `self.env.execute_query(query.select(SQL("%s, %s", query.table.id, query.table.name)))` |
| `get_public_method(model, name)` | RPC guard: `_`-prefixed, class/static methods and `@api.private` are not callable remotely |
| `env.website` | Current website from context `website_id` |
| `_explanation` model attribute | Verbose purpose of the model (`_explanation = "..."`), exposed as `ir.model.explanation` |
| Domain dynamic dates (since 18.5) | `[('date_deadline', '<', 'today')]` in view/filter domains |

## 17. Deprecated in 20.0 (still working, plan the change)

| Deprecated | Replacement |
|------------|-------------|
| `type="base64"` in XML data | `type="bytes"` |
| `from odoo.tools import ormcache` | `from odoo import api` (`@api.ormcache`) |
| `_check_field_access`, `_check_access` | `check_field_access`, `_access_domain` |
| `env.clear()`, `env.cache` | `env.transaction.clear()` / `reset()`, field methods |
| `transaction.clear_access_cache()` | `invalidate_access_cache()` |
| `Query.join()` / `left_join()`, `add_where(str)`, `order = str`, `select(str)` | `TableSQL._join()`, `add_join()`, `SQL` |
| `tools.sql.escape_psql` | `escape_like_value` |
| `tools.sql.fix_foreign_key`, `check_index_exist`, `reverse_order` | Removed after 20.0 |
| `ReadonlyDict` | `frozendict` |
| `image.base64_to_image`, `image.image_to_base64` | `binary_to_image`, `image_apply_opt` |
| Comparing booleans with non-booleans in domains (`('active', '=', 'True')`) | Use real booleans |
| `@route(type='json')` | `type='jsonrpc'` |
| `ir.attachment.check(mode)` | `check_access(mode)` |
| `ir.cron._notify_progress()` | `_commit_progress()` |
| `self._cr`, `self._uid`, `self._context`, `request.cr/uid/context` | `self.env.cr`, `self.env.uid`, `self.env.context` |
| Constraint attribute names ending in `_not_null` | Rename (clashes with PostgreSQL 18 NOT NULL names) |
| `--syslog` option | `--log-config` with a syslog handler |
| `convert_file(..., kind=...)`, `upstream_dependencies(known_deps=...)` | Drop the argument |

## Tooling: `odoo-bin upgrade_code`

```bash
# Dry run of all 19.x → 20.0 rewrites on your addons
./odoo-bin upgrade_code --from 19.0 --addons-path=/path/to/custom_addons --dry-run
# Apply them
./odoo-bin upgrade_code --from 19.0 --addons-path=/path/to/custom_addons
# OWL 2 → OWL 3 rewrite (no version prefix, run it explicitly)
./odoo-bin upgrade_code --script owl3-migration --addons-path=/path/to/custom_addons
# Restrict to one module
./odoo-bin upgrade_code --from 19.0 --addons-path=/path/to/custom_addons --glob '**/my_module/**'
```

| Script | Rewrite |
|--------|---------|
| `19.1-00-t-call` | Server QWeb: `t-set` children of `t-call` → `t-call` attributes |
| `19.3-00-account-groups`, `19.3-00-account-report-foldable` | Accounting data/report adaptations |
| `19.3-00-base64-in-xml` | `type="base64"` → `type="bytes"` for `file=` fields |
| `19.4-00-ir-access` | `ir.model.access.csv` + `ir.rule` records → `security/ir.access.csv` (logs cases to review) |
| `19.4-00-ormcache-on-transaction` | `registry.clear_cache` → `transaction.invalidate_ormcache` |
| `19.5-00-tuple-rec_names_search` | `_rec_names_search = [...]` → tuple |
| `owl3-migration` | useState/reactive → proxy, useRef → t-ref signals, useEffect → useLayoutEffect, t-esc → t-out, t-slot → t-call-slot, `this.` in templates, t-call params, useService → usePlugin (mapped services) |

The scripts are best effort: always review the diff, and check the `WARNING`/`ERROR` logs of `19.4-00-ir-access`.

Other CLI changes: the old `odoo-bin populate` became `odoo-bin duplicate` (copies existing records);
the new `populate` addon provides blueprint-based data generation (`populate/*.xml`,
`populate.blueprint` records).

## Modules Added / Removed (highlights)

| Community | |
|-----------|--|
| Merged (depend on the target instead) | `base_vat`, `base_iban` → `base`; `stock_picking_batch` → `stock`; `hr_org_chart` → `hr`; `website_sale_wishlist`, `website_sale_comparison*` → `website_sale` |
| Removed | `l10n_latam_base`, `hr_homeworking*`, `hr_hourly_cost`, `delivery_mondialrelay`, `transifex`, `iot_base` |
| Added | `base_report_wkhtmltox`, `base_report_paper_muncher`, `populate`, `printer`, `mail_tracking`, `purchase_alternative*`, `pos_stock`, `portal_discuss`, `fleet_maintenance`, `l10n_eu_account_vies` |

| Enterprise | |
|------------|--|
| Replaced | `industry_fsm*` (Field Service on project tasks) → `planning_field_service*` (Field Service on Planning) |
| Removed | `hr_work_entry_enterprise` and the `hr_work_entry_*` bridges, `helpdesk_fsm*`, several Belgian payroll connectors |
| Added | `ai_*` family (`ai_agentic`, `ai_mcp`, ...), `obox*`, `pricer`, `stock_pricer`, `maintenance_enterprise` |

Always check a dependency still exists in 20.0 before listing it in `depends`.

## Manifest (v20)

```python
{
    'name': 'Library',
    'version': '20.0.1.0.0',
    'category': 'Services',
    'summary': 'Manage library books',
    'author': 'Your Company',
    'website': 'https://example.com',
    'license': 'LGPL-3',
    'depends': ['mail'],
    'data': [
        'security/library_security.xml',
        'security/ir.access.csv',
        'views/library_book_views.xml',
        'views/library_menus.xml',
    ],
    'assets': {
        'web.assets_backend': [
            'library/static/src/**/*',
        ],
    },
    'installable': True,
    'application': True,
}
```

- Removed legacy keys: `demo_xml`, `init_xml`, `update_xml`, `images_preview_theme`. New keys:
  `other_files`, `iap_paid_service`.
- `author` and `license` are mandatory (warning and default when missing).
- Odoo's house rule: do not list `base` in `depends` (it is injected). OCA projects may keep it explicit.

## Version Detection Indicators

```python
# v20 indicators (Python / data)
'security/ir.access.csv' in manifest['data']
from odoo.tools import BinaryBytes
from zoneinfo import ZoneInfo
from odoo.http.stream import content_disposition
@api.ormcache(...)
self.env.transaction.invalidate_ormcache(...)
def _track_log_get_default_subtype(self, track_init_values):
```

```javascript
// v20 indicators (JS)
import { Component, proxy, signal, t, useProps } from "@odoo/owl";
props = useProps({ ... });
import { useLayoutEffect } from "@web/owl2/utils";
```

```xml
<!-- v20 indicators (XML) -->
<i class="oi" data-icon="check"/>
<card>...</card>
<kanban card_id="%(my_view_card)d"/>
<t t-call="tmpl" title.f="..."/>
```

## Common Errors in v20

| Error | Cause | Fix |
|-------|-------|-----|
| `KeyError: 'ir.model.access'` during install | Module ships `ir.model.access.csv` | Convert to `security/ir.access.csv` |
| `ImportError: cannot import name 'content_disposition' from 'odoo.http'` | Helper moved | `from odoo.http.stream import content_disposition` |
| `AssertionError: bearer_scope must be set for auth='bearer'` | Route without scope | Add `bearer_scope='...'` |
| `TypeError: ... use BinaryValue instead of bytes` | Writing raw bytes to a Binary field | `BinaryBytes(data)` or base64 `str` |
| Warning `Use raw, datas has beeen removed` | `ir.attachment` created with `datas` | Use `raw` |
| `ValueError: Invalid field 'report_file' in 'ir.actions.report'` | Old report action | Remove the field |
| `AttributeError: 'zoneinfo.ZoneInfo' object has no attribute 'localize'` | pytz API on `env.tz` | `dt.replace(tzinfo=tz)` / `astimezone` |
| `ModuleNotFoundError: No module named 'odoo.osv'` | `expression` helpers | `odoo.fields.Domain` |
| `AttributeError: 'Registry' object has no attribute 'clear_cache'` | Old cache API | `env.transaction.invalidate_ormcache()` |
| `Error: Component "X" defines a static "props" or "defaultProps"...` | OWL 2 props | `props = useProps({...})` |
| `TypeError: useState is not a function` | OWL 2 hook | `proxy({...})` |
| Blank value in a PDF/mail/website template | `t-esc` / `t-raw` server-side | `t-out` |
| Empty icon | FA class or `icon="fa-..."` | Material Symbols name in `data-icon` / `icon` |

## Best Practices for v20

1. Start from the official `<odoo_src>/skills/` guidelines for house rules.
2. Security in `security/ir.access.csv`; multi-company via a groupless restriction row with `company_ids`.
3. Methods private by default (`_` prefix); `@api.private` for public names that must not be RPC-callable.
4. Build domains with `odoo.fields.Domain`; never concatenate user-provided domains.
5. Raw SQL only when the ORM cannot do it; use `SQL`/`SQL.identifier`, or `_search()` + `Query.select(SQL(...))`.
6. Binary data as `BinaryBytes`/`raw`; time zones with `zoneinfo`.
7. `self.env._("Text %s", value)` for translations (static literals only).
8. Server QWeb: `t-out` only; `t-call` parameters as attributes.
9. OWL 3: `useProps` + `t`, `proxy`/`signal`, `this.` in templates, `t-out`, `t-call-slot`.
10. Type hints are optional (not enforced by the framework); use them on public APIs when useful.

## AI Agent Instructions (v20)

1. **CONFIRM** the target is 20.0 (`version` starts with `20.0.`) and read the source when unsure: `git grep` the API in the 20.0 tree before using it.
2. **NEVER** generate `ir.model.access.csv`, `ir.rule` records, `static props`, `useState`, `useRef`, `t-esc`, `datas`, `report_file`, `pytz`, `fa fa-*`.
3. **USE** `ir.access.csv`, `useProps`/`proxy`/`signal`, `t-out`, `raw`/`BinaryBytes`, `zoneinfo`, Material Symbols.
4. **CHECK** merged/removed modules (`base_vat`, `base_iban`, `stock_picking_batch`, `hr_org_chart`, `l10n_latam_base`...) before adding a dependency.
5. **RUN** `odoo-bin upgrade_code --from 19.0` (and `--script owl3-migration`) when migrating, then review manually.
6. **READ** the version-specific files: `odoo-model-patterns-20.md`, `odoo-owl-components-20.md`, `odoo-security-guide-20.md`, `odoo-module-generator-20.md`.
