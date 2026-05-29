---
title: "Design / Drawing Tool (e.g. Figma, Canva, Excalidraw)"
slug: "design-drawing-tool"
difficulty: "hard"
duration_mins: 45
tags:
  - system-design
premium: true
summary: "Design a browser-based design and drawing tool with layers, transforms, and smooth canvas interactions."
free_preview: false
status: draft
---

# Design / Drawing Tool (e.g. Figma, Canva, Excalidraw)

Browser-based design and drawing tools are interaction-heavy editor applications. The design should explain how to model elements, render the canvas, and keep editing responsive as the document grows.

## Question

Design a desktop-first browser-based design and drawing tool similar to Figma, Canva, Excalidraw, or tldraw. Users should be able to create and edit a single document containing common elements such as rectangles, images, frames or groups, and text.

Focus on the front-end architecture for the active editor experience: element modeling, canvas rendering, selection, transforms, layering, undo/redo, keyboard shortcuts, and viewport navigation such as zooming and panning. Assume the initial version is primarily single-user and in-memory during an editing session. Persistence, exporting, offline support, and real-time collaboration can be discussed as extensions rather than the core design.

You do not need to go deep on template marketplaces, comments, version history, advanced vector path editing, animation timelines, or full raster painting engines unless the interviewer asks.

## Real-life examples

Figma
Canva
Excalidraw
FigJam
## Requirements

Support a desktop-first editing surface for a single active document.
Users should be able to create and edit basic elements such as shapes, images, frames or groups, and text.
Support single-select and multi-select, moving, resizing, rotating, deleting, copying, and layer ordering operations.
Support viewport interactions such as zooming and panning across a document that may contain hundreds to low thousands of elements.
Support undo and redo for recent editing operations.
The design should explain how document state, selection state, and viewport state are represented on the client.
Focus on the base editing architecture first. Persistence, exporting, offline behavior, and collaboration can be covered as follow-up extensions.
Cover RADI first before diving into optimizations

During interviews, there won't be enough time to cover all these points. Hence you should first cover the RADI in RADIO, then move on to the various topics in the "Optimizations and deep-dive" section, without going into too much detail first.

If the interviewer pauses on a certain topic and wants you to dive deeper, only then do you go into them. Aim to provide a well-rounded solution that covers the most important ground first, before building on top of it.

## Requirements exploration

Before choosing a rendering stack or data model, it helps to clarify the core editing capabilities, navigation behavior, and non-functional constraints the tool needs to support.

#### What design features should be supported?

Add basic elements: rectangles, images, frames
Enter and edit text
Ability to move, resize, and rotate elements after creation
Change elements' styling properties like stroke color, fill color, and stroke thickness
#### What additional operations should be supported?

Element selection, including single-select and multi-select
Multi-element updates and deletion of selected items
Ordering operations such as bring-to-front and send-to-back
Undo/redo of recent actions
Keyboard shortcuts for common operations
#### What navigation features should be supported?

Zooming in and out of the canvas
Panning the canvas view via mouse drag or scrollbars
#### Should the document be persisted?

For now we can simply talk about in-memory edits and focus on the core application functionality. If there is time, we can discuss various persistence options like saving to the file system and to the cloud.

#### What are the non-functional requirements?

Performance: interactive updates should be smooth, even with dozens to hundreds of shapes
Accessibility: desktop keyboard interactions should work, including keyboard shortcuts and focus management, though full screen reader support for arbitrary shapes on the drawing surface is out of scope
Creative tools are part of the "editor tools" family, and the architecture of editor tools in general shares many similar aspects: rendering, state/data model, events, and actions. Therefore we recommend checking out the Rich Text Editor question first if time permits.

Many creative tools these days allow advanced features like real-time collaboration, conflict resolution, and offline usage. For this writeup, we primarily focus on the design aspects of the apps. For a deeper dive into real-time collaboration, refer to the article on Google Docs collaborative editor.

Background
Creative tools on the web may appear similar at first glance. Users drag shapes, edit text, or move elements around. Under the hood, however, each category has significant differences and therefore faces different tradeoffs in terms of element rendering and interaction overlays.

Let us first look at the categories, the various rendering approaches, and which rendering approaches are better for each category.

Diagramming / whiteboarding: Apps for creating structured visuals like flowcharts, org charts, UML diagrams, and wireflows. They focus on connectors, nodes, and alignment grids.
Graphic design: A broad category of tools for creating marketing visuals, posters, typography, and brand assets. They mix vector, text, and image editing with templates and effects.
UI design / product design: Tools for designing digital interfaces like apps, websites, and products with features like frames, components, constraints, and prototyping. They emphasize layouts and hierarchical layers.
Raster painting and pixel editors: Tools focused on freeform drawing and pixel-level editing. They simulate brushes, painting effects, and detailed bitmap control.
Diagramming and whiteboarding tools
Examples: Draw.io, FigJam, Excalidraw, tldraw, Lucidchart, Miro

Diagramming and whiteboarding tools are optimized for structured visuals. Their primary concern is connecting nodes with edges, arranging elements on an infinite canvas, and letting users rearrange ideas quickly. The rendering layer must support large, zoomable workspaces with potentially hundreds or thousands of shapes while keeping interactions fluid.

The challenge lies in balancing simplicity of interaction, where shapes and connectors should "just work", with performance, since real-time redrawing of connectors, arrowheads, and text labels can become expensive. Because users expect diagrams to remain crisp when zoomed, vector rendering dominates, but naive use of SVG can choke on too many elements.

Many tools solve this by mixing retained vector graphics for interactivity with a canvas layer for performance-heavy redraws. Whiteboarding also emphasizes immediacy; latency in drawing freehand strokes or moving sticky notes undermines the collaborative and brainstorming flow.

Graphic design tools
Examples: Canva, Figma Buzz, Figma Draw, Adobe Illustrator

Graphic design tools sit between diagramming and UI design. They need to support a wide variety of media: vector shapes, typography, and raster images, often combined with templates, stickers, and filters.

Unlike diagramming apps, which focus on structure, or UI design apps, which focus on fidelity to product UIs, graphic design tools emphasize flexibility and creativity. The rendering layer must be versatile enough to handle both crisp vector art and bitmap manipulations without breaking flow.

Challenges here include managing resource-heavy assets such as large images and high-resolution exports, providing real-time previews of filters and effects, and scaling templates across different formats like social media posts, posters, and presentations.

UI design and product design tools
Examples: Figma Design, Framer, Sketch, Adobe XD, Penpot, Webflow

UI design and product design tools face a very different set of requirements. Instead of a loose collection of shapes, these tools must handle deeply hierarchical layers, component systems, and layout constraints that mimic how code-driven interfaces behave.

Rendering is more demanding. Users expect to navigate files with tens of thousands of objects, complete with nested frames, masks, auto-layout rules, and real-time previews of responsive designs. This pushes the limits of what the browser can handle with plain DOM or SVG. Tools like Figma therefore rely on GPU-accelerated rendering (WebGL) for compositing, caching layers, and applying effects like blur or blending at scale.

Another unique challenge here is text rendering. UI design depends heavily on typography fidelity, ligatures, and accurate alignment, which means these tools often need custom text layout engines rather than relying solely on browser text. In addition, interaction models like "component overrides" and prototyping introduce complexity into how changes propagate through a deeply nested design tree.

Strictly speaking, tools like Framer and Webflow are more for prototyping and building websites, but they can also be used for UI and product design because the two are closely related.

Raster painting and pixel editors
Examples: Adobe Photoshop, Photopea, Microsoft Paint

Unlike vector-based tools, raster painting and pixel editors deal directly with bitmaps and pixel data. The core of the app is a brush engine, which simulates strokes, pressure sensitivity, blending modes, and even smudging in real time. Performance is paramount here because painting involves rapid, continuous updates to large pixel buffers, often with layers stacked on top of one another. Rendering must therefore use GPU acceleration, typically WebGL or WebGPU, to apply brushes and filters at interactive speeds.

Memory management becomes a unique challenge. Large canvases with multiple layers consume huge amounts of memory, and undo/redo stacks must be implemented carefully, often by storing deltas or compressed tiles instead of full images. Fidelity is also critical; artists expect sub-pixel accuracy, color consistency, and advanced features like non-destructive editing, such as adjustment layers and masks.

Unlike diagramming or UI design tools, where vectors and layouts dominate, raster editors push the rendering layer into the domain of high-performance graphics akin to game engines.

Each category emphasizes different priorities: diagramming stresses scalable vector rendering and fluid connectors, UI design pushes toward hierarchical layers and text fidelity at massive scale, graphic design requires a hybrid engine that mixes media types while staying approachable for non-technical users, and raster painting depends on raw GPU performance and memory-efficient brush simulation.

The following table summarizes the main properties of each tool category and highlights what makes them distinct in terms of rendering requirements and design constraints:

| Category | Diagramming / whiteboarding | Graphic design | UI design / product design | Raster painting and pixel editors |
| --- | --- | --- | --- | --- |
| Number of elements | Medium - tens to hundreds for large boards | Low - typically less than a hundred for common designs | Thousands - especially if there are many pages and elements are duplicated across flows | Few - tens, but each element has many pixels |
| --- | --- | --- | --- | --- |
| Main element types | Mostly simple vectors - rectangles, circles, arrows, text | Mix of vector paths, styled text, and raster images | Frames (layout-aware), text, images | Pixel buffers (bitmaps), brushes, masks |
| --- | --- | --- | --- | --- |
| Layout depth | Shallow - flat list or simple grouping | Moderate - mostly groups | Deep - hierarchical element trees, similar to a webpage | Flat layers - stacked with masks/adjustments |
| --- | --- | --- | --- | --- |
| Unique requirements | Drag/drop nodes, connect edges | Transforming, layering, applying filters | Flexbox / autolayout | Continuous freehand strokes, filters, pixel edits |
Glossary
Scene graph: The logical document model for everything on the canvas, including elements, hierarchy, transforms, and stacking order, independent of how the scene is rendered.
Canvas: An immediate-mode raster rendering surface where the app draws pixels manually with JavaScript and must redraw the scene on every update.
SVG: A retained-mode vector graphics system where each shape is a node in the DOM tree, stays crisp at any zoom level, and can be styled and event-bound like regular HTML.
WebGL: A low-level GPU-accelerated rendering API exposed through the <canvas> element, used for high-performance 2D and 3D graphics when DOM, SVG, or Canvas 2D cannot keep up.
Clipping: Restricting rendering so child content is only visible within a container's bounds, commonly used for frames, masks, and cropped content.
Snapping: Automatically adjusting an element's position, size, or rotation when it gets close to a meaningful alignment target such as an edge, center line, or grid line.
Guides: Visual alignment indicators shown during movement or resizing to help users understand spacing, centering, and snap targets.
Autolayout: A rule-based layout system popularized by Figma that automatically positions and sizes child elements inside a container such as a frame. It is highly similar to CSS Flexbox.
Approach
We will look at the various common rendering approaches and then determine which are suitable for the needs and constraints of each category.

### Rendering approach

The right rendering approach depends on the category of creative tool being built, the expected scene complexity, and how much browser-native behavior the product wants to retain.

DOM (HTML elements)
Rendering via the DOM means constructing drawings directly with regular HTML elements styled using CSS. For example, rectangles can be represented as <div> elements, text labels as <span> elements, and images as <img> tags. This approach relies heavily on the browser's layout and painting engine, with positioning handled through CSS transforms or absolute coordinates.

DOM-based rendering is simple to implement and integrates seamlessly with the rest of the web page. It provides all the benefits of accessibility, event handling, and styling via CSS. However, because every element adds overhead to the layout tree, performance deteriorates as the number of elements grows. Once you exceed a few hundred shapes, interactions such as panning and zooming can become sluggish.

Advantages:

Easiest to implement and requires no custom rendering logic
Built-in hit-testing and event handling
Can leverage CSS layout such as Flex and Grid for hierarchical layout constraints
Full browser support for text, images, and accessibility out of the box
Resolution independent and scales smoothly at any zoom
Disadvantages:

Limited scalability; performance drops with many shapes
Shapes are constrained to what HTML and CSS can represent, which rules out effects and compositing that need custom pixel-level control
Fine-grained control over rendering is difficult
The browser's default styles will affect the elements. Tools typically get around this by putting the elements in an <iframe>
Zooming and panning are harder to implement, typically done using CSS transforms such as scale() and translate()
Selection elements such as selection boxes and resize/rotation knobs will be affected by zoom level if rendered within the same layer as the elements
The biggest advantage of the DOM approach is the built-in event handling and the ability to leverage CSS flex layouts for hierarchical UIs. Website builders like Webflow lean heavily on DOM-based rendering for their published output to stay as close to the actual usable result as possible, though the editing canvas itself may combine DOM with additional overlays for performance and interaction.

SVG (Scalable Vector Graphics)
SVG is a retained-mode graphics system where each vector element, such as a line, circle, path, or text node, is a node in the DOM tree. Instead of simulating visuals with generic HTML, SVG exposes drawing primitives as first-class elements.

Because SVG is vector-based, graphics remain crisp at any zoom level. Each node can be styled with CSS and responds to events, making it a good fit for diagramming tools where users select, resize, and connect shapes. That said, SVG performance begins to struggle once the document contains thousands of nodes, since every element adds to the DOM. Complex scenes with freehand strokes, heavy filters, or continuous updates can overwhelm the rendering engine.

Advantages:

Resolution independent and scales smoothly at any zoom
Declarative markup that is easy to read and manipulate
Individual shapes are addressable, enabling straightforward hit-testing and interactivity
Native export to .svg files
Disadvantages:

DOM overhead makes it slow with thousands of elements
Less suitable for freehand drawing or raster-like effects
Since the number of elements will often stay low enough to avoid severe performance issues, SVG is widely used in apps like draw.io and Lucidchart, where interactivity and clarity at scale matter more than raw performance.

Canvas (2D context)
Canvas provides an immediate-mode raster rendering surface. Developers draw into a bitmap using JavaScript operations such as ctx.fillRect() or ctx.arc(). Unlike SVG, the canvas does not remember objects once they are drawn; every frame or update must be explicitly redrawn.

This model makes canvas faster than SVG for large numbers of simple shapes because it avoids DOM overhead. It also opens up direct pixel manipulation, which is vital for painting and freehand drawing applications. The main challenge is that developers must manage their own scene graph, implement hit-testing, and redraw everything when the user pans or modifies the scene.

Advantages:

High performance for hundreds to thousands of shapes
Supports freehand drawing and pixel-level operations
Fine-grained control over rendering
Disadvantages:

Text rendering and editing are difficult
Flex layout does not exist
No retained mode; the app must reimplement its own scene graph, hit-testing, and per-element event dispatch, which can be a huge undertaking
The canvas backing store is a bitmap, so naive zooming via CSS transform looks blurry; sharp zoom requires redrawing the scene at the new scale
Canvas is often chosen for simple drawing apps and painting tools where immediate rendering speed is more important than retaining semantic structure.

WebGL (GPU-accelerated)
WebGL exposes a low-level API for rendering with the GPU. It can draw both 3D and 2D graphics with immense flexibility and performance, but requires developers to work with shaders, buffers, and GPU pipelines. Like Canvas 2D, WebGL is accessed through the <canvas> element via a different rendering context, but the APIs are otherwise unrelated and most 2D primitives have to be rebuilt on top of shaders and buffers.

The advantage of WebGL is that it scales to tens of thousands of objects while maintaining smooth performance. It can also apply advanced visual effects like blending, shadows, and blurs. This power makes it the rendering backbone of professional tools like Figma Design, where users expect responsiveness even with enormous design files.

The downside is complexity. Similar to the canvas approach, you must build your own rendering engine, scene graph, and interaction models on top of the GPU layer.

Advantages:

Superior performance compared to DOM, SVG, and Canvas
Flexible enough to render both 2D and 3D
Disadvantages:

Shares Canvas's limitations, since text rendering and editing are difficult and flex layout does not exist
Steep learning curve and verbose setup
Requires building or adopting an engine for scene management
Debugging is difficult compared to higher-level approaches
WebGL is the go-to choice for high-performance, professional-grade design applications, where responsiveness and scalability outweigh the cost of engineering complexity.

Choosing a rendering approach
Public implementation details for creative tools are uneven, and many mature products use hybrid renderers that evolve over time. So any product examples should be read as rough tendencies, not as audited statements about an exact current stack.

A few useful patterns still emerge:

Diagramming / whiteboarding: tools in this bucket often start with SVG or hybrid SVG + Canvas layers because connectors, selection outlines, and crisp zoomable shapes are easier to implement there. Whiteboards with heavier freehand drawing often add Canvas-backed paths or other rasterized layers as boards grow.
UI / product design: large design editors often move toward GPU-backed or hybrid renderers because deep hierarchies, clipping, effects, auto-layout, and text fidelity push beyond what plain DOM or SVG handle comfortably at scale. Figma's public engineering writeup is a good example of that tradeoff.
Graphic design: the space is broader. Simpler template editors can go far with DOM and SVG, while more advanced tools often adopt hybrid or GPU-backed compositing once effects, media mixing, and export fidelity become more important.
Raster painting and pixel editors: these tools need a GPU-friendly raster pipeline because brushes, blending, filters, and large bitmap layers are fundamentally pixel operations.
There is no universally best renderer even within one product category. The right choice depends on the expected scene complexity, how much browser-native behavior you want to keep, and how much rendering infrastructure the team is willing to build.

Canvas and WebGL remain the most powerful options when you need full control over compositing, caching, and drawing performance. The tradeoff is that you must rebuild more of the browser's behavior yourself, especially around text editing, layout, accessibility, and hit testing.

In interviews, it is useful to quickly outline the unique needs and constraints of the tool you need to design, based on the product category or any differing requirements, then describe the various rendering approaches and propose a suitable one.

A reasonable recommendation for each common category:

Diagramming / whiteboarding: Start with SVG or a hybrid SVG + Canvas approach. SVG makes connectors, arrows, selection outlines, and crisp zoomable shapes much easier to implement, while a canvas layer can help with freehand drawing or performance-heavy redraws on large boards.
UI / product design: Prefer WebGL or a hybrid renderer with a GPU-backed scene plus DOM overlays for text editing and inputs. These tools need to handle very deep hierarchies, clipping, effects, auto-layout, and thousands of elements, which pushes beyond what plain DOM or SVG can comfortably handle at scale.
Graphic design: A hybrid approach is usually the most pragmatic choice. Simpler tools can get far with DOM and SVG, especially when template editing and accessibility matter, while more advanced tools benefit from WebGL for compositing, effects, and higher export fidelity.
Raster painting and pixel editors: Use WebGL or another GPU-accelerated raster pipeline. Brush strokes, blending, filters, and large bitmap layers are fundamentally pixel operations and are too performance-sensitive for DOM or retained vector approaches.
Default to a hybrid rendering architecture
If the interviewer asks for just one concrete design, a good default is to propose a hybrid architecture: use a rendering engine optimized for the main content layer, then use a separate interaction or editing overlay for handles, selections, guides, and text editing where needed. That gives a good balance between implementation complexity, performance, and future extensibility.

Assumptions
Since interviewers often ask about many kinds of creative tools, this article does not assume a specific tool or rendering approach. Instead, it focuses on the common aspects across these tools, and there are many.

We will also assume the app only supports design within a single page. Most of the time, pages are just multiple canvases, and the proposed architecture, data models, and APIs can be extended to support apps with multiple pages.

## Architecture / high-level design

Complex client apps:

Have lots of interaction sources such as keyboard, mouse, panels, menu bar, and background sync
Need predictable state across many views at once
Require advanced debugging, undo/redo, and sometimes multiplayer
Are built by large teams where clear boundaries and conventions reduce chaos
![Flux-like architecture](design-drawing-tool-pdf-assets/figures/flux-like-architecture.png)

Flux/Redux-like architectures are common in complex client apps because they enforce structure and predictability in a place where complexity would otherwise spiral out of control. The architecture discussed here is heavily based on the Flux library by Meta.

Single source of truth
All application state lives in stores.

No more scattering state across random components, services, or globals
Any part of the app can subscribe to the store to render the current truth
Great for debugging because you always know where the state is
Predictable state changes
State is updated only through actions processed by reducers.

Makes mutations explicit and traceable; you can see exactly which action caused which change
Avoids the chaos of direct mutations from multiple places
Leads to deterministic behavior: the same sequence of actions always produces the same state
![Unidirectional data flow](design-drawing-tool-pdf-assets/figures/unidirectional-data-flow.png)

Data moves in one direction: Action -> Dispatcher -> Store -> View. Once commands are introduced later in the article, the full flow becomes Input -> Command -> Action -> Dispatcher -> Store -> View.

Views do not directly mutate state; they issue commands or dispatch actions through the architecture
This breaks circular dependencies and makes data flow easy to reason about
Testability
Reducers, or equivalent state-updating functions, are pure functions: (state, action) -> newState.

Easy to unit test without rendering UI
Each action can be tested in isolation
In creative tools, you can validate that MOVE_ELEMENT produces the correct new coordinates without spinning up the view layer
Time-travel debugging and replay
Because state is immutable and every update is triggered by an action:

You can record the sequence of actions and replay them to reproduce bugs
"Time-travel" dev tools let you step forward and backward through state changes
This is invaluable in apps with lots of complex user interactions such as drawing tools, form builders, and data dashboards
Easier to implement undo/redo, history, and collaboration
Since every committed edit flows through the same mutation pipeline, the app can derive history entries and inverse patches in one place
Undo/redo becomes a predictable matter of reverting committed transactions instead of chasing ad-hoc mutations across the UI
Collaboration can derive explicit operations from committed commands and feed remote changes through the same reducer pipeline as local edits
Importance of commands
In this writeup, actions are the internal mutation units that update store state. User-facing inputs such as Delete, Paste, or Align Left usually sit one layer above that as commands, which later resolve into one or more actions or patches. Keeping that split clear matters in creative tools because the command layer is where the benefits below actually accrue.

Multiple sources, one consistent result
The same result can come from very different sources:

A mouse drag on the canvas
A keyboard shortcut, such as pressing the up arrow to nudge position
A property panel change, such as typing x = 20 into a field
A plugin command, AI-assisted operation, or collaboration message
Instead of letting each source mutate application state directly in its own way, all of them can issue the same command such as MOVE_SELECTION. The command layer then resolves that intent into the same underlying actions or patches (for example, a MOVE_ELEMENT action per affected element), so state updates stay consistent no matter where the request originated.

Decoupling interaction logic from state update logic
Source: Tools, UI panels, shortcuts, plugins, and collaboration code. They only know what the user or system wants.
Result: New document state, owned by the command layer, dispatcher, and store.
This decoupling means the UI can evolve, such as adding new gestures or shortcut keys, by firing commands from those places without having to touch the state logic, and vice versa.

Replayability and determinism
When the source is abstracted away, you can log accepted commands locally and derive explicit operations from the committed result:

Replay them later to get the same result
Sync them to other clients for real-time collaboration
The actual pointer moves or key presses are not needed once you have the committed intent; the state result is deterministic.

Clean mental model and improved maintainability
Developers do not have to care whether a resize came from a drag handle, keyboard shortcut, plugin, or remote collaborator. They only need to know that the same RESIZE_SELECTION command eventually produced the expected state updates. That is cleaner, easier to test, and more robust than direct mutations sprinkled across the codebase.

In essence, commands sit between the raw input source and the underlying actions that mutate document state.

Component responsibilities
Design Drawing Architecture

This diagram shows the high-level architecture of a creative tool/editor like a drawing or design app. Each box represents a subsystem, and the arrows show how data, commands, and actions flow. The overall system is heavily inspired by Flux architecture.

Let's proceed to break down each component's responsibilities.

View
The View is what the user sees and interacts with. It has two main parts:

Workspace UI
Toolbar: buttons to choose tools such as select, rectangle, text, and brush, or trigger commands such as undo, zoom, and export
Layers panel: shows the hierarchy or stacking order of objects and lets the user reorder, lock, or toggle visibility
Properties panel: displays and edits properties of the currently selected element, such as color, stroke width, and font
Canvas
Interaction overlay: renders guides, selection boxes, transformation handles, and similar ephemeral UI. It handles hit-testing and user gestures like clicking, dragging, and resizing, and can interpret pointer/keyboard events in terms of editor semantics
Elements: actually draws the shapes, images, and overlays to the screen using the selected rendering approach such as SVG, Canvas, or WebGL
Views do not update themselves directly; they listen to state changes and issue commands based on user input.

Commands and actions
Commands represent user-level intents such as DELETE_SELECTION, PASTE, or ALIGN_LEFT. Command handlers resolve those intents into one or more actions or patches, which are the low-level mutation units that actually update the store.

Commands can originate from various places:

Workspace UI: toolbar buttons, inspector fields, menus, and context menus
Keyboard shortcuts: copy, paste, delete, nudge, and similar commands
Canvas interactions: drag, resize, rotate, marquee select, and direct-manipulation gestures
Plugin, AI, or collaboration sources: external or remote requests that should reuse the same command path
Background events: autosave timers, asset-loaded notifications, and similar system-triggered events
These are not the only ways to fire commands. The important point is that every meaningful edit eventually funnels into the same set of internal mutation units. Encapsulating mutation operations as actions or patches keeps reducer logic reusable across the application.

Actions are the only way to mutate state, which is an important architectural decision for implementing undo/redo, plugins, and real-time collaboration.

Dispatcher
The Dispatcher is the central hub that receives actions from the command layer and other internal systems. Its job is to:

Validate the action
Pre-process the action, which is a good place to add middleware logic such as logging
Send it to the Store to update state
This enforces a unidirectional flow: Input -> Command -> Action -> Dispatcher -> Store -> Updated view.

Store
The Store is the single source of truth for application state, similar to a Flux/Redux store. It holds the elements model, selection, undo/redo stacks, and more.

Once an action is applied, the Store notifies its subscribers, and the View re-renders the UI panels and canvas to reflect changes such as a moved rectangle or a newly applied style.

Keyboard events (shortcuts) and keybinding system
This subsystem captures keystrokes like Ctrl+C, Delete, or arrow keys for nudging and maps them to commands. See Command system for the full intent-normalization flow.

Background events
Background events are non-user-triggered events that affect the document or editor, such as autosave timers, collaboration updates, or system signals like window resize and asset-loaded notifications. These can also generate commands or internal actions.

## Summary

The View shows the document and captures input
User interactions, keyboard events, plugins, collaboration code, and background events trigger commands or actions
Command handlers resolve intents into actions, and the Dispatcher funnels those actions into the Store
The Store pushes the new state back into the View for re-rendering
## Data model

The main aspects of the data model are:

Elements: Scene graph nodes placed on the page, such as frames, text layers, rectangles, and images, with stable IDs, hierarchy, transforms, styling, and z-order
Canvas state: Ephemeral app state that tracks viewport, selection, active tool, focus target, and in-progress gestures
Elements
There are many different element types in mature creative tools. For example, Figma's Plugin API exposes dozens of node types. We will only cover core types such as FRAME, TEXT, RECTANGLE, and IMAGE because they are the most common elements across creative tools. Elements share a small common base for identity, visibility, transforms, and stacking; each type adds its own payload.

Base element
| Name | Type | Description |
| --- | --- | --- |
| id | string | Stable identity referenced by selection, history, and constraints. This can be a globally increasing integer or a UUID. UUIDs are useful for real-time collaboration, where multiple users may generate new elements concurrently. |
| --- | --- | --- |
| type | "FRAME", "TEXT", "RECTANGLE", "IMAGE" | Discriminator for the concrete element kind. |
| --- | --- | --- |
| name? | string | Optional label used for layers panel display and search. |
| --- | --- | --- |
| parentId | string | null | Parent container ID. null means the element lives at the page root. |
| --- | --- | --- | --- |
| x | number | Horizontal position in the element's parent-local coordinate space. Root elements use page coordinates. |
| --- | --- | --- |
| y | number | Vertical position in the element's parent-local coordinate space. Root elements use page coordinates. |
| --- | --- | --- |
| width | number | Width in the element's local coordinate space. |
| --- | --- | --- |
| height | number | Height in the element's local coordinate space. |
| --- | --- | --- |
| rotation | number | Rotation angle in degrees. |
| --- | --- | --- |
| opacity | number | Opacity in the range 0..1, where 0 is fully transparent and 1 is fully opaque. |
| --- | --- | --- |
| visible | boolean | Whether the element should currently be rendered on the canvas. |
| --- | --- | --- |
| locked | boolean | Whether the element is protected from normal selection and editing interactions. |
A minimal TypeScript shape:


```typescript
type ElementId = string;
type ElementType = 'FRAME' | 'TEXT' | 'RECTANGLE' | 'IMAGE';
type SceneElement = FrameElement | TextElement | RectangleElement | ImageElement;

interface BaseElement {
```
id: ElementId;
type: ElementType;
name?: string;
parentId: ElementId | null;
x: number;
y: number;
width: number;
height: number;
rotation: number;
opacity: number;
visible: boolean;
locked: boolean;
}
The distinction between local and world coordinates matters. A child inside a frame stores x, y, width, height, and rotation relative to that frame, not as absolute page coordinates. Rendering, hit testing, snapping, and selection overlays often need world-space bounds, which can be derived by walking the parentId chain and composing ancestor transforms. Reparenting is therefore not just "move this ID to another list"; the editor usually also converts the element's transform from the old parent space into the new parent space so it stays visually stationary unless the command explicitly says otherwise.

We keep both a map and a doubly-linked list:


interface Page {
elements: DoublyLinkedList<ElementId>; // root-level z-order (frontmost last)
elementsMap: Record<ElementId, SceneElement>; // O(1) lookup
}
This hybrid structure is efficient: re-ordering operations are trivial on doubly-linked lists, while random access and updates are O(1) via the map.

elements stores only root-level order, not nested children
elements stores only the root-level order for the page. Nested hierarchy lives on container elements such as FRAME.children, so the full scene graph is still a tree even though each sibling list is linear.

The choice of data structure here is similar to the Rich Text Editor, which goes deeper into alternative representations and explains their pros and cons.

Rectangle element
A rectangle is a basic vector shape with optional rounded corners. It only needs to store the x/y/width/height/rotation instead of the coordinates of the 4 corners because those can be computed during rendering.


interface RectangleElement extends BaseElement {
type: 'RECTANGLE';
borderRadius?: number | { tl: number; tr: number; br: number; bl: number };
stroke?: {
width: number;
dash?: number[];
join?: 'miter' | 'round' | 'bevel';
cap?: 'butt' | 'round' | 'square';
};
fill?: string;
}
Text element
Text consists of content plus ranges with style references. It contains text content and format ranges.


interface TextElement extends BaseElement {
type: 'TEXT';
text: string;
ranges: Array<{
start: number;
end: number;
style: 'bold' | 'italic' | 'underline';
}>;
align: 'start' | 'center' | 'end';
lineHeight: number | 'auto';
overflow: 'clip' | 'ellipsis';
}
Image element
An image references an asset by ID and a placement rectangle. Fit and crop are purely logical; renderers resolve them.


interface ImageElement extends BaseElement {
type: 'IMAGE';
assetId: string; // Stable reference into the asset store / blob cache
fit: 'fill' | 'fit' | 'crop' | 'tile';
crop?: { x: number; y: number; w: number; h: number }; // in image pixels
borderRadius?: number | { tl: number; tr: number; br: number; bl: number };
}
Most of the time, the actual image bytes are uploaded to a separate file storage service and stored outside the document itself. The document keeps only an assetId reference, while an asset table, CDN URL resolver, or local blob cache maps that ID to actual bytes. If offline editing is required, the same assetId can point to a locally cached blob in IndexedDB until connectivity returns and the asset is uploaded.

Frame element
A frame is a container used for grouping and optional clipping. It can host children but still behaves like an element. It is highly similar to flexbox in CSS.


interface FrameElement extends BaseElement {
type: 'FRAME';
clipContent?: boolean; // acts as a mask for children
children: DoublyLinkedList<ElementId>; // ordered child IDs
direction: 'row' | 'column';
spacing: number;
padding: number | { top: number; right: number; bottom: number; left: number };
sizing: { width: 'hug' | 'fixed' | 'fill'; height: 'hug' | 'fixed' | 'fill' };
alignMain: 'start' | 'center' | 'end' | 'space-between';
alignCross: 'start' | 'center' | 'end' | 'stretch';
}
The page keeps one ordered list for root elements, and each frame keeps a separate ordered list for its own children. That split lets the scene graph stay hierarchical without giving up efficient sibling reordering.

Frames are primarily used in creative tools where hierarchy matters and the layout of elements affects one another. Drawing and diagram tools usually do not need frame elements.

Elements are represented in a rendering-agnostic fashion, so no SVG, Canvas, or WebGL specifics leak into the model. That makes it easier to swap rendering approaches and serialize the model into JSON for persistence.

elements[]


children[]

assetId


parentId


clipContent


assetId


Scene graph entity relationships
Canvas state
Selection is app state (temporary / ephemeral), not document state. It should live alongside viewport state, tool mode, focus state, and in-progress interaction state, and it should support multi-select naturally.


interface CanvasState {
viewport: ViewportState;
selection: SelectionState;
tool: ToolState;
focus: FocusState;
interaction: InteractionState | null;
}

interface ViewportState {
zoom: number;
x: number;
y: number;
}

```typescript
type ToolMode = 'select' | 'hand' | 'text' | 'rectangle' | 'image';

interface ToolState {
```
mode: ToolMode;
}

interface SelectionState {
ids: ElementId[]; // ordered, first is "primary" for handles/inspector
anchor?: ElementId; // last clicked element ID, for shift-range operations
}

```typescript
type FocusState =
| { kind: 'canvas' }
|  |
| {
```
kind: 'text';
elementId: ElementId;
range: { start: number; end: number };
composing: boolean; // IME / composition in progress
};

```typescript
type ResizeHandle = 'n' | 'ne' | 'e' | 'se' | 's' | 'sw' | 'w' | 'nw';

type InteractionState =
```
| { kind: 'marquee'; start: Vec2; current: Vec2 }
|  |
| {
kind: 'drag-selection';
startPointer: Vec2;
originalPositions: Record<ElementId, { x: number; y: number }>;
}
| {
kind: 'resize-selection';
handle: ResizeHandle;
startPointer: Vec2;
draftBounds: Rect;
}
| {
kind: 'rotate-selection';
pivot: Vec2;
startPointer: Vec2;
startAngle: number;
}
| { kind: 'pan'; startPointer: Vec2; startViewport: { x: number; y: number } };
}
This separation is important because many editor interactions have a preview phase before commit. While the user is dragging, resizing, rotating, or typing into a text box, the app needs temporary state such as the active handle, draft geometry, text caret range, or IME composition state. These are real editor states, but they should not be serialized into the document or treated as durable history entries until the interaction is committed.

Allowing multiple selection, and why it matters
Real editing workflows require transforming several elements at once, including move, align, distribute, and batch style changes. Restricting selection to a single element forces users into tedious, repetitive steps and complicates features like alignment or grouping into a frame.

Multi-select enables:

Group transforms: Transforming all selected items with shared resize/rotate handles within a bounding box
Bulk operations: Delete, duplicate, align, distribute, and set common fill/stroke/text styles
Nudging: Keyboard nudging and snapping that respect the combined bounds rather than just one element
Decoupling from the document
Selection should not be serialized with the document. It is ephemeral, user-session state. This separation keeps saved files clean and prevents merge conflicts if collaboration is added later. It also makes undo/redo simpler: history concerns document mutations, not focus or selection. That's not a hard rule though; Figma does track certain selection actions in the undo/redo stack.

Keep selection, viewport, and focus out of the durable document
Session state such as selection, viewport, active tool, and in-progress gestures should never flow into the serialized document. Mixing it in pollutes saved files, creates meaningless merge conflicts once collaboration is added, and forces every undo entry to carry noise that has nothing to do with the document itself. Keep one model for the document and a separate CanvasState for the current session.

Using plain objects vs. classes for elements
Elements in a creative tool can be represented either as plain data objects (simple JSON-like structures) or as classes (objects with both data and methods).

Plain objects
Advantages:

Easy to serialize and deserialize to JSON, which is critical for persistence, collaboration, and undo/redo
Rendering-agnostic and portable, and work well with Redux-like immutable stores
Simple to diff, patch, and clone
Keeps the model declarative: elements describe what they are, not how they behave
Disadvantages:

No built-in behavior; all logic such as hit-testing, transforms, and rendering must live in separate utility functions or systems
Can become verbose if many helper functions are required
Classes
Advantages:

Encapsulate both data and behavior, such as element.hitTest(point) or element.render(ctx)
Can enforce invariants through constructors and methods
More familiar to developers coming from OOP backgrounds
Disadvantages:

More brittle for undo/redo and collaboration, since behavior is embedded with state
Can blur concerns if rendering logic leaks into the data model
Default to plain objects for element models in interviews
In interviews, it is usually fine to go with plain objects. When the app becomes complicated and logic needs to be shared across elements, class instances can become a better choice due to inheritance.

Note that Rich text editor nodes in editors like Lexical are class instances.

Rendering-agnostic and serializable
No element stores renderer-specific constructs such as SVG path strings, Canvas image data, or WebGL buffers. Everything is logical geometry plus transforms and style references.

That gives several benefits:

Renderers can be swapped more easily: SVG backends can generate <rect> and <text>, while Canvas and WebGL can tessellate quads or glyphs from the same model
The whole page serializes cleanly as compact JSON
Assets can stay as separate blobs referenced by URL or asset ID from within the document
## Interface definition (API)

For a drawing tool, the most important API is not a broad HTTP surface. It is a structured editor API shared by the canvas, inspector, keyboard shortcuts, menus, clipboard flows, plugins, and any later persistence or collaboration adapters. This follows the same general pattern used in the Rich Text Editor, Collaborative Spreadsheet (Google Sheets), and Collaborative Editor (Google Docs) solutions: keep one local operation layer inside the editor, then let other systems consume that boundary instead of mutating UI state directly.

Core API shape
A useful mental model is a DrawingEditor facade. UI surfaces talk to this facade to read state, dispatch commands, and subscribe to committed updates. Internally it can wrap a Redux-like store, dispatcher, and command registry, but the surface exposed to the rest of the product should stay compact and stable.


```typescript
type CommandId =
| 'MOVE_SELECTION'
|  |
| 'GROUP_SELECTION'
|  |
| 'SET_FILL'
|  |
| 'INSERT_IMAGE'
|  |
| 'ZOOM_TO_FIT'
|  |
| 'UNDO'
```
| 'REDO';

interface DrawingDocument {
page: Page;
}

interface EditorSnapshot {
document: DrawingDocument;
canvas: CanvasState;
}

interface DrawingEditor {
getDocument(): DrawingDocument;
getCurrentPage(): Page;
getElementById(id: ElementId): SceneElement | null;
getCanvasState(): CanvasState;
getSelection(): SelectionState;
getViewport(): ViewportState;
canExecute(commandId: CommandId): boolean;

dispatchCommand(commandId: CommandId, payload?: unknown): void;
runTransaction(label: string, fn: () => void): void;

serializeDocument(): string;
loadDocument(serialized: string): void;
subscribe(listener: (snapshot: EditorSnapshot) => void): () => void;
registerCommand?(commandId: CommandId, handler: (payload: unknown) => void): () => void;
}
The exact names are not critical in an interview. What matters is that every editor surface shares one facade for reads, one command boundary for writes, and one subscription boundary for observing committed changes.

Initialization APIs
Initialization should look more like constructing an editor instance than like calling a backend endpoint. Similar to the Rich Text Editor article, the editor should accept the minimum state and configuration needed to mount the active editing surface.

| Name | Type | Description |
| --- | --- | --- |
| rootElement | HTMLElement | Mount point for the active editor surface and overlays. |
| --- | --- | --- |
| initialDocument | DrawingDocument | Initial scene graph for the current file or page. |
| --- | --- | --- |
| initialViewport | ViewportState | Initial zoom and pan state for the canvas. |
| --- | --- | --- |
| readonly | boolean | Whether mutating commands should be blocked at startup. |
| --- | --- | --- |
| commandRegistry? | CommandDescriptor[] | Optional command metadata for menus, shortcuts, and command palette integration. |
| --- | --- | --- |
| listeners? | { onUpdate?; onError? } | Optional update and error hooks used by surrounding app code. |
The rendering approach itself should stay internal to the editor implementation. Whether the scene is drawn with DOM, SVG, Canvas, or WebGL should not leak into the API contract because the rest of the product should only care about editor state and editor intents.

Querying APIs
Read-side APIs should expose the logical document and editor state, not renderer-specific objects. These can be methods on the editor facade or selectors in a Redux-like architecture.

getDocument(): Returns the current document model for persistence, export, or dev tooling.
getCurrentPage(): Returns the active page or canvas state being edited.
getElementById(id): Looks up one element by stable ID, typically through the elementsMap.
getCanvasState(): Returns viewport, selection, active tool, focus target, and any in-progress gesture state.
getSelection(): Returns the current committed selection state, including selected IDs and primary selection.
getViewport(): Returns current zoom and pan values.
canExecute(commandId): Lets menus, toolbars, and keyboard handlers ask whether a command is currently valid.
This split is useful because view code can stay declarative. The inspector can read getSelection(), the canvas overlay can read getCanvasState().interaction, the minimap can read getViewport(), and menus can check canExecute('GROUP_SELECTION') without duplicating business logic.

Mutation and command APIs
This is the core of the editor API. Mutations should enter through commands, not through raw direct state edits scattered across the UI. Commands represent user-level intents such as move, group, style, paste, or zoom. Internally, command handlers can resolve those intents into lower-level actions plus explicit document operations or patches, and transactions batch those internal updates into one logical edit for history and analytics.

For most editor surfaces, dispatchCommand(commandId, payload) is the primary write API. Keyboard shortcuts, toolbar buttons, inspector fields, context menus, clipboard flows, and plugins should all use the same command entry point.

runTransaction(label, fn) is useful for editor-owned flows that need to batch several low-level mutations into one commit, such as paste, import, or alignment operations that touch many elements at once. Thin helpers like undo() and redo() can exist, but they should just delegate to the same command layer instead of becoming a second mutation system.


editor.dispatchCommand('MOVE_SELECTION', { dx: 24, dy: 0 });
editor.dispatchCommand('SET_FILL', { color: '#2563eb' });
editor.dispatchCommand('GROUP_SELECTION');

editor.runTransaction('Paste imported image', () => {
editor.dispatchCommand('INSERT_IMAGE', {
assetId: 'asset_42',
x: 120,
y: 80,
});
editor.dispatchCommand('ZOOM_TO_FIT');
});
One important distinction: commands are local editor intents, not the persistence or wire format. A command like MOVE_SELECTION depends on ephemeral local state such as the current selection and focus target, so it should not be logged or sent over the network directly. After command handling, the editor should produce explicit operations with stable IDs and concrete payloads, such as:


```typescript
type DocumentOperation =
```
| { type: 'MOVE_ELEMENTS'; elementIds: ElementId[]; dx: number; dy: number }
|  |
| { type: 'SET_FILL'; elementIds: ElementId[]; color: string }
|  |
| { type: 'INSERT_ELEMENT'; parentId: ElementId | null; index: number; element: SceneElement }
| --- |
| { type: 'SET_TEXT_CONTENT'; elementId: ElementId; text: string; ranges: TextElement['ranges'] };
Those explicit operations are what history, autosave, offline replay, and collaboration should record or transmit. The later Command system section goes deeper on registry design, coalescing, and context-aware enablement. The API surface here is intentionally simpler: expose one stable way to express intent, and keep the reducer/transaction machinery behind that boundary.

Serialization and subscription APIs
Like the Rich Text Editor solution, the editor should expose explicit serialization and subscription hooks rather than forcing external code to reach into internal stores.

serializeDocument(): Returns a durable representation of the scene graph, usually JSON or a JSON-derived string.
loadDocument(serialized): Replaces the current document state from a serialized representation.
subscribe(listener): Registers a listener for committed editor updates and returns an unsubscribe function.
registerCommand(commandId, handler): Optional extension hook for editor-owned surfaces such as menus, palettes, or import/export adapters. This is an editor extension point, not a full plugin SDK.
These APIs form the boundary that persistence, export, offline recovery, and collaboration code can consume later. That is the same architectural direction as the Collaborative Editor (Google Docs) and Collaborative Spreadsheet (Google Sheets) articles: local editor operations happen first, then sync layers observe or serialize those committed changes instead of inventing a second mutation path.

In the first version of a drawing tool, this entire API can stay local to the browser tab. If cloud persistence, offline replay, or real-time collaboration are added later, the same command and transaction stream becomes the sync boundary. That keeps the architecture consistent and avoids splitting the product into one mutation path for local editing and another for synchronized editing.

## Optimizations and deep dive

The architecture so far covers the main building blocks of a modern creative tool. This section groups the harder implementation details into the areas most likely to come up in follow-up discussion:

Rendering and interaction internals: This section explains how the editor draws the scene, layers interaction UI, hit tests objects, and keeps direct manipulation responsive.
Editing model and command flow: This section covers commands, transactions, selection state, history, undo/redo, readonly behavior, and predictable document mutations.
Persistence, sync, and extensibility: This section discusses local persistence, autosave, collaboration, conflict handling, export, import, and plugin boundaries.
Accessibility: This section explains what partial accessibility support means for Canvas and WebGL-heavy creative tools.
This section is deeper than the minimum interview answer
The optimizations section is long, but it can go even deeper if we really want to delve into implementation details. We drew the line at the depth you would possibly have to go in an interview setting. Aim to provide a well-rounded solution that covers the most important ground first, before building on top of it.

Rendering and interaction internals
This group covers how the editor draws the scene, layers interaction UI on top, and keeps direct manipulation responsive as files grow.

Canvas rendering
This section covers some of the nuances and complexities of using HTML Canvas as the rendering approach. It assumes some familiarity with the canvas drawing paradigm and the basic HTML Canvas APIs.

New to HTML Canvas? Start with the appendix primer
If you're not familiar with HTML Canvas, check out the Appendix section for a brief introduction to HTML Canvas APIs.

Flex layout / autolayout
Figma's autolayout is a dynamic layout system that automatically arranges and resizes design elements based on predefined rules, similar to CSS Flexbox. When applied to a frame, it becomes a smart container that adjusts spacing, alignment, and sizing of child elements automatically. For example, a button will resize when you change its text while maintaining consistent padding.

This feature eliminates manual positioning work and makes designs more responsive and maintainable. Elements can be set to fill space, hug content, or stay at a fixed size, while the container handles spacing automatically. It is particularly useful for building components like navigation bars or cards that need to adapt when content changes, making designs behave more like actual code implementations.

However, HTML Canvas provides only drawing APIs. How does Figma achieve autolayout in canvas? The answer is that Figma painstakingly reimplements a flex layout algorithm:

Layout engine: Figma implements a custom layout engine (in C++ compiled to WebAssembly) similar to CSS Flexbox. This engine calculates positions, sizes, and spacing for elements marked as autolayout containers.
Virtual layout tree: Figma maintains a virtual representation of the layout hierarchy separate from the visual rendering tree. This allows layout calculations without directly manipulating canvas elements.
React Native's Yoga engine uses the same layout approach
Another popular technology that uses a similar approach is React Native. React Native uses the Yoga layout engine to convert flexbox elements into absolute coordinates before painting them to a native app's screen. It is worth looking at React Native and Yoga side by side to see how that style of layout engine works.

Text rendering and editing
The difficulty does not end there. Canvas provides only basic text rendering with the fillText() and strokeText() methods. Many common text features such as text decoration, text shadows, or complex text effects require manual implementation. Advanced typography features like ligatures, kerning pairs, or OpenType features are not directly accessible.

Canvas has no built-in text layout engine, so the app must manually handle:

Line breaking and wrapping by detecting word boundaries and splitting text across lines
Text measurement for correct positioning and bounds
Multi-line alignment for left, center, right, and justified text
Baseline alignment across different font sizes and families
Figma is a useful case study in how far this problem can go. In Building a professional design tool on the web, Figma describes using a WebGL-backed renderer and a C++ codebase compiled with Emscripten so it could control rendering quality, memory layout, and cross-browser consistency more directly. The broader lesson is that once a creative tool needs high-fidelity, browser-independent text behavior at scale, text layout stops being a small widget problem and starts to become part of the rendering engine itself.

That level of investment is unusual. Most teams should not assume they need to build a Figma-like text stack from day one.

A practical compromise is hybrid text editing. In display mode, text is rendered into Canvas or WebGL like the rest of the scene. When the user double-clicks a text box or enters edit mode, the app temporarily places a real DOM text surface such as a textarea, input, or contenteditable element on top of the canvas at the exact same position, size, and transform. The browser then handles caret movement, selection, IME composition, copy/paste, spellcheck, accessibility, and other difficult text-editing behaviors. Once editing is complete, the app commits the updated text back into the document model, removes the DOM overlay, and resumes rendering the text through the graphics engine. This is often a good tradeoff between engineering effort and performance because it avoids building a full text editor inside Canvas or WebGL from day one while still preserving a fast scene renderer for the rest of the app.

Pretext is a new text measurement and layout library that can also be useful in this setup. It is not a full text editor, so it should be thought of as an emerging building block rather than a complete solution. It can be used to compute text height, line breaks, and rich inline layout without depending on DOM reflow, which is helpful for Canvas or WebGL renderers that need accurate text layout in display mode. In a creative tool, Pretext can drive line wrapping, hit testing, selection geometry, and text box sizing before anything is painted. The @chenglou/pretext/rich-inline helper is especially relevant when inline content such as mentions, chips, code spans, or mixed text styles needs to flow correctly. In a hybrid architecture, Pretext can handle layout and measurement while the DOM overlay continues to handle the actual editing interaction.

Event handling
Canvas event handling requires manual implementation because the canvas is just a bitmap; the browser does not know what objects you have drawn or where they are.

Canvas elements still receive standard DOM events, but you need to map event coordinates back to your own drawn objects:


const canvas = document.getElementById('myCanvas');
const ctx = canvas.getContext('2d');

canvas.addEventListener('click', (event) => {
const rect = canvas.getBoundingClientRect();
const x = event.clientX - rect.left;
const y = event.clientY - rect.top;

// Check if the click hit any objects.
checkHitDetection(x, y);
});
A common hit detection strategy is the bounding box method: store rectangular bounds for each object and check whether click coordinates fall within them.


const objects = [
{ x: 10, y: 10, width: 100, height: 50, type: 'button' },
{ x: 150, y: 20, width: 80, height: 30, type: 'text' },
];

function checkHitDetection(mouseX, mouseY) {
for (const obj of objects) {
if (
mouseX >= obj.x &&
mouseX <= obj.x + obj.width &&
mouseY >= obj.y &&
mouseY <= obj.y + obj.height
) {
handleObjectClick(obj);
break;
}
}
}
The bounding box approach requires more complex calculations if the element is rotated or has an irregular shape.

Pixel-perfect detection is another plausible hit detection strategy, used by Konva.js. It involves drawing an object using unique colors on an invisible canvas overlay for precise hit testing. See their article Hit Region Detection for HTML5 Canvas and How to Listen to Click Events on Canvas Shapes.

Cursor display
Displaying the right cursor for each element is also tricky. Because the HTML <canvas> element is just a bitmap, it does not know about objects like text boxes or draggable handles. You cannot assign a cursor per element directly like in CSS. Instead, you need to detect which element the pointer is currently over and set canvas.style.cursor = "grab" or "text", "move", "crosshair", and so on.

Zooming and panning
Thankfully, zooming and panning are not especially difficult to achieve with HTML Canvas. To implement zooming and panning, you do not move already-drawn pixels. Instead, you control the coordinate system with transforms and then redraw the scene.

We've created a canvas-based zoom/pan demo here: Canvas zoom/pan demo

When a canvas is zoomed and panned, the screen coordinates no longer match world coordinates. Mouse clicks have to be converted back into world coordinates so the cursor selects the correct objects after zoom and pan.

Always convert pointer events into world coordinates before hit testing
Pointer events arrive in screen space, but the scene graph stores geometry in world space, and nested frames add their own local coordinate systems on top of that. Converting the pointer to world space once at the input boundary, then into the element's parent-local space only when committing a transform, avoids a whole class of off-by-zoom and off-by-pan bugs in drag, resize, snapping, and selection math.

Overall, Canvas is a good choice for drawing-heavy tools, large freeform surfaces, and products that need a custom rendering pipeline with tight control over performance. The tradeoff is that the app must rebuild many browser features itself, especially around text editing, accessibility, hit testing, and input behavior. For simpler or more text-centric tools, DOM or SVG is often faster to build and easier to maintain, but Canvas becomes attractive once rendering flexibility and scene size matter more than browser-native primitives.

DOM rendering
DOM rendering delegates much more of layout, hit testing, text editing, and accessibility to the browser, which makes many interactions simpler to implement than in Canvas or WebGL.

Native text editing and accessibility
This is the biggest advantage of DOM rendering. If text elements are backed by real browser primitives such as input, textarea, or contenteditable, the app gets caret movement, text selection, IME composition, copy/paste, spellcheck, focus management, and screen reader semantics largely for free. That makes DOM especially attractive for text-heavy tools, template editors, and lighter-weight whiteboards where editing fidelity matters more than extreme scene size.

Even when the full scene is not editable all the time, using DOM elements for text still reduces the amount of custom editor logic the app has to build. The browser already knows how to handle arrow-key movement, double-click word selection, accessibility tree generation, and platform-specific input behavior. Reproducing those details correctly in Canvas or WebGL is much harder.

Clipping, transforms, and stacking
DOM rendering also maps naturally to frames and nested containers. A frame can become an absolutely positioned element, clipping can be implemented with overflow: hidden, and rounded clipping can often be achieved with border-radius. For simpler tools, this is much easier than manually implementing masking and clipping in Canvas.

However, complexity returns once the design surface becomes deeply nested. CSS transforms create new coordinate spaces, transform-origin affects how scaling and rotation behave, and stacking contexts created by transform, opacity, filter, or position can make z-index reasoning surprisingly tricky. In practice, once elements live inside multiple transformed parents, selection math, drag handles, and alignment guides often need getBoundingClientRect() or DOMMatrix-style calculations to map between local and world coordinates correctly.

Event handling and hit testing in the DOM
Basic hit testing is much easier in the DOM because the browser already knows which element is under the pointer. Each element can receive pointer or mouse events directly, so for simple interactions there is no need to write a manual hit-testing loop like there is in Canvas.

That said, DOM interaction is not entirely free of complexity. Overlapping layers, selection overlays, and nested transformed elements can cause events to land on the wrong node unless pointer-events is configured carefully. During drag interactions, the app will often need pointer capture so movement continues even if the cursor leaves the original element. It is also common to temporarily disable browser text selection with CSS such as user-select: none while dragging, otherwise native selection behavior can interfere with move and resize operations.

### Performance and scaling limits

DOM rendering is easy to start with, but each visible element is a real browser node that participates in style calculation, layout, paint, and memory usage. As files grow, those costs compound. Large scenes with hundreds or thousands of elements, especially when they include nested frames, shadows, filters, or blurs, can become sluggish because the browser has to maintain and repaint a large render tree.

Effects are a particular pain point. Shadows, CSS filters, masks, and semi-transparent overlapping layers can force expensive paint and compositing work. Deep nesting also increases the cost of layout invalidation when parent size or transform changes ripple down the subtree. Techniques such as flattening the DOM structure, avoiding unnecessary wrappers, and virtualizing offscreen regions can help, but there is still a ceiling. That is why many large professional creative tools eventually move the main scene renderer to Canvas or WebGL once DOM overhead becomes the bottleneck.

Zooming and panning
When rendering through the DOM, zooming and panning can be implemented using CSS transforms on the scene container. This is convenient because the browser handles the visual transformation, and child elements automatically move together with the scene.

However, the coordinate system still changes after transformation, so events like clicks need to be transformed back into world coordinates. Overlay elements such as selection boxes or resize handles also need to stay in sync with the transformed scene, which becomes harder when the scene is nested inside other transformed or scrollable containers.

Overall, DOM is a good choice for simpler creative tools, lightweight whiteboards, template editors, and products where text editing and accessibility are more important than rendering massive scenes. For large-scale product design tools with deep hierarchies and thousands of elements, DOM often becomes the bottleneck and teams move the primary scene renderer to Canvas or WebGL.

Interaction overlay
The interaction overlay layer exists to separate user interaction concerns from rendering of the elements. This layer draws things that are not part of the document: selection outlines, resize and rotate handles, guides, snapping indicators, pivot points, and marquee boxes.

These visuals help the user understand what is selected and what operations are available, but they do not get serialized into the saved file or exported image.

Why it is usually a separate layer
In practice, the interaction overlay is often implemented as a separate layer stacked above the main scene, for example, as a second transparent <canvas>, an SVG overlay, or an absolutely positioned DOM layer. Keeping it separate from the elements layer makes the architecture much cleaner.

The main reason is redraw frequency. Ephemeral visuals such as hover outlines, selection boxes, drag handles, and snap guides change much more often than the document itself. If they were painted directly into the same canvas as the scene, every pointer move could force a full redraw of all elements. With a separate overlay layer, the app can update only the interaction visuals while leaving the main scene untouched.

This also keeps coordinate systems and responsibilities clearer. The elements layer is responsible for rendering the document, while the overlay is responsible for manipulation UI and temporary interaction state. That separation makes it easier to clear and repaint handles, keep controls above all elements regardless of document z-order, and render screen-space affordances whose size should stay visually consistent across zoom levels.

Zoom level is another major reason to keep the overlay separate. The document itself should scale with zoom, but interaction controls often should not. For example, resize handles, rotation knobs, selection outlines, and cursor indicators usually need to remain usable at a fixed on-screen size even when the user is zoomed far in or far out. A separate overlay layer makes that easier: the app can project element bounds from world coordinates into screen coordinates, then draw the manipulation UI in screen space instead of letting it scale blindly with the scene.

Input handles
The interaction overlay is a good candidate to own hit-testing for selection and transformation handles. Instead of mixing that logic into shapes themselves, the interaction overlay knows where handles are and interprets pointer events. It translates raw pointer, mouse, and touch events into higher-level edit intents such as "move this object", "rotate selection by 15 degrees", or "scale from center". This keeps input logic unified and prevents accidental interference with the elements rendering code.

Depending on the rendering approach used for the elements layer, the responsibilities of the interaction overlay can differ. If the elements are rendered using the DOM, it can be easier to attach event listeners directly to DOM elements while rendering transformation handles on a separate layer and handling transformations there. If the elements layer is rendered using HTML Canvas, the interaction logic has to be implemented manually anyway, so it can be cleaner to put interaction logic inside the interaction overlay.

### Performance and UX considerations:


Handle ergonomics: handles should remain a reasonable size regardless of zoom level so that elements can still be manipulated even when heavily zoomed out. When zoomed out, fewer handles can be rendered, such as only corners and not edges. The hitbox can also be slightly larger than the visible handle.
Large scenes make hit testing expensive: when manual hit testing is required, spatial indexing may be needed. We cover that in Performance.
Guides and snapping
Guides and snapping are interaction aids that make alignment and spacing easier.

Guides are visual helper lines that appear while you move or resize an element. Their purpose is to show relationships between the active element and nearby elements, artboard edges, or predefined rules. They do not affect position directly; they are visual indicators.

Examples:

A vertical line through the center of the canvas when you center an object
Lines showing equal spacing between three objects
Snapping is a behavior where an element's position or size is automatically adjusted when it is near a significant alignment target. Its purpose is to help objects align precisely without requiring pixel-perfect dragging.

Examples:

When you drag a box near another box's edge, it "sticks" so the edges align
Objects snapping to grid lines or the artboard center
Rotation snapping every 45 or 90 degrees
Snapping changes behavior because the element's position is corrected to a target. Guides usually accompany snapping so the user can see why the element jumped.

Guides and snapping behavior operate by tracking distances between the current element and surrounding elements:

Track bounding boxes for every element using x, y, width, and height
While dragging or resizing one element, compute distances to the nearest edges of nearby elements
If a distance is small, for example, less than 10px, display a measurement line
If within a tighter threshold, for example, 5px, snap the element's coordinate and show a guide line
### Performance and UX considerations:


During each drag, you do not want O(n^2) checks against every element. Use spatial indexing such as a quadtree, R-tree, or grid bucketing to quickly find nearby elements.
Avoid over-snapping: if multiple snap targets are found, pick the strongest one
Use easing: do not lock completely, allow the user to break free by dragging past the threshold
Hybrid renderer choices
To recap, the elements layer contains what the user is creating, while the interaction overlay contains guides and ephemeral UI handles for manipulating elements.

A hybrid approach means the elements layer and the interaction overlay do not have to use the same rendering technology. Common combinations include:

Canvas or WebGL scene + DOM overlay: Strong choice when the main scene needs performance, but text editing, focus, clipboard behavior, and accessibility should still rely on browser primitives.
Canvas or WebGL scene + SVG overlay: Useful when selection boxes, handles, guides, and connector paths benefit from crisp vector rendering and easy styling.
DOM scene + separate DOM or SVG overlay: Still useful even when the main scene is already in the DOM, because ephemeral interaction UI can stay isolated from document content and be managed independently.
The key idea is to choose the renderer for each layer based on what that layer needs to do. The scene layer optimizes for drawing the document, while the overlay optimizes for manipulation, feedback, and temporary interaction state.

### Performance

Performance in creative tools is mostly about keeping editing interactions responsive even as the scene grows large and the rendering, layout, and hit-testing work become more expensive.

Redrawing only visible parts
Large design files should not force the renderer to redraw everything on every update. The first optimization is to make rendering proportional to what the user can actually see. For large scenes, the renderer should draw only what intersects the current viewport, plus a small overscan buffer so fast pans and scrolls do not expose empty gaps.

This idea shows up differently depending on the rendering approach. DOM-based implementations may use virtualization or windowing to mount only visible elements, while Canvas and WebGL renderers usually perform explicit culling before drawing. In both cases, partial invalidation matters too: if only one object moves, the app should avoid rebuilding unrelated parts of the scene where possible. A separate interaction overlay helps here because ephemeral selection and guide updates can be redrawn independently from the main scene.

The goal is simple: work should scale with what the user can actually see and interact with, not with the total size of the document.

Throttling/debouncing updates
Throttle and debounce solve different problems and should not be used interchangeably.

Throttle is for continuous streams such as pointermove, scroll, window resize, or collaborator cursor updates. In direct manipulation flows like dragging, throttling should usually be aligned with requestAnimationFrame so that the app updates at most once per frame without feeling laggy.
Debounce is for work that should only happen after activity settles, such as autosave, expensive layout recalculation, persistence, or non-urgent background processing.
For creative tools, this distinction matters a lot. Debouncing direct drawing, dragging, or resize previews too aggressively makes the editor feel sluggish and disconnected from the pointer. Throttling is the better default for interactive previews, while debouncing is better for follow-up work that does not need to happen immediately. This is related to the transaction coalescing discussion in Command system, but it is not the same thing: the UI may render many intermediate preview states while history still collapses them into one logical edit.

Align direct-manipulation redraws with requestAnimationFrame
Raw pointermove fires faster than the display can paint, so repainting the scene on every event wastes work and can actually feel laggier than rAF-aligned updates. Batch pointer-driven preview state and flush it once per frame via requestAnimationFrame so the renderer runs at most once per refresh. Keep debouncing for background work like autosave, text measurement, or thumbnail regeneration where immediate response is not needed.

Spatial indexing
Scanning every element on every pointer move does not scale. As scenes grow, the editor needs a spatial index so hit testing and nearby-element queries do not become O(n) across the full document.

Spatial indexing is useful for:

Hit testing which element is under the pointer
Marquee or box selection across many objects
Snapping and guide lookup during drag or resize
Finding nearby elements, connectors, or alignment targets
Common approaches include quadtrees, R-trees, and simpler uniform-grid or grid-bucketing schemes. The exact structure is less important than the core idea: maintain a fast mapping from regions of space to candidate elements. That index must also be updated incrementally whenever elements move, resize, appear, disappear, or otherwise change their bounds.

Caching and layerization
Not every frame should recompute and redraw the entire scene from scratch. Expensive content can often be cached once and reused until it becomes invalid. For example, a complex group, frame, or tile can be prerendered into an offscreen surface and then reused as a cached texture or bitmap during panning and zooming.

The same principle applies to text and layout work. Text measurement, line breaking, and layout results are often expensive enough to cache, especially when the underlying text and styles have not changed. Separating the editor into layers, such as background, scene, and interaction overlay, also helps because each layer can be invalidated independently. Typical invalidation triggers include style changes, transform changes, text or content edits, and zoom changes when the cache is zoom-sensitive.

Off-main-thread work
The main thread should prioritize input, animation, and composition. Heavy background work should move off the main thread whenever possible so pointer input, dragging, and typing remain responsive.

Good candidates include image decoding, serialization and deserialization, compression, large layout or text calculations, and other computationally expensive helpers implemented in Web Workers or WASM. The article also mentions some of these ideas in Offline editing, but the broader performance principle is the same: expensive work should not compete directly with interactive editing on the UI thread.

Editing model and command flow
Once the scene can be rendered and manipulated efficiently, the next question is how the editor turns user intent into predictable document mutations, history entries, and protected readonly behavior.

Single-element operations
Single-element operations are the baseline editing workflow. Most user actions begin with one primary selection, and multi-element behavior is best understood as an extension of these same mechanics.

Select and inspect
The simplest flow is selecting one element. A click hit-tests the scene, updates SelectionState.ids to contain that element ID, clears any marquee interaction, and sets the primary selection used by handles and the inspector. At that point, the properties panel can read that one element's current geometry, style, visibility, and lock state directly from the document model.

This matters because many other single-element operations depend on a stable primary selection:

Direct manipulation on canvas, such as move, resize, and rotate
Inspector edits, such as changing x, y, width, height, fill, stroke, or opacity
Contextual actions, such as delete, duplicate, lock, hide, bring forward, or send backward
Move / resize / rotate
Direct manipulation usually has two phases: preview and commit.

Pointer down on an element body or transform handle starts an InteractionState, such as drag-selection, resize-selection, or rotate-selection.
Pointer move updates draft geometry every frame so the overlay and scene show immediate feedback.
Pointer up commits one logical document edit and clears the interaction state.
The implementation is usually easiest if pointer coordinates are first converted into the same world space used by hit testing and overlay rendering, then mapped back into the selected element's parent-local space before committing. That avoids mixing screen coordinates, viewport transforms, and nested frame transforms in the same calculations.

Moving: Movement is the simplest transform. On pointer down, capture:

The pointer's starting world coordinate
The selected element's original local x and y
The selected element's parentId, so deltas can later be converted back into that parent's coordinate space
On every pointer move:

Compute worldDelta = currentPointerWorld - startPointerWorld
If the parent is transformed, convert that delta into the parent's local coordinate space
Preview x = startX + dx, y = startY + dy
Apply snapping if the moved bounds are close to grid lines, guides, or nearby elements
The key point is that move should preserve the element's size and rotation and only translate its origin. Locked elements should not enter this flow at all, and hidden elements usually should not be directly movable from the canvas.

Resizing: Resizing is more subtle because the active handle determines which edges move and which remain fixed. For a single selected element, a common mental model is "drag one side or corner while the opposite side stays anchored".

Typical implementation steps:

On pointer down, store the element's original bounds and which handle was grabbed, such as e, sw, or n
Convert the pointer into the element's local axis if the element is rotated
Update only the edges controlled by that handle
Recompute x, y, width, and height from the new edges
Examples:

Dragging the east handle changes width but keeps the west edge fixed
Dragging the northwest handle changes x, y, width, and height together because the southeast corner remains anchored
Holding modifiers can alter the rule, such as resize-from-center or preserve-aspect-ratio
There are a few important edge cases:

Enforce a minimum width and height so the box does not collapse into invalid geometry
Decide whether negative width or height is allowed; many editors instead flip the active handle when the drag crosses past the opposite edge
For rotated elements, project pointer movement into the element's local coordinate system before updating edges
For text or autolayout frames, resizing may trigger secondary layout behavior rather than just changing a raw rectangle
Rotating: Rotations are usually computed relative to a pivot point, often the center of the element's bounds. On pointer down, store:

The pivot point in world coordinates
The pointer's starting angle relative to the pivot
The element's starting rotation
On every pointer move:

Compute currentAngle = angle(pointerWorld - pivotWorld)
Compute deltaAngle = currentAngle - startAngle
Preview rotation = startRotation + deltaAngle
Optionally snap to common increments such as 15 or 45 degrees
This works well because the rotation gesture only depends on the relationship between the pivot and the pointer, not on the element's width or height. If the product supports rotate-around-custom-pivot behavior, then that pivot becomes part of the interaction state too.

Commit and history behavior
Even though pointer move may update preview geometry dozens of times, the commit step should still produce one logical edit. A drag that moves a rectangle from (10, 20) to (110, 60) should create one committed transform operation, not one history entry per frame. The same rule applies to resize and rotate.

For a single selected element, the committed result usually becomes one explicit operation with a stable target ID, for example:


```typescript
type DocumentOperation =
```
| { type: 'UPDATE_ELEMENT_TRANSFORM'; elementId: ElementId; x: number; y: number; width: number; height: number; rotation: number }
|  |
| { type: 'SET_ELEMENT_STYLE'; elementId: ElementId; patch: Partial<Pick<RectangleElement, 'fill' | 'stroke'>> };
The command that triggers this may still be selection-oriented, such as MOVE_SELECTION or RESIZE_SELECTION, because the same command can later support multi-select. But once resolved, the durable operation should target explicit element IDs and explicit transform values or deltas.

Inspector and menu edits
Not every single-element edit comes from dragging on the canvas. Many come from typing into the inspector or using menus:

Typing x = 120 or width = 240
Changing fill color, border radius, font size, or line height
Toggling visible or locked
Executing duplicate, delete, or reordering commands
These should all reuse the same command path as direct manipulation. For example, dragging a rectangle to x = 120 and typing x = 120 in the inspector should not mutate state through different code paths. They should converge on the same normalized transform or property-update logic so validation, history, analytics, persistence, and collaboration all stay consistent.

Text edit mode as a special single-element case
Text layers deserve special mention because selecting a text element and editing a text element are not the same state. A single click may only select the text box, while double-click or Enter can switch FocusState into text-edit mode for that one element. In that mode, commands like Delete, arrow keys, paste, and IME composition should affect the text range inside that element rather than deleting the whole layer.

Single-element operations therefore need command routing that distinguishes:

Element-level editing: move, resize, rotate, delete layer, set fill/stroke
Text-content editing within one selected text element
That is one reason the earlier FocusState and InteractionState distinctions are important: a single selected element can still be in very different editing modes.

Multi-element operations
Multi-element operations build on top of multi-select. Once SelectionState.ids contains multiple elements, the editor should treat many user actions as one logical command so undo/redo, persistence, and collaboration reason about them as a single edit instead of a pile of unrelated updates.

Grouping
Grouping lets several selected elements behave like one unit for move, resize, rotate, hide, lock, and reorder operations, while preserving each child's relative transform and local z-order. Conceptually, grouping creates a logical parent or wrapper around the current selection, even if the internal data model does not expose a formal GROUP element type.

Ungrouping does the opposite: it removes that wrapper while keeping the children visually in place. Users also usually need a drill-down interaction, where they first select the group as a whole and then enter the group, often via double-click or a keyboard shortcut, to edit an individual child. Commands such as GROUP_ELEMENTS and UNGROUP_ELEMENTS should each be recorded as a single history item.

Bulk selection and editing
When multiple elements are selected, the editor computes one aggregate bounding box and shows shared transformation handles around it. Transform operations should apply deltas to each selected element rather than assigning every child the same absolute x, y, width, or height. For example, move applies the same translation to all selected elements, while resize and rotate typically use a shared pivot and preserve each element's relative placement within the selection.

Bulk operations commonly include delete, duplicate, move, rotate, lock, hide, reorder, and shared style updates such as fill, stroke, opacity, or text properties. The inspector also needs to handle mixed state: if selected elements disagree on a property, the UI should show a mixed value until the user applies a new one. Internally, the editor may expand a bulk edit into multiple UPDATE_ELEMENT patches, but it should batch them into one command or transaction so undo/redo treats the whole action as one step.

Alignment / distribution
Alignment and distribution operate relative to the combined selection bounds, a containing frame or artboard, or a chosen anchor element such as the primary selection. Common actions include align left, right, top, bottom, horizontal center, vertical center, and distribute elements evenly along the horizontal or vertical axis.

The implementation idea is straightforward: compute the bounds of the participating elements, exclude locked or hidden elements, sort them along the target axis, and then update positions according to the requested rule. Alignment snaps multiple elements to the same reference edge or center line. Distribution keeps their order but recomputes gaps so spacing becomes even. These commands are some of the clearest examples of why a stable selection model matters: they depend on consistent bounds calculation, predictable batching, and a clean command system.

Multi-element operations are one of the main reasons creative tools need both a reliable selection model and a consistent command system. Users think of these edits as one action, even when the implementation touches many elements underneath, so the architecture should preserve that mental model.

Copying / pasting / dropping
Copy/paste in creative tools looks simple to users but is surprisingly complex under the hood because it touches every part of the system: the data model, serialization, input/output formats, and editing commands.

Some of the complexities:

Selection scope: decide what to copy, including elements, groups, hidden or locked items, and dependencies
Data representation: internal format versus OS clipboard, which may include native JSON, SVG/HTML, or bitmap data
References and dependencies: handle shared styles, components, and external assets by linking, cloning, or breaking references
Coordinate space: preserve original transforms versus pasting relative to the viewport or cursor
Creative tools go beyond text copy by putting multiple data formats onto the system clipboard and letting the target app decide what to consume.

System clipboard supports MIME types: when you copy, the app can register several representations of the same content, such as text/plain, text/html, image/png, plus a custom app-specific type
Native format: stored so the same app can paste with full fidelity, including layers, styles, and components
Standard formats: added so other apps can consume something usable
Text editors -> text/plain or text/html
Image editors -> image/png or image/svg+xml
Vector tools -> image/svg+xml or application/pdf
Using common apps as examples:

Figma and Sketch: Copying vector objects puts SVG, PNG, and a hidden JSON blob on the clipboard for the source app itself
Google Docs: Copying text provides plain text, rich text such as HTML/RTF, and often embedded images separately
Photoshop: Copying a layer produces a rasterized bitmap plus Photoshop's private format
Here is a minimal example of copying an object to the clipboard using the modern Clipboard API:


// Copy an object as JSON into the clipboard.
async function copyObject(obj: unknown) {
const blob = new Blob([JSON.stringify(obj)], { type: 'application/json' });
const item = new ClipboardItem({ 'application/json': blob });
await navigator.clipboard.write([item]);
}

// Example usage.
copyObject({ id: 'rect1', type: 'RECT', width: 100, height: 50 });
And to read it back:


async function pasteObject() {
const items = await navigator.clipboard.read();

for (const item of items) {
if (item.types.includes('application/json')) {
const blob = await item.getType('application/json');
const text = await blob.text();
return JSON.parse(text);
}
}

return null;
}

// Example usage.
pasteObject().then((obj) => console.log(obj));
This practice is important for:

Interoperability: Content can be pasted into other apps
Fidelity: Pasting back into the same app does not lose layers or styles
Adaptability: Allows each target app to pick the best format it understands
Tools allow copying beyond text by serializing the same selection into multiple clipboard formats, both standard and proprietary, so pasting works both inside the app and across different apps.

Command system
Commands are high-level editor intents. We have used actions to describe the Flux/reducer mutation mechanism inside the app. Commands sit one level above that: keyboard shortcuts, menus, panels, plugins, and collaboration code dispatch commands, and a command handler resolves them into one or more internal actions or patches within a single transaction. The main benefit is simple: many input sources, one consistent mutation path.

Many sources, one command
The same editing intention can originate from many places:

Keyboard shortcuts, such as pressing the arrow keys to nudge a selection
Toolbar or properties panel changes, such as typing x = 20
Right-click context menus
Command palette actions
Plugins, AI assistants, or background sync flows
The important thing is that these sources should not all mutate state in their own custom way. Instead, they normalize user intent into the same command. For example, both arrow-key nudging and typing x = 20 in an inspector can become MOVE_SELECTION. A toolbar fill picker and a pasted style can both become SET_FILL. A menu click and a keyboard shortcut can both become GROUP_SELECTION.

Command registry
User-facing commands should be registered with metadata instead of being hardcoded independently in every UI surface. That metadata can drive shortcut resolution, command palette listing, context menu enablement, discoverability, and even analytics.

A minimal shape could look like this:


```typescript
type CommandId = 'MOVE_SELECTION' | 'SET_FILL' | 'GROUP_SELECTION';

interface CommandDescriptor {
```
id: CommandId;
label: string;
shortcuts?: string[];
group?: 'Arrange' | 'Style' | 'Edit';
when: (state: AppState) => boolean;
execute: (payload: unknown, ctx: CommandContext) => void;
}

const commands: CommandDescriptor[] = [
{
id: 'GROUP_SELECTION',
label: 'Group selection',
shortcuts: ['Mod+G'],
group: 'Arrange',
when: (state) => state.selection.ids.length > 1,
execute: (_, ctx) => ctx.dispatch({ type: 'GROUP_ELEMENTS' }),
},
];
The exact API does not matter much in an interview. What matters is the architecture: a central registry gives the app one place to answer questions like "Is this command currently allowed?", "What shortcut should trigger it?", and "Should it appear in the command palette right now?"

Dispatch flow and handlers
A typical command flow looks like this:

A source dispatches a command ID with an optional payload.
The dispatcher validates preconditions and current context.
A handler resolves the command into document updates.
Those updates are committed as one transaction.
Post-commit listeners or observers react, for example to update UI state, analytics, autosave, or presence.
View
Store (document + history)
Dispatcher
Command layer
Source (shortcut / menu / gesture)
View
Store (document + history)
Dispatcher
Command layer
Source (shortcut / menu / gesture)
dispatchCommand(id, payload)
Resolve intent, begin transaction
Action 1 (e.g. MOVE_ELEMENTS)
Validate (readonly? locked? focus?)
Apply reducer
Action 2 (e.g. UPDATE_STYLE)
Apply reducer
Commit transaction (1 history entry)
Notify subscribers
Re-render scene + overlay


Command dispatch flow with transaction coalescing
Validation is important because the same command ID can be invalid in one context and valid in another. Common checks include:

The current tool or focus target
Whether the app is in text editing mode or canvas selection mode
Whether the selected elements are locked or hidden
Whether the app is in readonly mode
It is also useful to think of commands as something that can be reacted to by multiple handlers or listeners. A specific handler might claim a command first, while more generic fallback handlers only run if no earlier handler has handled it. That idea keeps command processing extensible without forcing every command through one giant switch statement.

Transactions, batching, and coalescing
This is the core reason command systems matter in creative tools. Many visible interactions generate a stream of low-level updates, but users still expect them to behave as one logical action.

Examples:

Dragging a selection may produce dozens of intermediate position updates while the pointer moves.
Resize and rotate gestures may continuously update preview geometry.
Alignment, grouping, or bulk style changes may touch many elements at once.
Coalesce intermediate updates by intent, not event count
These should still execute inside one transaction boundary. Internally, the command may dispatch multiple updates, but history should coalesce by intent, not by raw event count. That is why APIs such as commitUpdate() or explicit transaction wrappers are so useful: they let the app separate "previewing many intermediate states" from "recording one user-level edit". We cover the actual history mechanics in more detail in Undo/redo history.

Context-aware commands
Commands are also context-sensitive. The meaning of Delete depends on what the user is doing:

If a text editor is active, Delete should delete selected text.
If canvas selection is active, Delete should remove selected elements.
If the selection is locked or the app is readonly, Delete should do nothing.
That means command enablement predicates should consult state such as:

Current selection
Active tool
Focus target
Readonly status
Locked or hidden state
This context awareness should not live only in the UI. It should be enforced in the command system itself so menus, the command palette, and keyboard shortcuts all behave consistently. Unavailable commands should usually be hidden or disabled rather than only failing at execution time. Readonly enforcement is covered more specifically in Readonly mode.

Keyboard shortcuts, palette, and menus as thin adapters
Keyboard shortcuts, context menus, and command palettes should be treated as thin adapters over the command layer, not as separate places that contain mutation logic. Their job is only to:

Resolve user intent
Map it to a command ID and payload
Dispatch the command
This keeps behavior consistent across the product. It also makes power-user features easier to build because the command palette can simply list registered commands and their shortcuts, instead of inventing a parallel action model. The same abstraction is also helpful for plugins, AI features, and collaboration code: they can trigger the same command layer as the built-in UI without special-case mutation paths.

A consistent command system is what makes undo/redo, readonly enforcement, plugins, and real-time collaboration tractable in large creative tools. Once every meaningful edit goes through the same intent layer, the app has one consistent place to validate, batch, record, replay, and transmit changes. The discussions in Undo/redo history and Real-time collaboration build directly on that foundation.

Undo/redo history
Undo/redo is the mechanism for reverting recent user commands in the active editing session. It is local, interaction-oriented history: the goal is to let users quickly step backward and forward through their recent edits without affecting the long-term document record.

There are two common implementation approaches. One is to store immutable state snapshots in undoStack and redoStack, then move a pointer or shift snapshots between stacks as the user undoes and redoes. The other is to store inverse commands or inverse patches so each committed edit knows how to revert itself. Both approaches can work, but for a creative tool built around commands and transactions, a transaction-based command history is usually the better fit.

In that model, each history item represents one committed user intent, even if it expands into multiple low-level updates internally. A drag operation may emit many preview position updates, but the final committed move should produce one history item. The same is true for grouping, alignment, or bulk style changes across many elements. This is exactly why commitUpdate()-style APIs or explicit transaction boundaries are useful: they let the app coalesce noisy intermediate updates into one logical undo step, which is consistent with Command system.

Granularity matters a lot for user experience. Dragging a shape across the canvas should undo once, not 50 times. Repeated typing or keyboard nudging may be coalesced into fewer history items if they are part of one continuous editing intention. At the same time, transient UI state such as hover, focus rings, or interaction overlays usually should not go into history. Selection state is often excluded too, though some products selectively include certain selection transitions because they are meaningful in the editing flow.

A new edit after an undo should clear the redo branch
One practical rule is that creating a new edit after an undo should clear the redo branch, because the user has started a different future. In multiplayer scenarios, undo/redo should also generally operate only on the current user's own committed actions, not on peers' edits. The collaboration implications are discussed further in Real-time collaboration.

Version history
Version history is different from undo/redo. Undo/redo is for quickly correcting recent editing actions, while version history is a durable, document-level record used for auditing, reviewing, and restoring older states of the document.

It is useful to distinguish between revisions and versions. Revisions are the granular internal commits or accepted operation batches that advance the document state. Versions are the user-facing checkpoints shown in the product UI, often grouping many revisions together into a more meaningful timeline. A version entry usually contains a revision or version ID, timestamp, author or authors, and optionally a summary, label, or thumbnail preview.

The storage model should match the persistence approach described in Data persistence: keep an append-only revision log for granular changes and write periodic snapshots so restoring an older state does not require replaying the entire log from the beginning. This gives the system both an efficient synchronization model and the foundation for product-level history features.

Restoring an old version should usually create a new current revision rather than rewriting or deleting past history. That keeps the audit trail intact and avoids breaking references to earlier revisions. On top of that core model, products can add features like named versions, autosaved checkpoints, and diff or preview views between versions.

In short, undo/redo is short-lived editor history, while version history is durable document history. They are related, but they serve different user needs and should be modeled separately. Version history depends on durable storage and revisioning, not just local editor state, which is why it connects more naturally to the persistence and collaboration sections than to the active-session undo stack.

Readonly mode
Readonly mode is a mode where the current user cannot modify elements on the canvas.

It can be implemented by disabling every action source that causes element mutation, but this is brittle and cumbersome because every new interaction source has to remember to check the mode before firing an event.

Since every change has to come from an action, the most convenient place to enforce readonly behavior is in the dispatcher. The dispatcher can discard any action that would change element state. This guarantees that, no matter the action source, the document cannot be modified.

Enforce readonly at the dispatcher, not at every input source
Gating readonly in the UI layer by disabling buttons, ignoring shortcuts, or short-circuiting drag handlers is fragile: every new input surface, plugin hook, or collaboration path becomes a fresh chance to leak a write through. Reject mutating actions centrally in the dispatcher so readonly is enforced once, for every source, including sources that do not exist yet.

Persistence, sync, and extensibility
Once the editor can render and mutate state predictably, the remaining concerns are how that state is saved, synchronized, exported, and safely opened up to extensions.

Data persistence
The design document can be persisted in two ways: locally to the device file system, and to the cloud, eventually backed by a database. Both approaches share similarities and differences.

Local persistence: Tools like Figma and Excalidraw support saving the entire design document as a file, such as .fig and .excalidraw, that can be transferred around, similar to the pre-cloud era of Microsoft Office files.

These tools treat the durable part of the file as the scene graph plus document metadata such as pages, styles, components, and asset references. Session-specific details such as the current selection, temporary overlays, or current zoom are often kept outside the durable model or stored separately as optional app metadata. Because the core model is structured data, it can be written to disk and later restored by parsing it back into memory.

Excalidraw: .excalidraw files are human-readable JSON that can be versioned with Git or transferred across machines. The architectural idea is straightforward: serialize the drawable elements plus the relevant document metadata, then load that JSON back into memory later.
Figma: .fig files represent a more complex, tool-specific packaged document format because the underlying model includes pages, components, styles, vector data, text, and asset references at much larger scale. The exact encoding matters less than the architectural point: the saved file is still a serialized representation of the document model, not a dump of renderer-specific objects.
In both cases, the key idea is that the design document is model-driven rather than tied to the renderer. Because state is captured in plain data structures, saving to a file is serialization, and opening a file is deserialization. That is what makes these formats easy to transfer between devices or share outside the cloud service.

Server persistence: So far the discussion has focused on what goes on in the client. In modern, real-world design apps there is also a server component. Documents are periodically persisted to the cloud instead of only being stored on the local file system.

The local document is usually treated as the source of truth and mirrored to the server using a mix of snapshots, deltas, and asset uploads. The server becomes a durable, shareable copy that other devices or collaborators can fetch.

The main entities to persist are:

Document: scene graph or elements such as frames, shapes, text, and images-as-refs, plus app-level metadata such as title and modifiedAt, and sometimes a compact operation log for history or multi-device merges
Assets: large binaries such as images, thumbnails, raster tiles, and fonts, usually stored out-of-line and referenced by assetId in the document
Document persistence to the server usually uses one or more of these models:

Snapshot model: periodically serialize the whole document and PUT it to the server. This is simple, robust, and good for single-user files.
Pros: Trivial to reason about and restore; easy to version
Cons: Heavy uploads for large docs; harder to merge concurrent edits
Delta / operation model: append small explicit document operations or patches, and occasionally compact them into a snapshot on the server
Pros: Efficient bandwidth usage and a better merge story across devices
Cons: Requires server logic to apply and compact operations while guarding consistency
Most mature tools use both: stream operations for responsiveness and checkpoint a snapshot every N operations or M minutes.

When persisting to the server, conflicts can occur. Depending on how many users can work on a document:

Single-user, multi-device: Last-writer-wins per field is often enough, and operations make rebasing easier
Multi-user, true concurrent collaboration: Use OT or CRDT on the server or in a relay that merges operations
Various conflict resolution policies have been introduced in detail in the Google Docs system design article. Also refer to Real-time collaboration for more detail when a design document is worked on by multiple users at the same time.

Real-time collaboration
First, align on what real-time collaboration means:

Each participant runs the same editor locally, keeping a replica of the document state
When a user performs an edit such as drawing a rectangle, resizing an image, or changing text, the local UI first issues a command such as MOVE_SELECTION or INSERT_IMAGE
The command handler resolves that local intent into one or more explicit document operations such as MOVE_ELEMENTS, INSERT_ELEMENT, or SET_TEXT_CONTENT
Those explicit operations, not ephemeral commands, are what get persisted or sent to other online clients. There are multiple viable client-server architectures; refer to the Google Docs article.
Each client applies the operation to local state. With the right concurrency model, such as Operational Transformation or CRDTs, everyone eventually converges on the same document
This allows multiple users to draw, type, or edit objects simultaneously while seeing each other's changes in near real time
Real-time collaboration is extremely tricky to implement for complex apps like creative tools. It is effectively a distributed systems problem:

Latency: Edits must feel instant locally, but updates from others arrive with network delay
Conflicts: Two people editing the same object, such as resizing versus deleting, may cause diverging states
Consistency: All replicas must converge to the same state, even if commands arrive out of order
Complex operations: Creative tools support vectors, images, groups, constraints, and more, which makes conflict resolution harder than it is for plain text
Scalability: Hundreds of simultaneous edits per second must be synchronized without slowing the UI
Apps that use a command system will find real-time collaboration easier to implement:

Instead of directly mutating state from many UI surfaces, every local edit first flows through the same command layer
That command layer then produces explicit operations with stable target IDs and payloads
Those operations are:
Serializable: easy to send across the network
Reversible: can support undo/redo by applying inverse operations or patches
Composable: multiple operations can be batched into transactions
Because both local UI updates and remote updates ultimately resolve into the same operation pipeline, the editor does not need separate document-mutation logic for "my action" versus "their action"
This uniform abstraction makes collaboration, persistence, and history easier to implement
Client B (store + view)
Sync server (OT/CRDT)
Client A (store + view)
Client B (store + view)
Sync server (OT/CRDT)
Client A (store + view)
Presence (cursors, selection) flows on the same channel but outside durable history
dispatchCommand(MOVE_SELECTION)
Apply op locally (optimistic)
Send op (rev N, clientId)
Transform / merge against concurrent ops
Ack + canonical rev N+1
Broadcast transformed op
Apply op through same reducer pipeline


Multiplayer edit with optimistic local apply and remote broadcast
A concrete Figma case study is useful here. In How Figma's multiplayer technology works, Figma describes a client/server system where each client keeps a local replica, syncs over WebSockets, and applies local edits optimistically. Conflict resolution is not one-size-fits-all: Figma uses different rules depending on the data, such as last-writer-wins for simple object properties, but more careful handling for tree structure changes like moving an element to a new parent or reordering siblings, where naive last-writer-wins would produce broken results. The details are specific to Figma's document model, but the broader takeaway is that a visual editor needs an explicit sync model for object properties, tree structure, offline rebasing, and undo semantics.

Canva's engineering writing is useful for a different reason: it highlights the transport and reliability side of collaboration at scale, such as maintaining bidirectional streaming channels and pushing updates promptly to many connected clients. That is a reminder that "real-time collaboration" is both a document-merge problem and an infrastructure problem.

Regardless of algorithm, awareness features such as cursors, selection highlights, and presence indicators usually stay ephemeral and sit outside the durable document state.

Even though the shared document can be updated by multiple people at the same time, undo/redo should only operate on a single user's own changes.

Offline editing
Offline editing works best when the app is local-first: every edit applies instantly to an in-memory document and is durably written to a local database such as IndexedDB, with the network treated as a best-effort mirror.

In practice, that means keeping the scene graph in memory for responsiveness and persisting it to IndexedDB as a versioned JSON snapshot plus a tiny append-only operation log. The log makes autosave cheap and crash-safe. On restart, the app loads the latest snapshot and replays the tail of operations to recover exactly where the user left off.

Assets like images or raster tiles live as blobs in IndexedDB and are referenced by stable IDs from the JSON so the document itself stays small. When connectivity resumes, the assets are uploaded to file storage and the document points to external sources rather than local blobs. Alternatively, flows that require immediate communication with external file storage can be disabled while offline.

To make the app reliably load offline, ship an app shell with a service worker that caches core scripts, styles, fonts, and a lightweight home screen that lists recent files from a local index and shows thumbnails. The editor should hydrate only what is needed, such as the current page and nearby assets, and stream the rest on demand so cold starts are fast. Heavy work such as JSON serialization/deserialization, compression, and image decoding should run in Web Workers or WASM so periodic saves never block pointer input or typing.

When connectivity returns, a background sync reconciles local state with the server. A simple and robust approach is to upload new assets first, then send queued document operations and let the server compact them. If multiple devices edit the same file, fetch the latest snapshot, rebase local operations on top, and retry. For true concurrent edits, introduce OT or CRDT, but many single-user or handoff scenarios work with last-writer-wins at the field level plus clear "updated to latest" messaging.

user edits

op acked by server

connection lost

edit while offline

more edits appended to local log

network returns

assets pending

assets uploaded

no asset changes

server rev ahead

server accepts queue

rebased on latest snapshot

cannot auto-merge

user resolves

Synced

EditingOnline

OfflineQueued

Reconnecting

UploadingAssets

ReplayingOps

Rebasing

ConflictPrompt


Offline editing and sync state
Good UX is necessary too: show statuses like "Saved locally - offline" versus "Synced", keep work uninterrupted during network flaps, and recover gracefully from crashes. With these pieces, offline becomes the default experience: edits are instant and durable on the device, and cloud sync is an additive convenience rather than a prerequisite for getting work done.

Exporting
Export is distinct from local persistence
Export in creative tools like Figma, Excalidraw, and Canva refers to exporting the document, or selected parts of it, into a chosen target format such as PNG, SVG, or PDF. This is not to be confused with local persistence, which is about saving the document to the device and allowing the app to open the file later to resume editing.

Because creative tools keep a scene graph or element tree that is rendering-agnostic rather than being tied directly to <canvas> or <svg>, exporting becomes easier:

The same data model can be rendered interactively in the editor
The document can also be re-rendered into offline export formats
When a user selects export as PNG, SVG, or PDF, the tool runs a pipeline:

Traverse the scene graph and collect all elements, including transformations, opacity, masks, and effects
Resolve styles so the output matches the on-screen result, including fonts, colors, gradients, and shadows
Perform rasterization or vector serialization
PNG / JPEG export -> render the scene to an off-screen canvas at the requested resolution, then serialize to an image buffer
SVG export -> serialize vector shapes into <path>, <rect>, and <text> elements with inline styles
PDF export -> use a PDF backend such as PDFKit, Skia, or Cairo so each vector/text element maps to a corresponding PDF primitive
Package the output
Apply compression, such as PNG optimization or JPEG quality settings
Bundle fonts or replace them with outlines where necessary, especially for PDF and SVG
Handle slices or frames if exporting only a portion of the design
A popular client-side library for DOM-based rendering approaches is html2canvas, which can export a DOM result as an image. However, it has known issues keeping up with the latest CSS standards, and results can vary across browsers. For reliable exporting, it is generally better to generate the result on the server.

Plugin system
A plugin system lets a creative tool expose external extension points without stuffing every workflow into the core editor. In practice, plugins are useful for automating repetitive editor work, importing or exporting data, integrating external services, and supporting company-specific internal workflows that would otherwise bloat the product for everyone else. Figma is a useful reference model here: its plugin architecture extends the editor in powerful ways while still preserving a clear boundary between third-party code and the product's internal mutation pipeline.

Runtime model: Following Figma's plugin runtime model, a plugin usually has two execution surfaces. The main plugin logic runs in a sandboxed runtime that can access the editor's scene or document API, but does not get unrestricted browser APIs. If the plugin needs richer controls, it can also open an optional UI in an iframe or similar browser-like environment. That UI can use normal browser capabilities such as forms, network requests, and local rendering primitives, but it does not directly access the scene graph. The two sides communicate through explicit message passing.

This split is useful because it creates a stable extension boundary. The sandbox side can be given carefully designed editor capabilities, the UI side can stay flexible enough for custom workflows, and the message channel makes the crossing between the two explicit. For a creative tool, that improves security isolation, keeps the extension surface more stable over time, and still allows plugins to ship purpose-built interfaces instead of being limited to menu-only actions.

Lifecycle and execution constraints: In Figma-style systems, plugins are usually user-initiated, short-lived actions rather than always-on background processes. A user explicitly launches a plugin command, the platform runs one plugin action at a time, and the plugin is expected to close when its work is done. Users can also cancel execution.

These constraints matter more than they might seem at first. They avoid hidden background mutation paths, reduce the chance that third-party code quietly harms performance, and keep the editor's mental model simple: plugins are invoked tools, not independent agents that continue mutating the file after the user has moved on. That simplicity becomes especially important once undo/redo, readonly enforcement, and collaboration are layered on top.

Manifest, commands, and permissions: The manifest is the contract between the plugin and the host editor. In a Figma-style architecture, it declares the plugin entrypoint, optional UI entrypoint, menu commands or subcommands, which editor or environment the plugin targets, document access policy, network access restrictions, and relaunch buttons or command variants. This is how the platform exposes explicit capabilities instead of handing plugins unrestricted access to internal product APIs.

Large files add another important constraint: a plugin cannot assume the entire document is already loaded in memory. Figma's dynamic page loading model is a good example. Plugins that need to inspect multiple pages should load and traverse additional pages incrementally and asynchronously rather than assuming whole-document access from the start. That keeps the platform responsive even when a file is very large.

Integration with the editor core: Architecturally, plugins should be treated as another command source, alongside shortcuts, menus, context menus, and inspector panels. If a plugin wants to rename layers, insert generated assets, align elements, or update styles, it should trigger the same high-level commands as the built-in UI. Those commands should still flow through the normal dispatcher, validation logic, and transaction boundary before state is committed.

Plugins must not mutate editor state through a side door
The important rule is that plugins should not mutate internal editor state through a side door. If plugin-triggered edits reuse the same command and transaction path, undo/redo remains consistent, readonly mode still blocks writes, analytics and logging still see the action, and persistence plus collaboration continue to operate on the same mutation stream. The plugin API is external, but the editor's mutation pipeline should remain singular.

Safety and performance guardrails: Plugin APIs should be narrower and more stable than internal product APIs. Figma's launch posts make this principle explicit: third-party developers need an API surface that will not break every time the product evolves, and plugins should not harm performance or the user experience. That is why a mature plugin platform should expose intentionally designed extension APIs rather than simply leaking internal objects and methods.

In practice, that means capability-based and permission-based access, no uncontrolled background execution, incremental access patterns for large files, batching scene mutations instead of issuing many tiny writes, and avoiding eager whole-document scans unless the user clearly asked for them. The goal is not to maximize raw power. The goal is to let plugins extend the editor safely while still fitting the product's performance and consistency guarantees.

Distribution and ecosystem: Once the runtime boundary is clean, distribution can stay modular. Creative tools can support public community plugins, private internal plugins for organizations, and independently versioned plugin updates without coupling every extension release to the editor's main release train.

### Accessibility

Accessibility in a creative tool is usually a partial-support architecture, not a promise that every pixel on a Canvas or WebGL surface is exposed as a fully semantic document tree. In a practical first version, keyboard support, deterministic focus management, visible focus indicators, accessible editor chrome, and DOM-backed text editing should be first-class. Full screen reader semantics for arbitrary shapes on the drawing surface can remain out of scope.

Keyboard accessibility
Keyboard behavior should be treated as a core interaction model, not as a thin layer added after pointer interactions are complete.

Maintain a logical tab order across the main editor chrome: toolbar, layers panel, properties panel, canvas focus target, and any open dialogs or menus.
Once the canvas focus target is active, support core editing shortcuts such as delete, duplicate, nudge with arrow keys, zoom shortcuts, undo/redo, and mode switches like Enter to begin editing selected text or Escape to leave an active interaction mode.
Keep focus movement deterministic: Tab should move between major UI regions, while arrow keys and editor shortcuts operate within the currently active region instead of unpredictably escaping it.
Document the main shortcuts in menus, tooltips, or a command palette so keyboard access is discoverable rather than hidden.
Focus management and visible focus
The editor should maintain one clear active surface at a time: either the surrounding chrome, the canvas selection surface, or a DOM text-edit overlay. That maps naturally to the FocusState discussed earlier.

The canvas itself should have a focusable wrapper or focus proxy so users can intentionally enter and leave canvas interaction mode without relying on the mouse.
Focus should visibly move when the user switches from selecting an object to editing a text layer. Entering text-edit mode should transfer DOM focus to the text overlay; exiting should return focus to the canvas selection surface while preserving the selection.
Toolbar buttons, layer rows, inspector controls, menus, and any keyboard-manipulable canvas affordances should have clear visible focus indicators so users always know which surface owns keyboard input.
Composite widgets such as toolbars or layers lists may use roving tabindex or similar managed-focus patterns so keyboard navigation stays efficient without exploding the tab order.
Screen reader support for the editor shell
Even if the drawing surface itself is not fully semantic, the surrounding editor shell should still be screen-reader friendly.

Toolbars, buttons, menus, layers panels, zoom controls, and property fields should expose clear accessible names. Icon-only controls should use labels such as aria-label.
The layers panel is especially important because it can act as a structured, accessible representation of document objects even when the canvas is only partially accessible. If a rectangle is selected on the canvas, the corresponding layer row should reflect that selected state too.
Use status or live regions for concise announcements of meaningful state changes such as "rectangle selected", "3 items selected", "layer locked", "entered text editing", "saved locally", or "synced". These announcements should be short and meaningful rather than narrating every pointer move.
Property controls should expose labels and current values clearly so non-visual users can still inspect and edit geometry and style fields through the editor chrome.
Text editing overlays and IMEs
This is where the hybrid rendering approach pays off. When the user edits text, the editor should temporarily rely on a real DOM text surface so caret movement, range selection, composition events, copy/paste, spellcheck, and platform input behavior remain accessible. That same DOM surface is also the best place to support IMEs correctly.

This is the same general principle described in the Rich Text Editor article: text editing, focus management, keyboard behavior, and IME composition are much easier to implement accessibly when the browser owns the active editing surface instead of a custom bitmap renderer. For a drawing tool, that does not mean the whole scene must become DOM-based; it means text edit mode should intentionally switch to a DOM-backed overlay for that one focused element.

Canvas / WebGL limitation and fallback strategy
Canvas and WebGL surfaces are fundamentally bitmap-oriented, so assistive technologies cannot automatically infer the semantic meaning of every drawn object. That is one of the main architectural costs of those rendering approaches.

A practical fallback strategy is:

Keep the main scene renderer optimized for performance
Expose accessible surrounding UI for tools, layers, properties, menus, and status
Use DOM-backed overlays for text editing and other browser-native interactions
Optionally synchronize the currently selected object into accessible shell UI such as the layers panel and properties panel
That is similar to the accessibility tradeoff discussed in the Collaborative Spreadsheet (Google Sheets) article: dense, high-performance surfaces often need an accessible shell and editor overlays even when the primary rendering layer is highly customized. A future product could go further with an accessible DOM mirror for more of the scene, but that is a significantly larger investment and not something this writeup needs to claim for a first version.

Summary
A Figma- or Canva-style editor is one of the widest front-end system design prompts you can draw: rendering, input, data model, history, persistence, offline, and multiplayer all meet on a single canvas. Most candidates drown by drifting into implementation minutiae (WebGL shaders, CRDT algebra, text shaping) before they have committed to a shape for the document, the mutation path, and the rendering split. Use the order below to keep the discussion on architecture.

Frame the problem before drawing anything. State that the product is a large, hierarchical design document under heavy direct manipulation, and that the job is one responsive mutation path that undo/redo, autosave, offline, and multiplayer can all reuse. This sets the bar that later decisions must clear.

Pick the rendering pipeline first. Use hybrid Canvas/WebGL for the elements layer, DOM overlay for text edit mode, and a separate interaction overlay for selection boxes, handles, guides, and marquees. Justify it by comparing DOM/SVG's ergonomics against Canvas/WebGL's scalability once scenes reach thousands of elements, and keep IME, spellcheck, and screen readers on the DOM layer to avoid rebuilding the browser.

Commit to a Flux-like command pipeline as the core mutation path. Editor views emit user-level commands like MOVE_SELECTION, GROUP_SELECTION, or SET_FILL; a single dispatcher validates and enforces readonly; the store holds document, selection, and history; commands resolve into low-level actions and DocumentOperations that mutate state. This one-way flow is what makes persistence, sync, and history tractable, because every mutation passes through the same command boundary.

Split the data model into durable document state and ephemeral session state. Model the scene graph as a tree of SceneElements (FRAME, TEXT, RECTANGLE, IMAGE) on a Page that carries both an elementsMap for O(1) lookup and a doubly-linked list for cheap reordering, with children nested on container frames. Keep viewport, selection, tool, focus, and in-progress InteractionState inside CanvasState, and keep CanvasState out of the serialized document so saved files, history entries, and multiplayer merges stay clean.

Design the editor as a DrawingEditor facade instead of a broad HTTP API. Reads go through getDocument, getSelection, getCanvasState, and canExecute; writes go through dispatchCommand and runTransaction; observers go through subscribe. DocumentOperations with stable element IDs, not transient commands, feed autosave, offline replay, and real-time sync, so one operation format serves every dependent system.

Treat history, multiplayer, and offline as consequences of the pipeline. Undo/redo is a transaction-based local stack where one committed intent becomes one entry regardless of preview updates, a new edit after undo clears the redo branch, and in multiplayer undo scopes to the current user's own edits. Multiplayer applies operations optimistically, ships them over a sync channel, and merges with last-writer-wins on simple properties plus careful handling for tree structure changes, with presence kept outside durable history. Offline is local-first: an IndexedDB snapshot, an append-only operation log, cached asset blobs, and a background reconcile that uploads assets, replays operations, and rebases against the server snapshot on reconnect.

Earn performance claims with named techniques, not adjectives. Call out viewport culling with an overscan buffer and partial invalidation; rAF-aligned throttling for pointer-driven preview state with debouncing reserved for autosave and thumbnail regeneration; spatial indexing via quadtree, R-tree, or grid buckets for hit testing, marquee selection, and snapping; prerendered group and tile caches with independent invalidation for scene and overlay layers; and Web Workers or WASM for image decoding, serialization, compression, and heavy layout. Tie each one to the specific interaction it keeps at 60 fps.

If time is short, start with the hybrid rendering split, the single dispatcher with DocumentOperations as the unit of persistence and sync, and the document/CanvasState separation. The remaining topics can be reconstructed from those three decisions.

Appendix
This appendix covers supporting implementation details that are useful context but not central to the main architecture discussion.

How HTML Canvas works
HTML Canvas is a powerful HTML element that provides a drawing surface where graphics, animations, and interactive visualizations can be created using JavaScript. Think of it as a blank digital canvas that you can paint on programmatically.

The canvas element itself is simple HTML:


<canvas id="myCanvas" width="400" height="300"></canvas>
To draw on it, you need to get the 2D rendering context in JavaScript:


const canvas = document.getElementById('myCanvas');
const ctx = canvas.getContext('2d');
How to draw basic shapes:


// Rectangle
ctx.fillRect(10, 10, 100, 50); // Filled rectangle
ctx.strokeRect(150, 10, 100, 50); // Rectangle outline

// Circle/Arc
ctx.beginPath();
ctx.arc(100, 150, 30, 0, 2 * Math.PI); // x, y, radius, startAngle, endAngle
ctx.fill();

// Line
ctx.beginPath();
ctx.moveTo(200, 100);
ctx.lineTo(300, 200);
ctx.stroke();
How to set styling:


ctx.fillStyle = 'red'; // Fill color
ctx.strokeStyle = 'blue'; // Stroke color
ctx.lineWidth = 3; // Line thickness
ctx.font = '20px Arial'; // Font for text
How to render text:


ctx.fillText('Hello Canvas!', 50, 200); // Filled text
ctx.strokeText('Outline text', 50, 230); // Text outline
Paths, for complex shapes:


ctx.beginPath();
ctx.moveTo(50, 250);
ctx.lineTo(100, 200);
ctx.lineTo(150, 250);
ctx.closePath();
ctx.fill(); // Creates a triangle
Canvas is immediate-mode graphics. Once you draw something, it becomes pixels. Unlike SVG, there is no retained object model, so animations typically clear and redraw the entire canvas each frame.

There is also a zoom/pan demo here: Canvas zoom/pan demo

## References

The references below group design-tool engineering posts, editor and layout foundations, plugin platform docs, multiplayer case studies, and related system design write-ups on this site, so it is easier to map each source back to the part of the article it informed.

Creative tool architecture and rendering
These sources describe the high-level rendering and architectural choices behind large browser-based design tools.

Building a professional design tool on the web for the high-level rendering and architecture tradeoffs behind large browser-based design tools.
Konva.js for a practical retained-scene abstraction over HTML Canvas.
Hit Region Detection for HTML5 Canvas and How to Listen to Click Events on Canvas Shapes for canvas hit-testing patterns when the browser does not provide per-shape events.
html2canvas for one common client-side export approach when the editor surface is DOM-based.
Editor state, actions, and layout systems
These references back the command and state flow and layout primitives discussed in the article.

Flux in-depth overview for the unidirectional action and dispatcher model used as a reference for command and state flow in the article.
React Native for an example of separating a platform-agnostic UI model from the underlying rendering target.
Yoga layout engine for a production-tested flexbox-style layout engine that is relevant to frame and auto-layout discussions.
Plugins and extension architecture
These docs describe the plugin runtime, manifest, and launch principles that inform the extension architecture.

Figma Developer Docs: Introduction / Plugins for the overall plugin model, editor targeting, and user-invoked execution model.
Figma Developer Docs: How Plugins Run for the sandboxed runtime, optional iframe UI, message passing, and lifecycle behavior.
Figma Developer Docs: Plugin Manifest for manifest-declared entrypoints, commands, relaunch buttons, document access, and network restrictions.
Figma Blog: Plugins are coming to Figma for the launch principles around API stability, security, and performance.
Figma Blog: Introducing Figma Plugins for examples of the kinds of workflows plugins enable in practice.
Collaboration and multiplayer
These case studies describe how shared editing is implemented in production design tools.

How Figma's multiplayer technology works for the operational model behind real-time collaborative editing in complex design files.
Enabling real-time collaboration with RSocket for a case study on the transport and reliability layer behind shared editing at Canva.
Related system design articles
These articles on this site cover adjacent editor and collaboration problems that share parts of the design-tool architecture.

Rich Text Editor for adjacent editor-surface tradeoffs around document models, commands, and browser editing constraints.
Collaborative Spreadsheet (Google Sheets) for another example of an operation layer shared by local UI and later collaboration or persistence adapters.
Collaborative Editor (Google Docs) for the deeper discussion of optimistic local edits, revisioning, and real-time conflict resolution.
#### The Component Library Procrastinator Tee — Why reuse when you can rebuild?