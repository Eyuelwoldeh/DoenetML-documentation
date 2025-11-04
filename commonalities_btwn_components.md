## 1. Theres an Action methods pattern

### All action methods follow this structure:

```javascript
async actionName({
    
    param1,
    param2,
    transient,
    actionId,
    sourceInformation = {},
    skipRendererUpdate = false,
}) {
    // performs an action using coreFunctions
    return await this.coreFunctions.performUpdate({
        updateInstructions: [{
            updateType: "updateValue",
            componentIdx: this.componentIdx,
            stateVariable: "variableName",
            value: newValue,
        }],
        transient,           // temporary vs permanent
        actionId,
        sourceInformation,
        skipRendererUpdate,
        event: {             // record event (if not transient)
            verb: "interacted",
            object: {
                componentIdx: this.componentIdx,
                componentType: this.componentType,
            },
            result: { /* what happened */ },
        },
    });
}
```

**some common action types:**
- **Move actions**: `moveText`, `moveNumber`, `movePoint`
- **Click actions**: `textClicked`, `numberClicked`, `pointClicked`
- **Focus actions**: `textFocused`, `numberFocused`, `pointFocused`
- **Update actions**: Component-specific interactions

**Transient parameter:**
- `true`: Temporary update (while dragging)
- `false`: Permanent update (on release)

## 2. 