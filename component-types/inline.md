# Inline Components

things that appear inline in text content, like a number, a math expression, or a string. they render directly inside a paragraph or sentence.

**base class:** `InlineComponent` (or extend an existing one like `Number`)

**examples:** `Number.js`, `Integer.js`, `Text.js`, `Boolean.js`, `Math.js`

**renderer:** usually a simple `.tsx` file that renders the `text` or `value` state variable as a string

---

## basic structure

```js
import InlineComponent from "./abstract/InlineComponent";

export default class MyComponent extends InlineComponent {
    static componentType = "myComponent";
    // rendererType defaults to componentType. override if you want to reuse an existing renderer:
    // static rendererType = "number";

    // createAttributesObject defines what attributes this component accepts.
    // always call super() first to inherit the base attributes (like hide, fixed, etc.)
    static createAttributesObject() {
        let attributes = super.createAttributesObject();

        // this attribute creates a boolean state variable called "myAttr" automatically.
        // the user writes <myComponent myAttr="true" /> to set it.
        attributes.myAttr = {
            createComponentOfType: "boolean",  // can be "boolean", "number", "text", "math", etc.
            createStateVariable: "myAttr",     // the name of the state variable that gets created
            defaultValue: false,               // what value to use if the attribute is not provided
            public: true,                      // lets other components reference $myComp.myAttr
        };

        return attributes;
    }

    // returnChildGroups defines what types of children this component can have.
    // for example, <myComponent>42</myComponent> has a string child "42".
    // each group is a named bucket that children get sorted into.
    static returnChildGroups() {
        return [
            {
                group: "strings",           // name you use in returnDependencies
                componentTypes: ["string"], // what component types go in this bucket
            },
            {
                group: "numbers",
                componentTypes: ["number"],
            },
        ];
    }

    static returnStateVariableDefinitions() {
        // always start by getting the parent class's state variables.
        // this includes things like "hidden", "disabled", "label", etc.
        let stateVariableDefinitions = super.returnStateVariableDefinitions();

        stateVariableDefinitions.value = {
            public: true,       // can be referenced by other components as $myComp.value
            forRenderer: true,  // this state variable will be sent to the renderer
            returnDependencies: () => ({
                // declare what other data this state variable depends on.
                // when dependencies change, the definition function runs again.
                stringChildren: {
                    dependencyType: "child",
                    childGroups: ["strings"],
                    variableNames: ["value"],
                },
                numberChildren: {
                    dependencyType: "child",
                    childGroups: ["numbers"],
                    variableNames: ["value"],
                },
                myAttr: {
                    dependencyType: "stateVariable",
                    variableName: "myAttr",
                },
            }),
            definition({ dependencyValues }) {
                let totalChildren =
                    dependencyValues.stringChildren.length +
                    dependencyValues.numberChildren.length;

                if (totalChildren === 0) {
                    return { setValue: { value: null } };
                }

                if (dependencyValues.stringChildren.length === 1 && totalChildren === 1) {
                    // string children are raw strings, NOT component objects.
                    // access them directly as strings.
                    let str = dependencyValues.stringChildren[0];
                    let parsed = Number(str);
                    if (Number.isFinite(parsed)) {
                        return { setValue: { value: parsed } };
                    }
                    return { setValue: { value: NaN } };
                }

                if (dependencyValues.numberChildren.length === 1 && totalChildren === 1) {
                    // number/math/text children ARE component objects.
                    // access their value through stateValues.
                    return {
                        setValue: {
                            value: dependencyValues.numberChildren[0].stateValues.value,
                        },
                    };
                }

                return { setValue: { value: NaN } };
            },
        };

        return stateVariableDefinitions;
    }
}
```

---

## the renderer

inline component renderers are usually very short. they just read the `text` state variable and render it as a React element.

```tsx
// packages/doenetml/src/Viewer/renderers/myComponent.tsx
import React from "react";
import useDoenetRenderer from "../useDoenetRenderer";

export default function MyComponent(props: any) {
    const { id, SVs } = useDoenetRenderer(props);

    if (SVs.hidden) return null;

    return (
        <span id={id}>
            {SVs.text}
        </span>
    );
}
```

`SVs` is the object of all state variables marked `forRenderer: true`. the renderer just reads from it and displays.

---

## key things to know

- `forRenderer: true` on a state variable means its value will be packed up and sent to the browser renderer
- `public: true` means other DoenetML components can access this variable using the `$` syntax, like `$myComp.value`
- string children (the raw text between tags) are plain JS strings. access them directly from the array: `dependencyValues.stringChildren[0]`. do NOT call `.stateValues.value` on them
- number, math, and text children are component objects. access their values like `dependencyValues.numberChildren[0].stateValues.value`
- always check the array length before accessing index 0, otherwise you get a runtime error if there are no children
- if you're extending an existing component (like Integer extends Number), you can replace a parent state variable by deleting the old one and adding your own: `delete stateVariableDefinitions.value; stateVariableDefinitions.value = { ... }`

---

## the `text` vs `value` distinction

most inline components have both a `value` state variable (the actual data type, like a JS number) and a `text` state variable (what gets displayed as a string). they are often different things.

for example, `Integer` with `representation="binary"` has:
- `value = 9` (a number, always stored in decimal internally)
- `text = "1001"` (the string shown to the user)

`text` is usually the one with `forRenderer: true` since the renderer shows it as a string. override `text` when your component needs to display differently depending on state:

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

## extending an existing inline component

the most common pattern. you inherit everything from the parent and just override what you need.

```js
import NumberComponent from "./Number";

export default class Integer extends NumberComponent {
    static componentType = "integer";
    static rendererType = "number"; // reuse the number renderer

    static createAttributesObject() {
        let attributes = super.createAttributesObject();

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

        // option 1: delete and replace the parent's version
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

        // option 3: just add new variables without touching existing ones
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

## inverse definitions

if your component can be used in an answer or input context, you need an inverse definition. this tells the system how to go from a desired output back to the right internal state.

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

put parsing and utility functions outside the class at the bottom of the file. keep them pure and validate input early.

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

1. **using `||` for numeric fallbacks**: 0 is falsy, so `parsedValue || NaN` breaks when the value is 0. use `Number.isNaN(parsedValue) ? NaN : parsedValue` instead

2. **forgetting `forRenderer: true`**: if you don't set this on `text`, the renderer won't get the updated value

3. **accessing string children wrong**: `dependencyValues.stringChildren[0]` is already a string. do not try to read `.stateValues.value` on it

4. **not checking array length**: always check `dependencyValues.stringChildren.length > 0` before reading index 0. if there are no children, you'll get undefined or a crash
