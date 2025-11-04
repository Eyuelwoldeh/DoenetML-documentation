# Key Differences in Doenet Component Structures

This document highlights the important differences between the `Text`, `Point`, `Number`, and `LineSegment` components in Doenet. While they share a common structure, their specific implementations and purposes lead to significant distinctions.

## 1. Base Class Inheritance

A primary difference is the base class from which they inherit. This determines their fundamental behavior and how they are rendered.

- **`GraphicalComponent`:** `Point` and `LineSegment` inherit from this class. These components are intended to be rendered within a graphical context, such as a graph.
- **`InlineComponent`:** `Text` and `Number` inherit from this class. These components are rendered inline with other text-based content.

**Example:**

```javascript
// Point.js
export default class Point extends GraphicalComponent { ... }

// Text.js
export default class Text extends InlineComponent { ... }
```

## 2. Implicit Property

The `variableForImplicitProp` static property is used to determine which state variable is assigned the component's content when no explicit property is specified. This is a feature of components that can contain content directly.

- **`Text` and `Number`** have this property, as they can represent a value directly from their children.
- **`Point` and `LineSegment`** do not have this property, as their primary value (coordinates) is typically defined through attributes.

**Example:**

```javascript
// Number.js
static variableForImplicitProp = "value";

// Text.js
static variableForImplicitProp = "value";
```

## 3. Attributes (`createAttributesObject`)

The attributes defined for each component are highly specific to their function.

- **`Point`:** Has attributes like `x`, `y`, `z`, and `coords` to define its position.
- **`LineSegment`:** Has an `endpoints` attribute to define the two points that form the segment.
- **`Number`:** Has attributes like `valueOnNaN` and `renderAsMath` to control its numerical behavior and appearance.
- **`Text`:** Has an `isLatex` attribute to determine if the text content should be parsed as LaTeX.

## 4. State Variables (`returnStateVariableDefinitions`)

The state variables reflect the core data and properties of each component.

- **`Point`:** Key state variables include `xs` (the coordinates as an array), `coords` (the coordinates as a vector), and `numDimensions`.
- **`LineSegment`:** Key state variables include `endpoints`, `length`, and `slope`.
- **`Number`:** Key state variables include `value`, `math` (a math-expression representation), and `text`.
- **`Text`:** Key state variables include `value` (the text content), `math` (a math-expression representation), and `number`.

## 5. Child Groups (`returnChildGroups`)

Each component defines which types of children it can accept and how they are grouped. This is crucial for how the component processes its content.

- **`Point`:** Accepts `pointsAndVectors` and `constraints`.
- **`Text`:** Accepts `textLike` children, which include strings and other text-related components.
- **`Number`:** Accepts a variety of child groups, including `strings`, `numbers`, `maths`, `texts`, and `booleans`, and its behavior changes based on the children present.
- **`LineSegment`:** Does not define custom child groups in the same way, as its definition is primarily through attributes.

## 6. Syntactic Sugar (`returnSugarInstructions`)

Some components provide "sugar" to make the DoenetML syntax more convenient. This is handled by the `returnSugarInstructions` static method.

- **`Point` and `Number`** have sugar instructions to simplify how they are defined. For example, a point can be defined with coordinates in parentheses like `(1,2)`, and a number can have math expressions directly as children.
- **`LineSegment` and `Text`** do not have these specific sugar instructions in the analyzed files.

## Summary of Key Differences

| Feature                       | `LineSegment`         | `Number`                 | `Point`                 | `Text`            |
| ----------------------------- | --------------------- | ------------------------ | ----------------------- | ----------------- |
| **Base Class**                | `GraphicalComponent`  | `InlineComponent`        | `GraphicalComponent`    | `InlineComponent` |
| **`variableForImplicitProp`** | ❌                    | `value`                  | ❌                      | `value`           |
| **Primary Attributes**        | `endpoints`           | `valueOnNaN`             | `x`, `y`, `z`, `coords` | `isLatex`         |
| **Core State Variables**      | `endpoints`, `length` | `value`, `math`          | `xs`, `coords`          | `value`, `text`   |
| **Child Groups**              | (none)                | `numbers`, `maths`, etc. | `pointsAndVectors`      | `textLike`        |
| **Syntactic Sugar**           | ❌                    | ✅                       | ✅                      | ❌                |
