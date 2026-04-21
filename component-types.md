# Component Types

a guide for making new DoenetML components, organized by what kind of component you want to build.

every component lives in:
```
packages/doenetml-worker-javascript/src/components/
```

every component is a JS class that extends one of the abstract base classes in `abstract/`. the base class you extend determines what kind of component you're building.

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

## pick your type

- **[inline components](component-types/inline.md)** -- things that show up inline in text, like a number or a math expression
- **[graphical components](component-types/graphical.md)** -- things that show up on a JSXGraph board inside a `<graph>` tag
- **[composite components](component-types/composite.md)** -- components that expand into other components, like lists and sequences
- **[input components](component-types/input.md)** -- things the user can type into or interact with
- **[block components](component-types/block.md)** -- block-level layout components like paragraphs, figures, tables

---

## registering a new component

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

## quick reference: attribute patterns

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

## quick reference: state variable return values

```js
// set a computed value
return { setValue: { myVar: 42 } };

// use essential (stored) value or a default
return { useEssentialOrDefaultValue: { myVar: { defaultValue: 0 } } };
```

## quick reference: child access

```js
// string children are raw strings
let str = dependencyValues.stringChildren[0];  // "hello"

// number/math/text children are objects with stateValues
let num = dependencyValues.numberChildren[0].stateValues.value;  // 42

// math children need evaluate_to_constant() to get a JS number
let val = dependencyValues.mathChildren[0].stateValues.value.evaluate_to_constant();

// attribute components are accessed via stateValues too
let count = dependencyValues.myAttr.stateValues.numComponents;
```

---

## build and test

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

open `http://localhost:8012/` and edit `packages/test-viewer/src/test/testCode.doenet` to test your component.
