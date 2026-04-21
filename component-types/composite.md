# Composite Components

components that expand into other components. they don't render themselves -- they generate replacement components.

**base class:** `CompositeComponent`

**examples:** `NumberList.js`, `MathList.js`, `Sequence.js`, `Select.js`, `TextList.js`

**renderer:** none! they expand into child components that each have their own renderer.

---

## basic structure

```js
import CompositeComponent from "./abstract/CompositeComponent";

export default class MyList extends CompositeComponent {
    static componentType = "myList";

    // this tells the system which state variable triggers replacement updates
    static stateVariableToEvaluateAfterReplacements = "readyToExpandWhenResolved";

    // when another component uses this as an attribute, shadow this variable
    static stateVariableToBeShadowed = "items";

    static returnChildGroups() {
        return [
            { group: "numbers", componentTypes: ["number"] },
        ];
    }

    static returnStateVariableDefinitions() {
        let stateVariableDefinitions = super.returnStateVariableDefinitions();

        stateVariableDefinitions.numComponents = {
            public: true,
            returnDependencies: () => ({
                children: {
                    dependencyType: "child",
                    childGroups: ["numbers"],
                },
            }),
            definition({ dependencyValues }) {
                return {
                    setValue: {
                        numComponents: dependencyValues.children.length,
                    },
                };
            },
        };

        // this is the standard trigger pattern
        // when numComponents goes stale, it tells the system to update replacements
        stateVariableDefinitions.readyToExpandWhenResolved = {
            returnDependencies: () => ({
                numComponents: {
                    dependencyType: "stateVariable",
                    variableName: "numComponents",
                },
            }),
            markStale: () => ({ updateReplacements: true }),
            definition() {
                return { setValue: { readyToExpandWhenResolved: true } };
            },
        };

        return stateVariableDefinitions;
    }

    // creates the initial replacement components
    static async createSerializedReplacements({ component, workspace, nComponents }) {
        let numComponents = await component.stateValues.numComponents;
        let replacements = [];

        for (let i = 0; i < numComponents; i++) {
            replacements.push({
                type: "serialized",
                componentType: "number",
                componentIdx: nComponents++,
                attributes: {},
                doenetAttributes: {},
                children: [],
                state: {},
            });
        }

        workspace.numComponents = numComponents;
        return { replacements, errors: [], warnings: [], nComponents };
    }

    // updates replacements when things change
    static async calculateReplacementChanges({ component, workspace, nComponents }) {
        // simplest approach: just recreate all replacements
        let result = await this.createSerializedReplacements({
            component, workspace, nComponents,
        });
        return {
            replacementChanges: [{
                changeType: "add",
                changeTopLevelReplacements: true,
                firstReplacementInd: 0,
                numberReplacementsToReplace: component.replacements.length,
                serializedReplacements: result.replacements,
            }],
            nComponents: result.nComponents,
        };
    }
}
```

---

## how it works conceptually

when the system encounters a composite component, it doesn't try to render it. instead it asks the component to produce a list of replacement components. those replacements are what actually get rendered.

for example, `<numberList>1 2 3</numberList>` doesn't render as a list element. it expands into three separate `<number>` components that each render individually.

the lifecycle is:
1. component is created
2. `createSerializedReplacements()` runs to produce the initial set of replacements
3. whenever `readyToExpandWhenResolved` goes stale, `calculateReplacementChanges()` runs
4. the system diffs the changes and updates the rendered output

---

## the trigger pattern

the `readyToExpandWhenResolved` + `markStale` combo is the standard way to tell the system "hey, my replacements need to be updated":

```js
stateVariableDefinitions.readyToExpandWhenResolved = {
    returnDependencies: () => ({
        // depend on whatever controls the number/type of replacements
        numComponents: {
            dependencyType: "stateVariable",
            variableName: "numComponents",
        },
    }),
    // this is the key part -- tells the system to call calculateReplacementChanges
    markStale: () => ({ updateReplacements: true }),
    definition() {
        return { setValue: { readyToExpandWhenResolved: true } };
    },
};
```

you also need to set the static property:
```js
static stateVariableToEvaluateAfterReplacements = "readyToExpandWhenResolved";
```

this tells the system which state variable to evaluate to decide if replacements need updating.

---

## replacement changes

the `calculateReplacementChanges` method returns an array of change objects. each one describes what to add or remove:

```js
// replace all existing replacements with new ones
{
    changeType: "add",
    changeTopLevelReplacements: true,
    firstReplacementInd: 0,
    numberReplacementsToReplace: component.replacements.length,
    serializedReplacements: newReplacements,
}

// add new replacements at the end
{
    changeType: "add",
    changeTopLevelReplacements: true,
    firstReplacementInd: component.replacements.length,
    numberReplacementsToReplace: 0,
    serializedReplacements: additionalReplacements,
}

// remove replacements from the end
{
    changeType: "delete",
    changeTopLevelReplacements: true,
    firstReplacementInd: desiredCount,
    numberReplacementsToDelete: currentCount - desiredCount,
}
```

for simple cases, the easiest approach is just to replace everything every time. for performance-sensitive cases (like large lists), you can compute a more surgical diff.

---

## shadowing

`stateVariableToBeShadowed` controls what happens when another component uses this composite as an attribute value:

```js
static stateVariableToBeShadowed = "items";
```

this means if someone writes `<myComponent list="$myList" />`, the system will shadow the `items` state variable from the composite.

---

## workspace

the `workspace` object is passed to both `createSerializedReplacements` and `calculateReplacementChanges`. it persists between calls so you can store bookkeeping info:

```js
// in createSerializedReplacements:
workspace.numComponents = numComponents;
workspace.previousItems = [...items];

// in calculateReplacementChanges:
let previousCount = workspace.numComponents;
// compare with current to figure out what changed
```

---

## common mistakes with composite components

1. **forgetting `stateVariableToEvaluateAfterReplacements`** -- without this static property, the system doesn't know to check for replacement updates

2. **forgetting `markStale: () => ({ updateReplacements: true })`** -- this is what actually triggers the replacement update. without it, `readyToExpandWhenResolved` is just a normal state variable

3. **not tracking `nComponents`** -- every replacement needs a unique `componentIdx`. pass `nComponents` through and increment it for each new replacement, then return the updated count

4. **mutating the workspace incorrectly** -- workspace persists between calls. if you store arrays or objects in it, be careful about mutation vs replacement
