# Odoo Module Generator - Version 20.0

```
╔══════════════════════════════════════════════════════════════════════════════╗
║  ODOO 20.0 MODULE GENERATION PATTERNS                                        ║
║  This file contains ONLY Odoo 20.0 specific patterns.                        ║
║  DO NOT use these patterns for other versions.                               ║
║  Verified against the 20.0 branch (community + enterprise)                   ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

## Version 20.0 Requirements

- **Python**: 3.12 - 3.14, PostgreSQL 16+
- **Security**: `security/ir.access.csv` (`ir.access`); no `ir.model.access.csv`, no `ir.rule`
- **Frontend**: OWL 3 (`useProps` + `t`, `proxy`, `signal`), Material Symbols icons, Hoot tests
- **Server QWeb**: `t-out` only, `t-call` parameters as attributes
- **Binary data**: `BinaryBytes` / `ir.attachment.raw`
- **Views**: `<list>`, `<chatter/>`, direct `invisible`/`readonly`/`required`, optional `card` view for kanban

## Module Structure

```
library/
├── __init__.py
├── __manifest__.py
├── controllers/
│   ├── __init__.py
│   └── library.py
├── data/
│   └── library_data.xml
├── models/
│   ├── __init__.py
│   └── library_book.py          # one model per file, named after the model
├── report/
│   ├── library_book_reports.xml    # report actions, paperformat
│   └── library_book_templates.xml  # QWeb templates
├── security/
│   ├── library_security.xml     # privileges, groups
│   └── ir.access.csv            # permissions and restrictions
├── static/
│   ├── description/icon.png
│   ├── src/
│   │   └── dashboard/           # one folder per feature: js + xml + scss
│   │       ├── dashboard.js
│   │       ├── dashboard.xml
│   │       └── dashboard.scss
│   └── tests/
│       └── dashboard.test.js
├── tests/
│   ├── __init__.py
│   └── test_library_book.py
├── views/
│   ├── library_book_views.xml
│   └── library_menus.xml
└── wizard/
    ├── __init__.py
    ├── library_book_lend.py
    └── library_book_lend_views.xml
```

Optional in 20.0: `populate/` with `populate.blueprint` XML records (requires the `populate` addon).

## `__manifest__.py`

```python
{
    'name': 'Library',
    'version': '20.0.1.0.0',
    'category': 'Services',
    'summary': 'Manage the books of a library',
    'author': '{Author}',
    'maintainers': ['{github_user}'],
    'website': '{Website}',
    'license': 'LGPL-3',
    'depends': ['mail'],
    'data': [
        # security first: groups, then access rows
        'security/library_security.xml',
        'security/ir.access.csv',
        'data/library_data.xml',
        'views/library_book_views.xml',
        'wizard/library_book_lend_views.xml',
        'report/library_book_reports.xml',
        'report/library_book_templates.xml',
        'views/library_menus.xml',
    ],
    'assets': {
        'web.assets_backend': [
            'library/static/src/**/*',
        ],
        'web.assets_unit_tests': [
            'library/static/tests/**/*',
        ],
    },
    'installable': True,
    'application': True,
}
```

- `author` and `license` are mandatory. Legacy keys `demo_xml`, `init_xml`, `update_xml` are gone.
- Do not depend on modules merged or removed in 20.0 (`base_vat`/`base_iban` → `base`, `stock_picking_batch` → `stock`, `hr_org_chart` → `hr`, `l10n_latam_base` removed).

## Model (`models/library_book.py`)

```python
from odoo import api, fields, models
from odoo.exceptions import UserError


class LibraryBook(models.Model):
    _name = 'library.book'
    _description = "Library Book"
    _inherit = ['mail.thread', 'mail.activity.mixin']
    _order = 'name'
    _check_company_auto = True
    _rec_names_search = ('name', 'isbn')

    _isbn_company_uniq = models.Constraint(
        'UNIQUE(isbn, company_id)',
        "The ISBN must be unique per company.",
    )

    name = fields.Char(required=True, tracking=True, index='trigram')
    isbn = fields.Char(string="ISBN", copy=False)
    active = fields.Boolean(default=True)
    state = fields.Selection([
        ('available', "Available"),
        ('borrowed', "Borrowed"),
        ('lost', "Lost"),
    ], default='available', required=True, tracking=True)
    company_id = fields.Many2one('res.company', required=True, index=True, default=lambda self: self.env.company)
    author_id = fields.Many2one('res.partner', check_company=True)
    user_id = fields.Many2one('res.users', string="Librarian", default=lambda self: self.env.user)
    tag_ids = fields.Many2many('library.tag')
    color = fields.Integer()
    loan_count = fields.Integer(compute='_compute_loan_count')

    def _compute_loan_count(self):
        counts = dict(self.env['library.loan']._read_group(
            [('book_id', 'in', self.ids)], ['book_id'], ['__count'],
        ))
        for book in self:
            book.loan_count = counts.get(book, 0)

    @api.ondelete(at_uninstall=False)
    def _unlink_except_borrowed(self):
        if any(book.state == 'borrowed' for book in self):
            raise UserError(self.env._("You cannot delete a borrowed book."))

    def action_mark_lost(self):
        self.ensure_one()
        self.state = 'lost'

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
```

More ORM patterns: `odoo-model-patterns-20.md`.

## Security

`security/library_security.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <record id="res_groups_privilege_library" model="res.groups.privilege">
        <field name="name">Library</field>
        <field name="category_id" ref="base.module_category_services"/>
    </record>

    <record id="group_library_user" model="res.groups">
        <field name="name">User</field>
        <field name="sequence">10</field>
        <field name="privilege_id" ref="res_groups_privilege_library"/>
        <field name="implied_ids" eval="[Command.link(ref('base.group_user'))]"/>
    </record>

    <record id="group_library_manager" model="res.groups">
        <field name="name">Administrator</field>
        <field name="sequence">20</field>
        <field name="privilege_id" ref="res_groups_privilege_library"/>
        <field name="implied_ids" eval="[Command.link(ref('group_library_user'))]"/>
        <field name="user_ids" eval="[Command.link(ref('base.user_admin'))]"/>
    </record>
</odoo>
```

`security/ir.access.csv`:

```csv
id,name,model_id,group_id/id,operation,domain
access_library_book_user,library.book user,library.book,library.group_library_user,r,
access_library_book_manager,library.book manager,library.book,library.group_library_manager,crud,
library_book_rule_company,library.book multi-company,library.book,,crud,"[('company_id', 'in', company_ids)]"
access_library_tag_user,library.tag user,library.tag,library.group_library_user,r,
access_library_tag_manager,library.tag manager,library.tag,library.group_library_manager,crud,
access_library_book_lend_user,library.book.lend user,library.book.lend,library.group_library_user,crud,
```

Details: `odoo-security-guide-20.md`.

## Views (`views/library_book_views.xml`)

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <record id="library_book_view_list" model="ir.ui.view">
        <field name="name">library.book.view.list</field>
        <field name="model">library.book</field>
        <field name="arch" type="xml">
            <list sample="1">
                <field name="name"/>
                <field name="author_id"/>
                <field name="state" widget="badge" decoration-success="state == 'available'" decoration-danger="state == 'lost'"/>
                <field name="company_id" groups="base.group_multi_company" optional="hide"/>
            </list>
        </field>
    </record>

    <record id="library_book_view_form" model="ir.ui.view">
        <field name="name">library.book.view.form</field>
        <field name="model">library.book</field>
        <field name="arch" type="xml">
            <form>
                <header>
                    <button name="action_mark_lost" type="object" string="Mark as Lost"
                            invisible="state != 'borrowed'" groups="library.group_library_manager"
                            confirm="Mark this book as lost?" confirm-title="Lost book"/>
                    <field name="state" widget="statusbar" statusbar_visible="available,borrowed"/>
                </header>
                <sheet>
                    <div class="oe_button_box" name="button_box">
                        <button name="action_view_loans" type="object" class="oe_stat_button" icon="book">
                            <field name="loan_count" string="Loans" widget="statinfo"/>
                        </button>
                    </div>
                    <widget name="web_ribbon" title="Archived" bg_color="text-bg-danger" invisible="active"/>
                    <div class="oe_title">
                        <h1><field name="name" placeholder="e.g. Dune"/></h1>
                    </div>
                    <group>
                        <group>
                            <field name="author_id"/>
                            <field name="isbn"/>
                        </group>
                        <group>
                            <field name="user_id"/>
                            <field name="tag_ids" widget="many2many_tags" options="{'color_field': 'color'}"/>
                            <field name="company_id" groups="base.group_multi_company"/>
                        </group>
                    </group>
                </sheet>
                <chatter/>
            </form>
        </field>
    </record>

    <record id="library_book_view_card" model="ir.ui.view">
        <field name="name">library.book.view.card</field>
        <field name="model">library.book</field>
        <field name="arch" type="xml">
            <card>
                <templates>
                    <t t-name="card">
                        <field name="name" class="fw-bold fs-5"/>
                        <field name="author_id"/>
                        <footer>
                            <field name="tag_ids" widget="many2many_tags"/>
                            <field name="user_id" widget="many2one_avatar_user" class="ms-auto"/>
                        </footer>
                    </t>
                </templates>
            </card>
        </field>
    </record>

    <record id="library_book_view_kanban" model="ir.ui.view">
        <field name="name">library.book.view.kanban</field>
        <field name="model">library.book</field>
        <field name="arch" type="xml">
            <kanban card_id="%(library.library_book_view_card)d" default_group_by="state" sample="1"/>
        </field>
    </record>

    <record id="library_book_view_search" model="ir.ui.view">
        <field name="name">library.book.view.search</field>
        <field name="model">library.book</field>
        <field name="arch" type="xml">
            <search>
                <field name="name"/>
                <field name="author_id"/>
                <filter name="filter_available" string="Available" domain="[('state', '=', 'available')]"/>
                <filter name="filter_mine" string="My Books" domain="[('user_id', '=', uid)]"/>
                <separator/>
                <filter name="filter_archived" string="Archived" domain="[('active', '=', False)]"/>
                <filter name="groupby_state" string="Status" context="{'group_by': 'state'}"/>
            </search>
        </field>
    </record>

    <record id="library_book_action" model="ir.actions.act_window">
        <field name="name">Books</field>
        <field name="res_model">library.book</field>
        <field name="view_mode">list,kanban,form</field>
        <field name="help" type="html">
            <p class="o_view_nocontent_smiling_face">Add your first book</p>
        </field>
    </record>
</odoo>
```

`views/library_menus.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <menuitem id="library_menu_root" name="Library" web_icon="library,static/description/icon.png"
              groups="library.group_library_user"/>
    <menuitem id="library_book_menu" name="Books" parent="library_menu_root"
              action="library_book_action" sequence="10"/>
</odoo>
```

View rules for 20.0:
- Button and stat-button `icon` = Material Symbols names (`book`, `build`, `edit_square`); `fa-*` no longer renders.
- Inline icons: `<i class="oi" data-icon="check" title="Available"/>` (a `title`/`aria-label` or text is required).
- `remaining_days` → `relative_date`; `selection_badge` → `badges_selection` / `badges_many2one`.
- A kanban view may still embed `<templates><t t-name="card">...</t></templates>`; `card_id` lets you reuse a card view.

## Report

`report/library_book_reports.xml` (no `report_file` field in 20.0):

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <record id="library_book_report_action" model="ir.actions.report">
        <field name="name">Book Sheet</field>
        <field name="model">library.book</field>
        <field name="report_type">qweb-pdf</field>
        <field name="report_name">library.library_book_report</field>
        <field name="print_report_name">'Book - %s' % object.name</field>
        <field name="binding_model_id" ref="model_library_book"/>
        <field name="binding_type">report</field>
    </record>
</odoo>
```

`report/library_book_templates.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <template id="library_book_report_document">
        <t t-call="web.external_layout" layout_document_title.f="Book {{ doc.name }}">
            <div class="page">
                <p>Author: <span t-field="doc.author_id"/></p>
                <p>ISBN: <span t-out="doc.isbn or '-'"/></p>
            </div>
        </t>
    </template>

    <template id="library_book_report">
        <t t-call="web.html_container">
            <t t-foreach="docs" t-as="doc">
                <t t-call="library.library_book_report_document" t-lang="doc.user_id.lang"/>
            </t>
        </t>
    </template>
</odoo>
```

- `t-esc`/`t-raw` render nothing in 20.0: use `t-out`/`t-field`.
- Values for the called template go on the `t-call` node (`name="expr"`, `name.f="{{ }}"`, `name.translate="..."`).

## Data File with Binary Content

```xml
<odoo noupdate="1">
    <record id="library_tag_classic" model="library.tag">
        <field name="name">Classic</field>
        <field name="image" type="bytes" file="library/static/img/classic.png"/>
    </record>
</odoo>
```

## Controller

```python
from odoo.http import Controller, request, route
from odoo.http.stream import content_disposition


class LibraryController(Controller):

    @route('/library/book/<int:book_id>/sheet', type='http', auth='user')
    def book_sheet(self, book_id):
        book = request.env['library.book'].browse(book_id)
        book.check_access('read')
        pdf, _type = request.env['ir.actions.report']._render_qweb_pdf(
            'library.library_book_report_action', book.ids,
        )
        return request.make_response(pdf, headers=[
            ('Content-Type', 'application/pdf'),
            ('Content-Disposition', content_disposition(f"{book.name}.pdf")),
        ])
```

## OWL 3 Component (`static/src/dashboard/dashboard.js`)

```javascript
import { Component, onWillStart, proxy, useProps } from "@odoo/owl";
import { registry } from "@web/core/registry";
import { useService } from "@web/core/utils/hooks";
import { standardActionServiceProps } from "@web/webclient/actions/action_plugin";

export class LibraryDashboard extends Component {
    static template = "library.LibraryDashboard";
    props = useProps(standardActionServiceProps);
    state = proxy({ available: 0, borrowed: 0 });

    setup() {
        this.orm = useService("orm");
        onWillStart(async () => {
            const groups = await this.orm.call("library.book", "read_group", [[], ["state"], ["__count"]]);
            for (const [state, count] of groups) {
                this.state[state] = count;
            }
        });
    }
}

registry.category("actions").add("library.dashboard", LibraryDashboard);
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<templates xml:space="preserve">
    <t t-name="library.LibraryDashboard">
        <div class="o_library_dashboard d-flex gap-3 p-3">
            <div class="o_library_dashboard_kpi">
                <i class="oi" data-icon="book" title="Available"/>
                <span t-out="this.state.available"/>
            </div>
            <div class="o_library_dashboard_kpi">
                <i class="oi" data-icon="schedule" title="Borrowed"/>
                <span t-out="this.state.borrowed"/>
            </div>
        </div>
    </t>
</templates>
```

`read_group` here uses the 20.0 contract (tuples). Full OWL patterns: `odoo-owl-components-20.md`.

## Tests (`tests/test_library_book.py`)

```python
from odoo.exceptions import UserError
from odoo.tests import tagged

from odoo.addons.base.tests.common import BaseCommon


@tagged('post_install', '-at_install')
class TestLibraryBook(BaseCommon):

    @classmethod
    def setUpClass(cls):
        super().setUpClass()
        cls.book = cls.env['library.book'].create({'name': "Dune"})

    def test_cannot_delete_borrowed_book(self):
        self.book.state = 'borrowed'
        with self.assertRaises(UserError):
            self.book.unlink()
```

- Test files: `tests/test_*.py`, imported in `tests/__init__.py`.
- `SingleTransactionCase` was removed in 20.0; use `TransactionCase`/`BaseCommon` or `HttpCase`.
- JS unit tests (Hoot) go in `static/tests/**/*.test.js` and the `web.assets_unit_tests` bundle.

## v20 Checklist

- [ ] `version` starts with `20.0.`; `author`, `license` set
- [ ] `security/ir.access.csv` loaded after the groups file; every model covered; multi-company restriction
- [ ] No `ir.model.access.csv`, no `ir.rule`, no `report_file`, no `t-esc`/`t-raw`
- [ ] `models.Constraint`/`models.Index` instead of `_sql_constraints`
- [ ] `Domain` for domain building, `self.env._()` for translations
- [ ] Binary content via `BinaryBytes`/`raw`, XML files via `type="bytes"`
- [ ] Time zones via `zoneinfo`
- [ ] Icons with Material Symbols names
- [ ] OWL 3 syntax (`useProps`, `proxy`, `this.` in templates)
- [ ] Tests with `BaseCommon`/`TransactionCase`

## AI Agent Instructions (v20)

1. **GENERATE** the structure above; one model per file; feature folders under `static/src`.
2. **GENERATE** `security/ir.access.csv` (never `ir.model.access.csv`).
3. **USE** OWL 3 and Material Symbols; `t-out` everywhere.
4. **CHECK** every dependency exists in 20.0 before adding it.
5. **FOLLOW** OCA conventions of the target project (README, pre-commit, pylint-odoo) on top of these patterns.
