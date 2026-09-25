# Odoo OWL Migration Guide: 19.0 → 20.0 (Owl 2.8 → Owl 3)

```
╔══════════════════════════════════════════════════════════════════════════════╗
║  OWL MIGRATION GUIDE: 2.x → 3.x                                              ║
║  Odoo 19 ships Owl 2.8.x; Odoo 20 ships Owl 3.0 (alpha) + a compat layer    ║
║  Real example: web.CharField in both versions                                ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

> Owl 3 arrives with Odoo **20.0**. Odoo 19.0 still runs Owl 2 (`web/static/lib/owl/owl.js` 2.8.x).

## What the Compatibility Layer Does (and Does Not)

`web/static/src/owl2/owl3_compatibility_layer.js` is loaded right after Owl 3 in every bundle. It:

- re-adds `useEnv`, `useSubEnv`, `onWillRender`, `useComponent` and an Owl 2 style `useLayoutEffect(fn, deps)`;
- keeps `this.env` on components and `env` on `mount()`/`App`;
- supports `t-custom-portal` and `t-custom-model`.

It does **not** keep `static props`/`static defaultProps` (the component constructor throws), `useState`,
`reactive`, `useRef`, `useExternalListener`, `useChildSubEnv`, `onRendered`, nor implicit component scope in
templates. Treat `@web/owl2/utils` imports as transitional.

## Automatic Rewrite

```bash
./odoo-bin upgrade_code --script owl3-migration --addons-path=/path/to/addons --glob '**/my_module/**'
```

Steps performed: `useEffect` → `useLayoutEffect` (compat), `onWillRender`/`onRendered`/`useComponent`/`useEnv`/
`useSubEnv`/`useChildSubEnv`/`useRef`/`useExternalListener` imports moved to `@web/owl2/utils`,
`useState`/`reactive` → `proxy`, `t-portal` → `t-custom-portal`, `t-esc` → `t-out`, `t-ref` → `t-custom-ref`,
`t-model` → `t-custom-model`, `this.` added to template variables, `t-slot` → `t-call-slot`, parametric
`t-call` rewrite, and `useService(...)` → `usePlugin(...)` for converted services.

It produces compat-flavoured code (`t-custom-ref`, `@web/owl2/utils` imports of hooks that the current compat
module no longer exports): finish by hand with the native Owl 3 forms below.

## Real Example: `web.CharField`

### Before (Odoo 19, Owl 2)

```javascript
import { Component, useEffect, useExternalListener, useRef } from "@odoo/owl";
import { standardFieldProps } from "../standard_field_props";

export class CharField extends Component {
    static template = "web.CharField";
    static props = {
        ...standardFieldProps,
        autocomplete: { type: String, optional: true },
        isPassword: { type: Boolean, optional: true },
        placeholder: { type: String, optional: true },
        dynamicPlaceholder: { type: Boolean, optional: true },
    };
    static defaultProps = { dynamicPlaceholder: false };

    setup() {
        this.input = useRef("input");
        if (this.props.dynamicPlaceholder) {
            useExternalListener(document, "keydown", this.dynamicPlaceholder.onKeydown);
            useEffect(() => this.dynamicPlaceholder.updateModel(...));
        }
    }

    onBlur() {
        this.selectionStart = this.input.el.selectionStart;
    }
}
```

```xml
<t t-name="web.CharField">
    <t t-if="props.readonly">
        <span t-esc="formattedValue"/>
    </t>
    <t t-else="">
        <input class="o_input" t-att-placeholder="props.placeholder" t-on-blur="onBlur" t-ref="input"/>
    </t>
</t>
```

### After (Odoo 20, Owl 3)

```javascript
import { Component, onMounted, onPatched, signal, t, useListener, useProps } from "@odoo/owl";
import { standardFieldProps } from "../standard_field_props";

export const charFieldProps = {
    ...standardFieldProps,
    autocomplete: t.string().optional(),
    isPassword: t.boolean().optional(),
    placeholder: t.string().optional(),
    dynamicPlaceholder: t.boolean().optional(false),
};

export class CharField extends Component {
    static template = "web.CharField";
    props = useProps(charFieldProps);
    input = signal.ref();

    setup() {
        if (this.props.dynamicPlaceholder) {
            useListener(document, "keydown", this.dynamicPlaceholder.onKeydown);
            const updateModel = () => this.dynamicPlaceholder.updateModel(...);
            onMounted(updateModel);
            onPatched(updateModel);
        }
    }

    onBlur() {
        if (this.input()) {
            this.selectionStart = this.input().selectionStart;
        }
    }
}
```

```xml
<t t-name="web.CharField">
    <t t-if="this.props.readonly">
        <span t-out="this.formattedValue"/>
    </t>
    <t t-else="">
        <input class="o_input" t-att-placeholder="this.props.placeholder" t-on-blur="this.onBlur" t-ref="this.input"/>
    </t>
</t>
```

## Mapping Table

| Owl 2 (19.0) | Owl 3 (20.0) | Notes |
|--------------|--------------|-------|
| `static props = {...}` | `props = useProps({...})` | Throws if left as static |
| `{ type: String, optional: true }` | `t.string().optional()` | Types: `t.number`, `t.boolean`, `t.object`, `t.array`, `t.function`, `t.instanceOf`, `t.or`, `t.selection`... |
| `static defaultProps = { x: 1 }` | `x: t.number().optional(1)` | Defaults live in the schema |
| `static props = ["a", "b?"]` | `useProps({ a: t.any(), b: t.any().optional() })` or `useProps(["a", "b"])` | |
| `...Parent.props` | `...parentProps` | Export schemas as constants |
| `useState(obj)` / `reactive(obj)` | `proxy(obj)` | |
| - | `signal(v)`, `computed(fn)`, `signal.Array/Map/Set/Object` | Fine-grained reactivity |
| `useRef("x")` / `ref.el` | `x = signal.ref()` / `this.x()` | `t-ref="this.x"` |
| `useExternalListener(el, ev, fn)` | `useListener(el, ev, fn)` | Target may be a ref signal |
| `useEffect(fn, depsFn)` | `useLayoutEffect(fn, depsFn)` (compat) / `useOnChange(depsFn, fn)` / `useEffect(fn)` | Owl 3 `useEffect` has no deps argument |
| `useComponent()` | `useScope().component` | |
| `useEnv()` / `useSubEnv()` / `onWillRender()` | `@web/owl2/utils` | Transitional |
| `useChildSubEnv()` | `useSubEnv()` (compat) or plugins (`providePlugins`) | |
| `onRendered()` | `onPatched()` / `onMounted()` | |
| `t-esc` | `t-out` | |
| `t-slot` | `t-call-slot` | |
| `t-model="state.x"` | `t-model="this.xSignal"` / `t-model.proxy="this.state.x"` | |
| `t-portal` | `Portal` component / `t-custom-portal` | |
| Implicit component scope (`props.x`) | `this.props.x` | Loop and `t-set` variables stay bare |
| `useService("action")` | `useService("action")` or `usePlugin(ActionManagerPlugin)` | Both valid in 20.0 |
| `@web/core/orm_service` | `@web/core/orm_plugin` | `ORM` plugin class |
| `@web/webclient/actions/action_service` | `@web/webclient/actions/action_plugin` | `standardActionServiceProps` |
| `useAutofocus({ refName })` | `useAutofocus({ ref })` | Ref is a signal |
| `useChildRef`, `useForwardRefToParent`, `useRefListener` | removed | Pass ref signals as props |

## Services → Plugins

```javascript
// v19 service
registry.category("services").add("library_stats", {
    dependencies: ["orm"],
    start(env, { orm }) {
        return { fetch: () => orm.call("library.book", "get_stats") };
    },
});

// v20: the same service still works; the Owl 3 plugin form is
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
```

## Frontend Widgets → Interactions

```javascript
// v19 (legacy, removed in 20.0)
import publicWidget from "@web/legacy/js/public/public_widget";
publicWidget.registry.LibraryReserve = publicWidget.Widget.extend({
    selector: ".o_library_reserve",
    events: { click: "_onClick" },
    _onClick() { ... },
});

// v20
import { Interaction } from "@web/public/interaction";
import { registry } from "@web/core/registry";

export class LibraryReserve extends Interaction {
    static selector = ".o_library_reserve";
    dynamicContent = { _root: { "t-on-click": this.onClick } };
    onClick() { ... }
}
registry.category("public.interactions").add("library.reserve", LibraryReserve);
```

## Icons in Templates

```xml
<!-- v19 -->
<i class="fa fa-trash" title="Delete"/>
<i class="fa fa-spinner fa-spin"/>
<!-- v20 -->
<i class="oi" data-icon="delete" title="Delete"/>
<i class="oi oi-spin" data-icon="progress_activity" title="Loading"/>
```

## Migration Checklist

- [ ] Run `upgrade_code --script owl3-migration` and review
- [ ] Replace every `static props`/`defaultProps` by `useProps` with exported `t` schemas
- [ ] `useState`/`reactive` → `proxy`; refs → `signal.ref()`; listeners → `useListener`
- [ ] `useEffect(fn, deps)` → `useOnChange`/`useEffect`/compat `useLayoutEffect`
- [ ] Templates: `this.` prefix, `t-out`, `t-call-slot`, `t-ref="this.x"`
- [ ] Imports: `orm_plugin`, `action_plugin`
- [ ] `publicWidget`/jQuery → `Interaction` + DOM APIs
- [ ] FontAwesome → Material Symbols
- [ ] QUnit → Hoot

## Common Migration Errors

| Error | Fix |
|-------|-----|
| `Component "X" defines a static "props" or "defaultProps", which Owl 3 ignores...` | `props = useProps({...})` |
| `TypeError: useState is not a function` / `useRef is not a function` | `proxy` / `signal.ref()` |
| Template renders `undefined` / variable not found | Add `this.` to component members |
| Ref is always `null` | `t-ref="this.name"` with `name = signal.ref()`, read `this.name()` |
| Effect never re-runs | Owl 3 `useEffect` tracks reactive reads only; use `useOnChange(() => [deps], cb)` |
| Module loader reports a missing dependency `@web/core/orm_service` (or `.../action_service`) | `@web/core/orm_plugin` / `@web/webclient/actions/action_plugin` |
