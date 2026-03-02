# Creating DoenetML Components

> A guide for developers who want to extend DoenetML by creating or modifying components

## Introduction

DoenetML components are JavaScript classes that define interactive educational elements. This guide walks you through adding new features to existing components or creating entirely new ones.

### What You'll Learn

- How DoenetML components are structured
- Adding custom attributes to components
- Working with state variables and dependencies

---

## Component Architecture

Every DoenetML component follows a consistent structure:

```javascript
export default class MyComponent extends BaseComponent {
    static componentType = "myComponent";
    static rendererType = "text";
    
    static createAttributesObject() {
        // Define component attributes
    }
    
    static returnStateVariableDefinitions() {
        // Define reactive state
    }
}
```

### Key Concepts

**Attributes** are properties users set in DoenetML tags:
```xml
<integer representation="binary">1001</integer>
```

**State Variables** hold the component's reactive data and drive its behavior. They automatically update when dependencies change.

---

## Adding Attributes

Attributes let users configure your component's behavior.

### Basic Attribute Definition

```javascript
static createAttributesObject() {
    let attributes = super.createAttributesObject();

    attributes.myAttribute = {
        createPrimitiveOfType: "string",      // Type: string, number, boolean
        createStateVariable: "myAttribute",   // Auto-creates state variable
        defaultValue: "default",              // Fallback value
        public: true,                         // Allow $component.myAttribute
        forRenderer: true,                    // Pass to display renderer
    };

    return attributes;
}
```

### Real Example: Integer Representation

The `<integer>` component supports different number representations:

```javascript
attributes.representation = {
    createPrimitiveOfType: "string",
    createStateVariable: "representation",
    defaultValue: "decimal",
    public: true,
    forRenderer: true,
};
```

Now users can write:
```xml
<integer representation="binary">1001</integer>
<integer representation="hexadecimal">FF</integer>
```

---

## State Variables

State variables are the heart of DoenetML's reactivity.

### Anatomy of a State Variable

```javascript
stateVariableDefinitions.myVariable = {
    public: true,                    // Accessible via $component.myVariable
    forRenderer: true,               // Send to renderer for display
    
    returnDependencies: () => ({
        otherVar: {
            dependencyType: "stateVariable",
            variableName: "otherVariable",
        },
    }),
    
    definition({ dependencyValues }) {
        // Compute the value
        let result = transform(dependencyValues.otherVar);
        
        return {
            setValue: { myVariable: result },
        };
    },
};
```

### Working Example: Binary Display

```javascript
stateVariableDefinitions.binary = {
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
        let value = dependencyValues.value;
        let binaryString;
        
        if (value < 0) {
            binaryString = "-" + Math.abs(value).toString(2);
        } else {
            binaryString = value.toString(2);
        }
        
        return {
            setValue: { binary: binaryString },
        };
    },
};
```

<!-- TODO: Add section on inverse definitions for two-way binding -->

---

## Parsing Different Formats

When your component accepts multiple input formats, create helper functions:

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
    
    // Handle optional 0b prefix
    if (trimmed.startsWith("0b") || trimmed.startsWith("0B")) {
        trimmed = trimmed.substring(2);
    }
    
    // Validate format
    if (!/^[01]+$/.test(trimmed)) {
        return NaN;
    }
    
    let value = parseInt(trimmed, 2);
    return isNegative ? -value : value;
}
```

### Using Parsers in State Variables