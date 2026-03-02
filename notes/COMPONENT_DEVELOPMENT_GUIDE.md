# DoenetML Component Development Guide

## Table of Contents
1. [Introduction](#introduction)
2. [Component Architecture Overview](#component-architecture-overview)
3. [Adding New Attributes](#adding-new-attributes)
4. [Working with State Variables](#working-with-state-variables)
5. [Modifying Existing Components](#modifying-existing-components)
6. [Helper Functions and Utilities](#helper-functions-and-utilities)
7. [Testing Your Changes](#testing-your-changes)
8. [Common Patterns](#common-patterns)
9. [Case Study: Integer Component Binary/Hex Support](#case-study-integer-component-binaryhex-support)

---

## Introduction

This guide walks you through the process of adding new features to existing DoenetML components or creating new components from scratch. We'll use the Integer component's binary/hexadecimal representation feature as a real-world example.

### Prerequisites
- Basic JavaScript/TypeScript knowledge
- Understanding of reactive state management
- Familiarity with DoenetML syntax

---

## Component Architecture Overview

DoenetML components are JavaScript classes that extend base component classes. Each component has:

### 1. **Component Metadata**
```javascript
export default class MyComponent extends BaseComponent {
    static componentType = "myComponent";  // Tag name in DoenetML
    static rendererType = "text";          // How it's rendered
}
```

### 2. **Attributes** (via `createAttributesObject()`)
Attributes are properties set in the DoenetML tag:
```xml
<integer representation="binary">1001</integer>
         ^^^^^^^^^^^^^^^^^^^^^^^^
         This is an attribute
```

### 3. **State Variables** (via `returnStateVariableDefinitions()`)
State variables hold the component's data and are reactive:
- `value` - The component's primary value
- `text` - How it displays
- Custom state variables for specific functionality

### 4. **Dependencies**
State variables can depend on:
- Other state variables
- Attributes
- Child components
- Parent components

---

## Adding New Attributes

Attributes allow users to configure component behavior. Here's how to add them:

### Step 1: Override `createAttributesObject()`

```javascript
static createAttributesObject() {
    // Get parent class attributes first
    let attributes = super.createAttributesObject();

    // Add your new attribute
    attributes.myNewAttribute = {
        createPrimitiveOfType: "string",      // Type: "string", "number", "boolean", etc.
        createStateVariable: "myNewAttribute", // Creates a state variable with this name
        defaultValue: "default",               // Default value if not specified
        public: true,                          // Allow external access ($component.myNewAttribute)
        forRenderer: true,                     // Pass to renderer for display
    };

    return attributes;
}
```

### Step 2: Use the Attribute in State Variables

```javascript
returnDependencies: () => ({
    myNewAttribute: {
        dependencyType: "stateVariable",
        variableName: "myNewAttribute",
    },
}),
```

### Example: Adding `representation` Attribute to Integer

```javascript
static createAttributesObject() {
    let attributes = super.createAttributesObject();

    attributes.representation = {
        createPrimitiveOfType: "string",
        createStateVariable: "representation",
        defaultValue: "decimal",
        public: true,
        forRenderer: true,
    };

    return attributes;
}
```

**Result:** Users can now write:
```xml
<integer representation="binary">1001</integer>
```

---

## Working with State Variables

State variables are the heart of DoenetML components. They hold data and define how components behave.

### Anatomy of a State Variable

```javascript
stateVariableDefinitions.myVariable = {
    // 1. PUBLIC ACCESS
    public: true,  // Allow $component.myVariable access
    
    // 2. SHADOWING (copying behavior)
    shadowingInstructions: {
        createComponentOfType: "text",  // When copied, create this type
    },
    
    // 3. RENDERER
    forRenderer: true,  // Pass to renderer
    
    // 4. DEPENDENCIES
    returnDependencies: () => ({
        dependencyName: {
            dependencyType: "stateVariable",
            variableName: "otherVariable",
        },
    }),
    
    // 5. DEFINITION (compute the value)
    definition({ dependencyValues }) {
        // Your logic here
        let result = computeValue(dependencyValues.dependencyName);
        
        return {
            setValue: {
                myVariable: result,
            },
        };
    },
    
    // 6. INVERSE DEFINITION (optional - for two-way binding)
    inverseDefinition({ desiredStateVariableValues }) {
        return {
            success: true,
            instructions: [
                {
                    setDependency: "dependencyName",
                    desiredValue: desiredStateVariableValues.myVariable,
                },
            ],
        };
    },
};
```

### Types of State Variables

#### 1. **Simple Computed Variables**
```javascript
stateVariableDefinitions.doubled = {
    public: true,
    returnDependencies: () => ({
        value: {
            dependencyType: "stateVariable",
            variableName: "value",
        },
    }),
    definition({ dependencyValues }) {
        return {
            setValue: {
                doubled: dependencyValues.value * 2,
            },
        };
    },
};
```

#### 2. **Variables with Conditional Logic**
```javascript
definition({ dependencyValues }) {
    let result;
    
    if (dependencyValues.representation === "binary") {
        result = convertToBinary(dependencyValues.value);
    } else if (dependencyValues.representation === "hexadecimal") {
        result = convertToHex(dependencyValues.value);
    } else {
        result = String(dependencyValues.value);
    }
    
    return {
        setValue: { text: result },
    };
}
```

#### 3. **Variables with Parsing Logic**
```javascript
definition({ dependencyValues }) {
    let rawInput = dependencyValues.rawValue;
    let parsed = NaN;
    
    if (typeof rawInput === "string") {
        parsed = parseMyFormat(rawInput);
    } else if (typeof rawInput === "number") {
        parsed = rawInput;
    }
    
    return {
        setValue: { value: parsed },
    };
}
```

---

## Modifying Existing Components

When adding features to existing components, follow these patterns:

### Pattern 1: Extending Attributes

✅ **DO:**
```javascript
static createAttributesObject() {
    let attributes = super.createAttributesObject();
    
    // Add new attributes
    attributes.newAttribute = { /* config */ };
    
    return attributes;
}
```

❌ **DON'T:**
```javascript
// Don't replace parent attributes entirely
static createAttributesObject() {
    return {
        newAttribute: { /* config */ },
    };
}
```

### Pattern 2: Modifying Existing State Variables

When you need to change how an existing state variable works:

```javascript
static returnStateVariableDefinitions() {
    let stateVariableDefinitions = super.returnStateVariableDefinitions();
    
    // Option A: Rename the original
    renameStateVariable({
        stateVariableDefinitions,
        oldName: "value",
        newName: "valuePreRound",
    });
    
    // Option B: Override completely
    stateVariableDefinitions.value = {
        // Your new definition
        returnDependencies: () => ({
            valuePreRound: {
                dependencyType: "stateVariable",
                variableName: "valuePreRound",
            },
            representation: {
                dependencyType: "stateVariable",
                variableName: "representation",
            },
        }),
        definition({ dependencyValues }) {
            // New logic that uses the old variable
            let value = dependencyValues.valuePreRound;
            let representation = dependencyValues.representation;
            
            // Process based on representation
            return {
                setValue: { value: processValue(value, representation) },
            };
        },
    };
    
    return stateVariableDefinitions;
}
```

### Pattern 3: Adding New State Variables

```javascript
static returnStateVariableDefinitions() {
    let stateVariableDefinitions = super.returnStateVariableDefinitions();
    
    // Add your new state variables
    stateVariableDefinitions.myNewVariable = {
        public: true,
        shadowingInstructions: {
            createComponentOfType: "text",
        },
        returnDependencies: () => ({
            value: {
                dependencyType: "stateVariable",
                variableName: "value",
            },
        }),
        definition({ dependencyValues }) {
            return {
                setValue: {
                    myNewVariable: transform(dependencyValues.value),
                },
            };
        },
    };
    
    return stateVariableDefinitions;
}
```

---

## Helper Functions and Utilities

Place helper functions outside the class definition for reusability:

### Structure

```javascript
export default class MyComponent extends BaseComponent {
    // Class definition
}

// Helper functions below

/**
 * Parse binary string to integer
 * @param {string} str - Binary string (e.g., "1001", "0b1001", "-1001")
 * @returns {number} - Decimal integer or NaN if invalid
 */
function parseBinary(str) {
    if (!str || typeof str !== "string") {
        return NaN;
    }
    
    let trimmed = str.trim();
    let isNegative = false;
    
    // Handle negative sign
    if (trimmed.startsWith("-")) {
        isNegative = true;
        trimmed = trimmed.substring(1).trim();
    }
    
    // Handle optional prefix
    if (trimmed.startsWith("0b") || trimmed.startsWith("0B")) {
        trimmed = trimmed.substring(2);
    }
    
    // Validate format
    if (!/^[01]+$/.test(trimmed)) {
        return NaN;
    }
    
    // Parse
    let value = parseInt(trimmed, 2);
    return isNegative ? -value : value;
}
```

### Best Practices for Helper Functions

1. **Document parameters and return values**
2. **Handle edge cases** (null, empty string, invalid input)
3. **Validate input** before processing
4. **Return consistent types** (e.g., always return NaN for invalid numbers)
5. **Make them pure** (no side effects)

---

## Testing Your Changes

### Manual Testing in Test Viewer

Create a test file in `packages/test-viewer/src/test/testCode.doenet`:

```xml
<section>
    <title>Testing New Feature</title>
    
    <!-- Test basic functionality -->
    <integer name="a" representation="binary">1001</integer>
    <p>Binary: $a.text (should show 1001)</p>
    <p>Value: $a.value (should show 9)</p>
    <p>Decimal: $a.decimal (should show 9)</p>
    
    <!-- Test negative numbers -->
    <integer name="b" representation="binary">-1001</integer>
    <p>Negative binary: $b.text (should show -1001)</p>
    <p>Negative value: $b.value (should show -9)</p>
    
    <!-- Test with prefixes -->
    <integer name="c" representation="binary">0b1111</integer>
    <p>With prefix: $c.value (should show 15)</p>
    
    <!-- Test hexadecimal -->
    <integer name="d" representation="hexadecimal">FF</integer>
    <p>Hex: $d.text (should show ff)</p>
    <p>Value: $d.value (should show 255)</p>
    
    <!-- Test in answers -->
    <problem>
        <p>What is binary 1101 in decimal?</p>
        <answer>
            <award><integer representation="binary">1101</integer></award>
        </answer>
    </problem>
</section>
```

### What to Test

- ✅ Basic functionality
- ✅ Edge cases (0, negative numbers, large values)
- ✅ Invalid input handling
- ✅ Different attribute values
- ✅ Interaction with other components
- ✅ Copy/reference behavior (`$component.attribute`)
- ✅ Answer validation
- ✅ Renderer display

---

## Common Patterns

### Pattern: Multiple Representation State Variables

When you want users to access the same value in different formats:

```javascript
// Create state variables for each representation
stateVariableDefinitions.decimal = {
    public: true,
    shadowingInstructions: { createComponentOfType: "text" },
    returnDependencies: () => ({
        value: { dependencyType: "stateVariable", variableName: "value" },
    }),
    definition({ dependencyValues }) {
        return {
            setValue: { decimal: String(dependencyValues.value) },
        };
    },
};

stateVariableDefinitions.binary = {
    public: true,
    shadowingInstructions: { createComponentOfType: "text" },
    returnDependencies: () => ({
        value: { dependencyType: "stateVariable", variableName: "value" },
    }),
    definition({ dependencyValues }) {
        let value = dependencyValues.value;
        let binary = value >= 0 
            ? value.toString(2) 
            : "-" + Math.abs(value).toString(2);
        return {
            setValue: { binary: binary },
        };
    },
};

// Now users can access:
// $myInt.value -> 9
// $myInt.decimal -> "9"
// $myInt.binary -> "1001"
```

### Pattern: Conditional Display with `text` State Variable

```javascript
stateVariableDefinitions.text = {
    public: true,
    shadowingInstructions: { createComponentOfType: "text" },
    forRenderer: true,  // Important! Renderer uses this
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
        
        // Return the appropriate representation
        if (dependencyValues.representation === "binary") {
            text = dependencyValues.binary;
        } else if (dependencyValues.representation === "hexadecimal") {
            text = dependencyValues.hexadecimal;
        } else {
            text = dependencyValues.decimal;
        }
        
        return {
            setValue: { text: text },
        };
    },
};
```

### Pattern: Two-Way Binding with Inverse Definitions

When you want changes to propagate back:

```javascript
stateVariableDefinitions.text = {
    // ... dependencies and definition ...
    
    async inverseDefinition({
        desiredStateVariableValues,
        dependencyValues,
    }) {
        let desiredText = desiredStateVariableValues.text;
        let representation = dependencyValues.representation;
        
        // Parse based on current representation
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
                instructions: [
                    {
                        setDependency: "value",
                        desiredValue: desiredNumber,
                    },
                ],
            };
        } else {
            return { success: false };
        }
    },
};
```

### Pattern: Input Validation

```javascript
definition({ dependencyValues }) {
    let rawInput = dependencyValues.rawValue;
    let parsedValue = NaN;
    
    // Type checking
    if (typeof rawInput === "number") {
        parsedValue = rawInput;
    } else if (typeof rawInput === "string") {
        let trimmed = rawInput.trim();
        
        // Format validation
        if (isValidFormat(trimmed)) {
            parsedValue = parseFormat(trimmed);
        }
    }
    
    // Range validation (optional)
    if (Number.isFinite(parsedValue)) {
        if (parsedValue < MIN_VALUE || parsedValue > MAX_VALUE) {
            parsedValue = NaN;
        }
    }
    
    // Return with fallback
    return {
        setValue: {
            value: Number.isNaN(parsedValue) ? NaN : Math.round(parsedValue),
        },
    };
}
```

---

## Case Study: Integer Component Binary/Hex Support

Let's walk through the complete implementation of adding binary/hexadecimal representation to the Integer component.

### Goal

Allow integers to be specified and displayed in binary or hexadecimal:

```xml
<integer name="a" representation="binary">1001</integer>
<!-- Value stored internally as 9 (decimal) -->
<!-- Displays as "1001" -->
<!-- Can access $a.value (9), $a.binary (1001), $a.decimal (9), $a.hexadecimal (9) -->
```

### Step 1: Add the Attribute

```javascript
static createAttributesObject() {
    let attributes = super.createAttributesObject();

    attributes.representation = {
        createPrimitiveOfType: "string",
        createStateVariable: "representation",
        defaultValue: "decimal",
        public: true,
        forRenderer: true,
    };

    Object.assign(attributes, returnAnchorAttributes());
    return attributes;
}
```

**Why:**
- Users can now specify `representation="binary"` or `representation="hexadecimal"`
- Defaults to "decimal" for backward compatibility
- Public so it can be accessed as `$a.representation`

### Step 2: Modify Value Parsing

```javascript
static returnStateVariableDefinitions() {
    let stateVariableDefinitions = super.returnStateVariableDefinitions();

    // Rename original value to valuePreRound
    renameStateVariable({
        stateVariableDefinitions,
        oldName: "value",
        newName: "valuePreRound",
    });

    // Override value to add representation parsing
    stateVariableDefinitions.value = {
        public: true,
        shadowingInstructions: {
            createComponentOfType: "integer",
        },
        returnDependencies: () => ({
            valuePreRound: {
                dependencyType: "stateVariable",
                variableName: "valuePreRound",
            },
            representation: {
                dependencyType: "stateVariable",
                variableName: "representation",
            },
        }),
        definition({ dependencyValues }) {
            let valuePreRound = dependencyValues.valuePreRound;
            let representation = dependencyValues.representation;

            if (typeof valuePreRound === "number") {
                return {
                    setValue: { value: Math.round(valuePreRound) },
                };
            }

            let parsedValue = NaN;
            if (typeof valuePreRound === "string") {
                let trimmed = valuePreRound.trim();

                if (representation === "binary") {
                    parsedValue = parseBinary(trimmed);
                } else if (representation === "hexadecimal") {
                    parsedValue = parseHexadecimal(trimmed);
                } else {
                    parsedValue = Number(trimmed);
                }
            }

            return {
                setValue: {
                    value: Number.isNaN(parsedValue) ? NaN : Math.round(parsedValue),
                },
            };
        },
        // ... inverseDefinition ...
    };

    return stateVariableDefinitions;
}
```

**Why:**
- We rename the original to keep backward compatibility
- We check representation before parsing
- We use helper functions for clean code

### Step 3: Add Representation State Variables

```javascript
stateVariableDefinitions.binary = {
    public: true,
    shadowingInstructions: { createComponentOfType: "text" },
    returnDependencies: () => ({
        value: { dependencyType: "stateVariable", variableName: "value" },
    }),
    definition({ dependencyValues }) {
        let value = dependencyValues.value;
        let binaryString = "";

        if (Number.isNaN(value) || !Number.isFinite(value)) {
            binaryString = String(value);
        } else if (value === 0) {
            binaryString = "0";
        } else if (value < 0) {
            binaryString = "-" + Math.abs(value).toString(2);
        } else {
            binaryString = value.toString(2);
        }

        return {
            setValue: { binary: binaryString },
        };
    },
};

// Similar for hexadecimal and decimal...
```

**Why:**
- Users can now access `$a.binary` regardless of input representation
- Handles edge cases (NaN, Infinity, negative numbers)

### Step 4: Override text for Display

```javascript
stateVariableDefinitions.text = {
    public: true,
    shadowingInstructions: { createComponentOfType: "text" },
    forRenderer: true,  // Crucial for display
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

        return {
            setValue: { text: text },
        };
    },
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
                instructions: [
                    {
                        setDependency: "value",
                        desiredValue: Math.round(desiredNumber),
                    },
                ],
            };
        } else {
            return { success: false };
        }
    },
};
```

**Why:**
- `forRenderer: true` ensures the display shows the correct representation
- Inverse definition allows user input to be parsed correctly

### Step 5: Create Helper Functions

```javascript
function parseBinary(str) {
    if (!str || typeof str !== "string") {
        return NaN;
    }

    let trimmed = str.trim();
    let isNegative = false;

    if (trimmed.startsWith("-")) {
        isNegative = true;
        trimmed = trimmed.substring(1).trim();
    }

    if (trimmed.startsWith("0b") || trimmed.startsWith("0B")) {
        trimmed = trimmed.substring(2);
    }

    if (!/^[01]+$/.test(trimmed)) {
        return NaN;
    }

    let value = parseInt(trimmed, 2);
    return isNegative ? -value : value;
}

function parseHexadecimal(str) {
    if (!str || typeof str !== "string") {
        return NaN;
    }

    let trimmed = str.trim();
    let isNegative = false;

    if (trimmed.startsWith("-")) {
        isNegative = true;
        trimmed = trimmed.substring(1).trim();
    }

    if (trimmed.startsWith("0x") || trimmed.startsWith("0X")) {
        trimmed = trimmed.substring(2);
    }

    if (!/^[0-9a-fA-F]+$/.test(trimmed)) {
        return NaN;
    }

    let value = parseInt(trimmed, 16);
    return isNegative ? -value : value;
}
```

**Why:**
- Reusable across multiple state variables
- Handles common formats (with/without prefixes)
- Validates input before parsing

### Complete Flow

1. **User writes:** `<integer representation="binary">1001</integer>`
2. **Attribute created:** `representation = "binary"`
3. **Value parsing:** 
   - `valuePreRound = "1001"` (string)
   - `representation = "binary"`
   - Calls `parseBinary("1001")` → returns `9`
   - `value = 9` (stored as decimal)
4. **Representation state variables computed:**
   - `decimal = "9"`
   - `binary = "1001"`
   - `hexadecimal = "9"`
5. **Text determined:**
   - Since `representation = "binary"`, `text = binary = "1001"`
6. **Renderer displays:** "1001"

### Result

Users can now:
```xml
<integer name="a" representation="binary">1001</integer>
<p>Binary: $a.binary</p>       <!-- 1001 -->
<p>Decimal: $a.decimal</p>     <!-- 9 -->
<p>Hex: $a.hexadecimal</p>     <!-- 9 -->
<p>Value: $a.value</p>         <!-- 9 -->
<p>Display: $a.text</p>        <!-- 1001 -->
```

---

## Checklist for Component Updates

When adding a feature to a component, use this checklist:

- [ ] **Attributes**
  - [ ] Added to `createAttributesObject()`
  - [ ] Proper type specified
  - [ ] Default value set
  - [ ] Public access configured if needed

- [ ] **State Variables**
  - [ ] Dependencies declared
  - [ ] Definition logic implemented
  - [ ] Edge cases handled
  - [ ] Inverse definition added if needed
  - [ ] Public access enabled if needed
  - [ ] Shadowing instructions configured

- [ ] **Display**
  - [ ] `text` state variable updated if needed
  - [ ] `forRenderer: true` set where appropriate
  - [ ] Correct representation shown

- [ ] **Helper Functions**
  - [ ] Created outside class
  - [ ] Documented with JSDoc
  - [ ] Input validation included
  - [ ] Edge cases handled
  - [ ] Consistent return types

- [ ] **Testing**
  - [ ] Basic functionality tested
  - [ ] Edge cases tested
  - [ ] Invalid input handling tested
  - [ ] Component references tested
  - [ ] Answer validation tested (if applicable)

- [ ] **Documentation**
  - [ ] Comments added to complex logic
  - [ ] TODOs removed or addressed
  - [ ] Example usage documented

---

## Common Pitfalls

### ❌ Forgetting `forRenderer: true`

```javascript
// Won't display correctly
stateVariableDefinitions.text = {
    public: true,
    // Missing: forRenderer: true
    definition({ dependencyValues }) {
        return { setValue: { text: "..." } };
    },
};
```

### ❌ Not Handling Edge Cases

```javascript
// Will crash on null/undefined
function parse(str) {
    return parseInt(str, 2);  // str.trim() will fail if str is null
}

// Better:
function parse(str) {
    if (!str || typeof str !== "string") {
        return NaN;
    }
    return parseInt(str.trim(), 2);
}
```

### ❌ Modifying Parent Objects Directly

```javascript
// Don't do this
static createAttributesObject() {
    let attributes = super.createAttributesObject();
    attributes.existingAttribute.defaultValue = "changed";  // Mutates parent
    return attributes;
}
```

### ❌ Using || for Numeric Fallbacks

```javascript
// Bug: 0 is falsy, so it falls back incorrectly
return {
    setValue: { value: parsedValue || defaultValue },
};

// Better:
return {
    setValue: {
        value: Number.isNaN(parsedValue) ? defaultValue : parsedValue,
    },
};
```

---

## Additional Resources

- **Component Examples:** `packages/doenetml-worker-javascript/src/components/`
- **Base Classes:** Check `Number.js`, `Text.js`, etc. for common patterns
- **Utilities:** `packages/doenetml-worker-javascript/src/utils/`
- **Testing:** `packages/test-viewer/` for manual testing

---

## Summary

Key steps for updating/creating components:

1. **Understand the architecture** - attributes, state variables, dependencies
2. **Add attributes** via `createAttributesObject()` for user configuration
3. **Define state variables** with proper dependencies and definitions
4. **Handle edge cases** - null, invalid input, type checking
5. **Override display** by modifying the `text` state variable
6. **Create helper functions** for complex logic outside the class
7. **Test thoroughly** with edge cases and interactions
8. **Document your changes** with comments and examples

Follow these patterns and your component updates will integrate seamlessly into DoenetML!
