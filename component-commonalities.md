# Commonalities in Doenet Component Structures

This document outlines the common structural patterns found in Doenet components, using `Text`, `Point`, `Number`, and `LineSegment` as examples. Understanding these commonalities is crucial for developing new components and maintaining existing ones.

## Core Commonalities

All Doenet components share a fundamental structure. Here are the key commonalities:

### 1. Class-Based Structure

Doenet components are implemented as JavaScript classes. These classes inherit from a base component class, which provides them with a common set of functionalities. The components we analyzed inherit from either `GraphicalComponent` or `InlineComponent`.

**Example:**

```javascript
// Point.js
export default class Point extends GraphicalComponent {
  // ...
}

// Text.js
export default class Text extends InlineComponent {
  // ...
}
```

### 2. Static `componentType` Property

Every component has a static `componentType` property. This property is a string that uniquely identifies the type of the component.

**Example:**

```javascript
// Point.js
static componentType = "point";

// Text.js
static componentType = "text";
```

### 3. Constructor

The constructor of a Doenet component typically performs two main tasks:

1.  It calls the parent class's constructor using `super(args)`.
2.  It registers the component's actions (i.e., methods that can be called externally) by assigning them to `this.actions`.

**Example:**

```javascript
// Point.js
constructor(args) {
    super(args);

    Object.assign(this.actions, {
        movePoint: this.movePoint.bind(this),
        pointClicked: this.pointClicked.bind(this),
        pointFocused: this.pointFocused.bind(this),
    });
}
```

### 4. Static `createAttributesObject` Method

This static method defines the attributes that can be set on the component in a DoenetML document. Each attribute is an object that specifies its type and other properties.

**Example:**

```javascript
// Point.js
static createAttributesObject() {
    let attributes = super.createAttributesObject();
    attributes.draggable = {
        createComponentOfType: "boolean",
        createStateVariable: "draggable",
        defaultValue: true,
        public: true,
        forRenderer: true,
    };
    // ... other attributes
    return attributes;
}
```

A common attribute found in many components is `draggable`, which controls whether the user can interact with and move the component.

### 5. Static `returnStateVariableDefinitions` Method

This static method is the heart of a component's logic. It defines the component's internal state variables. Each state variable is an object that specifies how its value is calculated, its dependencies, and other properties.

**Example:**

```javascript
// Point.js
static returnStateVariableDefinitions() {
    let stateVariableDefinitions = super.returnStateVariableDefinitions();

    stateVariableDefinitions.numDimensions = {
        public: true,
        // ... definition
    };

    stateVariableDefinitions.coords = {
        public: true,
        // ... definition
    };

    // ... other state variables

    return stateVariableDefinitions;
}
```

### 6. Actions

Components define actions that can be triggered by user interactions or other events. These actions are methods of the component class and are registered in the constructor. Common actions include:

- `move<ComponentName>`: For moving the component.
- `<componentName>Clicked`: For handling click events.
- `<componentName>Focused`: For handling focus events.

### 7. Adapters

Components can have a static `adapters` property. This property allows a component to be treated as another type of component. For example, a `number` component can be adapted to a `math` or `text` component.

**Example:**

```javascript
// Number.js
static adapters = [
    {
        stateVariable: "math",
        stateVariablesToShadow: Object.keys(
            returnRoundingStateVariableDefinitions(),
        ),
    },
    "text",
];
```

## Summary Table

| Feature                              | `LineSegment` | `Number` |   `Point`   |  `Text`  | Description                                                         |
| ------------------------------------ | :-----------: | :------: | :---------: | :------: | ------------------------------------------------------------------- |
| **Inherits from**                    |  `Graphical`  | `Inline` | `Graphical` | `Inline` | All components extend a base class.                                 |
| **`componentType`**                  |      ✅       |    ✅    |     ✅      |    ✅    | A unique string identifier for the component type.                  |
| **`constructor`**                    |      ✅       |    ✅    |     ✅      |    ✅    | Initializes the component and registers actions.                    |
| **`createAttributesObject`**         |      ✅       |    ✅    |     ✅      |    ✅    | Defines the component's attributes.                                 |
| **`returnStateVariableDefinitions`** |      ✅       |    ✅    |     ✅      |    ✅    | Defines the component's state variables.                            |
| **Actions**                          |      ✅       |    ✅    |     ✅      |    ✅    | Methods that define the component's behavior in response to events. |
| **Adapters**                         |      ✅       |    ✅    |     ✅      |    ✅    | Allows the component to be treated as another type of component.    |

By following these common patterns, we can ensure that Doenet components are consistent, maintainable, and easy to integrate into the Doenet platform.
