# Odoo Module Migration Guide: 19.0 → 20.0

```
╔══════════════════════════════════════════════════════════════════════════════╗
║  MIGRATION GUIDE: Odoo 19.0 → 20.0                                           ║
║  File-by-file changes for a custom module                                    ║
║  Automate first (odoo-bin upgrade_code), then fix by hand                    ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

## Breaking Changes Summary

| Component | v19 | v20 | Action |
|-----------|-----|-----|--------|
| `__manifest__.py` | `19.0.x.y.z`, `ir.model.access.csv` | `20.0.x.y.z`, `ir.access.csv` | Bump, replace file |
| `security/` | ACL CSV + `ir.rule` XML | `security/ir.access.csv` | Convert (script) |
| `models/` | Removed aliases, `read_group`, `pytz`, base64 binaries | See model guide | Rewrite |
| `views/` | `icon="fa-*"`, `fa fa-*`, `remaining_days`, `selection_badge` | Material Symbols, `relative_date`, `badges_selection` | Rewrite |
| `report/` | `report_file`, `t-esc`, `t-set` inside `t-call` | No `report_file`, `t-out`, `t-call` attributes | Rewrite |
| `data/` | `type="base64" file=...` | `type="bytes"` | Script |
| `controllers/` | `from odoo.http import content_disposition, ...` | Submodule imports, `bearer_scope` | Rewrite |
| `static/src` | Owl 2, `publicWidget`, jQuery | Owl 3, `Interaction` | Script + rewrite |
| `static/tests` | QUnit leftovers | Hoot | Rewrite |
| `tests/` | `SingleTransactionCase` | `TransactionCase` / `BaseCommon` | Rewrite |
| `depends` | `base_vat`, `base_iban`, `stock_picking_batch`, `hr_org_chart`, `l10n_latam_base`... | Merged into `base`/`stock`/`hr` or removed | Update |

## Step 1: Run the Official Rewrites

```bash
./odoo-bin upgrade_code --from 19.0 --addons-path=/path/to/addons --glob '**/library/**' --dry-run
./odoo-bin upgrade_code --from 19.0 --addons-path=/path/to/addons --glob '**/library/**'
./odoo-bin upgrade_code --script owl3-migration --addons-path=/path/to/addons --glob '**/library/**'
git diff --stat
```

The `19.4-00-ir-access` step prints `INFO`/`WARNING`/`ERROR` lines: every `WARNING` or `ERROR`
needs a manual decision (orphan rules, ACLs interacting with rules of other groups, modified records of other modules).

## Step 2: Manifest

```python
# v19
{
    'name': 'Library',
    'version': '19.0.1.2.0',
    'depends': ['base', 'mail', 'base_vat'],
    'data': [
        'security/library_security.xml',
        'security/ir.model.access.csv',
        'security/library_rules.xml',
        'views/library_book_views.xml',
    ],
}

# v20
{
    'name': 'Library',
    'version': '20.0.1.0.0',
    'author': 'Your Company',
    'license': 'LGPL-3',
    'depends': ['mail'],          # base_vat merged into base
    'data': [
        'security/library_security.xml',
        'security/ir.access.csv',
        'views/library_book_views.xml',
    ],
}
```

## Step 3: Security

See `odoo-security-guide-19-20.md` for the conversion table. Delete `ir.model.access.csv` and the `ir.rule`
records once `ir.access.csv` is correct; review that every model (wizards included) still has a permission row.

## Step 4: Models

| Search for | Replace with |
|------------|--------------|
| `check_access_rights(` / `check_access_rule(` | `check_access(` / `has_access(` |
| `_filter_access_rules(` | `_filtered_access(` |
| `_check_recursion(` | `not ..._has_cycle(` |
| `toggle_active(` | `action_archive()` / `action_unarchive()` |
| `.read_group(` | `_read_group(domain, groupby, aggregates)` or `formatted_read_group(...)` |
| `registry.clear_cache(` | `transaction.invalidate_ormcache(` |
| `from odoo.osv import expression` | `from odoo.fields import Domain` |
| `import pytz` | `from zoneinfo import ZoneInfo` / `from datetime import UTC` |
| `'datas':` on `ir.attachment` | `'raw':` |
| `= base64.b64encode(` on Binary fields | `= BinaryBytes(` |
| `def _track_subtype(` | `def _track_log_get_default_subtype(` |
| `acc_number`, `acc_holder_name`, `bank_id` | `account_number`, `holder_name`, `bank_name` |
| `company_type`, `company_registry` | `is_company`, `additional_identifiers` |

Details: `odoo-model-patterns-19-20.md`.

## Step 5: Views

```xml
<!-- v19 -->
<button name="action_view_loans" type="object" class="oe_stat_button" icon="fa-book">
    <field name="loan_count" widget="statinfo"/>
</button>
<field name="date_due" widget="remaining_days"/>
<field name="priority_level" widget="selection_badge"/>
<span class="fa fa-exclamation-triangle text-warning" title="Late"/>

<!-- v20 -->
<button name="action_view_loans" type="object" class="oe_stat_button" icon="book">
    <field name="loan_count" widget="statinfo"/>
</button>
<field name="date_due" widget="relative_date"/>
<field name="priority_level" widget="badges_selection"/>
<span class="oi text-warning" data-icon="warning" title="Late"/>
```

Optional improvements in 20.0: move the kanban card to a reusable `<card>` view and reference it with
`<kanban card_id="%(library_book_view_card)d">`; group product/description columns with `<column>` in lists.

## Step 6: Reports and Server Templates

```xml
<!-- v19 -->
<record id="library_book_report_action" model="ir.actions.report">
    <field name="name">Book Sheet</field>
    <field name="model">library.book</field>
    <field name="report_type">qweb-pdf</field>
    <field name="report_name">library.library_book_report</field>
    <field name="report_file">library.library_book_report</field>
</record>

<template id="library_book_report_document">
    <t t-call="web.external_layout">
        <t t-set="layout_document_title">Book sheet</t>
        <div class="page"><h2 t-esc="doc.name"/></div>
    </t>
</template>

<!-- v20 -->
<record id="library_book_report_action" model="ir.actions.report">
    <field name="name">Book Sheet</field>
    <field name="model">library.book</field>
    <field name="report_type">qweb-pdf</field>
    <field name="report_name">library.library_book_report</field>
</record>

<template id="library_book_report_document">
    <t t-call="web.external_layout" layout_document_title.translate="Book sheet">
        <div class="page"><h2 t-out="doc.name"/></div>
    </t>
</template>
```

Same rules for mail templates (`body_html` QWeb) and website templates: `t-esc`/`t-raw` → `t-out`.

## Step 7: Data Files

```xml
<!-- v19 -->
<field name="image_1920" type="base64" file="library/static/img/logo.png"/>
<!-- v20 -->
<field name="image_1920" type="bytes" file="library/static/img/logo.png"/>
```

## Step 8: Controllers

```python
# v19
from odoo.http import Controller, content_disposition, request, route

class LibraryController(Controller):
    @route('/library/api/books', type='json2', auth='bearer')
    def api_books(self): ...

# v20
from odoo.http import Controller, request, route
from odoo.http.stream import content_disposition

class LibraryController(Controller):
    @route('/library/api/books', type='json2', auth='bearer', bearer_scope='rpc')  # API key scope must match
    def api_books(self): ...
```

## Step 9: Frontend

- Components: `odoo-owl-components-19-20.md` (props via `useProps`, `proxy`, signals, `this.` in templates).
- `@web/core/orm_service` → `@web/core/orm_plugin`; `@web/webclient/actions/action_service` → `@web/webclient/actions/action_plugin`.
- `publicWidget.Widget.extend({...})` → `class X extends Interaction` registered in `public.interactions`.
- jQuery calls → DOM APIs (`querySelector`, `addEventListener`, `classList`).
- `fa fa-*` in OWL templates → `oi` + `data-icon`.
- QUnit tests → Hoot (`@odoo/hoot`, `web.assets_unit_tests`).

## Step 10: Tests

```python
# v19
from odoo.tests.common import SingleTransactionCase, tagged

@tagged('post_install', '-at_install')
class TestLibrary(SingleTransactionCase): ...

# v20
from odoo.tests import tagged
from odoo.addons.base.tests.common import BaseCommon

@tagged('post_install', '-at_install')
class TestLibrary(BaseCommon): ...
```

## Step 11: Data Migration Scripts

Upgrading a database that contains your module also needs `migrations/20.0.1.0.0/` scripts when you rename
fields or convert stored data, for example:

```python
# migrations/20.0.1.0.0/post-migrate.py
def migrate(cr, version):
    # v19 kept the citizen id in a custom Char column; v20 stores it in additional_identifiers (jsonb)
    cr.execute("""
        UPDATE res_partner
           SET additional_identifiers = COALESCE(additional_identifiers, '{}'::jsonb)
                                        || jsonb_build_object('XX_CI', x_citizen_id)
         WHERE x_citizen_id IS NOT NULL
    """)
```

Standard Odoo data (ACL rows, attachments, base fields) is migrated by the official upgrade service; your
module's own data is your responsibility.

## Migration Checklist

### Automated
- [ ] `upgrade_code --from 19.0` applied and reviewed
- [ ] `upgrade_code --script owl3-migration` applied and reviewed

### Manifest
- [ ] Version `20.0.x.y.z`; `author`/`license` present
- [ ] `security/ir.access.csv` listed, old security files removed
- [ ] Dependencies exist in 20.0

### Python
- [ ] Removed ORM aliases replaced; `read_group` calls rewritten
- [ ] `pytz`, `datas`, base64 binaries, `odoo.osv`, `registry.clear_cache` gone
- [ ] Mail tracking overrides renamed; base model renames applied
- [ ] Controllers import from `odoo.http.*` submodules; `bearer_scope` added

### XML
- [ ] No `t-esc`/`t-raw` (server templates), no `report_file`
- [ ] Icons converted; removed widgets replaced; `type="bytes"` in data

### JavaScript
- [ ] Owl 3 syntax; no jQuery/`publicWidget`; Hoot tests

### Validation
- [ ] Module installs on a fresh 20.0 database
- [ ] Module upgrades from a 19.0 database copy
- [ ] Python and Hoot tests pass
