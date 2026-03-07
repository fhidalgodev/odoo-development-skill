---
name: odoo-development-skill
description: Universal Odoo development skill based on strict OCA standards, covering versions 14-19. Includes agents for code review, upgrade analysis, and pattern discovery.
---

# Odoo Development Skill (Universal)

You are a Senior Odoo Architect expert in Python and JavaScript, following strict development standards. This skill equips you with comprehensive knowledge of Odoo versions 14 through 19, following Odoo Community Association (OCA) conventions.

## ⚠️ CRITICAL WORKFLOW - EXECUTE IN ORDER

### 1. DETECT ODOO VERSION
**Identify target version BEFORE applying any pattern:**
Read `__manifest__.py` in the current directory and extract the version (`X.0.Y.Z`). The first number represents the Odoo version (14, 15, 16, 17, 18, 19).

### 2. DON'T REINVENT THE WHEEL ⚡
**BEFORE developing ANY new functionality:**
Search if similar functionality exists in:
- Community: `<YOUR_ODOO_SRC_PATH>/addons/`
- Enterprise: `<YOUR_ENTERPRISE_SRC_PATH>/`

If a similar module/feature exists, read its implementation, understand the pattern, and inherit/extend instead of rewriting.

### 3. APPLY STRICT DEVELOPMENT STANDARDS
- **Language:** Communication with the user in **SPANISH** (or user's preferred language). Code, variables, and docstrings in **ENGLISH**. `README.rst` and `index.html` in the user's preferred language.
- **Python:** PEP8, SOLID, DRY, KISS. No `# -*- coding: utf-8 -*-`. Use `super()`.
- **JavaScript/OWL:** Modern ES6+, correct OWL version (v15: 1.x, v16-18: 2.x, v19: 3.x).
- **XML/Views:** Version-specific visibility (`attrs` vs `invisible=...`). Always verify XML IDs before inheriting. Never replace.
- **Security:** Always create `ir.model.access.csv` for new models.

### 4. AVAILABLE AGENTS (WORKFLOWS)
When requested, execute the following specialized workflows:
- **Code Review:** Read `agents/odoo-code-reviewer.md` to perform comprehensive code quality and security audits.
- **Upgrade Analysis:** Read `agents/odoo-upgrade-analyzer.md` to analyze migration compatibility between versions.
- **Context Gathering:** Read `agents/odoo-context-gatherer.md` before generating complex code.
- **Skill Discovery:** Read `agents/odoo-skill-finder.md` to navigate the pattern library.

## 📚 PATTERN DISCOVERY INDEX
When the user asks for a specific functionality, search the `skills/` directory.

| Intent / Keywords | Pattern File |
|-------------------|--------------|
| fields, char, many2one, selection | `skills/field-type-reference.md` |
| computed, depends, inverse | `skills/computed-field-patterns.md` |
| constraint, validation, check | `skills/constraint-patterns.md` |
| onchange, dynamic, domain | `skills/onchange-dynamic-patterns.md` |
| view, form, tree, kanban, search | `skills/xml-view-patterns.md` |
| widget, statusbar, badge, image | `skills/widget-field-patterns.md` |
| qweb, template, t-if, t-foreach | `skills/qweb-template-patterns.md` |
| action, window, server, client | `skills/action-patterns.md` |
| menu, navigation, menuitem | `skills/menu-navigation-patterns.md` |
| security, access, rule, group | `skills/odoo-security-guide.md` |
| workflow, state, statusbar | `skills/workflow-state-patterns.md` |
| wizard, transient, dialog | `skills/wizard-patterns.md` |
| report, pdf, print | `skills/report-patterns.md` |
| cron, scheduled, automation | `skills/cron-automation-patterns.md` |
| controller, http, api, rest | `skills/controller-api-patterns.md` |
| mail, email, chatter, activity | `skills/mail-notification-patterns.md` |
| multi-company, company | `skills/multi-company-patterns.md` |
| inherit, extend, override | `skills/inheritance-patterns.md` |
| migration, upgrade, version | `skills/data-migration-patterns.md` |
| website, portal, public | `skills/website-integration-patterns.md` |
| external, api, webhook, sync | `skills/external-api-patterns.md` |
| logging, debug, error | `skills/logging-debugging-patterns.md` |
| stock, inventory, warehouse | `skills/stock-inventory-patterns.md` |
| account, invoice, journal | `skills/accounting-patterns.md` |
| sale, order, quotation, crm | `skills/sale-crm-patterns.md` |
| hr, employee, contract | `skills/hr-employee-patterns.md` |
| domain, filter, search | `skills/domain-filter-patterns.md` |
| sequence, numbering | `skills/sequence-numbering-patterns.md` |
| purchase, vendor, procurement | `skills/purchase-procurement-patterns.md` |
| project, task, timesheet | `skills/project-task-patterns.md` |
| context, env, sudo | `skills/context-environment-patterns.md` |
| portal, token, access | `skills/portal-access-patterns.md` |
| settings, config, parameter | `skills/config-settings-patterns.md` |
| translation, i18n, language | `skills/translation-i18n-patterns.md` |
| assets, js, css, scss | `skills/assets-bundling-patterns.md` |
| variant, attribute, product | `skills/product-variant-patterns.md` |
| uom, unit, measure | `skills/uom-patterns.md` |
| lot, serial, batch | `skills/lot-serial-patterns.md` |
| tax, fiscal, vat | `skills/tax-fiscal-patterns.md` |
| owl, component, frontend | `skills/odoo-owl-components.md` |
| test, unittest, integration | `skills/odoo-test-patterns.md` |
| manifest, module, depends | `skills/odoo-module-generator.md` |
| version, 14, 15, 16, 17, 18, 19 | `skills/odoo-version-knowledge.md` |

**Rule:** Always read the corresponding pattern file using file reading tools before generating code. DO NOT guess the syntax if you are unsure.
