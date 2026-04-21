# Input Components

things the user can type into or interact with.

**base class:** `Input` (which extends `InlineComponent`)

**examples:** `TextInput.js`, `MathInput.js`, `BooleanInput.js`, `ChoiceInput.js`

**renderer:** a form control (text field, checkbox, dropdown, etc.)

---

## how they differ from regular inline components

the key difference is that input components have essential state. this means their `value` persists even if nothing is providing it from the outside. when the user types something, the renderer calls an action, which calls `performUpdate`, which stores the new value as essential state, which the renderer reads back next render.

---

## basic structure

```js
import Input from "./abstract/Input";

export default class MyInput extends Input {
    constructor(args) {
        super(args);
        // actions the renderer will call when the user interacts
        Object.assign(this.actions, {
            updateValue: this.updateValue.bind(this),
        });
    }

    static componentType = "myInput";

    static createAttributesObject() {
        let attributes = super.createAttributesObject();
        // Input base already gives you: disabled, hidden, fixed, size, prefill, bindValueTo, etc.

        // add a prefill if you want to provide a starting value
        attributes.prefill = {
            createComponentOfType: "text",
            createStateVariable: "prefill",
            defaultValue: "",
            public: true,
        };

        return attributes;
    }

    static returnStateVariableDefinitions() {
        let stateVariableDefinitions = super.returnStateVariableDefinitions();

        stateVariableDefinitions.value = {
            public: true,
            forRenderer: true,
            // hasEssential: true means the value can persist without any child or
            // attribute providing it. this is what makes input components work.
            hasEssential: true,
            returnDependencies: () => ({
                prefill: {
                    dependencyType: "stateVariable",
                    variableName: "prefill",
                },
            }),
            definition({ dependencyValues }) {
                // useEssentialOrDefaultValue tells the system to use whatever was
                // stored in essential state (from a previous updateValue call),
                // and if nothing is stored yet, use the provided defaultValue.
                return {
                    useEssentialOrDefaultValue: {
                        value: { defaultValue: dependencyValues.prefill },
                    },
                };
            },
            // inverseDefinition is called when something tries to set this variable
            // from outside (like an answer check or a bindValueTo). it stores the
            // new value as essential state.
            inverseDefinition({ desiredStateVariableValues }) {
                return {
                    success: true,
                    instructions: [
                        {
                            setEssentialValue: "value",
                            value: desiredStateVariableValues.value,
                        },
                    ],
                };
            },
        };

        return stateVariableDefinitions;
    }

    // this action is called by the renderer when the user changes the input
    async updateValue({ newValue, actionId }) {
        await this.coreFunctions.performUpdate({
            updateInstructions: [
                {
                    updateType: "updateValue",
                    componentName: this.componentName,
                    stateVariable: "value",
                    value: newValue,
                },
            ],
            actionId,
            sourceInformation: { component: this },
        });

        // trigger any chained actions (like answer submission) if configured
        await this.coreFunctions.triggerChainedActions({
            triggeringAction: "updateValue",
            componentName: this.componentName,
            actionId,
        });
    }
}
```

---

## what Input gives you for free

the `Input` base class (which extends `InlineComponent`) provides:
- `disabled`: prevents user interaction
- `hidden`: hides the input
- `fixed`: prevents changes
- `size`: controls display width
- `prefill`: sets initial value
- `bindValueTo`: connects the value to another component
- answer/scoring integration: inputs can be connected to `<answer>` components

look at `Input.js` to see the full list of state variables and actions you get automatically.

---

## essential values

`hasEssential: true` enables a state variable to store its own value independently. without this, a state variable always recomputes from its dependencies and you cannot "store" user input.

```js
stateVariableDefinitions.value = {
    public: true,
    forRenderer: true,
    hasEssential: true,       // the value is stored internally
    returnDependencies: () => ({
        prefill: {
            dependencyType: "stateVariable",
            variableName: "prefill",
        },
    }),
    definition({ dependencyValues }) {
        // use whatever was stored, or fall back to the prefill value
        return {
            useEssentialOrDefaultValue: {
                value: { defaultValue: dependencyValues.prefill },
            },
        };
    },
};
```

`useEssentialOrDefaultValue` is how you say "use the stored value if there is one, otherwise use this default."

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

- `immediateValue`: updates on every keystroke (for live preview)
- `value`: updates when the user presses enter or clicks away

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

## common mistakes with input components

1. **forgetting `hasEssential: true`**: without this, the value can't be stored when the user types. the input will appear to do nothing

2. **forgetting `inverseDefinition`**: if you want other components (like `bindValueTo`) to be able to set this value from outside, you need an inverse definition that calls `setEssentialValue`

3. **not using `transient: true` for immediate updates**: keystroke-by-keystroke updates should be transient so they don't flood the undo history

4. **not handling `disabled`**: always pass `SVs.disabled` to the actual form element so the input respects the disabled state

5. **forgetting to commit on blur**: if you only commit on enter, users who click away will lose their input
