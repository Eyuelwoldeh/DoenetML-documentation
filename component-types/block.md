# Block Components

block-level layout components like paragraphs, figures, tables.

**base class:** `BlockComponent` (or `SectioningComponent` for sections)

**examples:** `P.js`, `Figure.js`, `Table.js`, `Image.js`, `Video.js`, `Section.js`

**renderer:** usually renders children inside a div/section/table HTML structure

---

## basic structure

block components are the simplest structurally. they mostly just accept and pass through children.

```js
import BlockComponent from "./abstract/BlockComponent";

export default class MyBlock extends BlockComponent {
    static componentType = "myBlock";

    static returnChildGroups() {
        return [
            // _base accepts basically anything as a child
            { group: "anything", componentTypes: ["_base"] },
        ];
    }
}
```

that's it for a minimal block component. it accepts any children and renders them inside a block-level element.

---

## adding attributes

if your block component needs configuration, add attributes the same way as any other component:

```js
static createAttributesObject() {
    let attributes = super.createAttributesObject();

    attributes.width = {
        createComponentOfType: "text",
        createStateVariable: "width",
        defaultValue: "100%",
        public: true,
        forRenderer: true,
    };

    attributes.border = {
        createComponentOfType: "boolean",
        createStateVariable: "border",
        defaultValue: false,
        public: true,
        forRenderer: true,
    };

    return attributes;
}
```

---

## the renderer side

block component renderers are usually straightforward React components that wrap children in an HTML element:

```jsx
import React from "react";
import useDoenetRenderer from "../useDoenetRenderer";

export default React.memo(function MyBlock(props) {
    let { name, id, SVs, children } = useDoenetRenderer(props);

    if (SVs.hidden) return null;

    let style = {};
    if (SVs.width) {
        style.width = SVs.width;
    }
    if (SVs.border) {
        style.border = "1px solid #ccc";
    }

    return (
        <div id={id} style={style}>
            {children}
        </div>
    );
});
```

the key difference from other renderers: block components usually just render `{children}` and let the child components handle their own rendering.

---

## sectioning components

`SectioningComponent` extends `BlockComponent` and adds support for titles, numbering, and collapsing. use it when your component is a structural container like a section or chapter.

```js
import SectioningComponent from "./abstract/SectioningComponent";

export default class MySection extends SectioningComponent {
    static componentType = "mySection";

    static returnChildGroups() {
        return [
            { group: "titles", componentTypes: ["title"] },
            { group: "anything", componentTypes: ["_base"] },
        ];
    }
}
```

`SectioningComponent` gives you:
- `title` support (renders a heading)
- `sectionNumber` state variable
- `collapsible` / `open` attributes for expand/collapse behavior
- nesting awareness (sections inside sections)

---

## what BlockComponent gives you for free

- `hidden` -- whether the component is visible
- `disabled` -- whether the component is interactive
- basic child rendering infrastructure

---

## child groups

the `returnChildGroups()` method defines what types of children your block can contain:

```js
static returnChildGroups() {
    return [
        // accept only specific types
        { group: "paragraphs", componentTypes: ["p"] },
        { group: "images", componentTypes: ["image"] },

        // or accept anything
        { group: "anything", componentTypes: ["_base"] },
    ];
}
```

using `["_base"]` as the component type means "accept any component". this is common for layout containers that don't care what's inside them.

if you want to accept both inline and block children, just use `["_base"]`. the system will figure out the rendering order.

---

## state variables for block components

most block components don't need many state variables since they're primarily layout containers. but if you need computed state:

```js
static returnStateVariableDefinitions() {
    let stateVariableDefinitions = super.returnStateVariableDefinitions();

    // example: count the number of child images
    stateVariableDefinitions.numImages = {
        public: true,
        returnDependencies: () => ({
            imageChildren: {
                dependencyType: "child",
                childGroups: ["images"],
            },
        }),
        definition({ dependencyValues }) {
            return {
                setValue: {
                    numImages: dependencyValues.imageChildren.length,
                },
            };
        },
    };

    return stateVariableDefinitions;
}
```

---

## common mistakes with block components

1. **forgetting to render `{children}`** -- if your renderer doesn't include `{children}`, the child components won't show up

2. **using the wrong base class** -- if your component needs title/section number support, use `SectioningComponent` not `BlockComponent`

3. **restrictive child groups** -- if users complain they can't put certain things inside your component, check your `returnChildGroups()`. using `["_base"]` is the safe catch-all

4. **not handling `SVs.hidden`** -- always check this and return null if hidden. otherwise the component will render even when the user sets `hidden="true"`
