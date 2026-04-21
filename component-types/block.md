# Block Components

components that take up a full block of space in the document, like a paragraph, a figure, a table, or a section. they are not inline and usually wrap children.

**base class:** `BlockComponent` (or `SectioningComponent` for things like sections and chapters)

**examples:** `P.js`, `Figure.js`, `Table.js`, `Image.js`, `Video.js`, `Section.js`

**renderer:** usually a React component that renders its children inside some HTML wrapper element

these are the structurally simplest type. they mostly just declare what children they accept and let the system handle rendering of those children.

---

## basic structure

```js
import BlockComponent from "./abstract/BlockComponent";

export default class MyBlock extends BlockComponent {
    static componentType = "myBlock";

    static createAttributesObject() {
        let attributes = super.createAttributesObject();

        // add any block-specific attributes here
        attributes.width = {
            createComponentOfType: "number",
            createStateVariable: "width",
            defaultValue: 100,
            public: true,
            forRenderer: true,
        };

        return attributes;
    }

    // declare the children this block can accept.
    // "_base" means any component type (the base of the whole tree).
    static returnChildGroups() {
        return [
            { group: "anything", componentTypes: ["_base"] },
        ];
    }

    static returnStateVariableDefinitions() {
        let stateVariableDefinitions = super.returnStateVariableDefinitions();
        // for most block components, you do not need to add much here.
        // the children render themselves, and this component just wraps them.
        return stateVariableDefinitions;
    }
}
```

---

## the renderer side

block component renderers are usually straightforward. they wrap their children in an HTML element:

```tsx
import React from "react";
import useDoenetRenderer from "../useDoenetRenderer";

export default function MyBlock(props: any) {
    // children from useDoenetRenderer is the rendered output of the component's children.
    // you usually just place it inside your wrapper.
    const { id, SVs, children } = useDoenetRenderer(props);

    if (SVs.hidden) return null;

    return (
        <div id={id} style={{ width: SVs.width }}>
            {children}
        </div>
    );
}
```

the key difference from other renderers: block components usually just render `{children}` and let the child components handle their own rendering.

---

## what BlockComponent gives you for free

- `hidden`: whether the component is visible
- `disabled`: whether the component is interactive
- basic child rendering infrastructure

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

1. **forgetting to render `{children}`**: if your renderer doesn't include `{children}`, the child components won't show up

2. **using the wrong base class**: if your component needs title/section number support, use `SectioningComponent` not `BlockComponent`

3. **restrictive child groups**: if users complain they can't put certain things inside your component, check your `returnChildGroups()`. using `["_base"]` is the safe catch-all

4. **not handling `SVs.hidden`**: always check this and return null if hidden. otherwise the component will render even when the user sets `hidden="true"`
