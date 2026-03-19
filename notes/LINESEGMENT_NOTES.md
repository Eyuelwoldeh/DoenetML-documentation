# LineSegment component notes

file: `packages/doenetml-worker-javascript/src/components/LineSegment.js`

extends `GraphicalComponent`. it draws a line segment on a graph with two endpoints.

---

## attributes

- `draggable` (boolean, default true) -- whether the whole segment can be dragged
- `endpointsDraggable` (boolean) -- whether the individual endpoints can be dragged. if not set, it falls back to whatever `draggable` is
- `endpoints` (pointList) -- lets you set the two endpoints directly as an attribute
- `showCoordsWhenDragging` (boolean, default true) -- shows coordinates while dragging
- `labelPosition` (text, default "upperright") -- where to place the label. valid values are "upperright", "upperleft", "lowerright", "lowerleft"

also inherits rounding attributes from the shared rounding utilities.

---

## state variables

### numDimensions
reads from the `endpoints` attribute component to figure out how many dimensions the points are in. always at least 2.

### unconstrainedEndpoints
a 2D array (2 points x numDimensions). this holds the raw unconstrained positions of the endpoints. if no `endpoints` attribute is given, it uses essential values (defaults to (1,0) and (0,0)).

this is separate from `endpoints` because constraints can shift positions around. the unconstrained version stores where the user actually put the point before any snapping or sticking happens.

```js
// array key format is "pointIndex,dimension", e.g. "0,0" is x of point 1
let x1 = unconstrainedEndpoints[0][0];
let y2 = unconstrainedEndpoints[1][1];
```

### haveConstrainedEndpoints
just checks whether the segment is in a sticky group. if true, the `endpoints` variable will go through constraint logic before returning values.

### endpoints
the actual public-facing endpoints. depends on `haveConstrainedEndpoints`:
- if false: just reads from `unconstrainedEndpoints` directly
- if true: applies sticky group constraints first

this is what you should read if you want the real position of the endpoints. it's public and accessible as `$seg.endpoint1`, `$seg.endpoint2`, `$seg.endpointX1_1`, etc.

the inverse definition handles dragging. if both endpoints move (whole segment drag), it tries to keep the relative positions fixed even if one endpoint is constrained.

### numericalEndpoints
same as `endpoints` but evaluated to plain JS numbers using `.evaluate_to_constant()`. this is what gets passed to the renderer since it can't deal with math expression objects.

```js
// these are regular numbers, not math-expression objects
let x = numericalEndpoints[0][0]; // x coord of first endpoint
```

### length
computes the euclidean distance between the two endpoints. if all values are numeric it does it directly with `Math.sqrt`. if some values are symbolic it builds a math expression tree instead.

the inverse definition for `length` is interesting: it keeps the midpoint fixed and scales the segment outward/inward along its current direction to reach the desired length.

### slope
just `(y2 - y1) / (x2 - x1)` using `numericalEndpoints`. only works in 2D, returns NaN otherwise.

### parallelCoords
a vector `(x2 - x1, y2 - y1)` representing the direction of the segment. used as an adapter so the segment can act as a direction component when referenced by other things.

### nearestPoint
returns a function that takes a point and returns the closest point on the segment to it. uses the parametric line formula to find `t`, then clamps `t` to [0, 1] so the result stays on the segment (not the infinite line through the endpoints).

### nearestPointAsLine
same math but does NOT clamp `t`, so it finds the nearest point on the infinite line through the endpoints, not just the segment.

### styleDescription / styleDescriptionWithNoun
builds a human-readable string like "thin dashed red line segment". reads from the selected style and checks the document theme (light vs dark mode) to pick the right color word.

---

## actions

### moveLineSegment
called when the user drags the segment. checks `draggable` and `endpointsDraggable` first.

if only one endpoint was given coords, it checks `endpointsDraggable` before allowing it.
if both endpoints were given coords (whole segment drag), it checks `draggable`.

there's extra logic for when the segment is dragged as a whole but one endpoint is constrained. in that case it computes how far the constrained endpoint actually moved vs how far it was asked to move, and applies that offset to the other endpoint so the segment stays rigid.

```js
// the action receives coords like this:
// point1coords = [x, y]
// point2coords = [x, y]
// if only one is defined, an individual endpoint was dragged
```

### lineSegmentClicked / lineSegmentFocused
these just trigger chained actions (like answer submission triggers) if the segment is not fixed. they use `componentIdx` from the argument rather than `this.componentIdx` so they work correctly when the segment has been adapted into another component type.

---

## adapter

the segment exposes a `parallelCoords` adapter that converts it into a `_directionComponent`. this means other components that accept a direction can use a line segment directly and it will just use the segment's direction vector.

---

## a few things worth knowing

the endpoints state variable uses two different code paths depending on `haveConstrainedEndpoints`. this is controlled by `stateVariablesDeterminingDependencies`, which tells the system to recompute dependencies when that value changes. it's a pattern used when you don't know ahead of time which dependencies you'll need.

the `workspace` object in the inverse definition is used to accumulate partial updates. when a user drags a constrained segment, sometimes the x and y components of a point come in as separate updates. the workspace lets the code wait and collect all of them before running the constraint logic.

array keys throughout this file use the format `"pointIndex,dim"` where both are zero-indexed. so the y-coordinate of the second endpoint is key `"1,1"`.
