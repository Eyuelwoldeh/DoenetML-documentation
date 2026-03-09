# Creating DoenetML Components

> a guide for devs who want to extend DoenetML by adding features to existing components or building new ones

## Introduction

DoenetML components are JavaScript classes that define interactive educational elements. each component has attributes users can set, state variables that hold reactive data, and a renderer that displays things on screen.

this guide uses the `Integer` component as a real example — specifically the work done to add binary and hexadecimal support.

### what you'll need to know first

- basic JavaScript
- what reactive state management roughly means
- basic familiarity with DoenetML syntax

---

## component architecture

every DoenetML component follows this structure:

```javascript
export default class MyComponent extends BaseComponent {
    static componentType = "myComponent"; // the tag name, e.g. <myComponent>
    static rendererType = "text";         // which renderer to use for display

    static createAttributesObject() {
        // define what attributes the tag can accept
    }

    static returnStateVariableDefinitions() {
        // define reactive state variables
    }
}
```

### where components live

all component files are in:

```
packages/doenetml-worker-javascript/src/components/
```

components extend each other. for example, `Integer.js` extends `Number.js`, which extends `InlineComponent`.

---

## step 1 — adding attributes

attributes are the things users put on a tag:

```xml
<integer representation="binary">1001</integer>
```

you define them in `createAttributesObject()`:

```javascript
static createAttributesObject() {
    // always call super first to get the parent's attributes
    let attributes = super.createAttributesObject();

    attributes.representation = {
        createPrimitiveOfType: "string",      // the type: "string", "number", "boolean"
        createStateVariable: "representation", // auto-creates a state variable with this name
        defaultValue: "decimal",              // fallback if the user doesn't set it
        public: true,                         // lets other components read it via $myInt.representation
        forRenderer: true,                    // passes it to the renderer
    };

    return attributes;
}
```

> **note:** always call `super.createAttributesObject()` and extend the result — don't replace it. if you do `return { myAttr: ... }` without calling super, you lose all the parent's attributes.

---

## step 2 — defining state variables

state variables hold the component's reactive data. they recompute automatically when their dependencies change. you define them in `returnStateVariableDefinitions()`.

### anatomy of a state variable

```javascript
stateVariableDefinitions.myVariable = {
    public: true,        // lets other components read it via $component.myVariable
    forRenderer: true,   // passes it to the renderer for display

    returnDependencies: () => ({
        // list everything this variable needs to compute itself
        someOtherVar: {
            dependencyType: "stateVariable",
            variableName: "someOtherVar",
        },
    }),

    definition({ dependencyValues }) {
        // compute and return the value
        let result = doSomethingWith(dependencyValues.someOtherVar);

        return {
            setValue: { myVariable: result },
        };
    },

    // optional — used for two-way binding (when you need changes to propagate back)
    async inverseDefinition({ desiredStateVariableValues, dependencyValues }) {
        return {
            success: true,
            instructions: [{
                setDependency: "someOtherVar",
                desiredValue: desiredStateVariableValues.myVariable,
            }],
        };
    },
};
```

### how child types work (important!)

when a user writes `<integer>1001</integer>`, the `1001` becomes a **string child**. string children are raw strings, not objects. this trips people up a lot:

```javascript
// string children — raw strings, access directly
let str = dependencyValues.stringChildren[0]; // "1001"

// text children — component objects, use .stateValues
let str = dependencyValues.textChildren[0].stateValues.value; // "1001"

// number children — component objects, use .stateValues
let num = dependencyValues.numberChildren[0].stateValues.value; // 1001

// math children — component objects, need to evaluate
let val = dependencyValues.mathChildren[0].stateValues.value.evaluate_to_constant();
```

> **don't do this:** `dependencyValues.stringChildren[0].stateValues.value` — strings don't have `.stateValues`, so this crashes.

---

## step 3 — overriding parent state variables

since `Integer` extends `Number`, it inherits all of Number's state variables. to replace one:

```javascript
static returnStateVariableDefinitions() {
    let stateVariableDefinitions = super.returnStateVariableDefinitions();

    // delete the parent's version
    delete stateVariableDefinitions.value;

    // define your own
    stateVariableDefinitions.value = {
        // your new definition here
    };

    return stateVariableDefinitions;
}
```

you can also use `renameStateVariable()` to keep the old one under a different name instead of deleting it:

```javascript
// renames "value" to "valuePreRound" so we can still use it
renameStateVariable({
    stateVariableDefinitions,
    oldName: "value",
    newName: "valuePreRound",
});
```

or you can just add new state variables without touching the existing ones.

---

## step 4 — adding representation state variables

a nice pattern for formatting is to expose each format as its own state variable. users can then access `$a.binary`, `$a.hexadecimal`, `$a.decimal` independently.

```javascript
stateVariableDefinitions.binary = {
    public: true,
    shadowingInstructions: {
        createComponentOfType: "text", // when this is copied/referenced, create a text component
    },
    returnDependencies: () => ({
        value: {
            dependencyType: "stateVariable",
            variableName: "value",
        },
    }),
    definition({ dependencyValues }) {
        let value = dependencyValues.value;

        // handle edge cases first
        if (Number.isNaN(value) || !Number.isFinite(value)) {
            return { setValue: { binary: String(value) } };
        }
        if (value === 0) {
            return { setValue: { binary: "0" } };
        }

        let binaryString = value < 0
            ? "-" + Math.abs(value).toString(2)
            : value.toString(2);

        return { setValue: { binary: binaryString } };
    },
};
```

do the same thing for `hexadecimal` and `decimal`.

---

## step 5 — updating the `text` state variable

the `text` state variable is what the renderer actually displays. if your component supports multiple representations, override it to show the right one.

```javascript
stateVariableDefinitions.text = {
    public: true,
    shadowingInstructions: { createComponentOfType: "text" },
    forRenderer: true,  // don't forget this — the renderer reads `text`
    returnDependencies: () => ({
        representation: {
            dependencyType: "stateVariable",
            variableName: "representation",
        },
        decimal: {
            dependencyType: "stateVariable",
            variableName: "decimal",
        },
        binary: {
            dependencyType: "stateVariable",
            variableName: "binary",
        },
        hexadecimal: {
            dependencyType: "stateVariable",
            variableName: "hexadecimal",
        },
    }),
    definition({ dependencyValues }) {
        let text = "";

        if (dependencyValues.representation === "binary") {
            text = dependencyValues.binary;
        } else if (dependencyValues.representation === "hexadecimal") {
            text = dependencyValues.hexadecimal;
        } else {
            text = dependencyValues.decimal;
        }

        return { setValue: { text } };
    },

    // inverse definition — parses user input back to decimal
    async inverseDefinition({ desiredStateVariableValues, dependencyValues }) {
        let desiredText = desiredStateVariableValues.text;
        let representation = dependencyValues.representation;
        let desiredNumber = NaN;

        if (representation === "binary") {
            desiredNumber = parseBinary(desiredText);
        } else if (representation === "hexadecimal") {
            desiredNumber = parseHexadecimal(desiredText);
        } else {
            desiredNumber = Number(desiredText);
        }

        if (Number.isFinite(desiredNumber)) {
            return {
                success: true,
                instructions: [{
                    setDependency: "value",
                    desiredValue: Math.round(desiredNumber),
                }],
            };
        } else {
            return { success: false };
        }
    },
};
```

---

## step 6 — helper functions

put utility functions outside the class at the bottom of the file. keep them focused, validate input early, and return consistent types.

```javascript
// parse a binary string like "1001", "0b1001", or "-1001" into a decimal integer
function parseBinary(str) {
    if (!str || typeof str !== "string") return NaN;

    let trimmed = str.trim();
    let isNegative = false;

    if (trimmed.startsWith("-")) {
        isNegative = true;
        trimmed = trimmed.substring(1).trim();
    }

    // handle optional 0b prefix
    if (trimmed.startsWith("0b") || trimmed.startsWith("0B")) {
        trimmed = trimmed.substring(2);
    }

    if (!/^[01]+$/.test(trimmed)) return NaN;

    let value = parseInt(trimmed, 2);
    return isNegative ? -value : value;
}

// parse a hex string like "FF", "0xFF", or "-1A" into a decimal integer
function parseHexadecimal(str) {
    if (!str || typeof str !== "string") return NaN;

    let trimmed = str.trim();
    let isNegative = false;

    if (trimmed.startsWith("-")) {
        isNegative = true;
        trimmed = trimmed.substring(1).trim();
    }

    // handle optional 0x prefix
    if (trimmed.startsWith("0x") || trimmed.startsWith("0X")) {
        trimmed = trimmed.substring(2);
    }

    if (!/^[0-9a-fA-F]+$/.test(trimmed)) return NaN;

    let value = parseInt(trimmed, 16);
    return isNegative ? -value : value;
}
```

---

## step 7 — building and testing

DoenetML uses a monorepo with a build chain. after changing a component file, you need to rebuild:

```bash
cd DoenetML
npm run build --workspace packages/doenetml-worker-javascript
npm run build --workspace packages/doenetml-worker
npm run build --workspace packages/doenetml
```

then start the test viewer:

```bash
cd packages/test-viewer
npm run dev
```

edit the test file at `packages/test-viewer/src/test/testCode.doenet` and open `http://localhost:8012/` to see results.

### example test file

```xml
<section>
  <title>integer representation tests</title>

  <!-- basic binary -->
  <integer name="a" representation="binary">1001</integer>
  <p>display: $a</p>              <!-- should show 1001 -->
  <p>value: $a.value</p>         <!-- should show 9 -->
  <p>decimal: $a.decimal</p>     <!-- should show 9 -->
  <p>hex: $a.hexadecimal</p>     <!-- should show 9 -->

  <!-- negative binary -->
  <integer name="b" representation="binary">-1001</integer>
  <p>negative value: $b.value</p> <!-- should show -9 -->

  <!-- hex -->
  <integer name="c" representation="hexadecimal">FF</integer>
  <p>hex value: $c.value</p>     <!-- should show 255 -->

  <!-- zero -->
  <integer name="d" representation="binary">0</integer>
  <p>zero: $d.binary</p>         <!-- should show 0 -->

  <!-- in an answer block -->
  <problem>
    <p>what is binary 1101 in decimal?</p>
    <answer>
      <award><integer representation="binary">1101</integer></award>
    </answer>
  </problem>
</section>
```

---

## return value formats

quick reference for what to return from definition functions:

```javascript
// set a computed value
return { setValue: { myVariable: 42 } };

// use the essential/default value when there are no children
return { useEssentialOrDefaultValue: { myVariable: { defaultValue: NaN } } };
```

from `inverseDefinition`:

```javascript
// update a dependency
return {
    success: true,
    instructions: [{
        setDependency: "stringChildren",
        desiredValue: "42",
        childIndex: 0,
        variableIndex: 0,
    }],
};

// set an essential value directly
return {
    success: true,
    instructions: [{
        setEssentialValue: "value",
        value: 42,
    }],
};

// signal failure (input was invalid)
return { success: false };
```

---

## how it all fits together — the integer example

here's the full data flow when a user writes `<integer representation="binary">1001</integer>`:

1. **attribute is read** — `representation = "binary"`
2. **string child is parsed** — `"1001"` is passed to `parseBinary()` → returns `9`
3. **value is stored** — internally stored as `9` (always decimal)
4. **representation variables computed** — `binary = "1001"`, `decimal = "9"`, `hexadecimal = "9"`
5. **text is determined** — since `representation = "binary"`, `text = "1001"`
6. **renderer displays** — `"1001"`

users can then access:

```xml
<integer name="a" representation="binary">1001</integer>
$a          <!-- 1001 (display) -->
$a.value    <!-- 9 -->
$a.binary   <!-- 1001 -->
$a.decimal  <!-- 9 -->
$a.hexadecimal  <!-- 9 -->
```

---

## common mistakes

**1. accessing string children like objects**
```javascript
// wrong — crashes because strings don't have .stateValues
let val = dependencyValues.stringChildren[0].stateValues.value;

// correct
let val = dependencyValues.stringChildren[0];
```

**2. forgetting to rebuild**

the test viewer uses the built output, not the source files. every change needs a rebuild or you'll be staring at old code wondering why nothing works.

**3. forgetting `forRenderer: true` on `text`**

if you override `text` without `forRenderer: true`, the renderer won't pick it up and the display won't update.

**4. using `||` as a numeric fallback**

```javascript
// bug — 0 is falsy, so this would replace 0 with the default
return { setValue: { value: parsedValue || NaN } };

// correct
return { setValue: { value: Number.isNaN(parsedValue) ? NaN : parsedValue } };
```

**5. circular dependencies**

if variable A depends on B and B depends on A, things break silently. sketch out your dependency graph before writing code.

**6. replacing parent attributes instead of extending them**

```javascript
// wrong — wipes out all parent attributes
static createAttributesObject() {
    return { myAttr: { ... } };
}

// correct
static createAttributesObject() {
    let attributes = super.createAttributesObject();
    attributes.myAttr = { ... };
    return attributes;
}
```

---

## component update checklist

use this when adding a new feature:

- [ ] attribute added to `createAttributesObject()` with correct type and default
- [ ] attribute is `public: true` if other components need to read it
- [ ] state variable dependencies all declared in `returnDependencies`
- [ ] definition handles edge cases (NaN, null, negative, zero, etc.)
- [ ] `inverseDefinition` added if two-way binding is needed
- [ ] `text` state variable updated if display changes
- [ ] `forRenderer: true` set on `text`
- [ ] helper functions placed outside the class
- [ ] rebuilt all packages before testing
- [ ] tested basic cases, edge cases, and invalid input

---

## additional resources

- **existing components:** `packages/doenetml-worker-javascript/src/components/` — look at `Number.js` and `Text.js` for common patterns
- **utilities:** `packages/doenetml-worker-javascript/src/utils/`
- **test viewer:** `packages/test-viewer/` for manual testing
