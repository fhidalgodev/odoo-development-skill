# Odoo Version Knowledge - Complete Reference (All Versions)

This document provides a comprehensive reference for Odoo version differences, deprecations, and migration paths across all supported versions.

```
╔══════════════════════════════════════════════════════════════════════════════╗
║  COMPLETE DEPRECATION AND CHANGE REFERENCE                                   ║
║  Versions: 14.0 - 20.0                                                       ║
║  Use version-specific files for detailed implementation patterns.            ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

## Complete Deprecation Timeline

### Decorators

| Decorator | v14 | v15 | v16 | v17 | v18 | v19 | v20 |
|-----------|-----|-----|-----|-----|-----|-----|-----|
| `@api.multi` | ⚠️ DEP | ❌ REM | ❌ | ❌ | ❌ | ❌ | ❌ |
| `@api.one` | ⚠️ DEP | ❌ REM | ❌ | ❌ | ❌ | ❌ | ❌ |
| `@api.returns` | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ REM | ❌ |
| `@api.model` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `@api.model_create_multi` | ✅ | ⚠️ REC | ⚠️ REC | ✅ REQ | ✅ REQ | ✅ REQ | ✅ REQ |
| `@api.depends` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `@api.constrains` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `@api.onchange` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `@api.depends_context` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `@api.ondelete` | ➖ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `@api.private` | ➖ | ➖ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `@api.readonly` | ➖ | ➖ | ➖ | ➖ | ✅ | ✅ | ✅ |
| `@api.ormcache` (was `tools.ormcache`) | ➖ | ➖ | ➖ | ➖ | ➖ | ➖ | ✅ |

Legend: ✅ = Supported, ⚠️ DEP = Deprecated, ⚠️ REC = Recommended, ⚠️ OPT = Optional, ✅ REQ = Required, ❌ REM = Removed, ➖ = Not available

### Field Attributes

| Attribute | v14 | v15 | v16 | v17 | v18 | v19 | v20 |
|-----------|-----|-----|-----|-----|-----|-----|-----|
| `track_visibility` | ⚠️ DEP | ❌ REM | ❌ | ❌ | ❌ | ❌ | ❌ |
| `tracking` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `oldname` | ⚠️ DEP | ❌ REM | ❌ | ❌ | ❌ | ❌ | ❌ |
| `check_company` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Type hints on fields | ➖ | ➖ | ➖ | ➖ | ⚠️ OPT | ⚠️ OPT | ⚠️ OPT |
| `compute_sql`, `init_storage`, callable `copy` | ➖ | ➖ | ➖ | ➖ | ➖ | ➖ | ✅ |
| Binary value type | base64 | base64 | base64 | base64 | base64 | base64 | `BinaryValue` |

### View Attributes

| Attribute | v14 | v15 | v16 | v17 | v18 | v19 | v20 |
|-----------|-----|-----|-----|-----|-----|-----|-----|
| `attrs` | ✅ | ✅ | ⚠️ DEP | ❌ REM | ❌ | ❌ | ❌ |
| `states` | ✅ | ✅ | ⚠️ DEP | ❌ REM | ❌ | ❌ | ❌ |
| Direct `invisible` | ➖ | ➖ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Direct `readonly` | ➖ | ➖ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Direct `required` | ➖ | ➖ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Python expressions | ➖ | ➖ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `<list>` (was `<tree>`) | ➖ | ➖ | ➖ | ➖ | ✅ | ✅ | ✅ |
| `<chatter/>` | ➖ | ➖ | ➖ | ➖ | ✅ | ✅ | ✅ |
| `card` view type, `<kanban card_id>` | ➖ | ➖ | ➖ | ➖ | ➖ | ➖ | ✅ |
| Button `icon="fa-..."` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ (Material Symbols names) |
| Server QWeb `t-esc` / `t-raw` | ✅ | ✅ | ⚠️ DEP | ⚠️ DEP | ⚠️ DEP | ⚠️ DEP | ❌ REM |

### x2many Operations

| Pattern | v14 | v15 | v16 | v17 | v18 | v19 | v20 |
|---------|-----|-----|-----|-----|-----|-----|-----|
| Tuple commands `(0, 0, {...})` | ✅ | ✅ | ⚠️ | ⚠️ | ⚠️ | ⚠️ | ⚠️ |
| `Command.create({...})` | ➖ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `Command.update(id, {...})` | ➖ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `Command.delete(id)` | ➖ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `Command.unlink(id)` | ➖ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `Command.link(id)` | ➖ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `Command.clear()` | ➖ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `Command.set([ids])` | ➖ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

Tuple commands are still accepted by the ORM (`Command` is an `IntEnum`); `Command` is the readable form.

### Model Attributes

| Attribute | v14 | v15 | v16 | v17 | v18 | v19 | v20 |
|-----------|-----|-----|-----|-----|-----|-----|-----|
| `_check_company_auto` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `_parent_store` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `_order` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `_rec_name` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `_sql_constraints` | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ ignored | ❌ ignored |
| `models.Constraint` / `models.Index` | ➖ | ➖ | ➖ | ➖ | ➖ | ✅ | ✅ |
| `_explanation` | ➖ | ➖ | ➖ | ➖ | ➖ | ➖ | ✅ |

### ORM Methods

| Method | v14 | v15 | v16 | v17 | v18 | v19 | v20 |
|--------|-----|-----|-----|-----|-----|-----|-----|
| `name_get()` | ✅ | ✅ | ✅ | ⚠️ DEP | ❌ REM | ❌ | ❌ |
| `check_access_rights` / `check_access_rule` | ✅ | ✅ | ✅ | ✅ | ⚠️ DEP | ⚠️ DEP | ❌ REM |
| `check_access` / `has_access` | ➖ | ➖ | ➖ | ➖ | ✅ | ✅ | ✅ |
| `read_group(domain, fields, groupby, lazy)` | ✅ | ✅ | ✅ | ✅ | ✅ | ⚠️ DEP | ❌ (new tuple signature) |
| `_read_group(domain, groupby, aggregates)` | ➖ | ➖ | ➖ | ✅ | ✅ | ✅ | ✅ |
| `toggle_active()` | ✅ | ✅ | ✅ | ✅ | ✅ | ⚠️ DEP | ❌ REM |
| `registry.clear_cache()` | ➖ | ➖ | ➖ | ✅ | ✅ | ✅ | ❌ (`transaction.invalidate_ormcache`) |

### SQL Operations

| Pattern | v14 | v15 | v16 | v17 | v18 | v19 | v20 |
|---------|-----|-----|-----|-----|-----|-----|-----|
| Raw SQL strings with params | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ (prefer `SQL`) |
| `SQL()` builder | ➖ | ➖ | ➖ | ✅ | ✅ | ✅ REC | ✅ REC |
| `SQL.identifier()` | ➖ | ➖ | ➖ | ✅ | ✅ | ✅ | ✅ |
| `odoo.osv.expression` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ REM (`fields.Domain`) |

### JavaScript/OWL

| Pattern | v14 | v15 | v16 | v17 | v18 | v19 | v20 |
|---------|-----|-----|-----|-----|-----|-----|-----|
| `odoo.define()` | ✅ | ⚠️ DEP | ❌ REM | ❌ | ❌ | ❌ | ❌ |
| ES modules | ➖ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `@odoo-module` annotation | ➖ | ✅ REQ | ✅ REQ | ✅ REQ | ⚠️ OPT | ⚠️ OPT | ⚠️ OPT |
| OWL 1.x | ➖ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| OWL 2.x | ➖ | ➖ | ✅ | ✅ | ✅ | ✅ | ❌ |
| OWL 3.x | ➖ | ➖ | ➖ | ➖ | ➖ | ➖ | ✅ |
| jQuery / `publicWidget` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ REM |
| `Interaction` (frontend) | ➖ | ➖ | ➖ | ➖ | ➖ | ✅ | ✅ |
| QUnit | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ REM |
| Hoot | ➖ | ➖ | ➖ | ➖ | ✅ | ✅ | ✅ |
| FontAwesome CSS | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ (Material Symbols) |

### Security/Rules

| Pattern | v14 | v15 | v16 | v17 | v18 | v19 | v20 |
|---------|-----|-----|-----|-----|-----|-----|-----|
| `ir.model.access.csv` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ REM |
| `ir.rule` records | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ REM |
| `security/ir.access.csv` (`ir.access`) | ➖ | ➖ | ➖ | ➖ | ➖ | ➖ | ✅ |
| `company_ids` in rule domains | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `allowed_company_ids` in rule domains | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `res.users.groups_id` / `res.groups.users` | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ (`group_ids` / `user_ids`) | ❌ |
| `res.groups.privilege` | ➖ | ➖ | ➖ | ➖ | ➖ | ✅ | ✅ |
| `base.group_everyone` | ➖ | ➖ | ➖ | ➖ | ➖ | ➖ | ✅ |

`allowed_company_ids` is a **context key**, not a variable of the rule evaluation context
(`user`, `time`, `company_ids`, `company_id`).

## Python Version Requirements

| Odoo Version | Python Min | Python Max / Recommended | PostgreSQL Min |
|--------------|------------|--------------------------|----------------|
| 14.0 | 3.6 | 3.8 | - |
| 15.0 | 3.7 | 3.10 | - |
| 16.0 | 3.7 | 3.10 | - |
| 17.0 | 3.10 | 3.11 | - |
| 18.0 | 3.10 | 3.12 | - |
| 19.0 | 3.10 | 3.12 | 13 |
| 20.0 | 3.12 | 3.12 (max 3.14) | 16 |

## Manifest Changes Across Versions

### v14-v15 Manifest
```python
{
    'name': 'Module',
    'version': '15.0.1.0.0',
    'depends': ['base'],
    'data': ['views/views.xml'],
}
```

### v16+ Manifest (Assets)
```python
{
    'name': 'Module',
    'version': '18.0.1.0.0',
    'depends': ['base'],
    'data': ['views/views.xml'],
    'assets': {
        'web.assets_backend': [
            'module/static/src/**/*.js',
            'module/static/src/**/*.xml',
            'module/static/src/**/*.scss',
        ],
    },
}
```

### v20 Manifest (security file)
```python
{
    'name': 'Module',
    'version': '20.0.1.0.0',
    'author': 'Your Company',
    'license': 'LGPL-3',
    'depends': ['mail'],
    'data': [
        'security/module_security.xml',
        'security/ir.access.csv',
        'views/views.xml',
    ],
    'assets': {
        'web.assets_backend': ['module/static/src/**/*'],
    },
}
```

## Migration Path Summary

### v14 → v15
1. Remove `@api.multi` decorator
2. Replace `track_visibility` with `tracking`
3. Adopt OWL 1.x for new components
4. Update Python to 3.7+

### v15 → v16
1. Adopt `Command` class for x2many
2. Move assets to manifest `assets` key
3. Start using direct `invisible`/`readonly`
4. Migrate to OWL 2.x patterns

### v16 → v17
1. **MUST** remove all `attrs` usage
2. **MUST** remove all `states` usage
3. **MUST** use `@api.model_create_multi`
4. Convert to Python expression syntax
5. Update Python to 3.10+

### v17 → v18
1. `<tree>` → `<list>`, `<chatter/>`
2. `check_access_rights`/`check_access_rule` → `check_access` (old ones deprecated)
3. Start using `SQL()` builder
4. Type hints optional (recommended on public APIs)

### v18 → v19
1. `_sql_constraints` → `models.Constraint` / `models.Index`
2. `groups_id` → `group_ids`, `users` → `user_ids`, categories → `res.groups.privilege`
3. `read_group` deprecated → `_read_group` / `formatted_read_group`
4. OWL stays 2.x (2.8)

### v19 → v20
1. **MUST** convert `ir.model.access.csv` + `ir.rule` to `security/ir.access.csv`
2. **MUST** migrate OWL 2 → OWL 3 (`useProps`, `proxy`, signals, `this.` in templates)
3. **MUST** replace server `t-esc`/`t-raw` by `t-out`; `t-call` parameters as attributes
4. **MUST** replace `ir.attachment.datas` by `raw`; Binary values via `BinaryBytes`
5. Rewrite `read_group` calls; drop removed ORM aliases
6. `pytz` → `zoneinfo`; FontAwesome → Material Symbols
7. Update Python to 3.12+ and PostgreSQL to 16+
8. Run `odoo-bin upgrade_code --from 19.0` and `--script owl3-migration`

## Quick Reference Cards

### v20 Model Template
```python
from odoo import api, fields, models
from odoo.fields import Domain


class MyModel(models.Model):
    _name = 'my.model'
    _description = "My Model"
    _check_company_auto = True

    _name_company_uniq = models.Constraint('UNIQUE(name, company_id)', "Name must be unique per company.")

    name = fields.Char(required=True)
    company_id = fields.Many2one('res.company', required=True, default=lambda self: self.env.company)

    @api.model_create_multi
    def create(self, vals_list):
        return super().create(vals_list)
```

### v20 View Template
```xml
<button name="action_confirm" type="object" icon="check" invisible="state != 'draft'"/>
<field name="partner_id" readonly="state != 'draft'" required="type == 'invoice'"/>
<kanban card_id="%(my_model_view_card)d"/>
```

### v20 OWL Template
```javascript
import { Component, proxy, t, useProps } from "@odoo/owl";
import { registry } from "@web/core/registry";
import { useService } from "@web/core/utils/hooks";

export class MyComponent extends Component {
    static template = "module.Component";   // template: this.state.x, t-out
    props = useProps({ action: t.object().optional() });
    state = proxy({ loading: false });
    setup() { this.orm = useService("orm"); }
}
registry.category("actions").add("module.action", MyComponent);
```

## Error Messages Reference

| Error | Version | Cause | Fix |
|-------|---------|-------|-----|
| `@api.multi is deprecated` | v14 | Decorator still used | Remove decorator |
| `attrs is not supported` | v17+ | Using `attrs` in view | Use direct attributes |
| `states is not supported` | v17+ | Using `states` in view | Use `invisible` expression |
| `create() expects vals_list` | v17+ | Old create signature | Use `@api.model_create_multi` |
| `Model attribute '_sql_constraints' is no longer supported` | v19+ | Old constraints | `models.Constraint` |
| `KeyError: 'ir.model.access'` | v20 | ACL CSV in the manifest | `security/ir.access.csv` |
| `Component "X" defines a static "props"...` | v20 | OWL 2 props | `useProps` |
| `ImportError: cannot import name 'content_disposition' from 'odoo.http'` | v20 | Moved helper | `odoo.http.stream` |
| `TypeError: ... use BinaryValue instead of bytes` | v20 | Raw bytes on Binary field | `BinaryBytes(...)` |

---

**IMPORTANT**: This reference is for comparison purposes. Always use version-specific files for actual implementation patterns.
