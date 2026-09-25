# Odoo Model Patterns - Version Dispatcher

## CRITICAL: VERSION-SPECIFIC REQUIREMENTS

```
╔══════════════════════════════════════════════════════════════════════════════╗
║                                                                              ║
║   ⚠️  MANDATORY VERSION MATCHING ⚠️                                          ║
║                                                                              ║
║   You MUST use the version-specific model patterns that match your           ║
║   target Odoo version. Using patterns from the wrong version WILL            ║
║   cause errors or deprecated code warnings.                                  ║
║                                                                              ║
║   BEFORE implementing ANY model code, identify your target Odoo version      ║
║   and load the corresponding file. This is NOT optional.                     ║
║                                                                              ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

## Version-Specific Files

| Target Version | File to Use | Status |
|----------------|-------------|--------|
| Odoo 14.0 | `odoo-model-patterns-14.md` | Legacy |
| Odoo 15.0 | `odoo-model-patterns-15.md` | Legacy |
| Odoo 16.0 | `odoo-model-patterns-16.md` | Supported |
| Odoo 17.0 | `odoo-model-patterns-17.md` | Supported |
| Odoo 18.0 | `odoo-model-patterns-18.md` | Supported |
| Odoo 19.0 | `odoo-model-patterns-19.md` | Supported |
| Odoo 20.0 | `odoo-model-patterns-20.md` | Current |
| All versions | `odoo-model-patterns-all.md` | Core concepts |

## Migration Guides

| Migration Path | File |
|----------------|------|
| 14.0 → 15.0 | `odoo-model-patterns-14-15.md` |
| 15.0 → 16.0 | `odoo-model-patterns-15-16.md` |
| 16.0 → 17.0 | `odoo-model-patterns-16-17.md` |
| 17.0 → 18.0 | `odoo-model-patterns-17-18.md` |
| 18.0 → 19.0 | `odoo-model-patterns-18-19.md` |
| 19.0 → 20.0 | `odoo-model-patterns-19-20.md` |

## Quick Reference: Major Model Pattern Changes

### v14 Patterns
- `@api.multi` deprecated (still works)
- `track_visibility='onchange'`
- Single record `create(vals)`

### v15 Patterns
- `@api.multi` removed
- `tracking=True` replaces `track_visibility`
- Simplified chatter

### v16 Patterns
- `Command` class for x2many
- `@api.model_create_multi` recommended

### v17 Patterns
- `@api.model_create_multi` mandatory
- Enhanced ORM methods

### v18 Patterns
- `_check_company_auto = True`
- `check_company=True` on fields
- Type hints recommended
- `SQL()` builder recommended

### v19 Patterns
- `models.Constraint` / `models.Index` (`_sql_constraints` ignored)
- `res.users.group_ids`, `res.groups.user_ids`, `res.groups.privilege`
- `SQL()` builder recommended, type hints encouraged (not enforced)

### v20 Patterns
- `read_group` new tuple contract; `_read_group` in backend code
- Deprecated v18/v19 aliases removed (`check_access_rights`, `toggle_active`, `_check_recursion`...)
- `BinaryValue`/`BinaryBytes`, `ir.attachment.raw` (no `datas`)
- `zoneinfo` instead of `pytz`; `@api.ormcache`; `env.transaction.invalidate_ormcache()`
- Mail tracking hooks renamed (`_track_log_get_default_subtype`)

## Version Detection in Existing Code

| Indicator | Version |
|-----------|---------|
| `@api.multi` decorator | 14.0 |
| `track_visibility` | 14.0 |
| `tracking=True` | 15.0+ |
| Tuple syntax for x2many | 14.0-15.0 |
| `Command` class | 16.0+ |
| `_check_company_auto` | 18.0+ |
| Type hints on fields | 18.0+ |
| `models.Constraint` attributes | 19.0+ |
| `BinaryBytes`, `zoneinfo`, `@api.ormcache` | 20.0+ |

---

**REMINDER**: Always load the version-specific file before implementing model patterns.
