# Odoo Security Guide - Migration 19.0 → 20.0

```
╔══════════════════════════════════════════════════════════════════════════════╗
║  MIGRATION GUIDE: ODOO 19.0 → 20.0 SECURITY                                  ║
║  ir.model.access + ir.rule  →  ir.access                                     ║
║  Conversion rules taken from odoo/upgrade_code/19.4-00-ir-access.py          ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

## Overview

| Component | v19 | v20 | Migration |
|-----------|-----|-----|-----------|
| ACL | `ir.model.access` (CSV, `perm_*`) | `ir.access` permission row | **REQUIRED** |
| Record rule with groups | `ir.rule` (`groups`, `domain_force`, `perm_*`) | `ir.access` permission row with `domain` | **REQUIRED** |
| Global record rule | `ir.rule` without groups | `ir.access` restriction row (no group) | **REQUIRED** |
| ACL without group | Grants everybody | `base.group_everyone` permission row | **REQUIRED** |
| Code API | `check_access_rights`, `check_access_rule`, `_filter_access_rules` (deprecated) | removed | **REQUIRED** |
| `ir.rule._compute_domain(model, mode)` | Available | `env[model]._access_domain(mode)` | **REQUIRED** |
| `ir.model.access.check(model, mode, raise_exception)` | Available | `env[model].has_access(mode)` / `check_access(mode)` | **REQUIRED** |
| Groups, privileges, field `groups` | 19.0 syntax | Unchanged | - |

## Automatic Conversion

```bash
./odoo-bin upgrade_code --script 19.4-00-ir-access --addons-path=/path/to/addons --dry-run
./odoo-bin upgrade_code --script 19.4-00-ir-access --addons-path=/path/to/addons
```

The script reads each module's `ir.model.access.csv` and `ir.rule`/`ir.model.access` XML records, writes
`security/ir.access.csv` (or `ir.access.csv` when the module has no `security/` folder), updates the
manifest and removes the converted files/records. Log levels: `INFO` normal, `WARNING` partially handled
(check by hand), `ERROR` not handled.

## Conversion Rules

1. **ACL of group G, operations O, and no rule of G (or of groups implied by G) for an operation** → permission row `G, O_unrestricted`, no domain.
2. **Group rule R (group H, domain D, operations Or) combined with an ACL of group G** where H implies G or G implies H → permission row on the more specific group with operations `Or ∩ O` and domain `D`.
3. **Global rule (no group)** → restriction row with the same operations and domain.
4. **ACL without group** → permission row on `base.group_everyone`.
5. **Group rule without any compatible ACL** → dropped with a `WARNING` (it granted nothing in v19 either).
6. **Rule with a false domain** (`[(0, '=', 1)]`) → ignored.
7. **Records that modify another module's ACL/rule** (`other_module.xmlid`) → skipped with a `WARNING`: redo them by hand.
8. **Inactive ACL/rule** → skipped (`ERROR` log).
9. Rows made redundant by another row (same or implied group, superset operations, same or no domain) are deduplicated.

`ir.rule` `perm_*` flags default to `True` (the rule applies to that operation); `ir.model.access` `perm_*`
flags default to `False` (the operation is not granted).

The script can emit several rows that are equivalent once OR-ed (for instance a row with
`[(1, '=', 1)]` next to a row with a narrower domain for the same group): simplify them by hand, the
examples below show the simplified result.

## Conversion Examples

| v19 | v20 `ir.access.csv` row(s) |
|-----|----------------------------|
| ACL `G`: `1,1,1,1`, no rule | `G,crud,` |
| ACL `G`: `1,1,0,0` + rule `G` (default perms) `[('user_id','=',user.id)]` | `G,ru,"[('user_id', '=', user.id)]"` |
| ACL `G`: `1,1,1,1` + rule `G` only `perm_read` `D` | `G,r,D` and `G,cud,` |
| ACL `G`: `1,1,1,1` + rule `G_manager` (implies `G`) `[(1,'=',1)]` + rule `G` `D` | `G,crud,D` and `G_manager,crud,` |
| Global rule `[('company_id','in',company_ids)]` | `,crud,"[('company_id', 'in', company_ids)]"` (no group) |
| ACL without group `1,0,0,0` | `base.group_everyone,r,` |
| ACL `base.group_portal`: `1,0,0,0` + portal rule `D` | `base.group_portal,r,D` |

### Full Example

```csv
# v19: security/ir.model.access.csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_library_book_user,library.book user,model_library_book,library.group_library_user,1,1,1,0
access_library_book_manager,library.book manager,model_library_book,library.group_library_manager,1,1,1,1
access_library_book_portal,library.book portal,model_library_book,base.group_portal,1,0,0,0
```

```xml
<!-- v19: security/library_rules.xml -->
<odoo>
    <record id="library_book_rule_user" model="ir.rule">
        <field name="name">Own books</field>
        <field name="model_id" ref="model_library_book"/>
        <field name="domain_force">['|', ('user_id', '=', user.id), ('user_id', '=', False)]</field>
        <field name="groups" eval="[Command.link(ref('library.group_library_user'))]"/>
    </record>
    <record id="library_book_rule_manager" model="ir.rule">
        <field name="name">All books</field>
        <field name="model_id" ref="model_library_book"/>
        <field name="domain_force">[(1, '=', 1)]</field>
        <field name="groups" eval="[Command.link(ref('library.group_library_manager'))]"/>
    </record>
    <record id="library_book_rule_portal" model="ir.rule">
        <field name="name">Portal: member books</field>
        <field name="model_id" ref="model_library_book"/>
        <field name="domain_force">[('member_ids', 'in', [user.partner_id.id])]</field>
        <field name="groups" eval="[Command.link(ref('base.group_portal'))]"/>
    </record>
    <record id="library_book_rule_company" model="ir.rule">
        <field name="name">Multi-company</field>
        <field name="model_id" ref="model_library_book"/>
        <field name="domain_force">['|', ('company_id', '=', False), ('company_id', 'in', company_ids)]</field>
    </record>
</odoo>
```

```csv
# v20: security/ir.access.csv
id,name,model_id,group_id/id,operation,domain
access_library_book_user,library.book user,library.book,library.group_library_user,cru,"['|', ('user_id', '=', user.id), ('user_id', '=', False)]"
access_library_book_manager,library.book manager,library.book,library.group_library_manager,crud,
access_library_book_portal,library.book portal,library.book,base.group_portal,r,"[('member_ids', 'in', [user.partner_id.id])]"
library_book_rule_company,Multi-company,library.book,,crud,"[('company_id', 'in', company_ids + [False])]"
```

## Semantics to Double-Check

| Situation | Why it matters |
|-----------|----------------|
| Restrictions apply to **every** user (admins included, superuser excepted) | A too-strict global row can lock everybody out |
| Permissions of different groups are OR-ed | A broad row on an implied group widens access for all implying groups |
| `write` never checks `read` | Restrict every operation that must be restricted, not only `r` |
| `operation` letters order | `ru`, `cru`, `crud`... (`ur` is not a valid value) |
| `model_id` column | Model technical name (`library.book`), not `model_library_book` |
| Lines/children | Prefer `('parent_id', 'access', 'read')` over duplicating the parent's domain |

## Python Code Referencing the Old Models

```python
# v19
domain = self.env['ir.rule']._compute_domain('library.book', 'read')
allowed = self.env['ir.model.access'].check('library.book', 'write', raise_exception=False)
self.env['ir.rule'].create({
    'name': 'Own books', 'model_id': self.env.ref('library.model_library_book').id,
    'domain_force': "[('user_id', '=', user.id)]",
    'groups': [Command.link(self.env.ref('library.group_library_user').id)],
})
self.check_access_rights('write')
self.check_access_rule('write')

# v20
domain = self.env['library.book']._access_domain('read')
allowed = self.env['library.book'].has_access('write')
self.env['ir.access'].create({
    'name': 'Own books',
    'model_id': self.env['ir.model']._get_id('library.book'),
    'group_id': self.env.ref('library.group_library_user').id,
    'operation': 'ru',
    'domain': "[('user_id', '=', user.id)]",
})
self.check_access('write')
```

Groups for an operation: `self.env['ir.access']._get_groups_with_access('library.book', 'write')`.

## Disabling Another Module's Access

```xml
<!-- v19 -->
<record id="product.product_comp_rule" model="ir.rule">
    <field name="active" eval="False"/>
</record>

<!-- v20: the converted record keeps its xml id, now an ir.access -->
<record id="product.product_comp_rule" model="ir.access">
    <field name="active" eval="False"/>
</record>
```

Check the xml id still exists in 20.0 (`git grep` the target module's `ir.access.csv`).

## No Change Required

- `res.groups` / `res.groups.privilege` definitions (19.0 syntax: `privilege_id`, `implied_ids`, `user_ids`)
- Field-level `groups=`, `fields.NO_ACCESS`
- View `groups=` attributes, menu `groups=`
- `sudo()`, `with_user()`, `with_company()` semantics
- `@api.private`, `_allow_sudo_commands`

## Migration Checklist

- [ ] Script `19.4-00-ir-access` executed; every `WARNING`/`ERROR` reviewed
- [ ] `security/ir.access.csv` in the manifest after the groups file; old files removed
- [ ] Each model (wizards included) has a permission row
- [ ] Multi-company restriction rows use `company_ids` (`+ [False]` when optional)
- [ ] Portal/public rows reviewed (no `c`/`u`/`d` without a strict domain)
- [ ] Code using `ir.rule`, `ir.model.access`, `check_access_rights`, `check_access_rule` rewritten
- [ ] Tests that create `ir.rule` records create `ir.access` records instead
- [ ] Access verified with non-admin users of each group
