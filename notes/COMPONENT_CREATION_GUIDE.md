# How to Create DoenetML Components - By Type

This guide covers how to make new components in DoenetML, organized by what kind of component you want to build. It assumes you are already somewhat familiar with the codebase but have not built a component from scratch before.

Every component's logic lives in:
`packages/doenetml-worker-javascript/src/components/`

Every component is a JavaScript class that extends one of the abstract base classes in `abstract/`. The base class you pick determines the type of component you're building, what lifecycle methods you get for free, and what the renderer can expect.

---

## The Inheritance Tree (simplified)

```
BaseComponent
├── InlineComponent          <- text, number, integer, math, boolean
│   ├── Input                <- textInput, mathInput, booleanInput
│   └── SingleCharacterInline <- br, nbsp, etc.
├── BlockComponent           <- p, figure, table, image, video
│   ├── SectioningComponent  <- section, chapter, etc.
│   └── BlockScoredComponent <- answer
├── GraphicalComponent       <- point, line, polygon, discreteGraph
├── CompositeComponent       <- numberList, mathList, sequence, select
└── ConstraintComponent      <- constrainToGrid, attractTo, etc.
```

---

## How Components Work in General

Before diving into each type, it helps to understand the overall flow:

1. DoenetML content is parsed into a tree of components.
2. Each component's `.js` file runs in a **web worker** (not the browser tab). This is where state is computed and stored.
3. State variables marked `forRenderer: true` get passed to the browser.
4. Each component's renderer (`.jsx` or `.tsx`) runs in the browser and draws the component using those state values.
5. When the user does something (clicks, drags, types), the renderer fires an **action** back to the worker, which updates the state, which sends new values to the renderer.

So the `.js` file is the brain, the renderer is the face, and actions are how they talk to each other.

---

## Type 1: Inline Components

**What they are:** components that appear inline in text content, like a number, a math expression, or a string of text. They render directly inside a paragraph or sentence.

**Base class:** `InlineComponent`, or sometimes extend an existing component like `Number` if your component is closely related.

**Examples:** `Number.js`, `Integer.js`, `Text.js`, `Boolean.js`, `Math.js`

**Renderer:** usually a simple `.tsx` file that renders the component's `text` or `value` state variable as a string.

### How the .js file looks

```js
import InlineComponent from "./abstract/InlineComponent";

export default class MyComponent extends InlineComponent {
    static componentType = "myComponent";
    // rendererType defaults to componentType. Override if you want to reuse an existing renderer:
    // static rendererType = "number";

    // createAttributesObject defines what attributes this component accepts in DoenetML.
    // Always call super() first to inherit the base attributes (like hide, fixed, etc.)
    static createAttributesObject() {
        let attributes = super.createAttributesObject();

        // This attribute creates a boolean state variable called "myAttr" automatically.
        // The user writes <myComponent myAttr="true" /> to set it.
        attributes.myAttr = {
            createComponentOfType: "boolean",  // can be "boolean", "number", "text", "math", etc.
            createStateVariable: "myAttr",     // the name of the state variable that gets created
            defaultValue: false,               // what value to use if the attribute is not provided
            public: true,                      // lets other components reference $myComp.myAttr
        };

        return attributes;
    }

    // returnChildGroups defines what types of children this component can have.
    // For example, <myComponent>42</myComponent> has a string child "42".
    // Each group is a named bucket that children get sorted into.
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
        // Always start by getting the parent class's state variables.
        // This includes things like "hidden", "disabled", "label", etc.
        let stateVariableDefinitions = super.returnStateVariableDefinitions();

        stateVariableDefinitions.value = {
            public: true,           // can be referenced by other components as $myComp.value
            forRenderer: true,      // this state variable will be sent to the renderer
            returnDependencies: () => ({
                // Here you declare what other data this state variable depends on.
                // When dependencies change, the definition function runs again.
                stringChildren: {
                    dependencyType: "child",
                    childGroups: ["strings"],
                    variableNames: ["value"], // which state vars to pull from each child
                },
                numberChildren: {
                    dependencyType: "child",
                    childGroups: ["numbers"],
                    variableNames: ["value"],
                },
                // You can also depend on your own attributes:
                myAttr: {
                    dependencyType: "stateVariable",
                    variableName: "myAttr",
                },
            }),
            definition({ dependencyValues }) {
                // dependencyValues has everything you declared in returnDependencies.
                // Compute and return the value.

                let totalChildren =
                    dependencyValues.stringChildren.length +
                    dependencyValues.numberChildren.length;

                if (totalChildren === 0) {
                    return { setValue: { value: null } };
                }

                if (dependencyValues.stringChildren.length === 1 && totalChildren === 1) {
                    // String children are raw strings, NOT component objects.
                    // Access them directly as strings.
                    let str = dependencyValues.stringChildren[0];
                    let parsed = Number(str);
                    if (Number.isFinite(parsed)) {
                        return { setValue: { value: parsed } };
                    }
                    return { setValue: { value: NaN } };
                }

                if (dependencyValues.numberChildren.length === 1 && totalChildren === 1) {
                    // Number/math/text children ARE component objects.
                    // Access their value through stateValues.
                    return { setValue: { value: dependencyValues.numberChildren[0].stateValues.value } };
                }

                return { setValue: { value: NaN } };
            },
        };

        return stateVariableDefinitions;
    }
}
```

### How the renderer looks

The renderer for an inline component is usually very short. It just reads the `text` state variable (which most components compute automatically) and renders it as a React element.

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

`SVs` is the object of all state variables marked `forRenderer: true`. The renderer just reads from it and displays.

### Key things to know for inline components

- `forRenderer: true` on a state variable means its value will be packed up and sent to the browser renderer.
- `public: true` means other DoenetML components can access this variable using the `$` syntax, like `$myComp.value`.
- String children (the raw text between tags) are plain JS strings. Access them directly from the array: `dependencyValues.stringChildren[0]`. Do NOT call `.stateValues.value` on them.
- Number, math, and text children are component objects. Access their values like `dependencyValues.numberChildren[0].stateValues.value`.
- If you are extending an existing component (like Integer extends Number), you can replace a parent state variable by deleting the old one and adding your own: `delete stateVariableDefinitions.value; stateVariableDefinitions.value = { ... }`.
- The `text` state variable (what gets displayed as a string) is often separate from `value` (the actual data type). For example, `Integer` has `value = 9` (a number) but `text = "1001"` (what's shown when representation is binary).

---

## Type 2: Graphical Components

**What they are:** components that draw something on a JSXGraph board, which lives inside a `<graph>` tag. Examples include points, lines, polygons, circles, and custom graph objects.

**Base class:** `GraphicalComponent`

**Examples:** `Point.js`, `Line.js`, `Polygon.js`, `Polyline.js`, `DiscreteGraph.js`, `Circle.js`

**Renderer:** a `.jsx` file that gets the JSXGraph board from React context and uses it to draw and update objects.

### How the two files connect

The `.js` worker file computes coordinates and styling, marks them `forRenderer: true`, and the browser renderer receives them as `SVs`. The renderer creates and manages JSXGraph objects (points, lines, etc.) based on those values. If the component is draggable, the renderer fires an action back to the worker with the new position, and the worker validates and updates state, which flows back as new `SVs`.

### How the .js file looks

```js
import GraphicalComponent from "./abstract/GraphicalComponent";

export default class MyGraphThing extends GraphicalComponent {
    constructor(args) {
        super(args);
        // Register any actions the renderer will need to call back.
        // Actions are methods on this class that update state.
        Object.assign(this.actions, {
            myGraphThingClicked: this.myGraphThingClicked.bind(this),
            myGraphThingFocused: this.myGraphThingFocused.bind(this),
        });
    }

    static componentType = "myGraphThing";

    static createAttributesObject() {
        let attributes = super.createAttributesObject();
        // GraphicalComponent already gives you: hidden, fixed, layer, selectedStyle.

        // Add draggable if you want the user to be able to drag it.
        attributes.draggable = {
            createComponentOfType: "boolean",
            createStateVariable: "draggable",
            defaultValue: true,
            public: true,
            forRenderer: true,
        };

        // Use numberList to accept a space-separated list of numbers as an attribute.
        // The numberList component handles all the parsing for you.
        attributes.values = {
            createComponentOfType: "numberList",
        };

        return attributes;
    }

    static returnStateVariableDefinitions() {
        let stateVariableDefinitions = super.returnStateVariableDefinitions();

        // numElements: how many data points there are
        stateVariableDefinitions.numElements = {
            public: true,
            forRenderer: true,
            returnDependencies: () => ({
                values: {
                    dependencyType: "attributeComponent",
                    attributeName: "values",
                    variableNames: ["numComponents"],
                },
            }),
            definition({ dependencyValues }) {
                if (dependencyValues.values !== null) {
                    return {
                        setValue: {
                            numElements: dependencyValues.values.stateValues.numComponents,
                        },
                    };
                }
                return { setValue: { numElements: 0 } };
            },
        };

        // numericalPoints: the [x, y] pairs the renderer will draw
        // isArray: true means each element is computed individually.
        // The array state variable pattern uses returnArraySizeDependencies,
        // returnArraySize, returnArrayDependenciesByKey, and arrayDefinitionByKey.
        stateVariableDefinitions.numericalPoints = {
            isArray: true,
            entryPrefixes: ["numericalPoint"],
            forRenderer: true,
            returnArraySizeDependencies: () => ({
                numElements: {
                    dependencyType: "stateVariable",
                    variableName: "numElements",
                },
            }),
            returnArraySize({ dependencyValues }) {
                return [dependencyValues.numElements];
            },
            returnArrayDependenciesByKey({ arrayKeys }) {
                // Each element of the array declares its own dependencies.
                let globalDependencies = {
                    values: {
                        dependencyType: "attributeComponent",
                        attributeName: "values",
                        variableNames: ["numbers"],
                    },
                };
                return { globalDependencies };
            },
            arrayDefinitionByKey({ globalDependencyValues, arrayKeys }) {
                let numericalPoints = {};
                let numbers = globalDependencyValues.values
                    ? globalDependencyValues.values.stateValues.numbers
                    : [];

                for (let arrayKey of arrayKeys) {
                    let i = Number(arrayKey);
                    let y = numbers[i];
                    if (!Number.isFinite(y)) y = NaN;
                    numericalPoints[arrayKey] = [i, y]; // [index, value]
                }

                return { setValue: { numericalPoints } };
            },
        };

        return stateVariableDefinitions;
    }

    // Actions are called by the renderer to push updates back to state.
    async myGraphThingClicked({ actionId, sourceInformation = {} }) {
        // For click/focus actions, you usually just record that it happened.
        await this.coreFunctions.requestRecordEvent({
            verb: "interacted",
            object: { componentName: this.componentName, componentType: this.componentType },
            result: { focused: true },
        });
    }
}
```

### How the renderer looks

```jsx
// packages/doenetml/src/Viewer/renderers/myGraphThing.jsx
import React, { useContext, useEffect, useRef } from "react";
import useDoenetRenderer from "../useDoenetRenderer";
import { BoardContext } from "./graph";

export default React.memo(function MyGraphThing(props) {
    let { name, id, SVs, actions, callAction } = useDoenetRenderer(props);

    // board is the JSXGraph board provided by the parent <graph> component.
    // All drawing goes through this object.
    const board = useContext(BoardContext);

    // pointsJXG holds the actual JSXGraph point objects.
    // These are kept in a ref because updating them does not need to trigger a React re-render.
    let pointsJXG = useRef(null);

    // Clean up JSXGraph objects when this component is removed from the page.
    useEffect(() => {
        return () => {
            if (pointsJXG.current) {
                for (let pt of pointsJXG.current) {
                    board.removeObject(pt);
                }
                pointsJXG.current = null;
            }
        };
    }, []);

    if (board) {
        if (!pointsJXG.current) {
            // First render: create the JSXGraph objects from scratch.
            pointsJXG.current = [];
            for (let i = 0; i < SVs.numElements; i++) {
                let pt = board.create("point", SVs.numericalPoints[i], {
                    name: SVs.pointLabels ? SVs.pointLabels[i] : "",
                    withLabel: true,
                    fixed: SVs.fixed,
                    layer: 10 * SVs.layer + 1,
                });
                pointsJXG.current.push(pt);
            }
        } else {
            // Subsequent renders: update positions of existing points.
            // Also add new points if numElements increased, remove if it decreased.
            let prevCount = pointsJXG.current.length;

            // Add points if needed
            for (let i = prevCount; i < SVs.numElements; i++) {
                let pt = board.create("point", SVs.numericalPoints[i], {
                    name: SVs.pointLabels ? SVs.pointLabels[i] : "",
                    withLabel: true,
                    fixed: SVs.fixed,
                    layer: 10 * SVs.layer + 1,
                });
                pointsJXG.current.push(pt);
            }

            // Remove points if needed
            for (let i = SVs.numElements; i < prevCount; i++) {
                board.removeObject(pointsJXG.current[i]);
            }
            pointsJXG.current = pointsJXG.current.slice(0, SVs.numElements);

            // Move remaining points to their new positions
            for (let i = 0; i < SVs.numElements; i++) {
                pointsJXG.current[i].coords.setCoordinates(
                    JXG.COORDS_BY_USER,
                    SVs.numericalPoints[i],
                );
                pointsJXG.current[i].needsUpdate = true;
                pointsJXG.current[i].update();
            }

            board.updateRenderer();
        }
    }

    if (SVs.hidden) return null;
    return <><a name={id} /></>;
});
```

### Key things to know for graphical components

- `GraphicalComponent` gives you `hidden`, `fixed`, `layer`, and `selectedStyle` (color, thickness, etc.) for free.
- The `.js` file computes data in the worker. The `.jsx` file draws it in the browser. They do not run in the same context.
- Only state variables marked `forRenderer: true` are sent to the renderer. Everything else stays in the worker.
- `isArray: true` on a state variable means the DoenetML system handles it as an array. Each element is computed on demand, which helps with performance. The four functions (`returnArraySizeDependencies`, `returnArraySize`, `returnArrayDependenciesByKey`, `arrayDefinitionByKey`) work together to define it.
- Actions are registered in the constructor with `Object.assign(this.actions, { ... })`. The renderer calls them with `callAction({ action: actions.myAction, args: { ... } })`.
- For dragging: the renderer should NOT move the point on its own. Instead, it fires an action with the new coordinates, waits for the worker to update state, and the new position comes back via `SVs`. This is the standard "snap back" pattern seen in `discreteGraph.jsx`.
- JSXGraph objects must be manually removed when the component unmounts. Use the `useEffect` cleanup function for this.

---

## Type 3: Composite Components

**What they are:** components that expand into one or more other components. They do not render themselves. Instead, they produce "replacement" components that each have their own rendering.

**Base class:** `CompositeComponent`

**Examples:** `NumberList.js`, `MathList.js`, `Sequence.js`, `Select.js`, `TextList.js`

**Renderer:** none. The replacements they generate have their own renderers.

### How they work

A composite component has a `createSerializedReplacements` method that returns a list of serialized component definitions. These become the actual components in the tree. When the composite's data changes, `calculateReplacementChanges` runs to add, remove, or update those replacements.

This is how `<sequence from="1" to="5" />` expands into five `<number>` components, or how `<numberList>3 1 4 1 5</numberList>` expands into five individual `<number>` components.

### How the .js file looks

```js
import CompositeComponent from "./abstract/CompositeComponent";

export default class MyList extends CompositeComponent {
    static componentType = "myList";

    // This tells the system which state variable to watch for triggering replacement rebuilds.
    static stateVariableToEvaluateAfterReplacements = "readyToExpandWhenResolved";

    // When another component uses this as an attribute (like numberList is used
    // in graphical components), this is the state variable that gets "shadowed"
    // (exposed to the outside component).
    static stateVariableToBeShadowed = "numbers";

    static returnChildGroups() {
        return [
            { group: "numbers", componentTypes: ["number"] },
        ];
    }

    static returnStateVariableDefinitions() {
        let stateVariableDefinitions = super.returnStateVariableDefinitions();

        // Count the children so we know how many replacements to create.
        stateVariableDefinitions.numComponents = {
            public: true,
            returnDependencies: () => ({
                children: {
                    dependencyType: "child",
                    childGroups: ["numbers"],
                },
            }),
            definition({ dependencyValues }) {
                return { setValue: { numComponents: dependencyValues.children.length } };
            },
        };

        // readyToExpandWhenResolved is the standard trigger variable for composites.
        // When it goes stale (because numComponents changed), the system knows to
        // call calculateReplacementChanges.
        stateVariableDefinitions.readyToExpandWhenResolved = {
            returnDependencies: () => ({
                numComponents: {
                    dependencyType: "stateVariable",
                    variableName: "numComponents",
                },
            }),
            // markStale is special: returning updateReplacements: true tells the
            // system that replacements need to be recalculated when this goes stale.
            markStale: () => ({ updateReplacements: true }),
            definition() {
                return { setValue: { readyToExpandWhenResolved: true } };
            },
        };

        return stateVariableDefinitions;
    }

    // createSerializedReplacements runs once when the component is first created.
    // It returns a list of component definitions in "serialized" format.
    // nComponents is a running count of all components in the document, used to
    // give each replacement a unique index.
    static async createSerializedReplacements({ component, workspace, nComponents }) {
        let numComponents = await component.stateValues.numComponents;
        let replacements = [];

        for (let i = 0; i < numComponents; i++) {
            replacements.push({
                componentType: "number",
                componentIdx: nComponents++,
                attributes: {},
                doenetAttributes: {},
                children: [],
                state: {},
            });
        }

        workspace.numComponents = numComponents;
        return { replacements, errors: [], warnings: [], nComponents };
    }

    // calculateReplacementChanges runs when state changes.
    // The simplest approach is to just recreate all replacements.
    // For better performance in large components, you would do a diff and only
    // add/remove what changed.
    static async calculateReplacementChanges({
        component,
        workspace,
        nComponents,
        componentInfoObjects,
    }) {
        let result = await this.createSerializedReplacements({
            component,
            workspace,
            nComponents,
            componentInfoObjects,
        });

        return {
            replacementChanges: [
                {
                    changeType: "add",
                    changeTopLevelReplacements: true,
                    firstReplacementInd: 0,
                    numberReplacementsToReplace: component.replacements
                        ? component.replacements.length
                        : 0,
                    serializedReplacements: result.replacements,
                },
            ],
            nComponents: result.nComponents,
        };
    }
}
```

### Key things to know for composite components

- Composite components do not have renderers. They produce replacement components that do.
- `createSerializedReplacements` runs when the component first loads.
- `calculateReplacementChanges` runs when the component's dependencies change and replacements need updating.
- The `readyToExpandWhenResolved` pattern with `markStale: () => ({ updateReplacements: true })` is standard boilerplate. Copy it from any existing composite component.
- `stateVariableToBeShadowed` controls which state variable gets exposed when another component uses this one as an attribute value. For example, when you write `values="3 1 4 1 5"` on a graphical component and the attribute type is `numberList`, the graphical component gets access to the shadowed variable of the `numberList`.
- `workspace` is a plain object that persists between calls. Use it to store information between `createSerializedReplacements` and `calculateReplacementChanges`.

---

## Type 4: Input Components

**What they are:** components that let the user enter data. They have a renderer with a form control and they fire actions back to the worker when the user changes the value.

**Base class:** `Input`, which itself extends `InlineComponent`.

**Examples:** `TextInput.js`, `MathInput.js`, `BooleanInput.js`, `ChoiceInput.js`

**Renderer:** a form element in the browser (text field, math editor, checkbox, dropdown, etc.).

### How they differ from regular inline components

The key difference is that input components have **essential** state. This means their `value` persists even if nothing is providing it from the outside. When the user types something, the renderer calls an action, which calls `performUpdate`, which stores the new value as essential state, which the renderer reads back next render.

### How the .js file looks

```js
import Input from "./abstract/Input";

export default class MyInput extends Input {
    constructor(args) {
        super(args);
        // Actions the renderer will call when the user interacts.
        Object.assign(this.actions, {
            updateValue: this.updateValue.bind(this),
        });
    }

    static componentType = "myInput";

    static createAttributesObject() {
        let attributes = super.createAttributesObject();
        // Input base already gives you: disabled, hidden, fixed, size,
        // prefill, bindValueTo, and a few others.

        // Add a prefill if you want to provide a starting value.
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
            // attribute providing it. This is what makes input components work.
            hasEssential: true,
            returnDependencies: () => ({
                prefill: {
                    dependencyType: "stateVariable",
                    variableName: "prefill",
                },
            }),
            definition({ dependencyValues, usedDefault }) {
                // useEssentialOrDefaultValue tells the system to use whatever was
                // stored in essential state (i.e. from a previous updateValue call),
                // and if nothing is stored yet, use the provided defaultValue.
                return {
                    useEssentialOrDefaultValue: {
                        value: { defaultValue: dependencyValues.prefill },
                    },
                };
            },
            // inverseDefinition is called when something tries to set this variable
            // from outside (like an answer check or a bindValueTo). It stores the
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

    // This action is called by the renderer when the user changes the input.
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

        // Record that the user submitted an answer attempt if appropriate.
        await this.coreFunctions.triggerChainedActions({
            triggeringAction: "updateValue",
            componentName: this.componentName,
            actionId,
        });
    }
}
```

### Key things to know for input components

- `hasEssential: true` enables a state variable to store its own value independently. Without this, a state variable always recomputes from its dependencies and you cannot "store" user input.
- `useEssentialOrDefaultValue` is how you say "use the stored value if there is one, otherwise use this default."
- `inverseDefinition` is how updates from outside the component (like a bound variable) flow into the component. It runs when another component tries to set this variable.
- The `Input` base class handles a lot of answer-checking infrastructure. Look at `Input.js` to see what state variables and actions you get for free.

---

## Type 5: Block Components

**What they are:** components that take up a full block of space in the document, like a paragraph, a figure, a table, or a section. They are not inline and usually wrap children.

**Base class:** `BlockComponent`, or `SectioningComponent` for things like sections and chapters.

**Examples:** `P.js`, `Figure.js`, `Table.js`, `Image.js`, `Video.js`

**Renderer:** usually a React component that renders its children inside some HTML wrapper element.

These are the structurally simplest type. They mostly just declare what children they accept and let the system handle rendering of those children.

```js
import BlockComponent from "./abstract/BlockComponent";

export default class MyBlock extends BlockComponent {
    static componentType = "myBlock";

    static createAttributesObject() {
        let attributes = super.createAttributesObject();

        // Add any block-specific attributes here.
        // For example, a width for a figure:
        attributes.width = {
            createComponentOfType: "number",
            createStateVariable: "width",
            defaultValue: 100,
            public: true,
            forRenderer: true,
        };

        return attributes;
    }

    // Declare the children this block can accept.
    // "_base" means any component type (the base of the whole tree).
    static returnChildGroups() {
        return [
            { group: "anything", componentTypes: ["_base"] },
        ];
    }

    static returnStateVariableDefinitions() {
        let stateVariableDefinitions = super.returnStateVariableDefinitions();
        // For most block components, you do not need to add much here.
        // The children render themselves, and this component just wraps them.
        return stateVariableDefinitions;
    }
}
```

And the renderer:

```tsx
import React from "react";
import useDoenetRenderer from "../useDoenetRenderer";
import ChildrenByID from "../childrenById";

export default function MyBlock(props: any) {
    const { id, SVs, children } = useDoenetRenderer(props);

    if (SVs.hidden) return null;

    return (
        <div id={id} style={{ width: SVs.width }}>
            {children}
        </div>
    );
}
```

`children` from `useDoenetRenderer` is the rendered output of the component's children. You usually just place it inside your wrapper.

---

## Registering a New Component

After creating the `.js` file, you MUST register it in:
`packages/doenetml-worker-javascript/src/ComponentTypes.js`

Otherwise the DoenetML parser will not know the component type exists and will throw an error.

**Step 1:** Add an import near the top of the file, grouped with similar component imports:
```js
import MyComponent from "./components/MyComponent";
```

**Step 2:** Add it to the `componentTypeArray` list in the same file:
```js
const componentTypeArray = [
    // ... lots of existing components ...
    MyComponent,
    // ...
];
```

If your component has a renderer, create the `.jsx` or `.tsx` file in:
`packages/doenetml/src/Viewer/renderers/`

The renderer file name should match the `rendererType` static property on your class. If you do not set `rendererType`, it defaults to your `componentType`. By convention, renderer files use a lowercase first letter (e.g. `myComponent.tsx`).

---

## Quick Reference: Attribute Patterns

```js
// Option 1: attribute that creates a state variable automatically
// The system creates a child component of the given type and hooks it to a state variable.
attributes.myAttr = {
    createComponentOfType: "boolean",  // "number", "text", "math", "integer", etc.
    createStateVariable: "myAttr",     // name of the resulting state variable
    defaultValue: false,
    public: true,
    forRenderer: true,
};

// Option 2: attribute as a primitive string (no component created, just a raw string)
// Useful for mode/enum-style attributes like representation="binary"
attributes.mode = {
    createPrimitiveOfType: "string",
    createStateVariable: "mode",
    defaultValue: "decimal",
    public: true,
};

// Option 3: attribute that holds a complex component type
// The attribute is not reduced to a single value but kept as a whole component.
// Access its state variables via dependencyValues.values.stateValues.whatever
attributes.values = {
    createComponentOfType: "numberList",
};
```

## Quick Reference: State Variable Return Values

```js
// Return a computed value
return { setValue: { myVar: 42 } };

// Return multiple values at once
return { setValue: { myVar: 42, myOtherVar: "hello" } };

// Use essential (stored) state with a fallback default
return { useEssentialOrDefaultValue: { value: { defaultValue: 0 } } };

// Signal that the value could not be computed (not an error, just no value)
return { setValue: { myVar: null } };
```

## Quick Reference: Dependency Types

```js
returnDependencies: () => ({
    // Depend on a state variable from this same component
    myOtherVar: {
        dependencyType: "stateVariable",
        variableName: "myOtherVar",
    },

    // Depend on children in a named group
    numberChildren: {
        dependencyType: "child",
        childGroups: ["numbers"],
        variableNames: ["value"], // which state vars to pull from each child
    },

    // Depend on an attribute component (for complex attribute types like numberList)
    values: {
        dependencyType: "attributeComponent",
        attributeName: "values",
        variableNames: ["numbers", "numComponents"],
    },

    // Depend on the parent component's state
    parentValue: {
        dependencyType: "parentStateVariable",
        variableName: "someValue",
    },
}),
```

## Quick Reference: Child Access

```js
// String children are plain JS strings (the raw text between tags)
let str = dependencyValues.stringChildren[0];  // e.g. "hello" or "42"

// Number, math, text children are component objects with a stateValues property
let num = dependencyValues.numberChildren[0].stateValues.value;  // e.g. 42

// Math children often need to be converted to plain JS numbers
let val = dependencyValues.mathChildren[0].stateValues.value.evaluate_to_constant();

// Attribute components: access their state via stateValues
let count = dependencyValues.values.stateValues.numComponents;
let arr   = dependencyValues.values.stateValues.numbers;  // for numberList

// Always check length before accessing index 0 to avoid runtime errors
if (dependencyValues.stringChildren.length > 0) {
    let first = dependencyValues.stringChildren[0];
}
```

---

## Build and Test

After creating or modifying a component, rebuild the chain and test in the test viewer.

```bash
cd DoenetML

# Step 1: rebuild the worker-javascript package (your .js component lives here)
npm run build -w packages/doenetml-worker-javascript

# Step 2: rebuild the worker bundle (wraps the javascript worker)
npm run build -w packages/doenetml-worker

# Step 3: rebuild the main doenetml package (includes renderers)
npm run build -w packages/doenetml

# Step 4: start the test viewer dev server
cd packages/test-viewer
npm run dev
```

The test viewer dev server has hot module replacement for renderer files (`.jsx`/`.tsx`), so renderer-only changes might appear automatically. Changes to `.js` worker files always need the full rebuild chain above.

---
