# Odoo Version Knowledge: 19 to 20 Migration

```
╔══════════════════════════════════════════════════════════════════════════════╗
║  VERSION MIGRATION: 19.0 → 20.0                                              ║
║  Breaking changes, detection commands and migration procedure                ║
║  Verified by diffing the 19.0 and 20.0 source trees                          ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

## Real 19.0 Baseline

Compare against what 19.0 really ships (some older notes in this skill describe 19.0 differently):

| Topic | 19.0 (source) | 20.0 (source) |
|-------|---------------|---------------|
| Python / PostgreSQL | 3.10 - 3.14 / 13+ | 3.12 - 3.14 / 16+ |
| Owl | 2.8.x | 3.0 (alpha) + Owl 2 compatibility layer |
| Type hints | Optional | Optional |
| Raw `cr.execute(str, params)` | Allowed (`SQL` recommended) | Allowed (`SQL` recommended) |
| `_sql_constraints` | Ignored with a warning (use `models.Constraint`) | Same |
| Users / groups fields | `group_ids`, `all_group_ids`, `res.groups.user_ids`, `privilege_id` | Same |
| Record rule variables | `user`, `company_ids`, `company_id` | Same, in `ir.access` domains |
| ACL / rules | `ir.model.access` + `ir.rule` | `ir.access` |

## Breaking Changes Summary

| Category | Change | Impact | Auto-fix |
|----------|--------|--------|----------|
| Security | `ir.model.access` + `ir.rule` → `ir.access` | **CRITICAL** (install fails) | `upgrade_code --script 19.4-00-ir-access` |
| JS | Owl 2 → Owl 3 | **CRITICAL** (components crash) | `upgrade_code --script owl3-migration` (partial) |
| Server QWeb | `t-esc`/`t-raw` not compiled; `t-set` in `t-call` not passed | **CRITICAL** (silent blanks) | `19.1-00-t-call` (t-call only); `t-esc` by hand |
| Attachments | `ir.attachment.datas` removed (ignored) | **CRITICAL** (silent data loss) | Manual |
| Binary fields | `BinaryValue`; raw `bytes` rejected | High | Manual |
| ORM | `read_group` new signature | High | Manual |
| ORM | Deprecated v18/v19 aliases removed | High | Manual |
| HTTP | `odoo.http` helpers moved to submodules; `bearer_scope` mandatory | High | Manual |
| Time zones | `pytz` → `zoneinfo`, `env.tz` is `ZoneInfo` | High | Manual |
| Reports | `report_file` removed | High (install fails) | Manual |
| Views | FA icons not rendered; `remaining_days`, `selection_badge` removed | Medium | Manual |
| Mail | Tracking hooks renamed | Medium (silent) | Manual |
| Cache | `registry.clear_cache` removed; `ormcache` from `odoo.api` | Medium | `19.4-00-ormcache-on-transaction` |
| Data files | `type="base64"` → `type="bytes"` | Low (deprecation) | `19.3-00-base64-in-xml` |
| Tests | `SingleTransactionCase` removed | Medium | Manual |
| Base data | `res.bank`, `res.partner.bank`, `res.partner`, `res.company`, `res.currency`, `res.country` fields | Medium | Manual |
| Modules | `base_vat`/`base_iban` → `base`, `stock_picking_batch` → `stock`, `hr_org_chart` → `hr` (merged); `l10n_latam_base` removed | High (dependency missing) | Manual |

## Migration Procedure

```bash
# 0. Work on a branch of your addons repository
git switch -c 20.0-mig-library

# 1. Detect v19 patterns (see "Detection commands" below) and keep the report

# 2. Run the official rewrites (dry run first, then apply)
./odoo-bin upgrade_code --from 19.0 --addons-path=/path/to/addons --dry-run
./odoo-bin upgrade_code --from 19.0 --addons-path=/path/to/addons
./odoo-bin upgrade_code --script owl3-migration --addons-path=/path/to/addons

# 3. Review the diff, fix what the scripts cannot (list below), bump versions to 20.0.x.y.z

# 4. Install/upgrade on a 20.0 database and run tests
./odoo-bin -d mig20 -i library --test-enable --stop-after-init
```

## Detection Commands

```bash
# Security files that no longer load
grep -rln "ir.model.access.csv" --include=__manifest__.py .
grep -rln 'model="ir.rule"\|model="ir.model.access"' --include=*.xml .

# Removed / changed Python APIs
grep -rnE "check_access_rights|check_access_rule|_filter_access_rules|check_field_access_rights|toggle_active|_check_recursion|_check_m2m_recursion|copy_translations" --include=*.py .
grep -rnE "\.read_group\(|registry\.clear_cache|clear_all_caches|ormcache_context|from odoo\.osv|odoo\.tools\.query|pycompat" --include=*.py .
grep -rnE "import pytz|from pytz|\.localize\(" --include=*.py .
grep -rnE "'datas'|\.datas\b|base64\.b64encode" --include=*.py .
grep -rnE "from odoo\.http import .*(content_disposition|Stream|root|SessionExpiredException|serialize_exception|dispatch_rpc|Request)" --include=*.py .
grep -rn "auth='bearer'" --include=*.py .
grep -rnE "_track_subtype|_track_template\b|SingleTransactionCase" --include=*.py .

# XML / QWeb
grep -rnE 't-esc=|t-raw=' --include=*.xml .
grep -rn 'name="report_file"' --include=*.xml .
grep -rnE 'icon="fa-|class="fa fa-|widget="(remaining_days|selection_badge)"' --include=*.xml .
grep -rn 'type="base64"' --include=*.xml .

# JavaScript (Owl 2)
grep -rnE "static (props|defaultProps)|useState|useRef|useExternalListener|useChildSubEnv|reactive\(|onRendered" --include=*.js static/
grep -rnE 'publicWidget|\$\(|jQuery|QUnit' --include=*.js static/
grep -rnE "orm_service|actions/action_service" --include=*.js static/

# Data model renames
grep -rnE "acc_number|acc_holder_name|bank_id|company_registry|company_type|zip_required|res\.bank\b" --include=*.py --include=*.xml .
grep -rnE "'(base_vat|base_iban|l10n_latam_base|stock_picking_batch|hr_org_chart|website_sale_wishlist|website_sale_comparison)'" --include=__manifest__.py .
```

## 1. Security: ACL + Rules → `ir.access`

```csv
# v19: security/ir.model.access.csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_library_book_user,library.book user,model_library_book,library.group_library_user,1,1,0,0
access_library_book_manager,library.book manager,model_library_book,library.group_library_manager,1,1,1,1
```

```xml
<!-- v19: security/library_rules.xml -->
<record id="library_book_rule_own" model="ir.rule">
    <field name="name">library.book own</field>
    <field name="model_id" ref="model_library_book"/>
    <field name="domain_force">[('user_id', '=', user.id)]</field>
    <field name="groups" eval="[Command.link(ref('library.group_library_user'))]"/>
    <field name="perm_create" eval="False"/>
    <field name="perm_unlink" eval="False"/>
</record>
<record id="library_book_rule_all" model="ir.rule">
    <field name="name">library.book all</field>
    <field name="model_id" ref="model_library_book"/>
    <field name="domain_force">[(1, '=', 1)]</field>
    <field name="groups" eval="[Command.link(ref('library.group_library_manager'))]"/>
</record>
<record id="library_book_rule_company" model="ir.rule">
    <field name="name">library.book multi-company</field>
    <field name="model_id" ref="model_library_book"/>
    <field name="domain_force">[('company_id', 'in', company_ids)]</field>
</record>
```

```csv
# v20: security/ir.access.csv
id,name,model_id,group_id/id,operation,domain
access_library_book_user,library.book user,library.book,library.group_library_user,ru,"[('user_id', '=', user.id)]"
access_library_book_manager,library.book manager,library.book,library.group_library_manager,crud,
library_book_rule_company,library.book multi-company,library.book,,crud,"[('company_id', 'in', company_ids)]"
```

Full conversion rules: `odoo-security-guide-19-20.md`.

## 2. Python / ORM

| v19 | v20 |
|-----|-----|
| `records.check_access_rights('write')` | `records.browse().check_access('write')` |
| `records.check_access_rule('write')` | `records.check_access('write')` |
| `records._filter_access_rules('read')` | `records._filtered_access('read')` |
| `Model.check_field_access_rights('read', fnames)` | `record.check_field_access(field, 'read')` / `fields_get()` |
| `records._check_recursion()` | `not records._has_cycle()` |
| `records.toggle_active()` | `action_archive()` / `action_unarchive()` |
| `Model.read_group(domain, ['amount:sum'], ['partner_id'])` | `_read_group(domain, ['partner_id'], ['amount:sum'])` or `formatted_read_group(...)` |
| `self.env.registry.clear_cache()` | `self.env.transaction.invalidate_ormcache()` |
| `from odoo.tools import ormcache` | `from odoo import api` + `@api.ormcache` |
| `from odoo.osv import expression` + `expression.AND([...])` | `from odoo.fields import Domain` + `Domain.AND([...])` / `&` |
| `from odoo.tools.query import Query` | `from odoo.models import Query` |
| `pytz.timezone(tz).localize(dt)` | `dt.replace(tzinfo=ZoneInfo(tz))` |
| `record.bin_field = base64.b64encode(data)` | `record.bin_field = BinaryBytes(data)` |
| `attachment.create({'datas': b64})` | `attachment.create({'raw': data})` |
| `def _track_subtype(self, init_values)` | `def _track_log_get_default_subtype(self, track_init_values)` |
| `class T(SingleTransactionCase)` | `TransactionCase` / `BaseCommon` |

Details: `odoo-model-patterns-19-20.md`.

## 3. HTTP

```python
# v19
from odoo.http import Controller, request, route, content_disposition, Stream

@route('/api/books', type='json2', auth='bearer')
def api_books(self): ...

# v20
from odoo.http import Controller, request, route
from odoo.http.stream import Stream, content_disposition

@route('/api/books', type='json2', auth='bearer', bearer_scope='rpc')  # API key scope must match
def api_books(self): ...
```

`request.session.authenticate(env, credential)` → `odoo.http.session.authenticate(request.session, env, credential)`;
`request.session.logout()` → `odoo.http.session.logout(request.session)`.

## 4. Server QWeb and Reports

```xml
<!-- v19 -->
<record id="report_book" model="ir.actions.report">
    <field name="report_file">library.report_book</field>   <!-- remove in v20 -->
</record>
<t t-call="library.book_header">
    <t t-set="title">Book sheet</t>
    <t t-set="book" t-value="doc"/>
</t>
<span t-esc="doc.name"/>

<!-- v20 -->
<t t-call="library.book_header" title.translate="Book sheet" book="doc"/>
<span t-out="doc.name"/>
```

## 5. Views

```xml
<!-- v19 -->
<button name="action_open" type="object" icon="fa-bars" string="Open"/>
<field name="date_due" widget="remaining_days"/>
<field name="kind" widget="selection_badge"/>
<i class="fa fa-check"/>

<!-- v20 -->
<button name="action_open" type="object" icon="menu" string="Open"/>
<field name="date_due" widget="relative_date"/>
<field name="kind" widget="badges_selection"/>
<i class="oi" data-icon="check" title="Done"/>
```

Common FontAwesome → Material Symbols names used by core: `fa-bars` → `menu`, `fa-trash` → `delete`,
`fa-times` → `close`, `fa-check` → `check`, `fa-plus` → `add`, `fa-search` → `search`,
`fa-pencil`/`fa-edit` → `edit` / `edit_square`, `fa-arrow-right` → `arrow_forward`,
`fa-arrow-left` → `arrow_back`, `fa-chevron-right` → `chevron_forward`, `fa-download` → `download`,
`fa-warning` → `warning`, `fa-refresh` → `refresh` / `autorenew`, `fa-clock-o` → `schedule`,
`fa-money` → `payments`, `fa-shopping-cart` → `shopping_cart`, `fa-cog` → `settings`, `fa-envelope` → `mail`,
`fa-phone` → `phone`, `fa-user` → `person`, `fa-info-circle` → `info`, `fa-print` → `print`.
Brands and FA leftovers: `data-icon="oi_facebook"`, `oi_linkedin`, `oi_x`, `oi_github`, `oi_whatsapp`...
Only icons listed in `addons/web/tooling/icons/icons_wishlist.txt` exist in the shipped font subset.

## 6. JavaScript (Owl 2 → Owl 3)

```javascript
// v19
import { Component, useState, useRef } from "@odoo/owl";
export class BookCard extends Component {
    static template = "library.BookCard";
    static props = { book: Object, onOpen: { type: Function, optional: true } };
    setup() {
        this.state = useState({ open: false });
        this.titleRef = useRef("title");
    }
}
// template: <h3 t-ref="title" t-esc="props.book.name" t-on-click="toggle"/>

// v20
import { Component, proxy, signal, t, useProps } from "@odoo/owl";
export class BookCard extends Component {
    static template = "library.BookCard";
    props = useProps({ book: t.object(), onOpen: t.function().optional() });
    state = proxy({ open: false });
    titleRef = signal.ref();
}
// template: <h3 t-ref="this.titleRef" t-out="this.props.book.name" t-on-click="this.toggle"/>
```

Details: `odoo-owl-components-19-20.md`.

## 7. Base Data and Localization

| v19 | v20 |
|-----|-----|
| `res.partner.company_type == 'company'` | `res.partner.is_company` (computed from `commercial_partner_id` + `has_vat`) |
| `res.partner.company_registry` / `res.company.company_registry` | `additional_identifiers` (Json) |
| `res.bank` / `res.partner.bank.bank_id` | Removed; bank data on `res.partner.bank` (`bank_name`, address fields) |
| `res.partner.bank.acc_number` / `acc_holder_name` / `acc_type` | `account_number` / `holder_name` / `account_type` |
| `res.currency.date` | `rate_date` |
| `res.country.zip_required` | `zip_applicability` |
| `l10n_latam_base` identification types | Tax ID metadata in `odoo/tools/partner_identifiers.py` (`VE_RIF`, `PA_RUC`, ... for `vat`) + other identifiers in `additional_identifiers`, declared through `res.partner._get_all_additional_identifiers_metadata()` |
| `depends: ['base_vat']` / `['base_iban']` | Remove (merged into `base`) |

## Migration Checklist

- [ ] `upgrade_code --from 19.0` and `--script owl3-migration` executed and reviewed
- [ ] `security/ir.access.csv` replaces ACL CSV and `ir.rule` XML; warnings of the script handled
- [ ] No `t-esc`/`t-raw` in server templates; `t-call` parameters as attributes
- [ ] No `report_file`; report templates render
- [ ] `datas` → `raw`; binary writes use `BinaryBytes`
- [ ] `read_group` calls reviewed; removed ORM aliases replaced
- [ ] `pytz` removed; `env.tz` used as `ZoneInfo`
- [ ] `odoo.http` imports fixed; bearer routes have `bearer_scope`
- [ ] Icons converted to Material Symbols; removed widgets replaced
- [ ] Owl 3 components (no `static props`, no `useState`, `this.` in templates)
- [ ] Mail tracking overrides renamed
- [ ] Dependencies exist in 20.0; data-model renames applied
- [ ] Manifest `version` = `20.0.x.y.z`; tests green on 20.0

## GitHub Verification URLs

```
https://github.com/odoo/odoo/tree/20.0/odoo/upgrade_code
https://github.com/odoo/odoo/blob/20.0/odoo/addons/base/models/ir_access.py
https://github.com/odoo/odoo/tree/20.0/addons/web/static/src/owl2
https://github.com/odoo/odoo/tree/20.0/skills
```
