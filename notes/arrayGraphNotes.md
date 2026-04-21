# Array Graph Component - Notes

---

## Possible plan for array-graph component

The idea is a component that takes a list of numbers (an array) and plots them on a graph.
Index goes on the x-axis, value goes on the y-axis.

Example usage:
```xml
<graph>
  <arrayGraph name="ag" values="3 1 4 1 5" />
</graph>
```

This would plot points at (1, 3), (2, 1), (3, 4), (4, 1), (5, 5).

---

### Files to create

1. `packages/doenetml-worker-javascript/src/components/ArrayGraph.js`
   - the logic layer, runs in the web worker
   - reads a numberList attribute and computes (index, value) coordinates

2. `packages/doenetml/src/Viewer/renderers/arrayGraph.jsx`
   - the drawing layer, runs in the browser
   - uses JSXGraph to draw points on the board
   - can reference discreteGraph.jsx heavily since it already does this

---

### What ArrayGraph.js needs

- extend `GraphicalComponent` (same as DiscreteGraph does)
  - this gives you `hidden`, `fixed`, `layer`, `selectedStyle` for free

- one main attribute: `values` declared as `createComponentOfType: "numberList"`
  - this is the key insight: we do NOT parse the string ourselves
  - same pattern as how DiscreteGraph declares `vertices` as `_pointListComponent`
  - the `numberList` system handles all the parsing for free
  - the user writes `values="3 1 4 1 5"` and numberList turns it into individual numbers
  - to access the data: `dependencyValues.values.stateValues.numbers`
  - to get the count: `dependencyValues.values.stateValues.numComponents`

- state variables to compute and mark `forRenderer: true`:
  - `numElements` - reads from `values.stateValues.numComponents`
  - `numericalPoints` - array of `[index, value]` pairs as plain JS numbers
    - this is the main one the renderer uses, similar to `numericalVertices` in DiscreteGraph
    - x = index (1-based), y = the value at that index
    - check that values are finite numbers, skip NaN entries

- no dragging needed for a basic version
  - can add later if we want to let users rearrange values

- no edges needed for a basic version
  - could add a `connect` boolean attribute later to draw lines between points

---

### What arrayGraph.jsx needs

- use `useDoenetRenderer` hook to get `SVs` (same as all renderers)
- get `board` from `BoardContext` (same as discreteGraph)
- create one JSXGraph point per array element
- x-coordinate = index (1-based), y-coordinate = the value at that index
- SVs.numericalPoints is the array of [x, y] pairs, SVs.numElements is the count

- for managing the list of points, copy the add/remove pattern from
  `discreteGraph.jsx` - it already handles adding new points when count increases
  and removing them when count decreases

- no drag handlers needed for now (much simpler than discreteGraph)

---

### Open questions

- should the index start at 0 or 1? 
- should we support negative values? 
- do we want optional connecting lines?
- what happens if the array has non-numeric values? skip drawing that point

---

---

## DiscreteGraph.js and discreteGraph.jsx notes

### how the two files connect

DiscreteGraph.js (worker) computes state, marks some variables as `forRenderer: true`,
and those get sent to the browser. discreteGraph.jsx reads them as `SVs` and draws them.
When the user does something (drag, click), the renderer fires an action back to the worker.

---

### DiscreteGraph.js - top of the file

```js
// imports the base class for anything that can go inside a <graph>
import GraphicalComponent from "./abstract/GraphicalComponent";

// imports the sticky/constraint system for snapping points to things
import { returnStickyGroupDefinitions } from "../utils/constraints";
```

```js
export default class DiscreteGraph extends GraphicalComponent {
    constructor(args) {
        super(args);

        // this is where you register the actions the renderer can call back
        // each action is a method on this class
        Object.assign(this.actions, {
            moveDiscreteGraph: this.moveDiscreteGraph.bind(this),
            finalizeDiscreteGraphPosition: this.finalizeDiscreteGraphPosition.bind(this),
            discreteGraphClicked: this.discreteGraphClicked.bind(this),
            discreteGraphFocused: this.discreteGraphFocused.bind(this),
        });
    }

    static componentType = "discreteGraph"; // this is the tag name in DoenetML
```

---

### discreteGraph.jsx - top of the file

```jsx
// useDoenetRenderer is the hook that pulls state variables from the store
// SVs = state values, actions = the action functions registered in the .js file
let { name, id, SVs, actions, sourceOfUpdate, callAction } =
    useDoenetRenderer(props);

// tells the system to ignore drag actions if core isn't ready yet
DiscreteGraph.ignoreActionsWithoutCore = () => true;

// board comes from the parent <graph> component via context
// all the JSXGraph drawing happens through this board object
const board = useContext(BoardContext);
```

```jsx
// refs to hold the actual JSXGraph objects
// these are outside React state because updating them doesn't need a re-render
let pointsJXG = useRef(null);   // array of JSXGraph point objects
let edgesJXG = useRef(null);    // array of JSXGraph segment objects

// lastPositionsFromCore holds the positions that core last confirmed
// used to snap points back after a drag until core responds
let lastPositionsFromCore = useRef(null);
lastPositionsFromCore.current = SVs.numericalVertices;

// fixed.current / fixLocation.current / verticesFixed.current
// these determine what can be dragged, set from SVs each render
fixed.current = SVs.fixed;
fixLocation.current = !SVs.draggable || SVs.fixLocation || SVs.fixed;
verticesFixed.current = !SVs.verticesDraggable || SVs.fixed || SVs.fixLocation;
```

---

### DiscreteGraph.js - numVertices and numEdges state variables

```js
// numVertices just reads from the vertices attribute component
// the vertices attribute is a _pointListComponent which has a numPoints variable
stateVariableDefinitions.numVertices = {
    public: true,
    forRenderer: true,      // sent to the renderer
    returnDependencies: () => ({
        vertices: {
            dependencyType: "attributeComponent",
            attributeName: "vertices",
            variableNames: ["numPoints"],
        },
    }),
    definition: function ({ dependencyValues }) {
        if (dependencyValues.vertices !== null) {
            return {
                setValue: {
                    numVertices: dependencyValues.vertices.stateValues.numPoints,
                },
            };
        } else {
            return { setValue: { numVertices: 0 } };
        }
    },
};

// numEdges works the same way, just for the edges attribute
```

---

### discreteGraph.jsx - createDiscreteGraphJXG function

```jsx
function createDiscreteGraphJXG() {
    // guard: don't create if data isn't ready
    if (
        SVs.numericalVertices.length !== SVs.numVertices ||
        SVs.numericalVertices.some((x) => x.length !== 2)
    ) {
        return null;
    }

    // check all coords are real numbers
    let validCoords = true;
    for (let coords of SVs.numericalVertices) {
        if (!Number.isFinite(coords[0])) validCoords = false;
        if (!Number.isFinite(coords[1])) validCoords = false;
    }

    // build the JSXGraph attribute objects for points and edges
    // note: points go on a higher layer so they appear on top of edge lines
    jsxPointAttributes.current = {
        layer: 10 * SVs.layer + VERTEX_LAYER_OFFSET,
        fillColor: lineColor,
        size: normalizePointSize(SVs.selectedStyle.markerSize, SVs.selectedStyle.markerStyle),
        face: SVs.selectedStyle.markerStyle,
        showInfoBox: SVs.showCoordsWhenDragging,
        // ...more style stuff
    };

    // create a JSXGraph point for each vertex
    pointsJXG.current = [];
    for (let i = 0; i < SVs.numVertices; i++) {
        pointsJXG.current.push(
            board.create("point", [...SVs.numericalVertices[i]], pointAttributes)
        );
    }

    // attach drag/click/keyboard event handlers to each point
    for (let i = 0; i < SVs.numVertices; i++) {
        pointsJXG.current[i].on("drag", (e) => pointDragHandler(i, e));
        pointsJXG.current[i].on("up", () => upHandler(i));
        pointsJXG.current[i].on("down", (e) => downHandler(i, e));
        // ...etc
    }

    // create edge segments
    // edges are drawn slightly shorter so they don't visually overlap the point circles
    // the math here just moves each endpoint inward by (point radius / scale)
    for (let i = 0; i < SVs.numEdges; i++) {
        let edge = SVs.edges[i];
        // edge[0] and edge[1] are 1-based vertex indices
        let x0 = pointsJXG.current[edge[0] - 1].X();
        let y0 = pointsJXG.current[edge[0] - 1].Y();
        // ... compute shortened endpoints, then:
        edgesJXG.current.push(
            board.create("segment", [[newX0, newY0], [newX1, newY1]], edgeAttributes)
        );
    }
}
```

---

### DiscreteGraph.js - numericalVertices state variable

```js
// numericalVertices converts the math-expression vertices into plain JS numbers
// the renderer needs plain numbers to pass to JSXGraph
stateVariableDefinitions.numericalVertices = {
    isArray: true,
    forRenderer: true,
    returnArraySizeDependencies: () => ({
        numVertices: { dependencyType: "stateVariable", variableName: "numVertices" },
    }),
    returnArraySize({ dependencyValues }) {
        return [dependencyValues.numVertices];
    },
    returnArrayDependenciesByKey({ arrayKeys }) {
        // each key gets a dependency on the corresponding vertex state variable
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
            // if any coordinate is not a finite number, set the whole vertex to NaN
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

### discreteGraph.jsx - pointDragHandler

```jsx
function pointDragHandler(i, e) {
    let viaPointer = e.type === "pointermove";

    // ignore tiny accidental movements
    if (
        !viaPointer ||
        Math.abs(e.x - pointerAtDown.current[0]) > 0.1 ||
        Math.abs(e.y - pointerAtDown.current[1]) > 0.1
    ) {
        // record where the user dragged this point to
        pointCoords.current = {};
        pointCoords.current[i] = [pointsJXG.current[i].X(), pointsJXG.current[i].Y()];

        // fire the action back to core with the new coords
        // transient: true means "don't save this to history yet, still dragging"
        // skippable: true means "if another action is in progress, skip this one"
        callAction({
            action: actions.moveDiscreteGraph,
            args: {
                pointCoords: pointCoords.current,
                transient: true,
                skippable: true,
                sourceDetails: { vertex: i },
            },
        });

        // update connected edges visually while dragging
        // (searches through SVs.edges for edges that include point i)
        // ... edge update math ...

        // snap the point BACK to where core last said it was
        // this is the key pattern - the renderer doesn't move the point,
        // it waits for core to respond with the new position
        pointsJXG.current[i].coords.setCoordinates(
            JXG.COORDS_BY_USER,
            [...lastPositionsFromCore.current[i]]
        );
    }
}
```

---

### DiscreteGraph.js - moveDiscreteGraph action

```js
// this runs in the worker when the renderer fires moveDiscreteGraph
async moveDiscreteGraph({ pointCoords, transient, sourceDetails, actionId, ... }) {

    let numVerticesMoved = Object.keys(pointCoords).length;

    // check drag permissions
    if (numVerticesMoved === 1) {
        if (!(await this.stateValues.verticesDraggable)) return;
    } else {
        if (!(await this.stateValues.draggable)) return;
    }

    // convert plain number coords to math expressions
    // because the vertices state variable stores math expressions
    let vertexComponents = {};
    for (let ind in pointCoords) {
        vertexComponents[ind + ",0"] = me.fromAst(pointCoords[ind][0]);
        vertexComponents[ind + ",1"] = me.fromAst(pointCoords[ind][1]);
    }

    // performUpdate triggers the state variable system to recompute
    // which will send new forRenderer values back to the renderer
    await this.coreFunctions.performUpdate({
        updateInstructions: [{
            updateType: "updateValue",
            componentName: this.componentName,
            stateVariable: "vertices",
            value: vertexComponents,
            sourceDetails,
        }],
        transient,
        actionId,
        // ...
    });
}
```

---
