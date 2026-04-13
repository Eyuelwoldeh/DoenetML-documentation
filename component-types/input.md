# Input Components

things the user can type into or interact with.

**base class:** `Input` (which extends `InlineComponent`)

**examples:** `TextInput.js`, `MathInput.js`, `BooleanInput.js`, `ChoiceInput.js`

**renderer:** a form control (text field, checkbox, dropdown, etc.)

---

## basic structure

```js
import Input from "./abstract/Input";

export default class MyInput extends Input {
    constructor(args) {
        super(args);
        Object.assign(this.actions, {
            updateValue: this.updateValue.bind(this),
        });
    }

    static componentType = "myInput";

    static createAttributesObject() {
        let attributes = super.createAttributesObject();
        // Input base gives you: disabled, hidden, fixed, prefill, etc.
        return attributes;
    }

    static returnStateVariableDefinitions() {
        let stateVariableDefinitions = super.returnStateVariableDefinitions();

        stateVariableDefinitions.value = {
            public: true,
            forRenderer: true,
            hasEssential: true,
            defaultValue: "",
            returnDependencies: () => ({}),
            definition() {
                return {
                    useEssentialOrDefaultValue: { value: true },
                };
            },
        };

        return stateVariableDefinitions;
    }

    async updateValue({ newValue, actionId }) {
        await this.coreFunctions.performUpdate({
            updateInstructions: [{
                updateType: "updateValue",
                componentName: this.componentName,
                stateVariable: "value",
                value: newValue,
            }],
            actionId,
        });
    }
}
```

---

## what Input gives you for free

the `Input` base class (which extends `InlineComponent`) provides:
- `disabled` -- prevents user interaction
- `hidden` -- hides the input
- `fixed` -- prevents changes
- answer/scoring integration -- inputs can be connected to `<answer>` components

these are all set up automatically. you just need to define your `value` state variable and your action handlers.

---

## essential values

input components typically use `hasEssential: true` on their value state variable. this means the value persists even when nothing else defines it.

```js
stateVariableDefinitions.value = {
    public: true,
    forRenderer: true,
    hasEssential: true,       // the value is stored internally
    defaultValue: "",         // starting value if nothing else sets it
    returnDependencies: () => ({}),
    definition() {
        // "use whatever was stored, or fall back to the default"
        return {
            useEssentialOrDefaultValue: { value: true },
        };
    },
};
```

when the user types something and the `updateValue` action fires, `performUpdate` stores the new value as the essential value. on the next render, the definition reads it back with `useEssentialOrDefaultValue`.

---

## the action pattern

input components communicate user interactions back to core through actions. the standard flow:

1. user types or clicks something in the renderer
2. renderer calls `callAction({ action: actions.updateValue, args: { newValue } })`
3. the action method in the `.js` file calls `performUpdate()`
4. the state variable system stores the new essential value
5. new `forRenderer` values flow back to the renderer

```js
// in the .js file
async updateValue({ newValue, actionId, sourceInformation = {} }) {
    // you can validate or transform the value here before storing
    let processed = newValue.trim();

    await this.coreFunctions.performUpdate({
        updateInstructions: [{
            updateType: "updateValue",
            componentName: this.componentName,
            stateVariable: "value",
            value: processed,
        }],
        actionId,
        sourceInformation,
    });
}
```

---

## immediate vs committed values

some inputs (like `mathInput`) distinguish between what the user is currently typing and the committed value:

- `immediateValue` -- updates on every keystroke (for live preview)
- `value` -- updates when the user presses enter or clicks away

the pattern looks like:

```js
stateVariableDefinitions.immediateValue = {
    public: true,
    forRenderer: true,
    hasEssential: true,
    defaultValue: "",
    returnDependencies: () => ({}),
    definition() {
        return { useEssentialOrDefaultValue: { immediateValue: true } };
    },
};

stateVariableDefinitions.value = {
    public: true,
    forRenderer: true,
    hasEssential: true,
    defaultValue: "",
    returnDependencies: () => ({}),
    definition() {
        return { useEssentialOrDefaultValue: { value: true } };
    },
};
```

then you have two actions:

```js
// fires on every keystroke
async updateImmediateValue({ newValue, actionId }) {
    await this.coreFunctions.performUpdate({
        updateInstructions: [{
            updateType: "updateValue",
            componentName: this.componentName,
            stateVariable: "immediateValue",
            value: newValue,
        }],
        transient: true,  // don't save to undo history
        actionId,
    });
}

// fires on enter/blur
async updateValue({ actionId }) {
    let immediateValue = await this.stateValues.immediateValue;
    await this.coreFunctions.performUpdate({
        updateInstructions: [{
            updateType: "updateValue",
            componentName: this.componentName,
            stateVariable: "value",
            value: immediateValue,
        }],
        actionId,
    });
}
```

---

## the renderer side

input renderers are standard React form components. they get `SVs` and `callAction` from `useDoenetRenderer`:

```jsx
import React, { useRef } from "react";
import useDoenetRenderer from "../useDoenetRenderer";

export default React.memo(function MyInput(props) {
    let { name, id, SVs, actions, callAction } = useDoenetRenderer(props);
    let inputRef = useRef(null);

    if (SVs.hidden) return null;

    function handleChange(e) {
        callAction({
            action: actions.updateImmediateValue,
            args: { newValue: e.target.value },
        });
    }

    function handleBlur() {
        callAction({
            action: actions.updateValue,
            args: {},
        });
    }

    function handleKeyDown(e) {
        if (e.key === "Enter") {
            callAction({
                action: actions.updateValue,
                args: {},
            });
        }
    }

    return (
        <input
            id={id}
            ref={inputRef}
            value={SVs.immediateValue || ""}
            onChange={handleChange}
            onBlur={handleBlur}
            onKeyDown={handleKeyDown}
            disabled={SVs.disabled}
        />
    );
});
```

---

## prefill

inputs often support a `prefill` attribute that sets the initial value before the user types anything. this is usually handled by checking if there's a prefill dependency and using it as the default:

```js
stateVariableDefinitions.value = {
    public: true,
    forRenderer: true,
    hasEssential: true,
    returnDependencies: () => ({
        prefill: {
            dependencyType: "stateVariable",
            variableName: "prefill",
        },
    }),
    definition({ dependencyValues }) {
        return {
            useEssentialOrDefaultValue: {
                value: {
                    defaultValue: dependencyValues.prefill || "",
                },
            },
        };
    },
};
```

---

## common mistakes with input components

1. **forgetting `hasEssential: true`** -- without this, the value can't be stored when the user types. the input will appear to do nothing

2. **not using `transient: true` for immediate updates** -- keystroke-by-keystroke updates should be transient so they don't flood the undo history

3. **not handling `disabled`** -- always pass `SVs.disabled` to the actual form element so the input respects the disabled state

4. **forgetting to commit on blur** -- if you only commit on enter, users who click away will lose their input
