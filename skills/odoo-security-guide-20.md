# Odoo Security Guide - Version 20.0

```
╔══════════════════════════════════════════════════════════════════════════════╗
║  ODOO 20.0 SECURITY PATTERNS                                                 ║
║  ir.access (permissions + restrictions) replaces ir.model.access + ir.rule   ║
║  Source: odoo/addons/base/models/ir_access.py and <odoo_src>/skills/         ║
║  DO NOT use these patterns for Odoo 14-19.                                   ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

## What Changed

| Item | Odoo 19 | Odoo 20 |
|------|---------|---------|
| Model-level ACL | `ir.model.access` (`ir.model.access.csv`, `perm_*` columns) | `ir.access` row without domain |
| Record rules | `ir.rule` (`domain_force`, `groups`, `perm_*`) | `ir.access` row with a `domain` |
| Global rule | `ir.rule` without groups | `ir.access` row without group (*restriction*) |
| File | `security/ir.model.access.csv` + `security/*_rules.xml` | `security/ir.access.csv` (+ optional XML `ir.access` records) |
| Everyone | ACL without group | `base.group_everyone` (internal + portal + public) |
| Group fields | `res.groups.model_access`, `rule_groups` | `res.groups.access_ids` |
| API | `check_access_rights`, `check_access_rule` (deprecated) | `check_access`, `has_access`, `_filtered_access`, `_access_domain` |

`ir.model.access` and `ir.rule` **do not exist** in 20.0: a module shipping them fails to install.

## `ir.access` Semantics

| Concept | Rule |
|---------|------|
| Permission | Row **with** `group_id`. Rows are OR-ed over the user's groups. Its `domain` limits only what that row grants |
| Restriction | Row **without** `group_id`. AND-ed onto every user (superuser excepted). Never grants anything |
| Default | Deny. No permission row = no access (even for admins, except superuser) |
| `operation` | Required subset of `crud`, letters in that order: `c`, `r`, `u`, `d`, `cr`, `ru`, `cru`, `crud`, `rud`... |
| Domain context | `user` (context-free record), `time`, `company_ids` (active companies), `company_id` |
| `'access'` operator | `('field', 'access', 'read'\|'write'\|'create'\|'unlink')` on a many2one or `id`: the related record must be accessible for that operation |
| `write` | Checks write access only, never read: restrict **all** operations that need it |
| Formula | `Domain.OR(permission domains) & Domain.AND(restriction domains)` (see `BaseModel._access_domain`) |
| `_inherits` | Delegation models also require access to their parent models |
| Cache | Access records are cached (`stable` cache); changing them through the ORM invalidates automatically |

## `security/ir.access.csv`

```csv
id,name,model_id,group_id/id,operation,domain
access_library_book_user,library.book user,library.book,library.group_library_user,r,
access_library_book_own,library.book own books,library.book,library.group_library_user,ru,"[('user_id', '=', user.id)]"
access_library_book_manager,library.book manager,library.book,library.group_library_manager,crud,
access_library_book_portal,library.book portal,library.book,base.group_portal,r,"[('member_ids', 'in', [user.partner_id.id])]"
access_library_book_everyone,library.book published,library.book,base.group_everyone,r,"[('is_published', '=', True)]"
library_book_rule_company,library.book multi-company,library.book,,crud,"[('company_id', 'in', company_ids + [False])]"
access_library_loan_line_user,library.loan.line user,library.loan.line,library.group_library_user,crud,"[('loan_id', 'access', 'write')]"
access_library_wizard_user,library.book.wizard user,library.book.wizard,library.group_library_user,crud,
library_book_wizard_rule_own,library.book.wizard own,library.book.wizard,,crud,"[('create_uid', '=', user.id)]"
```

- `model_id` holds the model **technical name** (resolved through `ir.model` name search), not `model_x`.
- Group column header is `group_id/id` (external id). Empty = restriction.
- Load it after the groups file: `'security/library_security.xml', 'security/ir.access.csv'`.
- Transient models (wizards) need rows too.
- Multi-company models need a group-less restriction row; append `+ [False]` when `company_id` is optional.

## XML `ir.access` Records

Use XML for long domains, or to deactivate another module's access (the v20 equivalent of disabling an
`ir.rule`):

```xml
<odoo>
    <record id="access_library_book_sharing_portal" model="ir.access">
        <field name="name">library.book: portal members can edit shared books</field>
        <field name="model_id" ref="library.model_library_book"/>
        <field name="group_id" ref="base.group_portal"/>
        <field name="operation">ru</field>
        <field name="domain">[
            ('share_mode', '=', 'edit'),
            ('member_ids', 'in', [user.partner_id.id]),
        ]</field>
    </record>

    <!-- disable an access defined by another module (keep noupdate to survive updates) -->
    <record id="product.product_pricelist_comp_rule" model="ir.access">
        <field name="active" eval="False"/>
    </record>
</odoo>
```

## Groups and Privileges (unchanged since 19.0)

```xml
<odoo>
    <record id="res_groups_privilege_library" model="res.groups.privilege">
        <field name="name">Library</field>
        <field name="sequence">50</field>
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
        <field name="user_ids" eval="[Command.link(ref('base.user_root')), Command.link(ref('base.user_admin'))]"/>
    </record>
</odoo>
```

- `res.users.group_ids` / `res.groups.user_ids` (renamed in 19.0 from `groups_id` / `users`).
- New in 20.0: `base.group_everyone` (all users) and `base.group_user_regular` (regular vs light internal
  users, `res.users.role` = `light_user` / `regular_user` / `group_system`). Menus meant only for regular
  users use `groups="base.group_user_regular"`.

## Field-Level Access

```python
api_token = fields.Char(groups=fields.NO_ACCESS)          # hidden from everyone, admins included
salary = fields.Monetary(groups='library.group_library_manager')
member_email = fields.Char(related='member_id.email', related_sudo=False)  # related fields are sudo by default
```

- `groups` removes the field from views and `fields_get`, and raises on explicit read/write.
- Check programmatically with `record.check_field_access(field, 'read')` / `has_field_access(field, 'write')`.

## Access Checks in Code

```python
book.check_access('write')                                  # AccessError if not allowed (model + records)
if self.env['library.book'].has_access('create'):           # model level, boolean
    ...
readable = books._filtered_access('read')                   # replaces _filter_access_rules()
domain = self.env['library.book']._access_domain('read')    # Domain object
```

Removed in 20.0: `check_access_rights()`, `check_access_rule()`, `_filter_access_rules()`,
`_filter_access_rules_python()`, `check_field_access_rights()`, `ir.attachment.check()` (deprecated).

## Methods Exposed Through RPC

```python
class LibraryBook(models.Model):
    _inherit = 'library.book'

    def action_reserve(self):              # public: callable via RPC, validate everything
        self.ensure_one()
        self.check_access('write')
        ...

    def _compute_penalty(self):            # private by default
        ...

    @api.private
    def compute_penalty(self):             # public name kept for Python callers, blocked for RPC
        return self._compute_penalty()
```

`get_public_method()` refuses `_`-prefixed names, class/static methods and anything decorated
`@api.private` anywhere in the MRO.

## sudo() and Commands

```python
# BAD: attacker-controlled values written with superuser rights
record.sudo().write(post)

# GOOD: whitelist keys, narrowest scope, comment why
allowed = {key: post[key] for key in ('name', 'email') if post.get(key)}
record.sudo().write(allowed)  # sudo: portal users cannot write partner contact data
```

- x2many `Command` payloads executed under `sudo()` also run with sudo on the comodel, unless the comodel sets
  `_allow_sudo_commands = False` (`ir.access` itself does).
- `with_user()` / `with_company()` switches must never be attacker-driven.

## SQL and Domains

```python
from odoo.fields import Domain
from odoo.tools import SQL

# Domain injection: combine, never concatenate user lists
domain = Domain('company_id', 'in', self.env.companies.ids) & Domain(user_domain)

# Raw SQL over ORM-filtered rows (access rules and active_test stay in force)
query = self._search(domain)
rows = self.env.execute_query(query.select(query.table.id, query.table.name))

# Parameters and identifiers
self.env.cr.execute(SQL(
    "SELECT id FROM %s WHERE name ILIKE %s",
    SQL.identifier(self._table), f"%{escape_like_value(term)}%",
))
```

`SQL.identifier()` validates with `assert`, and Odoo 20 refuses to run under `python -O`; still never pass raw
user input as an identifier. (`escape_like_value` lives in `odoo.tools.sql`.)

## Controllers

```python
from odoo.http import Controller, request, route


class LibraryController(Controller):

    @route('/library/reserve/<int:book_id>', type='http', auth='user', methods=['POST'])
    def reserve(self, book_id, **post):
        book = request.env['library.book'].browse(book_id)
        book.check_access('write')
        book.action_reserve()
        return request.redirect(f'/library/book/{book_id}')

    @route('/library/api/books', type='json2', auth='bearer', bearer_scope='rpc', readonly=True)
    def api_books(self):
        return request.env['library.book'].search_read([], ['name', 'state'])
```

- `auth`: `user`, `bearer` (requires `bearer_scope` in 20.0), `public`, `none`.
- API keys are scoped: a key only authenticates routes whose `bearer_scope` equals its scope. Keys created from
  the user preferences use scope `rpc`; for a dedicated scope, extend `res.users.apikeys.description.scope`
  with `selection_add`.
- State-changing routes: `methods=['POST']`; keep CSRF on for `type='http'`
  (forms: `<input type="hidden" name="csrf_token" t-att-value="request.csrf_token()"/>`).
- `type='jsonrpc'` / `'json2'` have no CSRF token by design (JSON content type).

## XSS

- Server QWeb and OWL: `t-out` escapes; `t-raw` no longer exists, `t-esc` renders nothing server-side.
- Build HTML with `Markup("<b>{}</b>").format(value)` (never f-strings inside `Markup`).
- JS: never feed user data to `innerHTML`, `insertAdjacentHTML` or `markup(str)`; use the tagged template
  ``markup`<b>${name}</b>` `` or `htmlEscape` from `@odoo/owl`.

## Other Framework Rules

| Rule | Use |
|------|-----|
| Files | `odoo.tools.file_open()`, never builtin `open()` on influenced paths |
| Evaluation | `json.loads()` / `ast.literal_eval()`; `safe_eval` only for trusted privileged users |
| Secrets | `odoo.tools.consteq()` or a database lookup, never `==` |
| Dynamic fields | `record[field_name]`, never `getattr`/`setattr` on records |
| Serialization | `json`, never `pickle` |
| Defaults | No mutable default arguments |
| Model methods | Do not return rich objects (keys, handles) from public model methods |

## v20 Security Checklist

- [ ] `security/ir.access.csv` present, loaded after the groups file
- [ ] No `ir.model.access.csv`, no `ir.rule` records
- [ ] Every model (transient included) has at least one permission row
- [ ] Multi-company models have a group-less `company_ids` restriction row
- [ ] No `c`/`u`/`d` granted to `base.group_everyone`, `base.group_portal` or `base.group_public` without a strict domain
- [ ] Sensitive fields use `groups` (or `fields.NO_ACCESS`), sensitive related fields `related_sudo=False`
- [ ] Public methods validated or `@api.private`
- [ ] `auth='bearer'` routes declare `bearer_scope`
- [ ] SQL via `SQL`/`_search`, domains via `Domain`

## AI Agent Instructions (v20 security)

1. **GENERATE** `security/ir.access.csv` with header `id,name,model_id,group_id/id,operation,domain`.
2. **MODEL** each v19 ACL as a permission row and each v19 record rule as a row with a domain; global rules become restrictions.
3. **USE** `company_ids` (not `allowed_company_ids`) in domains.
4. **USE** `check_access` / `has_access` / `_filtered_access` / `_access_domain`.
5. **READ** `<odoo_src>/skills/odoo-security/SKILL.md` for audits.
