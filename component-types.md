# Component Types

a guide for making new DoenetML components, organized by what kind of component you want to build.

every component lives in:
```
packages/doenetml-worker-javascript/src/components/
```

every component is a JS class that extends one of the abstract base classes in `abstract/`. the base class you extend determines what kind of component you're building, what lifecycle methods you get for free, and what the renderer can expect.

---

## the inheritance tree (simplified)

```
BaseComponent
├── InlineComponent          -- text, number, integer, math, boolean
│   ├── Input                -- textInput, mathInput, booleanInput
│   └── SingleCharacterInline -- br, nbsp, etc.
├── BlockComponent           -- p, figure, table, image, video
│   ├── SectioningComponent  -- section, chapter, etc.
│   └── BlockScoredComponent -- answer
├── GraphicalComponent       -- point, line, polygon, discreteGraph
├── CompositeComponent       -- numberList, mathList, sequence, select
└── ConstraintComponent      -- constrainToGrid, attractTo, etc.
```

---

## how components work in general

before picking a type, it helps to understand the overall flow:

1. DoenetML content is parsed into a tree of components
2. each component's `.js` file runs in a web worker, not the browser tab. this is where state is computed and stored
3. state variables marked `forRenderer: true` get sent to the browser
4. each component's renderer (`.jsx` or `.tsx`) runs in the browser and draws the component using those state values
5. when the user does something (clicks, drags, types), the renderer fires an action back to the worker, which updates state, which sends new values back to the renderer

so: the `.js` file is the brain, the renderer is the face, and actions are how they talk to each other.

---

## pick your type

- **[inline components](component-types/inline.md)**: things that show up inline in text, like a number or a math expression
- **[graphical components](component-types/graphical.md)**: things that show up on a JSXGraph board inside a `<graph>` tag
- **[composite components](component-types/composite.md)**: components that expand into other components, like lists and sequences
- **[input components](component-types/input.md)**: things the user can type into or interact with
- **[block components](component-types/block.md)**: block-level layout components like paragraphs, figures, tables

---

## registering a new component

after creating the `.js` file, you MUST register it in:
`packages/doenetml-worker-javascript/src/ComponentTypes.js`

otherwise the DoenetML parser will not know the component type exists and will throw an error.

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

the renderer filename must match `rendererType` (lowercase first letter by convention). if you don't set `rendererType`, it defaults to your `componentType`.

---

## quick reference: attribute patterns

```js
// option 1: attribute that creates a state variable automatically
attributes.myAttr = {
    createComponentOfType: "boolean",  // "number", "text", "math", "integer", etc.
    createStateVariable: "myAttr",     // name of the resulting state variable
    defaultValue: false,
    public: true,
    forRenderer: true,
};

// option 2: attribute as a primitive string (no component created)
// useful for mode/enum-style attributes like representation="binary"
attributes.mode = {
    createPrimitiveOfType: "string",
    createStateVariable: "mode",
    defaultValue: "decimal",
    public: true,
};

// option 3: attribute that holds a complex component type
// access its state variables via dependencyValues.values.stateValues.whatever
attributes.values = {
    createComponentOfType: "numberList",
};
```

## quick reference: state variable return values

```js
// return a computed value
return { setValue: { myVar: 42 } };

// return multiple values at once
return { setValue: { myVar: 42, myOtherVar: "hello" } };

// use essential (stored) value or a default
return { useEssentialOrDefaultValue: { value: { defaultValue: 0 } } };

// signal that the value could not be computed (not an error, just no value)
return { setValue: { myVar: null } };
```

## quick reference: dependency types

```js
returnDependencies: () => ({
    // depend on a state variable from this same component
    myOtherVar: {
        dependencyType: "stateVariable",
        variableName: "myOtherVar",
    },

    // depend on children in a named group
    numberChildren: {
        dependencyType: "child",
        childGroups: ["numbers"],
        variableNames: ["value"], // which state vars to pull from each child
    },

    // depend on an attribute component (for complex attribute types like numberList)
    values: {
        dependencyType: "attributeComponent",
        attributeName: "values",
        variableNames: ["numbers", "numComponents"],
    },

    // depend on the parent component's state
    parentValue: {
        dependencyType: "parentStateVariable",
        variableName: "someValue",
    },
}),
```

## quick reference: child access

```js
// string children are raw strings (the text between tags)
let str = dependencyValues.stringChildren[0];  // "hello"

// always check length before accessing to avoid runtime errors
if (dependencyValues.stringChildren.length > 0) {
    let first = dependencyValues.stringChildren[0];
}

// number/math/text children are objects with stateValues
let num = dependencyValues.numberChildren[0].stateValues.value;  // 42

// math children need evaluate_to_constant() to get a plain JS number
let val = dependencyValues.mathChildren[0].stateValues.value.evaluate_to_constant();

// attribute components are accessed via stateValues
let count = dependencyValues.values.stateValues.numComponents;
let arr   = dependencyValues.values.stateValues.numbers;  // for numberList
```

---

## build and test

after creating or modifying a component:

```bash
cd DoenetML

# step 1: rebuild the worker-javascript package (your .js component lives here)
npm run build -w packages/doenetml-worker-javascript

# step 2: rebuild the worker bundle
npm run build -w packages/doenetml-worker

# step 3: rebuild the main doenetml package (includes renderers)
npm run build -w packages/doenetml

# step 4: start the test viewer dev server
cd packages/test-viewer
npm run dev
```

open `http://localhost:8012/` and edit `packages/test-viewer/src/test/testCode.doenet` to test your component.

the test viewer dev server has hot module replacement for renderer files (`.jsx`/`.tsx`), so renderer-only changes might appear without a full rebuild. changes to `.js` worker files always need the full rebuild chain above.
