# Composite Components

components that expand into one or more other components. they do not render themselves. instead, they produce "replacement" components that each have their own rendering.

**base class:** `CompositeComponent`

**examples:** `NumberList.js`, `MathList.js`, `Sequence.js`, `Select.js`, `TextList.js`

**renderer:** none. the replacements they generate have their own renderers.

---

## how they work

when the system encounters a composite component, it doesn't try to render it. instead it asks the component to produce a list of replacement components. those replacements are what actually get rendered.

this is how `<sequence from="1" to="5" />` expands into five `<number>` components, or how `<numberList>3 1 4 1 5</numberList>` expands into five individual `<number>` components.

the lifecycle is:
1. component is created
2. `createSerializedReplacements()` runs to produce the initial set of replacements
3. whenever `readyToExpandWhenResolved` goes stale, `calculateReplacementChanges()` runs
4. the system diffs the changes and updates the rendered output

---

## basic structure

```js
import CompositeComponent from "./abstract/CompositeComponent";

export default class MyList extends CompositeComponent {
    static componentType = "myList";

    // this tells the system which state variable to watch for triggering replacement rebuilds
    static stateVariableToEvaluateAfterReplacements = "readyToExpandWhenResolved";

    // when another component uses this as an attribute (like numberList is used
    // in graphical components), this is the state variable that gets exposed
    static stateVariableToBeShadowed = "numbers";

    static returnChildGroups() {
        return [
            { group: "numbers", componentTypes: ["number"] },
        ];
    }

    static returnStateVariableDefinitions() {
        let stateVariableDefinitions = super.returnStateVariableDefinitions();

        // count the children so we know how many replacements to create
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
                    setValue: { numComponents: dependencyValues.children.length },
                };
            },
        };

        // readyToExpandWhenResolved is the standard trigger variable for composites.
        // when it goes stale (because numComponents changed), the system knows to
        // call calculateReplacementChanges.
        stateVariableDefinitions.readyToExpandWhenResolved = {
            returnDependencies: () => ({
                numComponents: {
                    dependencyType: "stateVariable",
                    variableName: "numComponents",
                },
            }),
            // markStale is special: returning updateReplacements: true tells the
            // system that replacements need to be recalculated when this goes stale.
            markStale: () => ({ updateReplacements: true }),
            definition() {
                return { setValue: { readyToExpandWhenResolved: true } };
            },
        };

        return stateVariableDefinitions;
    }

    // createSerializedReplacements runs once when the component is first created.
    // it returns a list of component definitions.
    // nComponents is a running count of all components in the document, used to
    // give each replacement a unique index.
    static async createSerializedReplacements({ component, workspace, nComponents }) {
        let numComponents = await component.stateValues.numComponents;
        let replacements = [];

        for (let i = 0; i < numComponents; i++) {
            replacements.push({
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

    // calculateReplacementChanges runs when state changes.
    // the simplest approach is to recreate all replacements.
    // for better performance in large components, do a diff and only add/remove what changed.
    static async calculateReplacementChanges({
        component,
        workspace,
        nComponents,
        componentInfoObjects,
    }) {
        let result = await this.createSerializedReplacements({
            component,
            workspace,
            nComponents,
            componentInfoObjects,
        });

        return {
            replacementChanges: [
                {
                    changeType: "add",
                    changeTopLevelReplacements: true,
                    firstReplacementInd: 0,
                    numberReplacementsToReplace: component.replacements
                        ? component.replacements.length
                        : 0,
                    serializedReplacements: result.replacements,
                },
            ],
            nComponents: result.nComponents,
        };
    }
}
```

---

## the trigger pattern

the `readyToExpandWhenResolved` + `markStale` combo is the standard way to tell the system that replacements need updating. copy it from any existing composite component:

```js
stateVariableDefinitions.readyToExpandWhenResolved = {
    returnDependencies: () => ({
        // depend on whatever controls the number/type of replacements
        numComponents: {
            dependencyType: "stateVariable",
            variableName: "numComponents",
        },
    }),
    // this is the key part: tells the system to call calculateReplacementChanges
    markStale: () => ({ updateReplacements: true }),
    definition() {
        return { setValue: { readyToExpandWhenResolved: true } };
    },
};
```

you also need the static property:
```js
static stateVariableToEvaluateAfterReplacements = "readyToExpandWhenResolved";
```

---

## replacement changes

the `calculateReplacementChanges` method returns an array of change objects. each one describes what to add or remove:

```js
// replace all existing replacements with new ones
{
    changeType: "add",
    changeTopLevelReplacements: true,
    firstReplacementInd: 0,
    numberReplacementsToReplace: component.replacements ? component.replacements.length : 0,
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

for simple cases, replacing everything every time is fine. for performance-sensitive cases (like large lists), compute a more surgical diff.

---

## shadowing

`stateVariableToBeShadowed` controls what happens when another component uses this composite as an attribute value.

for example, when you write `values="3 1 4 1 5"` on a graphical component and the attribute type is `numberList`, the graphical component gets access to the shadowed variable of the `numberList`. that's how it can read `.stateValues.numbers` from it.

```js
static stateVariableToBeShadowed = "numbers";
```

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

1. **forgetting `stateVariableToEvaluateAfterReplacements`**: without this static property, the system doesn't know to check for replacement updates

2. **forgetting `markStale: () => ({ updateReplacements: true })`**: this is what actually triggers the replacement update. without it, `readyToExpandWhenResolved` is just a normal state variable

3. **not tracking `nComponents`**: every replacement needs a unique `componentIdx`. pass `nComponents` through, increment it for each new replacement, then return the updated count

4. **not guarding against null replacements**: when first expanding, `component.replacements` may be undefined. always use `component.replacements ? component.replacements.length : 0`

5. **mutating the workspace incorrectly**: workspace persists between calls. if you store arrays or objects in it, be careful about mutation vs replacement
