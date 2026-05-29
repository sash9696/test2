---
title: "Modal Dialog"
slug: "modal-dialog"
difficulty: "medium"
duration_mins: 30
tags:
  - system-design
  - accessibility
  - ui-component
premium: true
summary: "Design a modal/dialog component that shows a window overlaying the contents on the page."
free_preview: false
status: draft
---

# Modal Dialog

## Question

Design a modal/dialog component that shows content in a window overlaying the page.

Modal Dialog Example

## Real-life examples

Modal · Bootstrap v5.3
React Modal component - Material UI
Dialog — Radix UI
## Requirements exploration

Before designing the API, we scope the component by clarifying which parts it owns, how much the user can customize it, and which devices and content patterns it must support.

#### What are the components of the modal?

It's up to you to decide. At the basic level, it should allow customization of the contents. Whether to add built-in support for close buttons, titles, and footer actions will be up to you.

#### What amount of flexibility does the user have in customizing the design?

Users should be able to customize colors, fonts, padding, etc., of the modal elements to match their branding.

#### What devices will this component be used on?

All devices: mobile, tablet, desktop.

## Architecture / high-level design

Modals, like many components that reveal content, have a trigger element and content elements. Modals can be opened through user actions or background actions, so we should decouple the trigger source from the modal contents.

Basic modals that only include the contents don't have a complicated architecture. However, many modals from UI libraries have three distinct sections: header, body, and footer.

An example usage of modal components in React, with event handlers omitted:


<Modal>
<Modal.Header>My Modal Title</Modal.Header>
<Modal.Body>
<p>Modal body text goes here.</p>
</Modal.Body>
<Modal.Footer>
<button>Close</button>
<button>Confirm</button>
</Modal.Footer>
</Modal>
| Component | Role |
| --- | --- |
| Modal Root (Modal) | Root component, coordinates events between the inner components. |
| --- | --- |
| Modal Overlay | Component that renders the background overlay, usually dimming out the page contents. |
| --- | --- |
| Modal Header (Modal.Header) | The top section of the modal; contains the title and a close button. |
| --- | --- |
| Modal Body (Modal.Body) | The main contents of the modal. |
| --- | --- |
| Modal Footer (Modal.Footer) | Optional bottom section of the modal; usually contains close/submit buttons. |
In React, components can communicate with their parents using context or props. We choose to use context here since we are using a composition model and passing props is inconvenient. <Modal> should contain a context provider that provides the config options (<Modal>'s props) to all its child components in case they need them.

controls isOpen

renders via portal

provides config via context

provides config via context

provides config via context

Trigger button

Modal root

Portal mount node

Modal overlay

Modal contents

Modal.Header

Modal.Body

Modal.Footer


![Modal component hierarchy and context flow](modal-dialog-pdf-assets/figures/modal-component-hierarchy-and-context-flow.png)

## Data model

Note that for designing components, it might make sense to design the interface/API first, or both the data model and API concurrently. It depends on the component at hand. Feel free to jump between the two sections.

Modal components do not need much state. We'll build the modal as a controlled component, which is the usual approach taken by libraries. Hence, the open/closed state is managed outside the component.

| State | Type | Description |
| --- | --- | --- |
| previousFocusElement | HTMLElement | The DOM element in focus before the modal was shown. Read more on why this is needed in the Focus Management section. |
See below for configuration options which are also part of the data model.

Even though the persisted state is tiny, the modal moves through several observable phases during its lifecycle. Tracking them explicitly clarifies when side effects like the portal mount, focus trap, scroll lock, and background inertness should be set up and torn down.

isOpen = true

after mount + enter animation

Esc / overlay click / onClose

after exit animation


Opening


Closing

Capture previousFocusElement
Mount portal + overlay
Activate focus trap
Lock body scroll
Mark siblings aria-hidden

Release focus trap
Restore body scroll
Unhide siblings
Return focus to trigger


Modal lifecycle from closed to open and back
## Interface definition (API)

The API is split into props shared across subcomponents and props specific to each composition piece (root, overlay, header, body, footer).

General props
These props apply to most of the components.

| Prop | Type | Description |
| --- | --- | --- |
| children | React.Node | Children of the component. If using TypeScript/Flow, you can enforce specific components to be used as children. |
| --- | --- | --- |
| as | string | Component | In case there is a need to customize the underlying DOM element/component. |
| --- | --- | --- | --- |
| className | string | Classnames to add to components in case further visual customization is needed. May or may not be needed depending on theming approach. |
Modal
| Prop | Type | Description |
| --- | --- | --- |
| isOpen | boolean | Whether the modal is open or closed. |
| --- | --- | --- |
| onClose | Function | Callback to be triggered when the modal is closed, possibly from pressing the close button or hitting the "Escape" key. |
| --- | --- | --- |
| maxHeight | number | undefined | Max height of the modal. There should be a sensible default of around 80% of the viewport height. |
| --- | --- | --- | --- |
| width | number | undefined | Width of the modal. There should be a sensible default of 500-600px. |
Modal.Header
Basic version doesn't need props other than children.

Modal.Body
Basic version doesn't need props other than children.

Modal.Footer
Basic version doesn't need props other than children.

Customizing appearance
Guidance on designing good APIs for customizing UI components can be found in the Front End Interview Guidebook's UI Components API Design Principles Section.

Most of the modal's contents in the header/body/footer will be provided by the user; hence, there isn't that much default styling a modal component needs to provide.

## Optimizations and deep dive

The tricky parts of a modal are rendering outside the DOM hierarchy, focus management, accessibility, animations, localization, and stacked modal behavior:

Rendering: This section explains portals, stacking contexts, overlays, scroll locking, and the rendering constraints of floating UI.
Accessibility (a11y): This section covers focus trapping, keyboard behavior, screen-reader semantics, and ARIA modal guidance.
Animations and transitions: This section discusses entry and exit animations, independent overlay/content transitions, and reduced-motion concerns.
Internationalization (i18n): This section covers translated strings, text overflow, and direction-aware modal layouts.
Stacked modals: This section explains how multiple modal layers affect focus, dismissal, z-index, and background inertness.
Advanced topics: This section collects deeper modal concerns that may come up after the core design is covered.
Rendering
Modals float above the page, which creates two rendering challenges: escaping the parent's clipping and stacking contexts, and animating the modal's entry and exit smoothly.

Breaking out of DOM hierarchy
Rendering the modal is trickier than it seems due to the fact that modals are displayed over the page and do not follow the normal flow of page elements. It's important to render the modal outside of the DOM hierarchy of the parents, because if the parents contain styling that clips their contents, the modal contents might not be fully visible. Here's an example from the React docs demonstrating the issue.


In React, rendering outside the DOM hierarchy of the parent component can be achieved using React Portals. Common use cases of portals include tooltips, dropdown menus, and popovers.

Render the modal at the document root via a portal
Keeping the modal inside a parent subtree exposes it to ancestor overflow, transform, filter, and z-index stacking contexts, any of which can clip the modal or trap it behind other page chrome. Portaling into document.body (or a dedicated mount node) is the only reliable way to escape both the clipping and the stacking traps in one step.

Overlay
To help users focus on the content within the modal, there is usually an overlay/backdrop to dim out the page's contents. To render an element that covers the whole page, we can use the following CSS:


.modal-overlay {
/* Black color with some opacity. */
background-color: rgba(0, 0, 0, 0.7);
position: fixed;
top: 0;
left: 0;
right: 0;
bottom: 0;
}
Centered modal
To center the modal contents within the modal overlay, we can add the following styles to the .modal-overlay:


.modal-overlay {
/* Original styles are omitted. */
display: flex;
justify-content: center;
padding: 20px;
}
which will work with the following HTML structure.


<div className="modal-overlay">
<div className="modal-contents">...</div>
</div>
If vertical centering of the contents is desired, align-items: center can be added to .modal-overlay.

Here's a basic example of a modal with an overlay and optional vertical center mode:


Maximum height
Since the modal can contain a lot of contents, we can set a default maximum height for the modal such that the excess items will be scrollable within Modal.Body. This height can also be customized by specifying the maxHeight prop.

Scroll lock
When the modal is shown, the modal contents are in focus. To prevent the user from scrolling the background contents, the page should lock page-level scrolling. One way is to add overflow: hidden to the <body>.

Lock body scroll whenever the modal is open
Without a scroll lock, scroll events bubble to the underlying page and the background quietly shifts behind the dialog, especially disorienting on touch devices where overlay taps and scroll gestures overlap. Toggle overflow: hidden on <body> (or equivalent) on open and restore the previous value on close so nested or reopened modals do not leak their lock.

Rendering in HTML or JavaScript
The modal can either be:

Rendered into the HTML like Bootstrap's Modals. The modal is initially hidden from view via display: none / opacity: 0 / hidden attribute, and those styles are toggled when the modal is to be shown.
Rendered on the fly via JavaScript after the modal trigger button is activated.
The pro of rendering in the HTML first is better runtime performance due to fewer DOM operations needed to show the modal. The downside is that the HTML can be unnecessarily bloated, especially if the modal is never shown at all. Since modal contents usually contain secondary information, they shouldn't contribute towards SEO and don't need to be server-side rendered. The benefits of rendering the modal beforehand in HTML are relatively minor.

### Accessibility (a11y)

Modals must be usable with mouse, keyboard, and assistive tech. The subsections below cover each input modality and the ARIA semantics that tie them together.

Mouse interactions
Typically, clicking outside the modal (on the overlay) will close the modal. We have to ensure that clicks within the modal do not close the modal.


function clickListener(event) {
// No-op if clicked element is a
// descendant of the modal contents.
if ($modalContentsElement.contains(event.target)) {
return;
}

closeModal();
}

document.addEventListener('mousedown', clickListener);
document.addEventListener('touchstart', clickListener);
Remember to remove the clickListeners when the modal is closed.

Taken together with the keyboard and programmatic paths, the open and close flow touches the trigger, the portal, the focus trap, and the scroll lock in a specific order. The sequence below captures how these pieces fit together across a typical open-then-dismiss interaction.

Focus trap
Portal + overlay
Modal root (controller)
Trigger
Focus trap
Portal + overlay
Modal root (controller)
Trigger
Click / activate trigger
Set isOpen = true
Save previousFocusElement
Mount overlay + contents at document root
Activate trap, focus first element
Lock body scroll, mark siblings aria-hidden
Press Esc / click overlay / submit form
onClose()
Release trap
Unmount overlay + contents
Restore focus to previousFocusElement
Unlock body scroll, unhide siblings


Open and close flow across trigger, portal, and focus trap
Here's an example in React:


Focus management
The most complicated aspect of implementing a modal is probably focus management. Contents within modals are meant to be treated like a separate document; using the Tab key cycles through focusable elements within the dialog only, and focus can never be on elements outside of the component for as long as the modal is shown. This behavior/phenomenon is known as "focus trapping".

When a modal opens, focus moves to an element inside the modal. Generally, focus is set on the first focusable element, but there are some exceptions as mentioned in Dialog (Modal) pattern.

When a modal closes, focus returns to the element that opened the modal (unless the element no longer exists, in which case focus moves to another reasonable element).

How to implement focus management for modals:

When the modal is first opened, keep a reference to the element that opened the modal in the modal's state.
Focus on an element inside the modal.
Add keydown listeners for the Tab key that contain the following logic:
When the Tab key is pressed, determine if the focus should be shifted to the next or previous tabbable element by checking if the Shift key is also pressed (via the shiftKey value on the KeyboardEvent).
Find all tabbable elements within the modal.
From the currently focused element, find the next/previous tabbable element.
Focus on that new element.
When the modal is closed, hide the modal and move focus to the element that opened the modal.
The tricky part is the wraparound: the Tab handler has to detect the edges of the tabbable set and cycle focus back to the opposite end so it never escapes into background content. The flowchart below captures that decision for both Tab and Shift + Tab.

No

Yes

No

Yes

No

Yes

Yes

No

Keydown in modal

#### Key is Tab?


Let event propagate

Query all tabbable elements in modal

#### Shift held?


#### Focus on last tabbable?


preventDefault, focus first

Let browser advance focus

#### Focus on first tabbable?


preventDefault, focus last

Let browser move focus back


Focus trap wraparound for Tab and Shift+Tab
In practice, focus trapping can be done via focus-trap libraries. If using React, the react-focus-lock library is what Reach UI's Dialog component uses.

Always trap focus inside the modal and return it to the trigger on close
Skipping the focus trap lets Tab escape into the dimmed background, where keyboard and screen-reader users end up interacting with controls they cannot see. Failing to restore focus to the opening element on close is just as bad: it strands assistive tech at <body> and forces sighted keyboard users to tab back from the top of the page every time a dialog dismisses.

Keyboard interactions
The following is extracted from the Dialog (Modal) pattern:

| Key | Description |
| --- | --- |
| Tab | Moves focus to the next tabbable element inside the modal. If focus is on the last tabbable element inside the modal, moves focus to the first tabbable element inside the modal. |
| --- | --- |
| Shift + Tab | Moves focus to the previous tabbable element inside the modal. If focus is on the first tabbable element inside the modal, moves focus to the last tabbable element inside the modal. |
| --- | --- |
| Esc | Closes the modal. |
Focus trapping is essential for the required Tab-ing behavior; otherwise, the focus will "leak" out of the modal:

WAI-ARIA roles, states, and properties
The following is extracted from the Dialog (Modal) pattern.

The element that serves as the modal container has a role of dialog.
All elements required to operate the modal are descendants of the element that has role dialog.
The modal container element has aria-modal set to true.
The modal has either:
A value set for the aria-labelledby property that refers to a visible modal title.
A label specified by aria-label.
Optional aria-describedby property is set on the element with the dialog role to indicate which element or elements in the dialog contain content that describes the primary purpose or message of the dialog. Read the full guidance at Dialog (Modal) pattern.
role="dialog" alone is not enough; always pair it with aria-modal and a label
Screen readers need aria-modal="true" to treat the rest of the page as inert, and they need an aria-labelledby (pointing at the visible title) or aria-label to announce what the dialog is for. A <div> that sets role="dialog" without both pieces is effectively an unlabeled region that assistive tech cannot scope or describe.

<dialog> element
HTML now has a native <dialog> element that can be used to create modal dialogs, as it provides usability and accessibility features that must be replicated if using other elements for a similar purpose.

However, it is still relatively new and browser compatibility is not great. Moreover, behaviors like focus trapping still have to be manually implemented, which makes using a native <dialog> element less compelling.

<dialog> with showModal() handles inertness and Esc, but not focus trapping
When the element is opened via showModal() (not the open attribute), the browser adds the top layer, inertness of background content, and built-in Esc-to-close for free. Initial focus placement, cyclic Tab trapping, and returning focus to the trigger on close are still on you, so reach for <dialog> for its inertness benefits, not as a complete a11y solution.

Animations and transitions
If animation of the modal elements is desired and the transitions of the overlay and the contents need to be independent (e.g., the overlay fades in while the contents translate up vertically), the DOM structure will have to be changed and the contents cannot be rendered within the overlay's DOM. A structure similar to this will be required:


<div>
<!-- The overlay, rendered as a fixed sibling to the contents. -->
<div class="modal-overlay" aria-hidden="true"></div>
<!-- Full screen container to center the panel. -->
<div class="modal-contents-container">
<div class="modal-contents">...</div>
</div>
</div>
Here's an example of the entrance animation in React:


Exit transitions are non-trivial to implement properly in React because conditional rendering causes the DOM elements to be removed from the document when they are no longer needed on the page. Here's an example of a modal where the entrance and exit are both animated.


### Internationalization (i18n)

Since all user-facing strings are provided by the user, strings can be displayed as-is. However, do note that some strings can be long in certain languages, so overflowing text should either be truncated or wrapped. Text shouldn't overflow out of the modal elements. You usually don't want the modal title/footer to be more than a line long, so truncation is recommended here.

For RTL languages, the modal elements have to be horizontally flipped. To achieve this, the root modal component can take in a direction config option/prop and render the contents depending on the value.

Modal Dialog Right-to-left Example

Stacked modals
It is possible for modal contents to contain triggers that present even more modals, so the following needs to be considered:

Decide whether there should be an overlay per modal level, which will visually make the backdrop darker the higher the level of stacking.
Dismissing a modal via the Esc key or clicking outside the topmost modal's contents should only dismiss the topmost modal and not all the modals.
Dismissing a lower-layer modal should also dismiss all the modals on top of it (or make this behavior customizable).
Implementation-wise, a shared modal manager holds a stack of open modals, and only the topmost entry owns the active focus trap, the Esc handler, and the outside-click target. The layering below shows how overlays, contents, and focus responsibility fan out as deeper modals open.

Portal mount node

owns focus trap + Esc + outside-click

inert, focus trap paused

inert, focus trap paused

Page document

Body content - inert while any modal open

Overlay - level 1

Modal contents - level 1

Overlay - level 2, darker

Modal contents - level 2

Overlay - level 3, darkest

Modal contents - level 3 - top

Active modal


Stacking order and active focus trap across nested modals
Advanced topics
Tooltips and dropdown menus within a modal.
Alert dialog role and ARIA pattern.
## Summary

A modal is defined less by the box on the screen than by the three invisible systems wrapped around it. Portal rendering, focus management, and ARIA semantics sit behind a compound surface: Modal, Modal.Header, Modal.Body, and Modal.Footer coordinated through a root context, driven by isOpen and onClose, with maxHeight, width, as, and className for customization.

Render the modal through a portal. Mounting the dialog wherever the trigger lives exposes it to every ancestor overflow, transform, filter, and z-index stacking context, any of which can clip or bury it. Rendering through a React portal into document.body moves the overlay and contents to a neutral location where positioning is predictable, and an overflow: hidden scroll lock on <body> stops the page from drifting behind the dialog on touch. Lazy mounting keeps closed modals out of the initial HTML, and a stacked-modal manager lets only the topmost entry handle dismissal.

Trap focus inside the active modal. Opening the modal captures previousFocusElement, moves focus inside, and keeps Tab and Shift + Tab cycling within the tabbable set with edge wraparound so focus never escapes into hidden background content. Esc and overlay clicks both route through the same onClose path, and closing restores focus to the original trigger so keyboard users resume exactly where they left off. When modals stack, only the topmost owns the trap and the dismissal handlers.

Expose dialog semantics through ARIA and background inertness. The contents container carries role="dialog" and aria-modal="true", with aria-labelledby pointing at the visible title or aria-label as a fallback when the header is icon-only. Siblings of the portal container are marked inert where supported and aria-hidden elsewhere so assistive tech treats the rest of the page as unreachable while the dialog is open. RTL layouts flip the close affordance through logical properties, and long translations are absorbed by maxHeight-driven body scroll rather than clipped titles.

Start with the portal, focus trap, and ARIA semantics as the three concerns that make a modal reliable in production, and treat layout, animation, and theming as easier work.

## References

The references below group modal and dialog sources by theme, so it is easier to map each source back to the part of the article it informed.

W3C ARIA authoring practices
This subsection anchors the dialog design in the authoritative ARIA guidance for modal and alert dialogs.

Dialog (modal) pattern | W3C ARIA for the canonical roles, states, and keyboard behavior for modal dialogs.
Alert and message dialogs pattern | W3C ARIA for the stricter alertdialog variant used for urgent messages.
Focus management and the inert attribute
This subsection collects the platform primitives that enforce focus trapping and background inertness while a dialog is open.

<dialog>: The Dialog element | MDN for the native HTML element that handles top-layer rendering and modal semantics.
Inert attribute | MDN for removing background content from the tab order and accessibility tree while the dialog is open.
A CSS approach to trap focus inside of an element for a lightweight CSS-based technique for focus trapping.
createPortal | React for rendering the dialog outside its parent DOM hierarchy so it can escape stacking contexts.
Themed implementations
These libraries ship opinionated visual defaults and are useful when comparing component API ergonomics.

Modal | Bootstrap v5.3 for a widely used CSS-first modal implementation.
React Modal component | Material UI for a themed React modal with full keyboard and accessibility behavior.
Headless implementations
These libraries focus on behavior and accessibility without shipping styles, which is closer to what this article designs.

Dialog | Radix UI for a headless React dialog primitive with strong accessibility defaults.
Dialog (modal) | Reach UI for a minimal headless dialog implementation and its API shape.
Dialog (modal) | Headless UI for a Tailwind-friendly headless dialog with state and transition hooks.
Case studies
These writeups describe how product teams ship modals and dialogs in real applications.

Building a Dialog for Reddit Web for a production account of designing a reusable dialog system across a large web app.
Related system design articles
The articles below overlap with modal dialogs on portals, focus management, and dismissal behavior.

Dropdown menu for overlapping concerns around focus management, portals, and dismissal semantics in overlay components.