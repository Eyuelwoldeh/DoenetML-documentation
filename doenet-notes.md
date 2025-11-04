# DoenetML Components Explained for Python Developers

## Introduction
This guide explains how DoenetML components work, using comparisons to Python to help you understand JavaScript concepts. We'll look at three components: Text, Number, and Point.

---

## JavaScript vs Python Quick Reference

Before we dive in, here are some key JavaScript concepts compared to Python:

### Classes
**Python:**
```python
class MyClass:
    def __init__(self, args):
        self.property = args
    
    @staticmethod
    def static_method():
        pass
```

**JavaScript:**
```javascript
class MyClass {
    constructor(args) {
        this.property = args;
    }
    
    static staticMethod() {
        // code
    }
}
```
- JavaScript uses `constructor` instead of `__init__`
- JavaScript uses `static` keyword before methods (like `@staticmethod` in Python)
- Curly braces `{}` instead of indentation
- Semicolons `;` at end of statements (optional but common)

### Objects/Dictionaries
**Python:**
```python
my_dict = {
    "key": "value",
    "number": 42
}
print(my_dict["key"])  # Access with brackets
```

**JavaScript:**
```javascript
let myObject = {
    key: "value",
    number: 42
};
console.log(myObject.key);  // Can use dot notation
console.log(myObject["key"]);  // Or bracket notation
```
- JavaScript objects are like Python dicts
- Can access with `object.property` or `object["property"]`

### Arrow Functions
**Python:**
```python
my_func = lambda x: x + 1  # Lambda function
```

**JavaScript:**
```javascript
const myFunc = (x) => x + 1;  // Arrow function
// Or more verbose:
const myFunc = (x) => {
    return x + 1;
};
```
- `=>` is JavaScript's way of creating functions
- Similar to Python's lambda but more powerful

### Importing
**Python:**
```python
from my_module import my_function
import entire_module
```

**JavaScript:**
```javascript
import { myFunction } from "./my-module";  // Named import
import entireModule from "./entire-module";  // Default import
```

---

## Component Structure Overview

Think of a DoenetML component like a **class in Python** that represents a UI element. Each component:

1. **Extends a base class** (like inheritance in Python)
2. **Has static properties** (class attributes in Python)
3. **Has instance properties** (instance attributes in Python)
4. **Defines state variables** (reactive data that updates the UI when changed)
5. **Has action methods** (methods that respond to user interactions)

---

## 1. TEXT COMPONENT (Text.js)

### What It Does
The Text component displays text content. It can:
- Show plain text
- Be positioned on a graph
- Convert to mathematical expressions
- Convert to numbers
- Be interactive (clickable, draggable)

### Basic Structure

```javascript
export default class Text extends InlineComponent {
```
**Translation to Python:**
```python
class Text(InlineComponent):  # Inherits from InlineComponent
```

- `export default` means "this is what you get when you import this file"
- Like Python's main class in a module

---

### The Constructor

```javascript
constructor(args) {
    super(args);  // Call parent class constructor
    
    Object.assign(this.actions, {
        moveText: this.moveText.bind(this),
        textClicked: this.textClicked.bind(this),
        textFocused: this.textFocused.bind(this),
    });
}
```

**What's happening here:**

1. **`super(args)`** - Like Python's `super().__init__(args)`, calls parent constructor

2. **`Object.assign()`** - Merges objects together
   - Python equivalent: `dict1.update(dict2)` or `{**dict1, **dict2}`

3. **`.bind(this)`** - This is JavaScript-specific!
   - In Python, methods automatically have access to `self`
   - In JavaScript, you must explicitly bind `this` to keep the correct context
   - Think of it as ensuring the method knows which object it belongs to

**Python equivalent concept:**
```python
def __init__(self, args):
    super().__init__(args)
    # In Python, we don't need to bind 'self' - it's automatic
    self.actions = {
        'moveText': self.moveText,
        'textClicked': self.textClicked,
        'textFocused': self.textFocused
    }
```

---

### Static Properties (Class Attributes)

```javascript
static componentType = "text";
static includeBlankStringChildren = true;
static variableForImplicitProp = "value";
```

**Python equivalent:**
```python
class Text(InlineComponent):
    componentType = "text"  # Class attribute
    includeBlankStringChildren = True
    variableForImplicitProp = "value"
```

**What these mean:**

- **`componentType`**: A unique identifier for this component type (like "text", "number", "point")

- **`includeBlankStringChildren`**: When parsing the component's content, should we keep blank spaces?
  - Example: `<text>hello   world</text>` - keep those spaces? Yes!

- **`variableForImplicitProp`**: When someone references this component without specifying what part, which property should they get?
  - Example: `<copy source="myText" />` implicitly gets the "value" property

---

### The createAttributesObject Method

This is a **static method** (class method) that defines what attributes (properties) the component can have.

```javascript
static createAttributesObject() {
    let attributes = super.createAttributesObject();
    
    attributes.draggable = {
        createComponentOfType: "boolean",
        createStateVariable: "draggable",
        defaultValue: true,
        public: true,
        forRenderer: true,
    };
    
    return attributes;
}
```

**Breaking it down:**

1. **`let attributes = super.createAttributesObject()`**
   - Get parent class attributes first
   - Like calling `super().some_method()` in Python

2. **`attributes.draggable = { ... }`**
   - Adding a new attribute called "draggable"
   - The value is an object (like a Python dict) with configuration

3. **Configuration object explained:**
   ```javascript
   {
       createComponentOfType: "boolean",  // This attribute's value is true/false
       createStateVariable: "draggable",  // Creates a state variable with this name
       defaultValue: true,                // If not specified, use true
       public: true,                      // Can be accessed from outside
       forRenderer: true,                 // The UI renderer needs this info
   }
   ```

**Python equivalent concept:**
```python
@classmethod
def create_attributes_object(cls):
    attributes = super().create_attributes_object()
    
    attributes['draggable'] = {
        'createComponentOfType': 'boolean',
        'createStateVariable': 'draggable',
        'defaultValue': True,
        'public': True,
        'forRenderer': True
    }
    
    return attributes
```

**What attributes are defined:**

1. **draggable** - Can the text be dragged on a graph?
2. **layer** - Z-index (which layer is it on? Higher = on top)
3. **isLatex** - Should we parse the content as LaTeX math notation?

---

### Child Groups

```javascript
static returnChildGroups() {
    return [
        {
            group: "textLike",
            componentTypes: ["string", "text", "_singleCharacterInline", "_inlineRenderInlineChildren"],
        },
    ];
}
```

**What this means:**

Components can have children (nested elements). This defines what types of children are allowed.

**Example in DoenetML:**
```xml
<text>
    Hello <text>world</text>!  <!-- nested text component -->
</text>
```

**The return value is a list of group definitions:**
- **group name**: "textLike"
- **componentTypes**: An array (list) of allowed component types
  - "string" - plain text
  - "text" - nested text components
  - "_singleCharacterInline" - single character components
  - "_inlineRenderInlineChildren" - other inline components

---

### State Variables - The Heart of Reactivity

**Big Concept:** State variables are like **reactive properties** in Python frameworks. When a state variable changes, the UI automatically updates.

Think of it like this in Python:
```python
# Pseudo-code (not real Python)
class ReactiveText:
    @reactive_property
    def value(self):
        # When this changes, UI re-renders automatically
        return self._compute_value()
```

#### Structure of a State Variable Definition

Every state variable has this structure:

```javascript
stateVariableDefinitions.variableName = {
    // Configuration options
    public: true,           // Can be accessed from outside?
    shadowingInstructions: {},  // How to copy this?
    hasEssential: true,     // Can store a value independently?
    defaultValue: "",       // Default if nothing specified
    
    // Dependencies - what data do we need?
    returnDependencies: () => ({ ... }),
    
    // Definition - how to compute the value
    definition: function({ dependencyValues }) { ... },
    
    // Inverse - how to update when this changes
    inverseDefinition: function({ desiredStateVariableValues }) { ... }
};
```

---

#### The "value" State Variable

This is the most important state variable for Text - it holds the actual text content.

```javascript
stateVariableDefinitions.value = {
    public: true,  // Other components can read this
    shadowingInstructions: {
        createComponentOfType: this.componentType,
    },
    hasEssential: true,  // Can have its own stored value
    returnDependencies: () => ({
        textLikeChildren: {
            dependencyType: "child",
            childGroups: ["textLike"],
            variableNames: ["text"],
        },
    }),
    defaultValue: "",
    set: (x) => (x === null ? "" : String(x)),
    definition: function ({ dependencyValues }) {
        if (dependencyValues.textLikeChildren.length === 0) {
            return {
                useEssentialOrDefaultValue: { value: true },
            };
        }
        let value = textFromChildren(dependencyValues.textLikeChildren);
        return { setValue: { value } };
    },
    // ... more code
};
```

**Let's break this down step by step:**

1. **returnDependencies**
   ```javascript
   returnDependencies: () => ({
       textLikeChildren: {
           dependencyType: "child",
           childGroups: ["textLike"],
           variableNames: ["text"],
       },
   })
   ```
   
   **What this means:** "To compute my value, I need the text from all my textLike children"
   
   **Python equivalent concept:**
   ```python
   def get_dependencies(self):
       return {
           'textLikeChildren': {
               'dependencyType': 'child',
               'childGroups': ['textLike'],
               'variableNames': ['text']
           }
       }
   ```

2. **definition function**
   ```javascript
   definition: function ({ dependencyValues }) {
       if (dependencyValues.textLikeChildren.length === 0) {
           return { useEssentialOrDefaultValue: { value: true } };
       }
       let value = textFromChildren(dependencyValues.textLikeChildren);
       return { setValue: { value } };
   }
   ```
   
   **What this does:**
   - If no children exist, use the essential/default value (empty string)
   - If children exist, extract text from them using `textFromChildren` helper
   - Return an object telling the system to set this value
   
   **Python equivalent:**
   ```python
   def compute_value(self, dependency_values):
       if len(dependency_values['textLikeChildren']) == 0:
           return {'useEssentialOrDefaultValue': {'value': True}}
       
       value = text_from_children(dependency_values['textLikeChildren'])
       return {'setValue': {'value': value}}
   ```

3. **inverseDefinition function**
   
   This is called when someone **changes** the value and we need to update the source.
   
   ```javascript
   inverseDefinition: function ({
       desiredStateVariableValues,
       dependencyValues,
   }) {
       let numChildren = dependencyValues.textLikeChildren.length;
       
       if (numChildren === 1) {
           return {
               success: true,
               instructions: [{
                   setDependency: "textLikeChildren",
                   desiredValue: desiredStateVariableValues.value,
                   childIndex: 0,
                   variableIndex: 0,
               }],
           };
       }
       // ... more logic for multiple children
   }
   ```
   
   **What this means:**
   - Someone wants to change the text to a new value
   - If we have one child, tell that child to update to the new value
   - If we have multiple children or no children, different logic applies
   
   **Think of it like two-way data binding:**
   - `definition`: Child changes → Update my value
   - `inverseDefinition`: My value changes → Update child

---

#### Other Important State Variables in Text

1. **text** - Just mirrors the value (for consistency)
   ```javascript
   stateVariableDefinitions.text = {
       public: true,
       returnDependencies: () => ({
           value: {
               dependencyType: "stateVariable",
               variableName: "value",
           },
       }),
       definition: ({ dependencyValues }) => ({
           setValue: { text: dependencyValues.value },
       }),
   };
   ```
   **Simple!** Just returns whatever `value` is.

2. **math** - Converts text to a mathematical expression
   ```javascript
   stateVariableDefinitions.math = {
       returnDependencies: () => ({
           value: { dependencyType: "stateVariable", variableName: "value" },
           isLatex: { dependencyType: "stateVariable", variableName: "isLatex" },
       }),
       definition({ dependencyValues }) {
           let parser = dependencyValues.isLatex
               ? latexToMathFactory()  // LaTeX parser
               : textToMathFactory();   // Plain text parser
           
           let expression;
           try {
               expression = parser(dependencyValues.value);
           } catch (e) {
               expression = me.fromAst("\uFF3F");  // Error symbol
           }
           return { setValue: { math: expression } };
       },
   };
   ```
   
   **What this does:**
   - Takes the text value
   - Parses it as either LaTeX or plain text (depending on `isLatex`)
   - Returns a math expression object
   - If parsing fails, returns an error symbol

3. **number** - Converts text to a number
   ```javascript
   stateVariableDefinitions.number = {
       returnDependencies: () => ({
           math: { dependencyType: "stateVariable", variableName: "math" },
       }),
       definition({ dependencyValues }) {
           return {
               setValue: {
                   number: dependencyValues.math.evaluate_to_constant(),
               },
           };
       },
   };
   ```
   
   **Chain of dependencies:**
   - `value` (text) → `math` (expression) → `number` (numeric value)
   - Each builds on the previous one

---

### Action Methods

These are instance methods that respond to user interactions.

```javascript
async moveText({ x, y, z, transient, actionId, sourceInformation = {}, skipRendererUpdate = false }) {
    return await moveGraphicalObjectWithAnchorAction({
        x, y, z, transient, actionId, sourceInformation, skipRendererUpdate,
        componentIdx: this.componentIdx,
        componentType: this.componentType,
        coreFunctions: this.coreFunctions,
    });
}
```

**What's happening:**
- `async` - This is an asynchronous function (like Python's `async def`)
- Parameters use **destructuring** - like unpacking a dict in Python
- Calls a utility function to handle the actual move
- Passes along position (x, y, z) and other metadata

**Python equivalent:**
```python
async def move_text(self, x, y, z, transient, action_id, source_information=None, skip_renderer_update=False):
    if source_information is None:
        source_information = {}
    
    return await move_graphical_object_with_anchor_action(
        x=x, y=y, z=z,
        transient=transient,
        action_id=action_id,
        source_information=source_information,
        skip_renderer_update=skip_renderer_update,
        component_idx=self.component_idx,
        component_type=self.component_type,
        core_functions=self.core_functions
    )
```

---

## 2. NUMBER COMPONENT (Number.js)

### What It Does
The Number component displays numerical values. It can:
- Parse numbers from various sources (strings, math, other numbers)
- Round and format numbers for display
- Convert between number/math/text representations
- Handle boolean conversion (true=1, false=0)
- Be graphical (positioned on a graph)

### Key Differences from Text

While Text is simple (just stores text), Number is more complex because it:
1. **Parses multiple input formats** (strings, math expressions, booleans)
2. **Performs arithmetic** and evaluation
3. **Handles rounding** for display
4. **Validates numeric values** (checking for NaN, Infinity, etc.)

---

### Important Attributes

```javascript
attributes.convertBoolean = {
    createPrimitiveOfType: "boolean",
    createStateVariable: "convertBoolean",
    defaultValue: false,
};
```

**What this means:**
- If `true`: Treat text children as boolean expressions
  - "true and false" → 0 (false)
  - "true or true" → 1 (true)
- If `false`: Treat text children as numbers to parse

```javascript
attributes.valueOnNaN = {
    createPrimitiveOfType: "number",
    createStateVariable: "valueOnNaN",
    defaultValue: NaN,
};
```

**What this means:**
- What value to use when parsing fails (like dividing by zero)
- Default is `NaN` (Not a Number)
- Can be customized: `<number valueOnNaN="0">invalid</number>` → displays 0

---

### Sugar Instructions - Automatic Content Wrapping

**What is "sugar"?** It's automatic transformation of content for convenience.

```javascript
static returnSugarInstructions() {
    let sugarInstructions = super.returnSugarInstructions();
    
    sugarInstructions.push({
        childrenRegex: /..+/,  // Match 2 or more children
        replacementFunction: ({ matchedChildren, componentAttributes, nComponents }) => ({
            success: !componentAttributes.convertBoolean,
            newChildren: [{
                type: "serialized",
                componentType: "math",
                componentIdx: nComponents++,
                children: matchedChildren,
                attributes: {},
                doenetAttributes: {},
                state: {},
            }],
            nComponents,
        }),
    });
    
    return sugarInstructions;
}
```

**What this does:**
- If you have multiple children (like `<number>2 + 3</number>`)
- AND `convertBoolean` is false
- Automatically wrap them in a `<math>` component
- This allows: `<number>2 + 3</number>` to work like `<number><math>2 + 3</math></number>`

**Why?** So you can write arithmetic directly without wrapping in `<math>` tags!

---

### Child Groups in Number

```javascript
static returnChildGroups() {
    return [
        {
            group: "strings",
            componentTypes: ["string"],
        },
        {
            group: "numbers",
            componentTypes: ["number"],
        },
        {
            group: "maths",
            componentTypes: ["math"],
        },
        {
            group: "texts",
            componentTypes: ["text"],
        },
        {
            group: "booleans",
            componentTypes: ["boolean"],
        },
    ];
}
```

**Number accepts many child types:**
- **strings**: Plain text like "42"
- **numbers**: Other number components
- **maths**: Math expressions like `2 + 3`
- **texts**: Text components (can be converted to numbers)
- **booleans**: True/false (can be converted to 1/0)

---

### The "value" State Variable in Number

This is WAY more complex than Text's value because it handles so many input types!

```javascript
stateVariableDefinitions.value = {
    public: true,
    hasEssential: true,
    defaultValue: NaN,
    stateVariablesDeterminingDependencies: ["singleNumberOrStringChild"],
    // ... dependencies change based on what children exist!
}
```

**Key concept:** `stateVariablesDeterminingDependencies`
- The dependencies themselves are **dynamic**!
- Depends on whether we have a single simple child or multiple complex children
- Like having different `@property` decorators based on runtime conditions in Python

#### Case 1: Single Number or String Child

```javascript
returnDependencies({ stateValues }) {
    if (stateValues.singleNumberOrStringChild) {
        return {
            numberChild: {
                dependencyType: "child",
                childGroups: ["numbers"],
                variableNames: ["value"],
            },
            stringChild: {
                dependencyType: "child",
                childGroups: ["strings"],
                variableNames: ["value"],
            },
            // ... etc
        };
    }
    // ... different dependencies for complex cases
}
```

**Simple case logic:**
```javascript
definition({ dependencyValues }) {
    if (dependencyValues.numberChild.length === 1) {
        // Just copy the number child's value
        let number = dependencyValues.numberChild[0].stateValues.value;
        return { setValue: { value: number } };
    }
    
    if (dependencyValues.stringChild.length === 1) {
        // Try to parse the string as a number
        let number = Number(dependencyValues.stringChild[0]);
        if (Number.isNaN(number)) {
            // If that fails, try parsing as math expression
            number = me.fromAst(textToAst.convert(dependencyValues.stringChild[0]))
                .evaluate_to_constant();
        }
        return { setValue: { value: number } };
    }
    
    // No children - use essential/default value
    return { useEssentialOrDefaultValue: { value: true } };
}
```

**Python equivalent concept:**
```python
def compute_value(self, dependency_values):
    if len(dependency_values['numberChild']) == 1:
        # Simple case: just copy the number
        return {'setValue': {'value': dependency_values['numberChild'][0].state_values.value}}
    
    if len(dependency_values['stringChild']) == 1:
        # Try parsing string
        try:
            number = float(dependency_values['stringChild'][0])
        except ValueError:
            # Try math parsing
            try:
                number = me.from_ast(text_to_ast.convert(dependency_values['stringChild'][0])).evaluate_to_constant()
            except:
                number = self.value_on_nan
        return {'setValue': {'value': number}}
    
    return {'useEssentialOrDefaultValue': {'value': True}}
```

#### Case 2: Math Expression with Multiple Children

When you have `<number>2 + 3 * x</number>`:

```javascript
definition({ dependencyValues }) {
    // The parsedExpression is a tree structure
    // Example: ["*", 2, 3] represents "2 * 3"
    
    // Replace all variable references with actual values
    let replaceMath = function(tree) {
        if (typeof tree === "string") {
            // This is a variable reference - look it up in children
            let child = dependencyValues.mathChildrenByCode[tree];
            if (child !== undefined) {
                return child.stateValues.value.tree;
            }
            return tree;
        }
        if (!Array.isArray(tree)) {
            return tree;
        }
        // Recursively replace in the tree
        return [tree[0], ...tree.slice(1).map(replaceMath)];
    };
    
    try {
        let number = me.fromAst(
            replaceMath(dependencyValues.parsedExpression.tree)
        ).evaluate_to_constant();
        
        return { setValue: { value: number } };
    } catch (e) {
        return { setValue: { value: dependencyValues.valueOnNaN } };
    }
}
```

**What this does:**
1.  Takes the expression tree (like `["+", 2, 3]`)
2. Substitutes any variables with their actual values
3. Evaluates the whole thing to a number
4. If anything fails, uses `valueOnNaN`

---

### Rounding and Display

Number has sophisticated rounding for display:

```javascript
stateVariableDefinitions.valueForDisplay = {
    returnDependencies: () => ({
        value: { dependencyType: "stateVariable", variableName: "value" },
        displayDigits: { dependencyType: "stateVariable", variableName: "displayDigits" },
        displayDecimals: { dependencyType: "stateVariable", variableName: "displayDecimals" },
        displaySmallAsZero: { dependencyType: "stateVariable", variableName: "displaySmallAsZero" },
    }),
    definition: function ({ dependencyValues }) {
        let rounded = roundForDisplay({
            value: numberToMathExpression(dependencyValues.value),
            dependencyValues,
        }).evaluate_to_constant();
        
        return { setValue: { valueForDisplay: rounded } };
    },
};
```

**Example:**
- `value = 3.14159265359`
- `displayDigits = 3`
- `valueForDisplay = 3.14` (rounded to 3 significant digits)

Then the `text` state variable converts this to a string for display:

```javascript
stateVariableDefinitions.text = {
    definition: function ({ dependencyValues }) {
        let params = {};
        if (dependencyValues.padZeros) {
            // Add trailing zeros if needed
            if (Number.isFinite(dependencyValues.displayDecimals)) {
                params.padToDecimals = dependencyValues.displayDecimals;
            }
        }
        return {
            setValue: {
                text: numberToMathExpression(dependencyValues.valueForDisplay).toString(params),
            },
        };
    },
};
```

---

## 3. POINT COMPONENT (Point.js)

### What It Does
The Point component represents a point in 2D or 3D space. It can:
- Have coordinates (x, y) or (x, y, z)
- Be dragged around on a graph
- Have constraints applied (stay on a line, curve, etc.)
- Display labels at different positions
- Work with vectors interchangeably

### Key Concepts

**Points are more complex than numbers because:**
1. They have **multiple coordinates** (x, y, z) instead of a single value
2. They support **constraints** (restricting where the point can be)
3. They have **array state variables** (one entry per coordinate)
4. They handle **sticky groups** (points that move together)

---

### Attributes

```javascript
attributes.x = {
    createComponentOfType: "math",
};
attributes.y = {
    createComponentOfType: "math",
};
attributes.z = {
    createComponentOfType: "math",
};
attributes.xs = {
    createComponentOfType: "mathList",  // List of all coordinates
};
attributes.coords = {
    createComponentOfType: "coords",  // Special coords type
};
```

**Multiple ways to specify a point:**
- `<point x="2" y="3" />` - Individual x, y attributes
- `<point xs="2 3" />` - List of coordinates
- `<point coords="(2, 3)" />` - Coords notation
- `<point>(2, 3)</point>` - Sugar syntax (automatically parsed)

---

### Array State Variables

**Big new concept:** Array state variables have **multiple entries**, like a list in Python.

```javascript
stateVariableDefinitions.xs = {
    isArray: true,
    entryPrefixes: ["x"],  // Individual entries: x1, x2, x3
    returnArraySizeDependencies: () => ({
        numDimensions: {
            dependencyType: "stateVariable",
            variableName: "numDimensions",
        },
    }),
    returnArraySize({ dependencyValues }) {
        return [dependencyValues.numDimensions];  // Size based on dimensions
    },
    // ... more code
};
```

**What is this?**
- `xs` is an array state variable containing all coordinates
- Individual entries: `x1`, `x2`, `x3` (or using aliases: `x`, `y`, `z`)
- Array size determined by `numDimensions` (2D point = 2 entries, 3D = 3)

**Python equivalent concept:**
```python
class Point:
    @property
    def xs(self):
        # Returns a list: [x1, x2, x3]
        return [self.x1, self.x2, self.x3][:self.num_dimensions]
    
    # Aliases for convenience
    @property
    def x(self):
        return self.xs[0]  # Same as x1
    
    @property
    def y(self):
        return self.xs[1]  # Same as x2
```

---

### Array Definition By Key

Array state variables are defined **per entry**, not all at once:

```javascript
returnArrayDependenciesByKey({ arrayKeys }) {
    let dependenciesByKey = {};
    for (let arrayKey of arrayKeys) {
        let varEnding = Number(arrayKey) + 1;
        dependenciesByKey[arrayKey] = {
            unconstrainedX: {
                dependencyType: "stateVariable",
                variableName: `unconstrainedX${varEnding}`,
            },
            constraintsChild: {
                dependencyType: "child",
                childGroups: ["constraints"],
                variableNames: [`constraintResult${varEnding}`],
            },
        };
    }
    return { dependenciesByKey };
}
```

**What this means:**
- `arrayKeys` = `["0", "1", "2"]` for a 3D point
- For each key, we define separate dependencies
- Key "0" → depends on `unconstrainedX1` and `constraintResult1`
- Key "1" → depends on `unconstrainedX2` and `constraintResult2`
- etc.