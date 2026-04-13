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
