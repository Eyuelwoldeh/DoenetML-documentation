# Graphical Components

things that show up on a JSXGraph board inside a `<graph>` tag.

**base class:** `GraphicalComponent`

**examples:** `Point.js`, `Line.js`, `LineSegment.js`, `Polygon.js`, `Polyline.js`, `DiscreteGraph.js`, `Circle.js`

**renderer:** a `.jsx` file that uses `BoardContext` and creates JSXGraph objects

---

## basic structure

graphical components have two files:
1. a `.js` file in `components/` that does the math (runs in the web worker)
2. a `.jsx` file in `renderers/` that does the drawing (runs in the browser)

the `.js` file computes state, marks some variables as `forRenderer: true`, and those get sent to the browser. the `.jsx` file reads them as `SVs` and draws them. when the user does something (drag, click), the renderer fires an action back to the worker.

### the worker side (.js)

```js
import GraphicalComponent from "./abstract/GraphicalComponent";

export default class MyGraphThing extends GraphicalComponent {
    constructor(args) {
        super(args);
        // register actions the renderer can call
        Object.assign(this.actions, {
            moveMyThing: this.moveMyThing.bind(this),
            myThingClicked: this.myThingClicked.bind(this),
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
                    return {
                        setValue: {
                            numElements: dependencyValues.values.stateValues.numComponents,
                        },
                    };
                }
                return { setValue: { numElements: 0 } };
            },
        };

        // the actual data the renderer needs to draw
        stateVariableDefinitions.numericalPoints = {
            isArray: true,
            forRenderer: true,
            // ... array state variable pattern (see below)
        };

        return stateVariableDefinitions;
    }

    // actions are how the renderer communicates back
    async moveMyThing({ pointCoords, transient, actionId, sourceInformation = {} }) {
        if (!(await this.stateValues.draggable)) return;

        // convert plain number coords to math expressions
        let components = {};
        for (let ind in pointCoords) {
            components[ind + ",0"] = me.fromAst(pointCoords[ind][0]);
            components[ind + ",1"] = me.fromAst(pointCoords[ind][1]);
        }

        await this.coreFunctions.performUpdate({
            updateInstructions: [{
                updateType: "updateValue",
                componentName: this.componentName,
                stateVariable: "vertices",
                value: components,
            }],
            transient,
            actionId,
            sourceInformation,
        });
    }
}
```

---

## the renderer side (.jsx)

```jsx
import React, { useContext, useEffect, useRef } from "react";
import useDoenetRenderer from "../useDoenetRenderer";
import { BoardContext, VERTEX_LAYER_OFFSET } from "./graph";

export default React.memo(function MyGraphThing(props) {
    // SVs = state values, actions = the action functions registered in the .js file
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
                        name: "",
                        withLabel: false,
                        fixed: true,
                    })
                );
            }
        } else {
            // subsequent renders: move existing objects to new positions
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

---

## what GraphicalComponent gives you for free

- `hidden` -- whether the component is visible
- `fixed` -- whether the component can be interacted with at all
- `layer` -- z-ordering on the graph
- `selectedStyle` -- line color, width, style, marker size, marker style
- `labelPosition` -- where to place the label relative to the component
- `showCoordsWhenDragging` -- whether to show coordinates during drag
- label-related state variables

you don't need to define these yourself. just use them in your renderer via `SVs`.

---

## the numericalPoints / numericalVertices pattern

most graphical components store their positions as math expression objects internally (because they need to support symbolic math). but the renderer needs plain JS numbers to pass to JSXGraph.

so you create a `numericalSomething` state variable that converts them:

```js
stateVariableDefinitions.numericalVertices = {
    isArray: true,
    forRenderer: true,
    returnArraySizeDependencies: () => ({
        numVertices: {
            dependencyType: "stateVariable",
            variableName: "numVertices",
        },
    }),
    returnArraySize({ dependencyValues }) {
        return [dependencyValues.numVertices];
    },
    returnArrayDependenciesByKey({ arrayKeys }) {
        let dependenciesByKey = {};
        for (let arrayKey of arrayKeys) {
            dependenciesByKey[arrayKey] = {
                vertex: {
                    dependencyType: "stateVariable",
                    variableName: "vertex" + (Number(arrayKey) + 1),
                },
            };
        }
        return { dependenciesByKey };
    },
    arrayDefinitionByKey({ dependencyValuesByKey, arrayKeys }) {
        let numericalVertices = {};
        for (let arrayKey of arrayKeys) {
            // evaluate_to_constant() converts a math expression to a JS number
            let vert = dependencyValuesByKey[arrayKey].vertex.map(
                (x) => x.evaluate_to_constant()
            );
            // if any coordinate is not a finite number, fill with NaN
            if (!vert.every((x) => Number.isFinite(x))) {
                vert = Array(vert.length).fill(NaN);
            }
            numericalVertices[arrayKey] = vert;
        }
        return { setValue: { numericalVertices } };
    },
};
```

---

## the dragging pattern

this is how dragging works end to end:

1. user drags a point on the graph
2. renderer fires `callAction({ action: actions.moveMyThing, args: { pointCoords, transient: true } })`
3. the `.js` action method converts coords to math expressions and calls `performUpdate()`
4. the state variable system recomputes
5. new `forRenderer` values flow back to the renderer
6. renderer updates the JSXGraph objects with the new positions

during dragging, the renderer snaps the point BACK to where core last said it was. this is important -- the renderer doesn't move the point itself, it waits for core to respond with the validated position (which might be different due to constraints).

```jsx
function pointDragHandler(i, e) {
    // record where the user dragged to
    let newCoords = [pointsJXG.current[i].X(), pointsJXG.current[i].Y()];

    // fire the action back to core
    callAction({
        action: actions.moveMyThing,
        args: {
            pointCoords: { [i]: newCoords },
            transient: true,
            skippable: true,
        },
    });

    // snap point back to where core last confirmed
    pointsJXG.current[i].coords.setCoordinates(
        JXG.COORDS_BY_USER,
        [...lastPositionsFromCore.current[i]]
    );
}
```

---

## constrained vs unconstrained positions

some graphical components (like LineSegment, Polygon) support sticky groups and constraint snapping. the pattern is:

- `unconstrainedVertices` -- where the user actually put the point
- `haveConstrainedVertices` -- boolean, true if inside a sticky group
- `vertices` -- the public state variable. if constrained, runs through the constraint logic first. if not, just passes through from unconstrained

this means the renderer always reads from `vertices` (or `numericalVertices`), and the constraint system is transparent.

the `stateVariablesDeterminingDependencies` property tells the system to recompute dependencies when `haveConstrainedVertices` changes. this is a pattern used when you don't know ahead of time which dependencies you'll need.

---

## the add/remove pattern for dynamic counts

when the number of points changes between renders, you need to handle adding new JSXGraph objects and removing old ones:

```jsx
// track previous count in a ref
let previousNumPoints = useRef(0);

// on render:
let currentCount = SVs.numElements;
let prevCount = previousNumPoints.current;

if (currentCount > prevCount) {
    // add new points
    for (let i = prevCount; i < currentCount; i++) {
        pointsJXG.current.push(
            board.create("point", SVs.numericalPoints[i], pointAttributes)
        );
    }
} else if (currentCount < prevCount) {
    // remove extra points
    for (let i = currentCount; i < prevCount; i++) {
        board.removeObject(pointsJXG.current[i]);
    }
    pointsJXG.current.length = currentCount;
}

previousNumPoints.current = currentCount;
```

---

## sub-pattern: interactive graphical components with public mutable state

this is what you reach for when you need a graphical widget whose internal state (a) is publicly queryable from `<answer>` (e.g. `$mem.cellValue7`, `$mem.pointer1`), (b) can be mutated by the user typing or interacting, and (c) lives in arrays whose size is determined by an attribute. the `arrayComponent` (memory-diagram widget) was built this way, and the steps below capture every concrete trap that came up while building it.

the recipe has four moving parts:
1. public array state vars with **essential** storage
2. actions that mutate those vars
3. a renderer that draws on the JSXGraph board and fires actions on user input
4. registration in `ComponentTypes.js`

### 1. public, mutable array state variables

use this exact shape (this is the pattern matching `Vector.js` for `displacement2`):

```js
stateVariableDefinitions.cellValues = {
    public: true,
    shadowingInstructions: { createComponentOfType: "text" },
    isArray: true,
    entryPrefixes: ["cellValue"],          // -> $mem.cellValue1, $mem.cellValue2, ...
    forRenderer: true,                     // ship the whole array to the browser
    hasEssential: true,                    // store user-set values in an essential slot
    essentialVarName: "cellValues",        // name of that slot
    defaultValueByArrayKey: () => "?",     // default for any cell that hasn't been set

    returnArraySizeDependencies: () => ({
        size: { dependencyType: "stateVariable", variableName: "size" },
    }),
    returnArraySize({ dependencyValues }) {
        return [Math.max(0, Number(dependencyValues.size) || 0)];
    },

    returnArrayDependenciesByKey: () => ({}),

    // must return useEssentialOrDefaultValue for every requested key.
    // returning {} here is a silent footgun: the entry refs resolve to nothing
    // and you'll see "Neither value nor default value specified" errors.
    arrayDefinitionByKey({ arrayKeys }) {
        let useEssentialOrDefaultValue = {};
        for (let key of arrayKeys) {
            useEssentialOrDefaultValue[key] = { defaultValue: "?" };
        }
        return { useEssentialOrDefaultValue: { cellValues: useEssentialOrDefaultValue } };
    },

    // inverse: take desired changes from an action and persist them into essential storage
    async inverseArrayDefinitionByKey({ desiredStateVariableValues }) {
        return {
            success: true,
            instructions: [{
                setEssentialValue: "cellValues",
                value: Object.fromEntries(
                    Object.entries(desiredStateVariableValues.cellValues)
                          .map(([k, v]) => [k, String(v)]),
                ),
            }],
        };
    },
};
```

key points that cost real time to discover:

- `entryPrefixes: ["cellValue"]` is what makes `$mem.cellValue7` resolve. bracket-notation array refs like `$mem.cellValues[7]` do **not** currently resolve into a scalar usable by `<when>`/`<number>`. always design your entry prefix so it reads nicely (`cellValue7`, `pointer1`)
- doenetML's entry-prefix references are **1-indexed**. internal array keys are 0-indexed strings (`"0"`, `"1"`, ...). so `$mem.cellValue7` reads key `"6"`. plan your `<answer>` references accordingly
- `defaultValueByArrayKey` alone is **not** enough. you must also produce `useEssentialOrDefaultValue` from `arrayDefinitionByKey` for the requested keys, otherwise the entries never get instantiated
- `forRenderer: true` is required to expose the whole array as `SVs.cellValues` on the renderer side

### 2. actions that mutate the state

register actions in the constructor (rest of the class is unchanged):

```js
constructor(args) {
    super(args);
    Object.assign(this.actions, {
        setCellValue: this.setCellValue.bind(this),
        setPointerY: this.setPointerY.bind(this),
    });
}

async setCellValue({ index, value, actionId, sourceInformation = {}, skipRendererUpdate = false }) {
    await this.coreFunctions.performUpdate({
        updateInstructions: [{
            updateType: "updateValue",
            componentIdx: this.componentIdx,        // NOT this.componentName
            stateVariable: "cellValues",
            value: { [index]: String(value) },
        }],
        actionId,
        sourceInformation,
        skipRendererUpdate,
        event: {
            verb: "interacted",
            object: {
                componentIdx: this.componentIdx,   // same here
                componentType: this.componentType,
            },
            result: { index, value },
        },
    });
}
```

the single most likely source of a silent `TypeError: Cannot read properties of undefined (reading 'constructor')` deep in `Core.requestComponentChanges` is using `componentName: this.componentName`. the field is named `componentIdx` (numeric), and the property on `this` is `this.componentIdx`. reference `TextInput.js` if you're ever unsure of the field shape for an action.

### 3. renderer that draws on a JSXGraph board and fires actions

the renderer is a `.tsx` in `packages/doenetml/src/Viewer/renderers/`. the skeleton:

```tsx
import React, { useContext, useEffect, useRef } from "react";
import useDoenetRenderer from "../useDoenetRenderer";
import { BoardContext, LINE_LAYER_OFFSET, TEXT_LAYER_OFFSET } from "./graph";

export default React.memo(function MyWidget(props: any) {
    const { id, SVs, actions, callAction } = useDoenetRenderer(props);
    const board = useContext(BoardContext) as any;

    // refs to every JSXGraph object so they can be removed on unmount
    const jxgObjs = useRef<any[]>([]);
    const previousStructure = useRef<number | null>(null);

    useEffect(() => () => deleteAllJXG(), []);   // unmount cleanup

    function buildAll() { /* board.create(...) for every line/text/input */ }
    function deleteAllJXG() {
        jxgObjs.current.forEach(o => o && board?.removeObject(o));
        jxgObjs.current = [];
    }

    // build on first render or when structural attributes change.
    // for value-only updates (typing into a cell), sync DOM inputs without rebuilding.
    useEffect(() => {
        if (!board) return;
        const structureChanged = previousStructure.current !== SVs.size;
        if (structureChanged) {
            buildAll();
            previousStructure.current = SVs.size;
        } else {
            // sync SVs.cellValues into the existing inputs
        }
    }, [board, SVs.size, SVs.cellValues /* etc. */]);

    if (board) return null;       // graphics live on the board, not in the DOM tree
    return <a id={id} />;          // off-graph fallback element (use id, not name)
});
```

for typeable cells, `board.create("input", [x, y, initial, label], { fixed: true })` gives you an HTML `<input>` overlaid on the board. the reliable way to capture the user's value back into the worker:

```tsx
const dispatch = () => {
    const v = inp.rendNodeInput?.value;
    callAction({
        action: actions.setCellValue,
        args: { index: String(i), value: String(v ?? "") },
    });
};

// belt-and-suspenders: attach every plausible event. JSXGraph wraps its input
// in a way that means "change" alone occasionally doesn't fire.
inp.rendNodeInput?.addEventListener("change", dispatch);
inp.rendNodeInput?.addEventListener("blur",   dispatch);
inp.rendNodeInput?.addEventListener("keyup",  (e: any) => { if (e.key === "Enter") dispatch(); });
inp.on?.("change", dispatch);   // JSXGraph-native event, also fires in some cases
```

when you need a draggable pointer/arrow but want it **keyboard-accessible** instead of drag-only, render an input alongside the arrow where the user types a target value (e.g. an address). on `change`, parse the typed string against your address list and call `setPointerY` with the resulting row index. empty / invalid maps to `-1` (your "unplaced" sentinel) and hides the arrow via `visible: () => (SVs.pointerY || [])[p] >= 0`.

other renderer gotchas:

- all graphical positions and the `visible` flag should be **functions** when they depend on state, so JSXGraph re-evaluates them automatically when `board.update()` runs. e.g. `[() => xFn(), () => yFn()]` for a point's coords, `visible: () => SVs.foo >= 0` for conditional visibility
- when syncing `SVs.cellValues` back into the DOM inputs, skip inputs the user is currently typing in (`document.activeElement !== inp.rendNodeInput`), otherwise you wipe their text mid-keystroke
- the "off-graph fallback" element must use a valid HTML attribute. `<a name={id} />` is deprecated and gives a TypeScript error. use `<a id={id} />` instead

### 4. register the component + renderer

- worker side: add an import in `packages/doenetml-worker-javascript/src/ComponentTypes.js` and append it to the `componentTypeArray` (see "registering a new component" on the component-types page)
- renderer side: map the component type to its renderer in the renderer registry. look at how `point` or `discreteGraph` is wired up, usually a single object literal in the renderer barrel file

### build flow

every code change requires a build of the affected packages before the test viewer sees it. there is no HMR across the worker boundary.

| changed file | rebuild |
|--------------|---------|
| `MyComponent.js` (worker logic) | `doenetml-worker-javascript` then `doenetml-worker` |
| `myComponent.tsx` (renderer) | `doenetml` |
| both | all three |

then **hard-refresh** the test viewer (cmd/ctrl-shift-r). a normal refresh may use the cached bundle.

### debugging checklist when state isn't updating

1. open the browser console and add a temporary `console.log` inside the renderer's dispatch closure. if it doesn't fire, your event wiring is wrong (see "belt-and-suspenders" above)
2. if it does fire but state doesn't change, look for `TypeError: Cannot read properties of undefined (reading 'constructor')`. it means an update instruction is malformed. the two near-certain causes are `componentName` (should be `componentIdx`) and a missing/wrong `stateVariable` name
3. drop `<p>DEBUG cellValue7 = $mem.cellValue7</p>` into your test doenet to confirm the entry-prefix reference resolves on the doenetML side. if it prints as empty or `NaN`, your `arrayDefinitionByKey` is not returning `useEssentialOrDefaultValue` correctly
4. indices in `<when>`/`<answer>` are 1-based, renderer loop indices are 0-based, internal array keys are 0-based strings. mixing these up causes silent off-by-one wrong-answer marks

---

## real example: LineSegment


the LineSegment component is a good reference. it has:
- two endpoints (a fixed-size array, always 2 points)
- `draggable` and `endpointsDraggable` attributes
- `length` and `slope` computed state variables
- `nearestPoint` function for snapping other things to the segment
- `parallelCoords` adapter so it can act as a direction vector
- constraint support via `unconstrainedEndpoints` / `endpoints` split

see the [LineSegment notes](../notes/LINESEGMENT_NOTES.md) for a detailed walkthrough.

---

## common mistakes with graphical components

1. **not using `forRenderer: true`** -- the renderer only sees state variables marked with this flag

2. **forgetting to convert math expressions to numbers** -- the renderer can't use math expression objects. always create a `numerical*` version using `evaluate_to_constant()`

3. **moving points directly in the renderer** -- always let core validate the position first. snap back to the last confirmed position during drag

4. **not cleaning up JSXGraph objects** -- use a `useEffect` cleanup function to call `board.removeObject()` on unmount

5. **forgetting `board.updateRenderer()`** -- after updating JSXGraph object positions, call this so the board redraws
