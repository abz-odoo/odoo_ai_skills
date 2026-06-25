# Odoo SaaS-19.3 OWL Guide

Reference for the OWL 2 JavaScript component framework used in Odoo SaaS-19.3.
Source of truth: `/home/achraf/src/193/odoo/addons/web/static/src/`

> **SaaS-19.3 uses OWL 2.0.1** (`@odoo/owl: "^2.0.1"`)
> - Hooks-only API: no more `willStart()`, `mounted()` instance methods → use `onWillStart()`, `onMounted()` in `setup()`
> - `patch(obj, extension)` replaces `Component.patch()` static method
> - `useService()` hook replaces manual service injection
> - Deep reactivity via `useState()` / `reactive()`

---

## Table of Contents

1. [Component definition](#component-definition)
2. [Lifecycle hooks](#lifecycle-hooks)
3. [State and reactivity](#state-and-reactivity)
4. [Refs](#refs)
5. [Services](#services)
6. [Environment](#environment)
7. [Template syntax](#template-syntax)
8. [Patching existing components](#patching-existing-components)
9. [Registry system](#registry-system)
10. [Props validation](#props-validation)
11. [Import paths quick reference](#import-paths-quick-reference)

---

## Component definition

```javascript
import { Component, onWillStart, onMounted, xml } from "@odoo/owl";
import { useService } from "@web/core/utils/hooks";

export class MyComponent extends Component {
    // Inline template (small components)
    static template = xml`
        <div class="my-component">
            <span t-out="state.value"/>
            <button t-on-click="onClick">Click me</button>
        </div>
    `;

    // Or reference an external QWeb template by name
    // static template = "my_module.MyComponent";

    // Props schema — validated at runtime in debug mode
    static props = {
        recordId: Number,
        readonly: { type: Boolean, optional: true },
    };

    static defaultProps = {
        readonly: false,
    };

    // Declare child components used in the template
    static components = { ChildComponent };

    setup() {
        // All hooks must be called here (not in methods)
        this.orm = useService("orm");
        this.state = useState({ value: "" });

        onWillStart(async () => {
            this.state.value = await this.orm.read(
                "my.model", [this.props.recordId], ["name"]
            );
        });
    }

    onClick() {
        this.state.value = "clicked";
    }
}
```

---

## Lifecycle hooks

All hooks must be called **inside `setup()`**.

```javascript
import {
    onWillStart,
    onMounted,
    onPatched,
    onWillUpdateProps,
    onWillPatch,
    onWillDestroy,
    onError,
} from "@odoo/owl";

setup() {
    // Async setup before first render — good for data loading
    onWillStart(async () => {
        this.data = await fetchData();
    });

    // After DOM is mounted — can access DOM refs
    onMounted(() => {
        this.inputRef.el?.focus();
    });

    // After each re-render (patch)
    onPatched(() => {
        // Sync DOM side-effects
    });

    // Before props change — good for cleanup or async update
    onWillUpdateProps(async (nextProps) => {
        if (nextProps.recordId !== this.props.recordId) {
            this.data = await fetchData(nextProps.recordId);
        }
    });

    // Before re-render (patch) — synchronous
    onWillPatch(() => { });

    // Before component is destroyed
    onWillDestroy(() => {
        this.subscription?.unsubscribe();
    });

    // Catch errors from child components
    onError((error) => {
        console.error("Child error:", error);
        this.state.hasError = true;
    });
}
```

---

## State and reactivity

```javascript
import { useState, reactive } from "@web/owl2/utils";

setup() {
    // Reactive state — mutations trigger re-render
    this.state = useState({
        isOpen: false,
        items: [],
        count: 0,
    });

    // Mutate directly (reactive — no this.setState())
    this.state.isOpen = true;
    this.state.items.push({ id: 1, name: "new" });

    // Reactive object not tied to component lifecycle
    // Useful for shared state across components
    const shared = reactive({
        value: 42,
        onUpdate: () => console.log(shared.value),
    });
}
```

---

## Refs

```javascript
import { useRef } from "@web/owl2/utils";
import { useLayoutEffect } from "@web/owl2/utils";

setup() {
    // matches t-ref="myInput" in template
    this.inputRef = useRef("myInput");

    onMounted(() => {
        console.log(this.inputRef.el);  // HTMLElement or null
        this.inputRef.el?.focus();
    });

    // Layout effect: runs after each patch, with cleanup
    useLayoutEffect(
        (el) => {
            if (!el) return;
            const handler = () => console.log("resized");
            el.addEventListener("resize", handler);
            return () => el.removeEventListener("resize", handler);
        },
        () => [this.inputRef.el],  // dependency array
    );
}
```

Template:
```xml
<input t-ref="myInput" type="text"/>
```

---

## Services

### Accessing a service

```javascript
import { useService } from "@web/core/utils/hooks";

setup() {
    this.orm      = useService("orm");
    this.ui       = useService("ui");
    this.dialog   = useService("dialog");
    this.action   = useService("action");
    this.notification = useService("notification");
}
```

### Common service APIs

```javascript
// ORM service
const records = await this.orm.searchRead(
    "sale.order",
    [["state", "=", "sale"]],
    ["name", "partner_id"],
    { limit: 10 },
);

await this.orm.write("sale.order", [1, 2], { state: "cancel" });
await this.orm.create("sale.order", [{ partner_id: 1 }]);
await this.orm.call("sale.order", "action_confirm", [[1, 2]]);

// Notification service
this.notification.add("Saved!", { type: "success" });
this.notification.add("Error!", { type: "danger" });

// Dialog service
import { ConfirmationDialog } from "@web/core/confirmation_dialog/confirmation_dialog";
this.dialog.add(ConfirmationDialog, {
    title: "Are you sure?",
    body: "This cannot be undone.",
    confirm: () => this.doDelete(),
});

// Action service
await this.action.doAction("sale.action_quotations_with_onboarding");
await this.action.doAction({
    type: "ir.actions.act_window",
    res_model: "sale.order",
    views: [[false, "list"], [false, "form"]],
});
```

### Defining a service

```javascript
import { registry } from "@web/core/registry";

export const myService = {
    dependencies: ["orm", "notification"],
    async start(env, { orm, notification }) {
        return {
            async fetchData(id) {
                return orm.read("my.model", [id], ["name"]);
            },
            notify(msg) {
                notification.add(msg, { type: "info" });
            },
        };
    },
};

registry.category("services").add("my_service", myService);
```

### Event bus

```javascript
import { useBus } from "@web/core/utils/hooks";

setup() {
    // Attach listener — auto-removed on component destroy
    useBus(this.env.bus, "MY_EVENT", (payload) => {
        this.state.value = payload.value;
    });
}

// Emit
this.env.bus.trigger("MY_EVENT", { value: 42 });
```

---

## Environment

```javascript
import { useEnv, useSubEnv } from "@web/owl2/utils";

setup() {
    // Read current environment
    const env = useEnv();
    console.log(env.services);   // { orm, ui, ... }
    console.log(env.isSmall);    // Boolean — mobile breakpoint
    console.log(env.debug);      // "" | "1" | "assets"

    // Extend env for child components (does NOT affect parent or siblings)
    useSubEnv({
        myContext: { someValue: 42 },
    });
}
```

---

## Template syntax

```xml
<!-- Conditionals -->
<div t-if="state.isAdmin">Admin only</div>
<div t-elif="state.isManager">Manager only</div>
<div t-else="">Regular user</div>

<!-- Loops — t-key is REQUIRED for stable DOM diffing -->
<t t-foreach="state.items" t-as="item" t-key="item.id">
    <div t-out="item.name"/>
</t>

<!-- Output — t-out escapes HTML, t-html renders raw HTML -->
<span t-out="state.message"/>
<div t-html="state.htmlContent"/>

<!-- Attribute binding -->
<div t-att-class="{ 'active': state.isActive, 'disabled': props.readonly }"/>
<div t-att-style="{ color: state.color }"/>
<input t-attf-id="field-{{props.fieldName}}"/>  <!-- String interpolation -->

<!-- Event handling -->
<button t-on-click="onClick"/>
<button t-on-click="() => state.count++"/>
<input t-on-keyup.enter="onEnter"/>      <!-- .enter modifier -->
<div t-on-click.stop="onClickStop"/>     <!-- .stop = stopPropagation -->
<form t-on-submit.prevent="onSubmit"/>   <!-- .prevent = preventDefault -->

<!-- Two-way binding -->
<input t-model="state.search" type="text"/>
<input t-model.number="state.qty" type="number"/>  <!-- auto-parse to Number -->
<input t-model.lazy="state.val"/>                  <!-- on change, not keyup -->

<!-- Refs -->
<input t-ref="myInput"/>
<div t-ref="container"/>

<!-- Slots — render named content from parent -->
<t t-slot="default"/>
<t t-slot="header" defaultContent="No header"/>

<!-- Defining slot content (in parent template) -->
<MyDialog>
    <t t-set-slot="header"><h2>Title</h2></t>
    <p>Body content goes in the default slot.</p>
</MyDialog>

<!-- Dynamic component -->
<t t-component="state.currentView" t-props="state.viewProps"/>
```

---

## Patching existing components

```javascript
import { patch } from "@web/core/utils/patch";
import { FormController } from "@web/views/form/form_controller";

// Add/override methods on an existing component
patch(FormController.prototype, {
    setup() {
        // Call the original setup first
        super.setup(...arguments);
        // Then add your extensions
        this.myService = useService("my_service");
    },

    async saveRecord() {
        // Intercept save
        console.log("Before save");
        await super.saveRecord(...arguments);
        console.log("After save");
    },
});
```

> `patch()` stacks — multiple modules can patch the same object.
> If you need to undo: `const unpatch = patch(...); unpatch();`

---

## Registry system

```javascript
import { registry } from "@web/core/registry";

// ── Fields ──
registry.category("fields").add("my_widget", {
    component: MyWidget,
    displayName: "My Widget",
    supportedTypes: ["char", "text"],
    extractProps: ({ attrs, field }) => ({
        placeholder: attrs.placeholder || "",
    }),
});

// ── Views ──
registry.category("views").add("my_view", {
    type: "my_view",
    Controller: MyController,
    Renderer: MyRenderer,
});

// ── Services ──
registry.category("services").add("my_service", myServiceDef);

// ── Actions ──
registry.category("actions").add("my_action_tag", MyClientAction);

// ── Reading ──
const charDef = registry.category("fields").get("char");
const allFields = registry.category("fields").getEntries();  // [["name", def], ...]
```

---

## Props validation

```javascript
static props = {
    // Simple types
    name: String,
    count: Number,
    active: Boolean,
    onClick: Function,

    // Optional
    label: { type: String, optional: true },

    // Array with element type
    items: { type: Array, element: Object },

    // Object with shape
    record: {
        type: Object,
        shape: {
            id: Number,
            name: String,
            // "*": true  → allow extra keys
        },
    },

    // Union types
    value: [String, Number, Boolean],

    // Custom validator
    size: {
        type: String,
        validate: (v) => ["sm", "md", "lg"].includes(v),
    },

    // Slots
    slots: {
        type: Object,
        shape: {
            default: { optional: true },
            header:  { optional: true },
        },
    },
};
```

---

## Import paths quick reference

| What | Import |
|------|--------|
| `Component`, lifecycle hooks, `xml` | `@odoo/owl` |
| `useState`, `useRef`, `useEnv`, `useSubEnv`, `reactive`, `useLayoutEffect` | `@web/owl2/utils` |
| `useService`, `useBus`, `useAutofocus`, `useChildRef` | `@web/core/utils/hooks` |
| `patch` | `@web/core/utils/patch` |
| `registry` | `@web/core/registry` |
| `ConfirmationDialog` | `@web/core/confirmation_dialog/confirmation_dialog` |
| `standardFieldProps` | `@web/views/fields/standard_field_props` |
