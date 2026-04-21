# Graphical Components

things that draw something on a JSXGraph board, which lives inside a `<graph>` tag. examples include points, lines, polygons, circles, and custom graph objects.

**base class:** `GraphicalComponent`

**examples:** `Point.js`, `Line.js`, `LineSegment.js`, `Polygon.js`, `Polyline.js`, `DiscreteGraph.js`, `Circle.js`

**renderer:** a `.jsx` file that gets the JSXGraph board from React context and uses it to draw and update objects

---

## basic structure

graphical components have two files:
1. a `.js` file in `components/` that does the math (runs in the web worker)
2. a `.jsx` file in `renderers/` that does the drawing (runs in the browser)

the `.js` file computes coordinates and styling, marks them `forRenderer: true`, and those get sent to the browser. the `.jsx` file reads them as `SVs` and draws them. if the component is draggable, the renderer fires an action back to the worker with the new position, and the worker validates and updates state, which flows back as new `SVs`.

### the worker side (.js)

```js
import GraphicalComponent from "./abstract/GraphicalComponent";

export default class MyGraphThing extends GraphicalComponent {
    constructor(args) {
        super(args);
        // register any actions the renderer will need to call back.
        // actions are methods on this class that update state.
        Object.assign(this.actions, {
            myGraphThingClicked: this.myGraphThingClicked.bind(this),
            myGraphThingFocused: this.myGraphThingFocused.bind(this),
        });
    }

    static componentType = "myGraphThing";

    static createAttributesObject() {
        let attributes = super.createAttributesObject();
        // GraphicalComponent already gives you: hidden, fixed, layer, selectedStyle.

        // add draggable if you want the user to be able to drag it
        attributes.draggable = {
            createComponentOfType: "boolean",
            createStateVariable: "draggable",
            defaultValue: true,
            public: true,
            forRenderer: true,
        };

        // use numberList to accept a space-separated list of numbers as an attribute.
        // the numberList component handles all the parsing for you.
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

        // numericalPoints: the [x, y] pairs the renderer will draw.
        // isArray: true means the DoenetML system handles it as an array where each
        // element is computed individually. the four functions below define it:
        //   returnArraySizeDependencies: what state vars determine the array size
        //   returnArraySize: returns [length] based on those deps
        //   returnArrayDependenciesByKey: what each element depends on
        //   arrayDefinitionByKey: computes each element from its deps
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
                // use globalDependencies when all elements share the same source
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

    // actions are called by the renderer to push updates back to state
    async myGraphThingClicked({ actionId, sourceInformation = {} }) {
        await this.coreFunctions.requestRecordEvent({
            verb: "interacted",
            object: { componentName: this.componentName, componentType: this.componentType },
            result: { focused: true },
        });
    }
}
```

---

## the renderer side (.jsx)

```jsx
// packages/doenetml/src/Viewer/renderers/myGraphThing.jsx
import React, { useContext, useEffect, useRef } from "react";
import useDoenetRenderer from "../useDoenetRenderer";
import { BoardContext } from "./graph";

export default React.memo(function MyGraphThing(props) {
    // SVs = state values, actions = action functions registered in the .js file
    let { name, id, SVs, actions, callAction } = useDoenetRenderer(props);

    // board is the JSXGraph board provided by the parent <graph> component.
    // all drawing goes through this object.
    const board = useContext(BoardContext);

    // pointsJXG holds the actual JSXGraph point objects.
    // kept in a ref because updating them doesn't need to trigger a React re-render.
    let pointsJXG = useRef(null);

    // clean up JSXGraph objects when this component is removed from the page
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
            // first render: create the JSXGraph objects from scratch
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
            // subsequent renders: update positions and handle count changes
            let prevCount = pointsJXG.current.length;

            // add points if count increased
            for (let i = prevCount; i < SVs.numElements; i++) {
                let pt = board.create("point", SVs.numericalPoints[i], {
                    name: SVs.pointLabels ? SVs.pointLabels[i] : "",
                    withLabel: true,
                    fixed: SVs.fixed,
                    layer: 10 * SVs.layer + 1,
                });
                pointsJXG.current.push(pt);
            }

            // remove points if count decreased
            for (let i = SVs.numElements; i < prevCount; i++) {
                board.removeObject(pointsJXG.current[i]);
            }
            pointsJXG.current = pointsJXG.current.slice(0, SVs.numElements);

            // move remaining points to new positions
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

---

## what GraphicalComponent gives you for free

- `hidden`: whether the component is visible
- `fixed`: whether the component can be interacted with at all
- `layer`: z-ordering on the graph
- `selectedStyle`: line color, width, style, marker size, marker style
- `labelPosition`: where to place the label relative to the component
- `showCoordsWhenDragging`: whether to show coordinates during drag
- label-related state variables

you don't need to define these yourself. just use them in your renderer via `SVs`.

---

## the numericalPoints / numericalVertices pattern

most graphical components store their positions as math expression objects internally (because they need to support symbolic math). but the renderer needs plain JS numbers to pass to JSXGraph.

so you create a `numericalSomething` state variable that converts them using `evaluate_to_constant()`:

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
        // each element gets its own dependency on the corresponding vertex
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
            // if any coordinate is not a finite number, fill the whole vertex with NaN
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
3. the `.js` action method validates, converts coords, and calls `performUpdate()`
4. the state variable system recomputes
5. new `forRenderer` values flow back to the renderer
6. renderer updates the JSXGraph objects with the new positions

the renderer should NOT move the point on its own. instead it fires an action, waits for the worker to respond with the validated position (which may be different due to constraints), and then updates the visual. this is the "snap back" pattern:

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
    // (the new confirmed position will come back via SVs on the next render)
    pointsJXG.current[i].coords.setCoordinates(
        JXG.COORDS_BY_USER,
        [...lastPositionsFromCore.current[i]]
    );
}
```

---

## constrained vs unconstrained positions

some graphical components (like LineSegment, Polygon) support sticky groups and constraint snapping. the pattern is:

- `unconstrainedVertices`: where the user actually put the point
- `haveConstrainedVertices`: boolean, true if inside a sticky group
- `vertices`: the public state variable. if constrained, runs through the constraint logic first. if not, passes through from unconstrained

this means the renderer always reads from `vertices` (or `numericalVertices`), and the constraint system is transparent.

the `stateVariablesDeterminingDependencies` property tells the system to recompute dependencies when `haveConstrainedVertices` changes. it's a pattern used when you don't know ahead of time which dependencies you'll need.

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

1. **not using `forRenderer: true`**: the renderer only sees state variables marked with this flag

2. **forgetting to convert math expressions to numbers**: the renderer can't use math expression objects. always create a `numerical*` version using `evaluate_to_constant()`

3. **moving points directly in the renderer**: always let core validate the position first. snap back to the last confirmed position during drag

4. **not cleaning up JSXGraph objects**: use a `useEffect` cleanup function to call `board.removeObject()` on unmount

5. **forgetting `board.updateRenderer()`**: after updating JSXGraph object positions, call this so the board redraws
