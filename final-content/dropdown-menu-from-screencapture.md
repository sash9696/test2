---
title: "Dropdown Menu"
slug: "dropdown-menu"
difficulty: "medium"
duration_mins: 30
tags:
  - system-design
  - accessibility
  - ui-component
premium: true
summary: "Design a dropdown menu component that reveals a menu containing a list of actions."
free_preview: false
status: draft
---

# Dropdown Menu

## Question

Design a dropdown menu component that can reveal a menu containing a list of actions.

![Dropdown Menu Example](dropdown-menu-pdf-assets/figures/dropdown-menu-example.png)


## Real-life examples

Dropdowns · Bootstrap v5.3
React Menu component - Material UI
Dropdown Menu — Radix UI
## Requirements exploration

Before designing the API, we scope the component by clarifying how many menus can coexist, what content they hold, which devices they support, and the expected level of customization.

#### Can there be multiple dropdown menus open at once?

Yes, there can be.

#### What content will the menu contain?

Only text, no focusable elements.

#### Is there a maximum number of items allowed in the menu?

There's no fixed maximum, but preferably, there should not be more than 20 items for a better user experience.

#### What amount of flexibility does the user have in customizing the design?

Users should be able to customize colors, fonts, padding, etc., to match their branding.

#### What devices will this component be used on?

All devices: mobile, tablet, desktop.

## Architecture / high-level design

An example usage of the dropdown menu in React, with event handlers omitted.


<DropdownMenu>
<DropdownMenu.Button>Actions</DropdownMenu.Button>
<DropdownMenu.List>
<DropdownMenu.Item>New File</DropdownMenu.Item>
<DropdownMenu.Item>Save</DropdownMenu.Item>
<DropdownMenu.Item>Delete</DropdownMenu.Item>
</DropdownMenu.List>
</DropdownMenu>
| Component | Role |
| --- | --- |
| Dropdown Menu (DropdownMenu) | Root component, coordinates events between the inner components. |
| --- | --- |
| Menu Button (DropdownMenu.Button) | Button that toggles the display state of the DropdownMenu.List. |
| --- | --- |
| Menu List (DropdownMenu.List) | Contains the list of items. |
| --- | --- |
| Menu List Item (DropdownMenu.Item) | Individual list items. |
In React, components can communicate with their parents using context or props. We choose to use context here since we are using a composition model and passing props is inconvenient. <DropdownMenu> should contain a context provider that provides the state values (see Data Model section) to all its child components.


DropdownMenu (root + context provider)

MenuContext { isOpen, activeItem, setOpen, setActive }

DropdownMenu.Button (trigger)

DropdownMenu.List (role=menu)

DropdownMenu.Item (role=menuitem)

DropdownMenu.Item (role=menuitem)


![Compound component hierarchy and context flow](dropdown-menu-pdf-assets/figures/compound-component-hierarchy-and-context-flow.png)

Expose a compound-component API rather than a single monolithic prop tree
Splitting the component into DropdownMenu, DropdownMenu.Button, DropdownMenu.List, and DropdownMenu.Item lets consumers compose arbitrary trigger and item content while the root owns shared state via context. A single <DropdownMenu items={...} /> prop-based API quickly breaks down once items need icons, disabled states, or custom rendering.

## Data model

Note that when designing components, it might make sense to design the interface/API first, or both the data model and API concurrently. It depends on the component at hand. Feel free to jump between the two sections.

| State | Type | Description |
| --- | --- | --- |
| isOpen | boolean | Whether the menu is currently open or closed. |
| --- | --- | --- |
| activeItem | string | Menu item that is focused. We need this because hovering over menu items will change the active item. By keeping track of this value in state, we can respond to keyboard interactions, either focusing on the prev/next item or activating it. |
These state values are housed within <DropdownMenu> and provided to all components via React context.

See below for configuration options, which are also part of the data model.

## Interface definition (API)

The API surface is split into props shared across subcomponents and props specific to each composition piece (root, trigger, menu, item).

General props
These props apply to most of the components.

| Prop | Type | Description |
| --- | --- | --- |
| children | React.Node | Children of the component. If using TypeScript/Flow, you can enforce specific components to be used as children. |
| --- | --- | --- |
| as | string | Component | In case there is a need to customize the underlying DOM element/component. |
| --- | --- | --- | --- |
| className | string | Classnames to add to components in case further visual customization is needed. May or may not be needed depending on theming approach. |
DropdownMenu
| Prop | Type | Description |
| --- | --- | --- |
| isInitiallyOpen | boolean | Whether the menu is initially open or closed. |
| --- | --- | --- |
| size | string | Prop to customize size. Needed only if customization is desired. |
DropdownMenu.Button
| Prop | Type | Description |
| --- | --- | --- |
| onClick | function | Although the opening/closing will be handled within DropdownMenu, exposing an onClick prop is useful if additional logic needs to be performed (e.g., analytics logging). |
| --- | --- | --- |
| Other button props | * | Since this component is usually a <button>, it should also allow other props that <button>s expect, such as disabled. |
DropdownMenu.List
| Prop | Type | Description |
| --- | --- | --- |
| maxHeight | number | undefined | Max height of the menu list. There should be a sensible default of around 200-300px. |
| --- | --- | --- | --- |
| position | string | Position of the list relative to the button. |
DropdownMenu.Item
| Prop | Type | Description |
| --- | --- | --- |
| onClick | function | Triggered when item is activated. Possible responses include navigating to another page. |
| --- | --- | --- |
| disabled | boolean | Whether the item is disabled. Disabled items cannot be activated and are optionally skipped when interacting with the menu using a keyboard. |
Customizing Appearance
Guidance on designing good APIs for customizing UI components can be found in the Front End Interview Guidebook's UI Components API Design Principles Section.

## Optimizations and deep dive

The tricky parts of a dropdown menu are floating rendering, viewport-aware positioning, accessibility, and language flexibility:

Rendering: This section explains how to render the floating menu outside normal document flow while preserving positioning and layering behavior.
Automatic flipping when near the edge: This section covers viewport collision detection and placement changes when there is not enough space.
Accessibility (a11y): This section discusses pointer, keyboard, and assistive-technology behavior for a usable dropdown menu.
Internationalization (i18n): This section covers long translated labels, text direction, and layout flexibility for localized menus.
Rendering
Rendering the dropdown menu can be quite tricky because the menu is "floating" and does not exactly follow the normal flow of page elements.

Layout
There are two common ways to render the dropdown menu near the button. We've provided minimal code samples for each layout approach.

Relative to Button

In this approach, we wrap a <div> with position: relative around the <button> and the menu. The menu uses position: absolute, which positions it relative to its closest positioned ancestor, which is the root <div>.


This method is straightforward and does not require much calculation of elements and their positions relative to the page, but parent containers with overflow: hidden can clip the menu and its contents, or there could be z-index issues.

This approach is used by Headless UI and Bootstrap.

Relative to Page

In this approach, the menu is rendered as a direct child of the <body> and positioned absolute-ly to the page by getting the element's offsetTop and offsetLeft to get the coordinates of the <button> relative to the page and adding its height (offsetHeight) to get the final Y position to render the menu.

In React, this can be done using React Portals, which let you render outside the DOM hierarchy of the parent component. A typical use case for portals is when a parent component has an overflow: hidden or z-index style, but you need the child to visually "break out" of its container. Common examples include dropdown menus, tooltips, and modals.


The downside of this approach is that if the window is resized or if the contents of the page change such that the height of the page is shorter than when the menu was first shown, the menu's position will be incorrect. As a workaround, you can watch the window for height changes and re-render the menu with the correct position.

This approach is used by Radix UI and Reach UI.

Prefer portal-based rendering for production-grade dropdown menus
Rendering the menu as a child of <body> via a portal avoids clipping from ancestor overflow: hidden containers and sidesteps z-index stacking-context bugs that plague the relative-to-button approach. The position-recalculation cost on resize or scroll is a much easier problem to solve than chasing container-specific clipping across every page the component ships on.

Position
The component can also allow customization of alignment in all directions around the <button>. Examples of left/right-aligned menu lists can be found in the example below. Supporting more alignments is a matter of finding the right values for top/left/right/bottom/CSS translations to use.


Maximum height
Since there is no maximum allowable number of items in the menu, we can set a default maximum height for the menu so that excess items can be accessed by scrolling within the menu. This height can also be customized.

Rendering in HTML or JavaScript
The dropdown menu can either be:

Rendered into the HTML like Bootstrap's Dropdowns. The menu is initially hidden from view via display: none / opacity: 0 / hidden attribute, and those styles are toggled when the menu is to be shown.
Rendered on the fly via JavaScript after the menu button is activated.
The pro of rendering in the HTML first is better runtime performance due to fewer DOM operations needed to show the menu. The downside is that the HTML can be unnecessarily bloated, especially if the user never interacts with the dropdown menu at all. Since the menu items usually don't contribute towards SEO and there likely won't be that many elements, the benefits of rendering the menu beforehand in HTML are relatively minor.

Automatic flipping when near the edge
Smart dropdown menus will know their position relative to the viewport and can flip themselves when there is insufficient space to display the full menu in their current position. Reach UI's Menu has autoflipping implemented.

Optionally, scrolling can be disabled on the window when a menu is open (by adding overflow: hidden to <body>). This limits the user experience but avoids the need for automatic flipping, which can be complicated to implement. Material UI's Menu component takes this approach.

![Dropdown Menu Autoflipping example](dropdown-menu-pdf-assets/figures/dropdown-menu-autoflipping-example.png)


The core idea here is to know how tall the menu will be and autoflip when the bottom of the menu will exceed the viewport's height.

Viewport height: window.innerHeight.
Position of menu's bottom edge: which is a combination of:
Button position relative to viewport (buttonEl.getBoundingClientRect().y)
Button height (buttonEl.getBoundingClientRect().height)
Menu height (menuEl.getBoundingClientRect().height)
Spacing between button and menu
Here's an example of how to implement the menu autoflipping behavior in React. Make sure to make the preview height short enough in order to see the autoflipping in action.


### Accessibility (a11y)

Dropdown menus must be usable with mouse, keyboard, and assistive tech. The subsections below cover each input modality and the ARIA semantics that tie them together.

Mouse Interactions
Clicking on the button toggles the display state of the menu. Clicking outside an open menu will close that menu. We have to ensure that clicks within the menu do not close it.

Button click / Enter / Space / ArrowDown

Item activated (Enter / click)

Escape pressed

Click outside

Trigger blur / focus lost


![Menu open and close lifecycle](dropdown-menu-pdf-assets/figures/menu-open-and-close-lifecycle.png)

Pseudocode for how to implement close-on-click-outside behavior based on the React hook useOnClickOutside:


function clickListener(event) {
// No-op if clicked element is button or a
// descendant of the menu.
if (
$buttonElement.contains(event.target) ||
$menuElement.contains(event.target)
) {
return;
}

closeMenu();
}

document.addEventListener('mousedown', clickListener);
document.addEventListener('touchstart', clickListener);
Remember to remove the clickListeners when the menu is closed.

Here's how to implement click-outside-to-dismiss behavior in React:


Focus management
When the menu is open, focus is trapped and pressing Tab shouldn't shift focus to another element. When the menu is closed, focus returns to the button.

Enter / Space / ArrowDown opens menu

ArrowUp opens menu

ArrowDown / ArrowUp

ArrowDown / ArrowUp

ArrowDown / ArrowUp (optionally wraps)

Escape / outside click / item activated

Escape

Escape

TriggerFocused

FirstItemFocused

LastItemFocused

ItemFocused


Focus transitions across the trigger and menu items
Always return focus to the trigger button when the menu closes
If focus is lost to <body> on close, keyboard and screen-reader users are dumped back to the top of the document with no context about what just happened. Restoring focus to the triggering button is a core part of the WAI-ARIA Menu Button pattern and is what makes Esc-to-dismiss feel correct instead of broken.

Keyboard interactions
There are two WAI-ARIA patterns to follow: the Menu Button pattern and the Menu pattern. The latter is needed after the menu is open.

Button

When the focus is on the button:

| Key | Description |
| --- | --- |
| Enter | Opens the menu and places focus on the first menu item. |
| --- | --- |
| Space | Opens the menu and places focus on the first menu item. |
| --- | --- |
| ArrowDown (Optional) | Opens the menu and moves focus to the first menu item. |
| --- | --- |
| ArrowUp (Optional) | Opens the menu and moves focus to the last menu item. |
Menu

When the focus is on the menu items:

| Key | Description |
| --- | --- |
| Enter | Activates the item and closes the menu. |
| --- | --- |
| Space (Optional) | Activates the item and closes the menu. |
| --- | --- |
| ArrowDown | Moves focus to the next item, optionally wrapping from the last to the first. |
| --- | --- |
| ArrowUp | Moves focus to the previous item, optionally wrapping from the first to the last. |
| --- | --- |
| Home | If arrow key wrapping is not supported, moves focus to the first item. |
| --- | --- |
| End | If arrow key wrapping is not supported, moves focus to the last item. |
| --- | --- |
| Esc | Closes the menu that contains focus and returns focus to the button. |
End-to-end, a keyboard-driven open, navigate, and activate interaction ties these two patterns together:

DropdownMenu.Item
DropdownMenu.List
DropdownMenu (context)
DropdownMenu.Button
User (keyboard)
DropdownMenu.Item
DropdownMenu.List
DropdownMenu (context)
DropdownMenu.Button
User (keyboard)
Focus trigger, press Enter / Space
setOpen(true), setActive(firstItemId)
Render list, move DOM focus to first item
Press ArrowDown / ArrowUp
setActive(nextItemId)
Focus next item
Press Enter
onClick(item), setOpen(false)
Return focus to trigger


Keyboard-driven open, navigate, and select flow
WAI-ARIA roles, states, and properties
Button
The element that opens the menu has role button.
The element with role button has aria-haspopup set to either menu or true.
When the menu is displayed, the element with role button has aria-expanded set to true. When the menu is hidden, it is recommended that aria-expanded not be present. If aria-expanded is specified when the menu is hidden, it is set to false.
The element that contains the menu items displayed by activating the button has role menu.
Optionally, the element with role button has a value specified for aria-controls that refers to the element with role menu.
Menu/Menuitems
The element serving as the menu has a role of menu.
The items contained in a menu are child elements of the containing menu and have the menuitem role.
When a menu item is disabled, aria-disabled is set to true.
An element with role menu either has:
aria-labelledby set to a value that refers to the menuitem or button that controls its display.
A label provided by aria-label.
A styled <div> dropdown without role="menu" and menuitem is not accessible
Screen readers rely on the button + aria-haspopup + aria-expanded trio on the trigger, and role="menu" with child role="menuitem" elements on the list, to announce the component as an actionable menu. Skipping these roles produces a visually working dropdown that is effectively invisible to assistive tech, which is the single most common accessibility failure for this component.

### Internationalization (i18n)

Since all user-facing strings are provided by the user, strings can be displayed as-is. However, do note that some strings can be long in certain languages, so overflowing text should either be truncated or wrapped. Text shouldn't overflow outside the menu/button.

For RTL languages, the menu button and contents have to be horizontally flipped. To achieve this, the menu component can take in a direction config option/prop and render its contents depending on the value.

Dropdown Menu Right-to-left example

## Summary

A production dropdown is defined by focus restoration, portal rendering, and WAI-ARIA Menu Button semantics. Each of those is invisible when the menu is open on a desktop mouse, and each of them breaks the component for keyboard and assistive-tech users the moment the dropdown sits inside a scrollable card or a modal.

Use a compound API backed by shared menu context. DropdownMenu, DropdownMenu.Button, DropdownMenu.List, and DropdownMenu.Item keep the trigger, floating surface, and items as distinct subcomponents, while a shared MenuContext carrying isOpen and activeItem gives hover, arrow keys, and activation a single source of truth instead of duplicated local state.

Render the list through a portal and make positioning viewport-aware. The list mounts into <body> to escape overflow: hidden clipping and z-index wars, autoflips near viewport edges, and falls back to a scrollable surface via maxHeight when the options cannot fit.

Own focus and ARIA behavior explicitly. Focus is trapped inside the open menu and restored to DropdownMenu.Button on close, Escape, or outside click. The trigger exposes role="button", aria-haspopup, and aria-expanded, and the list carries role="menu" with role="menuitem" children so screen readers announce the menu as a menu.

Start with the compound API plus MenuContext, then explain the portal, focus restoration, and ARIA Menu Button roles as the decisions that make the component reliable in production.

## References

The references below group dropdown menu sources by theme, so it is easier to map each source back to the part of the article it informed.

W3C ARIA authoring practices
This subsection anchors the dropdown design in the authoritative ARIA guidance for menu buttons and menus.

Menu button pattern | W3C ARIA for the trigger-side roles, states, and keyboard interactions for a menu button.
Menu pattern | W3C ARIA for the menu-side roles, focus order, and keyboard model.
Themed implementations
These libraries ship opinionated visual defaults and are useful when comparing component API ergonomics.

Dropdowns | Bootstrap v5.3 for a widely used CSS-first dropdown implementation and its markup conventions.
React Menu component | Material UI for a themed React menu with full keyboard and accessibility behavior.
Headless implementations
These libraries focus on behavior and accessibility without shipping styles, which is closer to what this article designs.

Dropdown menu | Radix UI for a headless React primitive with strong accessibility defaults.
Menu button | Reach UI for a minimal headless menu button implementation and its API shape.
Menu (dropdown) | Headless UI for a Tailwind-friendly headless dropdown with state and transition hooks.
Positioning and floating elements
This subsection covers the browser APIs and libraries used to position the menu popup relative to its trigger.

Popover API | MDN for the native browser primitive that handles top-layer rendering and light dismiss.
Floating UI for the flip, shift, and collision-aware positioning library commonly used for menus.
How to build a dropdown menu with JavaScript | freeCodeCamp for a from-scratch tutorial that covers wiring up toggle, outside click, and keyboard behavior.
Related system design articles
The articles below overlap with dropdown menus on popup positioning, focus management, and dismissal behavior.

Modal dialog for overlapping concerns around focus management, portals, and dismissal semantics.
Autocomplete for how a menu-like popup pairs with an input and the combobox pattern.