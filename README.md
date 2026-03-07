# Odoo Development Universal Skill

A universal Odoo development skill for AI agents compatible with `skills.sh`. It provides fast code indexing, intelligent patterns, and strict adherence to Odoo Community Association (OCA) standards for versions 14 to 19.

This repository is a fork and adaptation of the Odoo plugin from [letzdoo/claude-marketplace](https://github.com/letzdoo/claude-marketplace), restructured to be 100% agnostic and compatible with any IDE that supports `skills.sh` (like Windsurf, Cursor, Cline, etc.).

## 🌟 Features

- **Version Awareness**: Covers Odoo 14 to 19, including breaking changes and deprecated methods.
- **OCA Standards Strict Adherence**: Enforces PEP8, DRY, KISS, and SOLID principles.
- **OWL Compatibility**: Knowledge of OWL 1.x (v15), 2.x (v16-18), and 3.x (v19).
- **Specialized Agents**: Includes 4 workflows (agents) for Context Gathering, Code Review, Upgrade Analysis, and Skill Discovery.
- **123 Skill Patterns**: Detailed copy-paste ready code snippets for every functional domain in Odoo.

## 📦 Installation

To install this skill globally in your environment using `skills.sh`:

```bash
npx skills add fhidalgodev/odoo-development-skill --global
```

Or install it for a specific project:

```bash
npx skills add fhidalgodev/odoo-development-skill
```

## 🧠 Architecture

```text
odoo-development-skill/
├── SKILL.md                 # Main entrypoint for skills.sh
├── README.md                # This documentation
├── agents/                  # 4 refactored workflows
│   ├── odoo-context-gatherer.md
│   ├── odoo-code-reviewer.md
│   ├── odoo-upgrade-analyzer.md
│   └── odoo-skill-finder.md
└── skills/                  # 123 pattern files
    ├── odoo-version-knowledge.md
    ├── field-type-reference.md
    ├── xml-view-patterns.md
    └── ...
```

## 🛠️ Usage

Once installed, your AI assistant will automatically read `SKILL.md` when Odoo-related tasks are requested. The assistant will:
1. Detect your Odoo version automatically.
2. Check existing Enterprise or Community modules before reinventing the wheel.
3. Fetch the exact pattern needed from the `skills/` folder.
4. Implement the solution adhering to OCA guidelines.

## 🤝 Contributing

Contributions, issues, and feature requests are welcome!

## 📝 License

This project is licensed under the MIT License - see the LICENSE file for details.
