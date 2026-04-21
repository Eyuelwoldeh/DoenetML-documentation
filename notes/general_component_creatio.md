# Creating Doenet Components by Type

a guide for making new components, organized by what kind of component you want to build.

every component lives in:
`packages/doenetml-worker-javascript/src/components/`

every component is a JS class that extends one of the abstract base classes in `abstract/`.

the base class you extend determines what kind of component you're building.

---

## The Inheritance Tree (simplified)

```
BaseComponent
├── InlineComponent          ← text, number, integer, math, boolean
│   ├── Input                ← textInput, mathInput, booleanInput
│   └── SingleCharacterInline ← br, nbsp, etc.
├── BlockComponent           ← p, figure, table, image, video
│   ├── SectioningComponent  ← section, chapter, etc.
│   └── BlockScoredComponent ← answer
├── GraphicalComponent       ← point, line, polygon, discreteGraph
├── CompositeComponent       ← numberList, mathList, sequence, select
└── ConstraintComponent      ← constrainToGrid, attractTo, etc.
```

---

## Type 1: Inline Components

**what they are:** things that show up inline in text, like a number or a math expression.

**base class:** `InlineComponent` (or sometimes extend an existing one like `Number`)

**examples:** `Number.js`, `Integer.js`, `Text.js`, `Boolean.js`, `Math.js`

**renderer:** usually a simple `.tsx` file that just renders the value as text

### how to make one

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
                let value = /* your logic here */;
                return { setValue: { value } };
            },
        };

        return stateVariableDefinitions;
    }
}
```

### key things for inline components
- `forRenderer: true` on a state variable means it gets sent to the renderer
- the renderer typically just displays the `text` or `value` state variable
- `public: true` means other components can reference it like `$myComponent.value`
- string children are raw strings, NOT objects — use `dependencyValues.stringChildren[0]` directly, not `.stateValues.value`
- number/math/text children ARE objects — use `child.stateValues.value`
- if extending an existing component (like Integer extends Number), you can `delete stateVariableDefinitions.value` to replace the parent's version

---

## Type 2: Graphical Components

**what they are:** things that show up on a JSXGraph board inside a `<graph>` tag.

**base class:** `GraphicalComponent`

**examples:** `Point.js`, `Line.js`, `Polygon.js`, `Polyline.js`, `DiscreteGraph.js`, `Circle.js`

**renderer:** a `.jsx` file that uses `BoardContext` and creates JSXGraph objects

### how to make one

```js
import GraphicalComponent from "./abstract/GraphicalComponent";

export default class MyGraphThing extends GraphicalComponent {
    constructor(args) {
        super(args);
        // register actions the renderer can call
        Object.assign(this.actions, {
            myAction: this.myAction.bind(this),
        });
    }

    static componentType = "myGraphThing";

    static createAttributesObject() {
        let attributes = super.createAttributesObject();
        // draggable is common for graphical components
        attributes.draggable = {
            createComponentOfType: "boolean",
            createStateVariable: "draggable",
            defaultValue: true,
            public: true,
            forRenderer: true,
        };
        // use numberList for a list of numbers, _pointListComponent for a list of points
        attributes.values = {
            createComponentOfType: "numberList",
        };
        return attributes;
    }

    static returnStateVariableDefinitions() {
        let stateVariableDefinitions = super.returnStateVariableDefinitions();

        // count of items
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
                    return { setValue: { numElements: dependencyValues.values.stateValues.numComponents } };
                }
                return { setValue: { numElements: 0 } };
            },
        };

        // the actual data the renderer needs to draw
        stateVariableDefinitions.numericalPoints = {
            isArray: true,
            forRenderer: true,
            // ... array state variable pattern (see DiscreteGraph for full example)
        };

        return stateVariableDefinitions;
    }

    // actions are how the renderer communicates back
    async myAction({ actionId, sourceInformation = {} }) {
        await this.coreFunctions.performUpdate({
            updateInstructions: [/* ... */],
            actionId,
            sourceInformation,
        });
    }
}
```

### the renderer side (jsx)

```jsx
import React, { useContext, useEffect, useRef } from "react";
import useDoenetRenderer from "../useDoenetRenderer";
import { BoardContext, VERTEX_LAYER_OFFSET } from "./graph";

export default React.memo(function MyGraphThing(props) {
    let { name, id, SVs, actions, callAction } = useDoenetRenderer(props);

    const board = useContext(BoardContext);
    let pointsJXG = useRef(null);

    // cleanup on unmount
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
            // first render: create JSXGraph objects
            pointsJXG.current = [];
            for (let i = 0; i < SVs.numElements; i++) {
                pointsJXG.current.push(
                    board.create("point", SVs.numericalPoints[i], {
                        name: SVs.pointLabels?.[i] || "",
                        withLabel: true,
                        fixed: true,
                    })
                );
            }
        } else {
            // update: move existing points, add/remove as needed
            for (let i = 0; i < SVs.numElements; i++) {
                pointsJXG.current[i].coords.setCoordinates(
                    JXG.COORDS_BY_USER, SVs.numericalPoints[i]
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

### key things for graphical components
- `GraphicalComponent` gives you `hidden`, `fixed`, `layer`, `selectedStyle`, and label stuff for free
- your `.js` file does the math, your `.jsx` file does the drawing
- always use `forRenderer: true` on the data the renderer needs
- actions use `this.coreFunctions.performUpdate()` to change state
- the renderer gets `SVs` (state values) from `useDoenetRenderer(props)`
- the renderer gets the JSXGraph `board` from `useContext(BoardContext)`
- for dragging: renderer sends coords via `callAction`, worker validates and updates, new SVs flow back
- add/remove pattern: track `previousCount` in a ref, add new objects if count increased, remove if decreased

---

## Type 3: Composite Components

**what they are:** components that expand into other components. they don't render themselves — they generate replacement components.

**base class:** `CompositeComponent`

**examples:** `NumberList.js`, `MathList.js`, `Sequence.js`, `Select.js`, `TextList.js`

**renderer:** none! they expand into child components that each have their own renderer.

### how to make one

```js
import CompositeComponent from "./abstract/CompositeComponent";

export default class MyList extends CompositeComponent {
    static componentType = "myList";

    static stateVariableToEvaluateAfterReplacements = "readyToExpandWhenResolved";

    // when another component uses this as an attribute, shadow this variable
    static stateVariableToBeShadowed = "items";

    static returnChildGroups() {
        return [
            { group: "numbers", componentTypes: ["number"] },
        ];
    }

    static returnStateVariableDefinitions() {
        let stateVariableDefinitions = super.returnStateVariableDefinitions();

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

        // this triggers replacement updates when it goes stale
        stateVariableDefinitions.readyToExpandWhenResolved = {
            returnDependencies: () => ({
                numComponents: {
                    dependencyType: "stateVariable",
                    variableName: "numComponents",
                },
            }),
            markStale: () => ({ updateReplacements: true }),
            definition() {
                return { setValue: { readyToExpandWhenResolved: true } };
            },
        };

        return stateVariableDefinitions;
    }

    // this creates the replacement components
    static async createSerializedReplacements({ component, workspace, nComponents }) {
        let numComponents = await component.stateValues.numComponents;
        let replacements = [];

        for (let i = 0; i < numComponents; i++) {
            replacements.push({
                type: "serialized",
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

    // this updates replacements when things change
    static async calculateReplacementChanges({ component, workspace, nComponents, ... }) {
        // simplest approach: just recreate all replacements
        let result = await this.createSerializedReplacements({ component, workspace, nComponents });
        return {
            replacementChanges: [{
                changeType: "add",
                changeTopLevelReplacements: true,
                firstReplacementInd: 0,
                numberReplacementsToReplace: component.replacements.length,
                serializedReplacements: result.replacements,
            }],
            nComponents: result.nComponents,
        };
    }
}
```

### key things for composite components
- they do NOT render themselves — they create "replacement" components that do
- `createSerializedReplacements` runs once to create the initial replacements
- `calculateReplacementChanges` runs when state changes to update them
- `readyToExpandWhenResolved` + `markStale: () => ({ updateReplacements: true })` is the standard trigger pattern
- `stateVariableToBeShadowed` controls what gets used when another component references this one as an attribute

---

## Type 4: Input Components

**what they are:** things the user can type into or interact with.

**base class:** `Input` (which extends `InlineComponent`)

**examples:** `TextInput.js`, `MathInput.js`, `BooleanInput.js`, `ChoiceInput.js`

**renderer:** a form control (text field, checkbox, dropdown, etc.)

### how to make one

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
                return { useEssentialOrDefaultValue: { value: true } };
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

### key things for input components
- `Input` base class gives you `disabled`, `hidden`, `fixed` and scoring-related stuff
- the renderer sends user input back via actions (like `updateValue`)
- `hasEssential: true` means the value persists even without children defining it
- `useEssentialOrDefaultValue` is how you say "use whatever was stored, or the default"

---

## Type 5: Block Components

**what they are:** block-level layout components like paragraphs, figures, tables.

**base class:** `BlockComponent` (or `SectioningComponent` for sections)

**examples:** `P.js`, `Figure.js`, `Table.js`, `Image.js`, `Video.js`

**renderer:** usually renders children inside a div/section/table HTML structure

these are the simplest structurally — they mostly just pass through children.

```js
import BlockComponent from "./abstract/BlockComponent";

export default class MyBlock extends BlockComponent {
    static componentType = "myBlock";

    static returnChildGroups() {
        return [
            { group: "anything", componentTypes: ["_base"] },
        ];
    }
}
```

---

## Registering a New Component

after creating the `.js` file, you MUST register it in:
`packages/doenetml-worker-javascript/src/ComponentTypes.js`

1. add an import at the top:
```js
import MyComponent from "./components/MyComponent";
```

2. add it to the `componentTypeArray`:
```js
const componentTypeArray = [
    // ... existing components ...
    MyComponent,
];
```

if your component has a renderer, create the `.jsx`/`.tsx` file in:
`packages/doenetml/src/Viewer/renderers/`

the renderer filename must match `rendererType` (lowercase first letter by convention).

---

## Quick Reference: Attribute Patterns

```js
// simple attribute that creates a state variable automatically
attributes.myAttr = {
    createComponentOfType: "boolean",     // "number", "text", "math", etc.
    createStateVariable: "myAttr",
    defaultValue: false,
    public: true,
    forRenderer: true,
};

// attribute as a primitive string (no component created)
attributes.mode = {
    createPrimitiveOfType: "string",
    createStateVariable: "mode",
    defaultValue: "default",
    public: true,
};

// attribute that accepts a complex component (like numberList, pointList)
attributes.values = {
    createComponentOfType: "numberList",
    // access via: dependencyValues.values.stateValues.numbers
};
```

## Quick Reference: State Variable Return Values

```js
// set a computed value
return { setValue: { myVar: 42 } };

// use essential (stored) value or a default
return { useEssentialOrDefaultValue: { myVar: { defaultValue: 0 } } };
```

## Quick Reference: Child Access

```js
// string children → raw strings
let str = dependencyValues.stringChildren[0];  // "hello"

// number/math/text children → objects with stateValues
let num = dependencyValues.numberChildren[0].stateValues.value;  // 42

// math children need evaluate_to_constant() to get a JS number
let val = dependencyValues.mathChildren[0].stateValues.value.evaluate_to_constant();

// attribute components → access via stateValues
let count = dependencyValues.myAttr.stateValues.numComponents;
```

---

## Build and Test

after creating or modifying a component:

```bash
cd DoenetML

# rebuild the chain
npm run build -w packages/doenetml-worker-javascript
npm run build -w packages/doenetml-worker
npm run build -w packages/doenetml

# start the test viewer
cd packages/test-viewer
npm run dev
```

---