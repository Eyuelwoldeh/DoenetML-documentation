# Inline Components

things that show up inline in text, like a number or a math expression.

**base class:** `InlineComponent` (or extend an existing one like `Number`)

**examples:** `Number.js`, `Integer.js`, `Text.js`, `Boolean.js`, `Math.js`

**renderer:** usually a simple `.tsx` file that just renders the value as text

---

## basic structure

```js
import InlineComponent from "./abstract/InlineComponent";

export default class MyComponent extends InlineComponent {
    static componentType = "myComponent";
    // rendererType defaults to the componentType, or you can override:
    // static rendererType = "number";

    static createAttributesObject() {
        let attributes = super.createAttributesObject();
        // add your attributes here
        attributes.myAttr = {
            createComponentOfType: "boolean",  // or "number", "text", "math"
            createStateVariable: "myAttr",
            defaultValue: false,
            public: true,
        };
        return attributes;
    }

    static returnChildGroups() {
        // what types of children this component accepts
        return [
            { group: "strings", componentTypes: ["string"] },
            { group: "numbers", componentTypes: ["number"] },
        ];
    }

    static returnStateVariableDefinitions() {
        let stateVariableDefinitions = super.returnStateVariableDefinitions();

        stateVariableDefinitions.value = {
            public: true,
            forRenderer: true,    // this gets sent to the browser
            returnDependencies: () => ({
                stringChildren: {
                    dependencyType: "child",
                    childGroups: ["strings"],
                    variableNames: ["value"],
                },
            }),
            definition({ dependencyValues }) {
                // compute the value from children
                let raw = dependencyValues.stringChildren[0] || "";
                return { setValue: { value: raw } };
            },
        };

        return stateVariableDefinitions;
    }
}
```

---

## key things to know

- `forRenderer: true` on a state variable means it gets sent to the renderer
- the renderer typically just displays the `text` or `value` state variable
- `public: true` means other components can reference it like `$myComponent.value`
- string children are raw strings, NOT objects -- use `dependencyValues.stringChildren[0]` directly, not `.stateValues.value`
- number/math/text children ARE objects -- use `child.stateValues.value`
- if extending an existing component (like Integer extends Number), you can `delete stateVariableDefinitions.value` to replace the parent's version
- the `text` state variable (what gets displayed as a string) is often separate from `value` (the actual data type). for example, `Integer` has `value = 9` (a number) but `text = "1001"` (what's shown when representation is binary)


---

## extending an existing inline component

this is the most common pattern. you inherit everything from the parent and just override what you need.

```js
import NumberComponent from "./Number";

export default class Integer extends NumberComponent {
    static componentType = "integer";
    static rendererType = "number"; // reuse the number renderer

    static createAttributesObject() {
        let attributes = super.createAttributesObject();

        // add a new attribute
        attributes.representation = {
            createPrimitiveOfType: "string",
            createStateVariable: "representation",
            defaultValue: "decimal",
            public: true,
            forRenderer: true,
        };

        return attributes;
    }

    static returnStateVariableDefinitions() {
        let stateVariableDefinitions = super.returnStateVariableDefinitions();

        // option 1: delete and replace
        delete stateVariableDefinitions.value;
        stateVariableDefinitions.value = {
            // your new definition
        };

        // option 2: rename and wrap
        // renameStateVariable({
        //     stateVariableDefinitions,
        //     oldName: "value",
        //     newName: "valuePreRound",
        // });

        // option 3: just add new ones without touching existing
        stateVariableDefinitions.binary = {
            public: true,
            shadowingInstructions: { createComponentOfType: "text" },
            returnDependencies: () => ({
                value: {
                    dependencyType: "stateVariable",
                    variableName: "value",
                },
            }),
            definition({ dependencyValues }) {
                let val = dependencyValues.value;
                if (!Number.isFinite(val)) {
                    return { setValue: { binary: String(val) } };
                }
                let str = val < 0
                    ? "-" + Math.abs(val).toString(2)
                    : val.toString(2);
                return { setValue: { binary: str } };
            },
        };

        return stateVariableDefinitions;
    }
}
```

---

## the `text` state variable

most inline components have a `text` state variable with `forRenderer: true`. this is what the renderer displays. if your component needs to show different things depending on state, override `text`.

```js
stateVariableDefinitions.text = {
    public: true,
    forRenderer: true,
    shadowingInstructions: { createComponentOfType: "text" },
    returnDependencies: () => ({
        value: {
            dependencyType: "stateVariable",
            variableName: "value",
        },
        representation: {
            dependencyType: "stateVariable",
            variableName: "representation",
        },
    }),
    definition({ dependencyValues }) {
        let text = "";
        if (dependencyValues.representation === "binary") {
            text = dependencyValues.value.toString(2);
        } else {
            text = String(dependencyValues.value);
        }
        return { setValue: { text } };
    },
};
```

---

## inverse definitions

if your inline component can be used in an answer or input context, you need an inverse definition. this tells the system how to go from a desired output back to the right internal state.

```js
stateVariableDefinitions.text = {
    // ... normal definition stuff ...

    async inverseDefinition({ desiredStateVariableValues, dependencyValues }) {
        let desiredText = desiredStateVariableValues.text;
        let parsed = Number(desiredText);

        if (Number.isFinite(parsed)) {
            return {
                success: true,
                instructions: [{
                    setDependency: "value",
                    desiredValue: Math.round(parsed),
                }],
            };
        }
        return { success: false };
    },
};
```

---

## helper functions

put parsing and utility functions outside the class at the bottom of the file. keep them pure and validate input.

```js
// parse a binary string into a decimal integer
function parseBinary(str) {
    if (!str || typeof str !== "string") return NaN;
    let trimmed = str.trim();
    if (!/^[01]+$/.test(trimmed)) return NaN;
    return parseInt(trimmed, 2);
}
```

---

## common mistakes with inline components

1. **using `||` for numeric fallbacks** -- 0 is falsy, so `parsedValue || NaN` breaks when the value is 0. use `Number.isNaN(parsedValue) ? NaN : parsedValue` instead

2. **forgetting `forRenderer: true`** -- if you don't set this on `text`, the renderer won't get the updated value

3. **accessing string children wrong** -- `dependencyValues.stringChildren[0]` is already a string, don't try to read `.stateValues.value` on it
