# Odoo OWL Components - Version 20.0 (OWL 3)

```
╔══════════════════════════════════════════════════════════════════════════════╗
║  ODOO 20.0 OWL 3 COMPONENT PATTERNS                                          ║
║  This file contains ONLY Odoo 20.0 frontend patterns.                        ║
║  Bundled Owl: 3.0.0-alpha.49 (web/static/lib/owl/owl.js + owl.d.ts)          ║
║  DO NOT use these patterns for Odoo 16-19 (Owl 2.x).                         ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

## Version 20.0 Frontend Stack

| Item | Value |
|------|-------|
| Owl | 3.x (signals, `proxy`, `useProps` + `t` schemas, plugins) |
| Compatibility layer | `web/static/src/owl2/owl3_compatibility_layer.js` + `@web/owl2/utils` (temporary) |
| Modules | Every file in `static/src` and `static/tests` is an ES module; `/** @odoo-module **/` is optional (`@odoo-module ignore` opts out) |
| Removed | jQuery, legacy `publicWidget`/`web/static/src/legacy`, QUnit, FontAwesome CSS |
| Tests | Hoot (`@odoo/hoot`, `@odoo/hoot-dom`, `@odoo/hoot-mock`) |
| Icons | `<i class="oi" data-icon="<material_symbol>"/>` |
| Official rules | `<odoo_src>/skills/odoo-web-guidelines/` (organize by feature, avoid getters, avoid `patch` inside Odoo) |

## Owl 2 → Owl 3 Cheat Sheet

| Owl 2 (Odoo 16-19) | Owl 3 (Odoo 20) |
|--------------------|-----------------|
| `static props = { x: { type: String, optional: true } }` | `props = useProps({ x: t.string().optional() })` |
| `static defaultProps = { x: "a" }` | `x: t.string().optional("a")` |
| `static props = { ...Parent.props, y: ... }` | `props = useProps({ ...parentProps, y: t.number() })` (exported schema) |
| `useState({...})`, `reactive({...})` | `proxy({...})` |
| - | `signal(value)` → read `sig()`, write `sig.set(v)`; `computed(() => ...)` |
| `useRef("name")` + `t-ref="name"` + `ref.el` | `name = signal.ref()` + `t-ref="this.name"` + `this.name()` |
| `useExternalListener(target, "ev", fn)` | `useListener(target, "ev", fn)` |
| `useEffect(fn, () => [deps])` | `useLayoutEffect(fn, () => [deps])` from `@web/owl2/utils` (compat), `useOnChange(() => [deps], fn)` or Owl 3 `useEffect(fn)` (auto-tracked) |
| `useComponent()` | `useScope().component` |
| `useEnv()`, `useSubEnv()`, `onWillRender()` | from `@web/owl2/utils` (compat) |
| `useChildSubEnv()`, `onRendered()` | removed |
| `props.x`, `state.y`, `onClick` in templates | `this.props.x`, `this.state.y`, `this.onClick` |
| `t-esc="value"` | `t-out="value"` (`t-esc` is a deprecated alias) |
| `t-slot="default"` | `t-call-slot="default"` (`t-set-slot` unchanged) |
| `t-model="state.value"` | `t-model="this.valueSignal"` (signal) or `t-model.proxy="this.state.value"` |
| `t-portal="selector"` | Owl 3 `Portal` / `t-custom-portal` (compat) |
| `useService("x")` | still works; `usePlugin(XPlugin)` for services converted to plugins |
| `registry.category("services").add(...)` | still supported (legacy starter); new global services: `class X extends Plugin` + `services.add(X)` |

Automatic rewrite: `./odoo-bin upgrade_code --script owl3-migration --addons-path=<addons>` (review the diff).

## Basic Component

```javascript
import { Component, onWillStart, proxy, signal, t, useProps } from "@odoo/owl";
import { _t } from "@web/core/l10n/translation";
import { registry } from "@web/core/registry";
import { useService } from "@web/core/utils/hooks";
import { Layout } from "@web/search/layout";
import { standardActionServiceProps } from "@web/webclient/actions/action_plugin";

export class LibraryDashboard extends Component {
    static template = "library.LibraryDashboard";
    static components = { Layout };

    props = useProps(standardActionServiceProps);
    state = proxy({ books: [], loading: true });
    search = signal("");

    setup() {
        this.orm = useService("orm");
        this.action = useService("action");
        this.notification = useService("notification");
        onWillStart(() => this.loadBooks());
    }

    async loadBooks() {
        this.state.loading = true;
        this.state.books = await this.orm.searchRead(
            "library.book",
            [["name", "ilike", this.search()]],
            ["name", "state", "author_id"],
            { limit: 80 }
        );
        this.state.loading = false;
    }

    layoutDisplay() {
        return { controlPanel: {} };
    }

    async onSearch() {
        await this.loadBooks();
        this.notification.add(_t("%s books found", this.state.books.length), { type: "info" });
    }

    openBook(bookId) {
        return this.action.doAction({
            type: "ir.actions.act_window",
            res_model: "library.book",
            res_id: bookId,
            views: [[false, "form"]],
        });
    }
}

registry.category("actions").add("library.dashboard", LibraryDashboard);
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<templates xml:space="preserve">
    <t t-name="library.LibraryDashboard">
        <Layout display="this.layoutDisplay()">
            <div class="o_library_dashboard p-3">
                <div class="d-flex gap-2 mb-3">
                    <input class="form-control" t-model="this.search" placeholder="Search a book"/>
                    <button class="btn btn-primary" t-on-click="this.onSearch">
                        <i class="oi" data-icon="search" title="Search"/>
                    </button>
                </div>
                <t t-if="this.state.loading">
                    <i class="oi oi-spin" data-icon="progress_activity" title="Loading"/>
                </t>
                <t t-else="">
                    <t t-foreach="this.state.books" t-as="book" t-key="book.id">
                        <div class="o_library_dashboard_card p-2 border rounded mb-2"
                             t-on-click="() => this.openBook(book.id)">
                            <span class="fw-bold" t-out="book.name"/>
                            <span t-if="book.author_id" class="text-muted ms-2" t-out="book.author_id[1]"/>
                        </div>
                    </t>
                </t>
            </div>
        </Layout>
    </t>
</templates>
```

Template rules in 20.0:
- Component members need `this.` (`this.props`, `this.state`, `this.method`); loop/`t-set` variables do not.
- Output with `t-out` (escaped unless the value is `markup`).
- Handlers: `t-on-click="this.method"` or an arrow function.
- CSS classes: `o_<module>_...` prefix, flat names.
- Check that an icon name exists in `addons/web/tooling/icons/icons_wishlist.txt` (shipped subset).

## Reactivity Primitives

```javascript
import { computed, effect, proxy, signal, untrack, useEffect, useOnChange } from "@odoo/owl";

const count = signal(0);           // count() to read, count.set(1) to write
const double = computed(() => count() * 2);
const items = signal.Array([]);    // also signal.Map, signal.Set, signal.Object
const model = proxy({ lines: [] }); // deep reactive object (replaces useState/reactive)

// inside setup():
useEffect(() => {                  // re-runs when any reactive value read inside changes
    document.title = `${count()} books`;
});
useOnChange(
    () => [this.props.bookId],     // only these dependencies are tracked
    (bookId) => this.loadBook(bookId)
);
const snapshot = untrack(() => count()); // read without subscribing
```

Official guideline: prefer a plain method or a `computed` over a JavaScript getter.

## Refs, Listeners and Lifecycle

```javascript
import { Component, onMounted, onWillUnmount, signal, useListener, useProps, t } from "@odoo/owl";

export class LibrarySearchBox extends Component {
    static template = "library.LibrarySearchBox";
    props = useProps({ onSearch: t.function(), placeholder: t.string().optional("") });
    input = signal.ref();

    setup() {
        useListener(window, "keydown", (ev) => this.onWindowKeydown(ev));
        onMounted(() => this.input()?.focus());
        onWillUnmount(() => {
            // cleanup
        });
    }

    onWindowKeydown(ev) {
        if (ev.key === "/" && this.input()) {
            ev.preventDefault();
            this.input().focus();
        }
    }

    onInputKeydown(ev) {
        if (ev.key === "Enter") {
            this.props.onSearch(this.input().value);
        }
    }
}
```

```xml
<t t-name="library.LibrarySearchBox">
    <input t-ref="this.input" class="form-control" t-att-placeholder="this.props.placeholder"
           t-on-keydown="this.onInputKeydown"/>
</t>
```

Event modifiers: `.stop`, `.prevent`, `.self`, `.capture`, `.synthetic`, `.passive` (no key modifiers).

Lifecycle hooks available: `onWillStart`, `onMounted`, `onWillUpdateProps`, `onWillPatch`, `onPatched`,
`onWillUnmount`, `onWillDestroy`, `onError`.

## Props Schemas (`t`)

```javascript
import { t, useProps } from "@odoo/owl";

export const libraryBadgeProps = {
    label: t.string(),
    count: t.number().optional(0),
    kind: t.selection(["info", "warning", "danger"]).optional("info"),
    tags: t.array(t.string()).optional([]),
    book: t.object({ id: t.number(), name: t.string() }).optional(),
    onClick: t.function().optional(),
    slots: t.object().optional(),
};

export class LibraryBadge extends Component {
    static template = "library.LibraryBadge";
    props = useProps(libraryBadgeProps);
}
```

Types: `t.any`, `t.string`, `t.number`, `t.boolean`, `t.array`, `t.object`, `t.strictObject`,
`t.record`, `t.tuple`, `t.function`, `t.instanceOf`, `t.constructor`, `t.literal`, `t.selection`,
`t.or`, `t.and`, `t.promise`, `t.signal`, `t.ref`, `t.customValidator`, plus `.optional(default)`.
Export the schema so subclasses and wrappers can spread it.

## Field Widget

```javascript
import { Component, t, useProps } from "@odoo/owl";
import { _t } from "@web/core/l10n/translation";
import { registry } from "@web/core/registry";
import { standardFieldProps } from "@web/views/fields/standard_field_props";

export const libraryRatingFieldProps = {
    ...standardFieldProps,
    maxStars: t.number().optional(5),
};

export class LibraryRatingField extends Component {
    static template = "library.LibraryRatingField";
    props = useProps(libraryRatingFieldProps);

    value() {
        return this.props.record.data[this.props.name] || 0;
    }

    stars() {
        return Array.from({ length: this.props.maxStars }, (_, index) => index + 1);
    }

    select(star) {
        if (!this.props.readonly) {
            this.props.record.update({ [this.props.name]: star });
        }
    }
}

registry.category("fields").add("library_rating", {
    component: LibraryRatingField,
    displayName: _t("Rating"),
    supportedTypes: ["integer"],
    extractProps: ({ options }) => ({ maxStars: options.max_stars }),
});
```

```xml
<t t-name="library.LibraryRatingField">
    <div class="o_library_rating_field d-flex gap-1">
        <t t-foreach="this.stars()" t-as="star" t-key="star">
            <i class="oi" t-att-class="{ 'oi-filled text-warning': star &lt;= this.value() }"
               data-icon="star" t-att-title="star" t-on-click="() => this.select(star)"/>
        </t>
    </div>
</t>
```

Usage in a view: `<field name="rating" widget="library_rating" options="{'max_stars': 10}"/>`.

## Extending Existing Components

```javascript
import { useProps } from "@odoo/owl";
import { CharField, charFieldProps } from "@web/views/fields/char/char_field";
import { registry } from "@web/core/registry";

export class LibraryIsbnField extends CharField {
    static template = "web.CharField";
    props = useProps({ ...charFieldProps });   // redeclare with the exported schema

    parse(value) {
        return super.parse(value).replaceAll("-", "");
    }
}

registry.category("fields").add("library_isbn", { component: LibraryIsbnField, supportedTypes: ["char"] });
```

- Views: `{ ...listView, Controller: MyListController }` registered in `registry.category("views")` and used
  with `js_class`.
- `patch(Target.prototype, {...})` from `@web/core/utils/patch` still exists; the official rule discourages it
  inside Odoo and prefers real extension points. Class fields (`props = useProps(...)`) cannot be patched:
  subclass instead.

## Services and Plugins

```javascript
// 1. Legacy registry service (still supported in 20.0)
import { registry } from "@web/core/registry";

registry.category("services").add("library_stats", {
    dependencies: ["orm"],
    start(env, { orm }) {
        return { fetch: () => orm.call("library.book", "get_stats") };
    },
});
// consumer: this.stats = useService("library_stats");

// 2. Owl 3 plugin (the direction core is taking)
import { Plugin, usePlugin } from "@odoo/owl";
import { services } from "@web/core/services";
import { ORM } from "@web/core/orm_plugin";

export class LibraryStatsPlugin extends Plugin {
    orm = usePlugin(ORM);

    fetch() {
        return this.orm.call("library.book", "get_stats");
    }
}
services.add(LibraryStatsPlugin);
// consumer: this.stats = usePlugin(LibraryStatsPlugin);
```

Core services already converted to plugins (import paths): `ORM` (`@web/core/orm_plugin`),
`ActionManagerPlugin` (`@web/webclient/actions/action_plugin`), `NotificationPlugin`
(`@web/core/notifications/notification_plugin`), `DialogPlugin` (`@web/core/dialog/dialog_plugin`),
`EffectPlugin`, `HotkeyPlugin`, `OverlayPlugin`, `PopoverPlugin`, `UIPlugin`, `TitlePlugin`.
`useService("orm" | "action" | "notification" | "dialog" | ...)` keeps working through wrappers.

## Other Registries (unchanged names)

`actions`, `fields`, `views`, `systray`, `main_components`, `user_menuitems`, `command_provider`,
`effects`, `formatters`, `parsers`, `lazy_components`, `public.interactions`, `public.interactions.edit`.

## Frontend (website / portal): Interactions

```javascript
import { Interaction } from "@web/public/interaction";
import { registry } from "@web/core/registry";

export class LibraryReserveButton extends Interaction {
    static selector = ".o_library_reserve";
    dynamicContent = {
        _root: {
            "t-on-click": this.onClick,
            "t-att-class": () => ({ disabled: this.reserved }),
        },
    };

    setup() {
        this.reserved = false;
    }

    async onClick() {
        this.reserved = true;
        await this.waitFor(fetch(`/library/reserve/${this.el.dataset.bookId}`, { method: "POST" }));
    }
}

registry.category("public.interactions").add("library.reserve_button", LibraryReserveButton);
```

`publicWidget` and jQuery no longer exist in 20.0: every frontend behaviour is an `Interaction`.

## Hoot Tests

```javascript
import { expect, test } from "@odoo/hoot";
import { click } from "@odoo/hoot-dom";
import { animationFrame } from "@odoo/hoot-mock";
import { defineModels, fields, models, mountView } from "@web/../tests/web_test_helpers";

class LibraryBook extends models.Model {
    _name = "library.book";
    name = fields.Char();
    rating = fields.Integer();
    _records = [{ id: 1, name: "Dune", rating: 2 }];
}

defineModels({ LibraryBook });

test("rating widget updates the value", async () => {
    await mountView({
        resModel: "library.book",
        resId: 1,
        type: "form",
        arch: `<form><field name="rating" widget="library_rating"/></form>`,
    });
    await click(".o_library_rating_field .oi:nth-child(4)");
    await animationFrame();
    expect(".o_library_rating_field .oi-filled").toHaveCount(4);
});
```

Manifest: `'web.assets_unit_tests': ['library/static/tests/**/*']`; tours in `web.assets_tests`.

## Assets (manifest)

```python
'assets': {
    'web.assets_backend': [
        'library/static/src/**/*',
    ],
    'web.assets_frontend': [
        'library/static/src/interactions/**/*',
    ],
    'web.assets_unit_tests': [
        'library/static/tests/**/*',
    ],
},
```

Organize `static/src` by feature (`dashboard/dashboard.js|xml|scss`), not by type
(`components/`, `services/`). Third-party libraries in `static/lib/`, unminified.

## v20 OWL Checklist

- [ ] No `static props` / `static defaultProps` (they throw); `props = useProps(schema)` with `t`
- [ ] No `useState`, `useRef`, `reactive`, `useExternalListener`, `useChildSubEnv`, `onRendered`
- [ ] `this.` in templates for component members; `t-out` instead of `t-esc`; `t-call-slot`
- [ ] Refs with `signal.ref()` + `t-ref="this.ref"`
- [ ] Icons with `oi` + `data-icon`, names present in the shipped subset
- [ ] `standardActionServiceProps` from `@web/webclient/actions/action_plugin`
- [ ] Frontend code as `Interaction`, no jQuery
- [ ] Hoot tests in `web.assets_unit_tests`

## AI Agent Instructions (v20 OWL)

1. **USE** `useProps` + `t`, `proxy`, `signal`, `computed`, `signal.ref()`, `useListener`.
2. **PREFIX** component members with `this.` in templates; **USE** `t-out` and `t-call-slot`.
3. **KEEP** `useService` for existing services; **USE** `usePlugin` when a plugin exists.
4. **IMPORT** compat helpers (`useLayoutEffect`, `useEnv`, `useSubEnv`, `onWillRender`) only from `@web/owl2/utils`, and treat them as transitional.
5. **DO NOT** generate OWL 2 syntax, jQuery, `publicWidget`, QUnit tests or FontAwesome classes.
6. **VERIFY** hooks and types in `addons/web/static/lib/owl/owl.d.ts` of the 20.0 tree.
