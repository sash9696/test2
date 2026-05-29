---
title: "Rich Text Editor"
slug: "rich-text-editor"
difficulty: "hard"
duration_mins: 40
tags:
  - system-design
  - performance
  - ui-component
premium: true
summary: "Design a rich text editor component used on websites like Medium and Gmail."
free_preview: false
status: draft
---

# Rich Text Editor

Design an extensible rich text editor component that allows users to create and edit text with various formatting options.

## Real-life examples

Lexical
Tiptap
Slate
Quill
Draft.js (predecessor of Lexical)
Rich text editors are also commonly known as WYSIWYG editors ("What you see is what you get"), as you can directly format text without using special markup or syntax.

This writeup goes deeper than a typical interview would demand
This writeup delves deeply into rich text editing, covering more than what is typically required for interviews. However, the concepts discussed here are valuable beyond rich text editing, providing insights that can enhance your ability to design complex UI applications.

## Requirements exploration

We scope the editor by clarifying which formatting features are in play, how content is styled and persisted, the target devices, and whether collaboration is part of the exercise.

#### What formatting features should be supported?

Block formatting: headings, paragraphs, blockquotes
Inline formatting: bold, underline, italics
#### How should the various elements be styled?

The browser's default stylesheet is a good starting point. If time permits, we can discuss how to allow further customizability of the elements.

#### Should the editor support media insertion?

The focus is on text formatting, but we can discuss inserting media objects like images, videos, etc.

#### What keyboard shortcuts should be supported?

The editor should support the common editing-related keyboard shortcuts (e.g. Copy, Cut, Paste, Redo). As a bonus, we can also explore customization and support for user-defined shortcuts.

#### What are the non-functional requirements?

Performance and user experience are important. The editor should reflect updates without lagging/stuttering.

![Overview of the Lexical editor](rich-text-editor-pdf-assets/figures/overview-of-the-lexical-editor.png)

There are many rich text editors around, and many editors are similar in design and implementation. Lexical is Meta's latest open source rich text editor library (an evolution of Draft.js) and is used across their products – the composer on facebook.com, workplace.com, etc. It is designed with extensibility, reliability, accessibility, and performance in mind.

Lexical's architecture allows developers to create customized text editing experiences that can scale in both size and functionality, making it suitable for a wide range of applications from simple text inputs to complex document editors.


For this article, we have studied Lexical's design and code in close detail, and our content is heavily based on Lexical's design.

Here's an overview of Lexical's design:

Editor instances: Lexical creates editor instances that attach to a single contenteditable element. Each instance manages its own state and updates.
Editor states: The framework uses a set of editor states to represent the current and pending states of the editor at any given time. This allows for efficient updates and undo/redo functionality.
Dependency-free core: Lexical is a lightweight, 22KB dependency-free library that can work with vanilla JavaScript or integrate seamlessly with other libraries like React.
Plugin architecture: One of Lexical's most powerful features is its plugin system. Plugins are independent and plug-and-play, allowing developers to extend functionality without affecting the core editor.
Node-based structure: Lexical uses a node-based structure to represent the content. Different types of nodes (e.g., text nodes, paragraph nodes) make up the document tree.
State management: When changes occur in the editor, Lexical updates its internal state. Plugins can register listeners to react to these state changes.
Serialization: Lexical can serialize its state to JSON, making it easy to save and restore editor content.
Framework-agnostic: While Lexical provides React bindings, it's designed to be framework-agnostic, allowing integration with other JavaScript frameworks like Vue.js, Angular, Svelte, etc.
Extensible: Simple, flexible core APIs are exposed to allow community-driven external plugins.
Collaboration support: Lexical can be extended to support real-time collaboration through plugins that translate editor state changes to and from collaborative protocols like Yjs.
Accessibility: The framework is built with accessibility in mind, following WCAG best practices and ensuring compatibility with screen readers and other assistive technologies.
Performance optimization: Lexical is designed for performance, handling large documents and complex editing operations efficiently using a virtual DOM-like approach.
Approach
As with most problems in Computer Science, there are multiple approaches to building a rich text editor, each with its own pros and cons.

As a reminder, rich text editors need to support:

Rendering rich text: Headings, paragraphs, bold, etc.
Rendering of cursors: Indication of the current typing position
WYSIWYG: What-you-see-is-what-you-get. Inline updates and preview of rich text
Cross-browser: Consistent experience across different browsers
To get a better sense of the various features a rich text editor supports, along with the user experience, try out Lexical's playground.

Rendering approaches
Let's consider the following approaches.

1. Form elements like <input> and <textarea>
This is an obvious no-go because, although these elements can render text and update inline, they only render plain text and are unable to render formatted text.

2. DOM with fake cursors
A regular DOM, containing the formatted text using HTML elements and styling, can be used to meet the rendering requirements.

However, all editors need to display a cursor to indicate the typing position. By using the regular DOM, cursors have to be faked by rendering a DOM element on top of the existing text. Rendering fake cursors isn't trivial at all:

Cursor position calculation: Cursor positioning has to be manually calculated. You have to know which character is being clicked on (not supported natively), then find the nearest position that is around/between characters. Adding to the complexity, different characters have different widths depending on the font size and family.
Cursor height calculation: The cursor has to match the height of the current line.
Rendering fake cursors involves a ton of complex text measurement that you have to implement yourself. This is supposed to be the browser's job, and you're reimplementing it!

Do not hand-roll cursor rendering on top of a regular DOM tree
Cursor positioning and line-height measurement are things the browser already solves for editable elements. Reimplementing them pixel by pixel is a deep rabbit hole that rarely pays off in an interview-scale design, so prefer contenteditable (or canvas, if you truly need it) so that text measurement stays the platform's problem, not yours.

3. contenteditable attribute
The contenteditable attribute to the rescue! Elements with the contenteditable="true" attribute are editable by the user. This attribute is available on every element. If you aren't familiar with the contenteditable attribute, read more about it here.

On top of the usual editing behavior and keyboard shortcuts, it even supports rich text formatting! You can use Ctrl/Cmd + B to bold selected text, and the text will be wrapped in a pair of <b> tags. The shortcuts for underline and italics are supported as well.

This is editable and format-able text thanks to contenteditable. You can even format the text (e.g. bold by using Ctrl/Cmd + B).

#### This seems to be a good starting point. What's the catch?


Limited formatting: Only inline text formatting is supported. No built-in way to add other elements like headings, blockquotes, etc.
Browser inconsistencies: Different browsers implement contenteditable functionality differently, leading to inconsistent behavior across platforms. This inconsistency can affect the user experience and result in unexpected formatting issues.
Unsafe format: The contents are essentially HTML. By storing the contents as HTML, you could be introducing security vulnerabilities like XSS (cross-site scripting) attacks.
Never persist raw contenteditable HTML as your document format
Round-tripping HTML straight from the DOM to your backend and back means any pasted <script>, javascript: URL, or dangerous attribute can be re-rendered for other users. Store a structured model instead (tree of typed nodes, JSON), and sanitize aggressively on both paste-in and any HTML export path before it reaches innerHTML or dangerouslySetInnerHTML.

However, contenteditable fulfills a lot of our desired rich text editor requirements, especially cursor rendering. Can we use contenteditable and patch the weaknesses and limitations? Yes! In fact, most popular rich text editors on the web are built on top of contenteditable.

4. Canvas with custom… everything
An advanced approach is to use a <canvas> element and render everything within it – styled text, layout, cursor, etc. This approach basically sidesteps a lot of what the browser provides and requires reimplementing everything within a canvas context. You can view it as an even tougher version of using DOM with fake cursors.

This approach is used by one of the world's most widely used document editors – Google Docs. In 2021, Google announced that Google Docs would be moving towards a canvas-based rendering approach to "improve performance and consistency in how content appears across different platforms".

Canvas renderers provide you with absolute full control over the rendering, but come at the cost of:

Complexity and maintenance burden: As mentioned, you'll have to implement layout, text rendering, cursor rendering, user inputs, etc., yourself.
Accessibility: Canvas-based editors have poor accessibility by default since screen readers cannot read what is within the <canvas>. Google had to implement accessibility features themselves on top of everything else that this approach required.
Incompatible with some existing extensions: Browser extensions that rely on inspecting and manipulating the contents in the DOM will cease to function since the contents are entirely within the <canvas> element.
Browser engines have to solve a much more general problem than rich text, and thus have extra overhead. Using canvas is a more performant approach, but few companies like Google have the resources to justify the performance benefits vs. the implementation cost. VS Code is another popular tool that uses canvas for rendering text, but only for the terminal.

#### Which rendering approach to choose?

| Approach | Rich formatting | Cursors | Implementation effort | Cross-browser |
| --- | --- | --- | --- | --- |
| <textarea> | No | Supported | N/A | Generally same |
| --- | --- | --- | --- | --- |
| DOM with fake cursors | Supported | Custom | High | Generally same |
| --- | --- | --- | --- | --- |
| contenteditable | Supported | Supported | Moderate | Some differences |
| --- | --- | --- | --- | --- |
| <canvas> | Custom | Custom | Very high | Generally same |
Given the tradeoffs of features vs. implementation effort, most rich text editor libraries use the contenteditable attribute on a root element and add event listeners to intercept browser events and standardize the result across browsers. With a contenteditable approach:

Default to contenteditable plus a custom model, not canvas
In an interview, the safe default is to build on contenteditable for rendering and cursors, while owning an editor-internal content model separately. Reserve canvas-based rendering for cases where you genuinely need Google Docs-level control over layout and cross-platform consistency, and call out the accessibility and extension-compatibility costs when you do.

Rendering rich text: This is available for free, since contenteditable renders using the various DOM elements.
Rendering cursors: Also available for free. The cursor's line height and positioning match the currently selected text.
Inline text updates: Text can be updated just like when using <input> and <textarea>. Some keyboard shortcuts are available for typical editing and even formatting (e.g. bold, underline).
Consistent experience across different browsers: Some browsers respond to certain user events while others don't. However, we can add event handlers for all these possible events (e.g. keystrokes, selection, clicks), intercept them, and normalize the behavior across browsers, operating systems, and devices.
Handling limited formatting: By default, only inline formatting is supported. However, since the result is DOM-based, we can programmatically insert any element necessary and style it for the desired result.
Backed by a different content model: While the rendered content result is DOM/HTML, the underlying model can be customized for performance optimizations and restricted to only certain supported elements.
Widely used: Some of the world's most widely used rich text editing surfaces, such as facebook.com's post composer, Gmail composer, and Medium's article editor, use contenteditable.
Model and state design
Now that we have decided on the rendering approach, let's discuss how we want to model the content state. Is content state even necessary? Can we just let the state be in the DOM and operate on the DOM nodes directly? It's certainly possible, but there are some issues with that:

No built-in state management features: The DOM does not provide built-in features for state management, such as undo/redo, or allow the use of complex data structures, which are sometimes necessary.
No good way of storing extra related fields: If extra fields are required for certain elements (e.g. type of custom elements), they have to be stored as data attributes on the DOM nodes, which can clutter up the DOM and adds overhead when they have to be read.
DOM is bloated: DOM elements are huge and have a ton of properties and methods. This can be bad for maintenance, and it's better to keep the API surface small.
Prone to tampering: Browser extensions often mess with the DOM and have unwanted consequences. By using the DOM as the underlying content model, third-party extensions can easily tamper with the source of truth.
Do not treat the DOM as the editor's source of truth
The DOM is what you render, not what you store. A separate in-memory model gives you undo/redo, serialization, headless testing, and, critically, protection from browser extensions and pasted markup quietly rewriting state. Every production-grade editor (Lexical, Slate, Tiptap/ProseMirror, Quill) keeps its own model and reconciles to the DOM.

A better approach is to model the content state in JavaScript and render to the DOM based on the content state. This content state will need to capture tree-like relationships, just like the DOM. An example could be:


{
type: "document",
children: [
{
type: "heading",
children: [{ type: "text", content: "Hello" }]
},
{
type: "paragraph",
children: [
{ type: "text", content: "This is John Doe. " },
{ type: "text", content: "He is strong", bold: true }]
},
]
}
The editor will iterate through this object, convert each entry to the relevant DOM nodes (e.g. heading -> <h1>, paragraph -> <p>), and render it to the screen.

Operating on a state model in JavaScript without relying on the DOM brings the following benefits:

Portability: The same data can be rendered on different platforms using platform-specific UI primitives.
Headless mode: Tests are easier to write, and state manipulation can be done on the server without an actual browser.
During initialization:

The editor initializes all required states (content state, selection state) in JavaScript.
The editor loops through the content state and renders it to the DOM.
Update approach
Updates to the editor will modify the underlying state model and result in DOM updates. However, how do we know what DOM updates need to be made? Do we blow away the entire DOM and re-render from scratch? That works but sounds wasteful and isn't great for performance, especially if the content is long and there are a lot of DOM nodes.

Can we just update the parts of the DOM that need to be updated? Here's how:

The editor intercepts the user event (e.g. input, keypress).
The editor converts the event into a supported command (e.g. inserting a character, deleting a character, etc.).
The editor makes a clone of the content state.
The editor modifies the cloned content state.
The editor compares the original content state with the cloned content state to determine the DOM changes required (also known as reconciliation).
The editor modifies the DOM based on the determined changes.
Such an update flow is similar to how React maintains a virtual DOM and only makes the minimum necessary DOM changes when updating based on the new state. This process is called reconciliation.

Lexical's DOM reconciler is independent of React
Lexical implements its own DOM reconciliation process that is independent of React. This allows Lexical to be UI-agnostic and used within React, Vue, Angular, and Svelte apps.

## Summary

In summary, most rich text editors on the web use contenteditable and take the following high-level approach:

Design an editor-specific model of the document content
Create a mapping between the model and DOM elements
Define a set of supported operations on this model (e.g. insert text at position, delete text, format text)
Translate user events (keypresses and mouse clicks) into a sequence of these supported operations
Update the DOM based on these operations, ideally with minimal DOM manipulation calls
We've briefly discussed the high-level approach. Covering the above is probably sufficient in the context of a 45-minute front end system design interview.

However, for a holistic understanding of how modern extensible rich text editors are designed, we recommend going through the rest of the article which dives deeper into the architecture, data entities, and available APIs.

Cover the five-step high-level approach before going deeper
In a 45-minute interview, getting the model, mapping, operations, event-to-operation translation, and DOM reconciliation on the board is usually the whole job. Everything below, including custom nodes, transforms, listeners, undo coalescing, and collaboration, is follow-up depth to reach for only once the core flow is clear.

## Architecture / high-level design

Rich text editor architecture

Editor
The editor consists of two parts: (1) the core, and (2) any additions added by developers (listeners and transformers).

Core
This rich text editor design is very extensible. The core contains the bare minimum logic needed for a user event -> state update -> DOM updates operation, while allowing for extension by providing commands that can be invoked from outside.

Breaking down the update flow:

Commands can be dispatched in multiple ways. They can come from:
DOM events on the content editor: These originate from a user interacting with the content DOM element (typing, deleting, moving the cursor, etc.). DOM events on the contenteditable DOM element are mapped to recognized commands (see LexicalEvents.ts). Refer to Lexical's list of supported commands that can be dispatched.
Programmatically via the toolbar: Dispatched programmatically (from the toolbar or within listeners/transforms), e.g. to change formatting or insert/remove nodes.
The editor receives the commands, makes a clone of the current content state (PendingEditorState), and updates the content state as per the command (see LexicalUpdates.ts). Both previous and current editor states are necessary for the DOM reconciliation process.
The reconciler receives both the current content state (ActiveEditorState) and the modified content state (PendingEditorState) and compares the two to create/update/remove the relevant DOM nodes corresponding to the new editor state. The contents within the contenteditable DOM element will then be modified (see LexicalReconciler.ts). The reconciler serves as a communication layer between the content state and the rendered UI (a contenteditable element). If a <canvas> UI approach is desired (like in Google Docs), most of the required changes will lie in the reconciler.
contenteditable DOM events

Toolbar / programmatic

Command dispatcher

Update: clone + mutate PendingEditorState

Transforms on dirty nodes

Reconciler: diff Active vs Pending

contenteditable DOM

Listeners


Editor core pipeline from input sources to rendered DOM
Listeners
Listeners are a mechanism that allows the editor to inform the developer when a certain operation has occurred. Since commands are dispatched as part of the editor update flow, developers can add listeners for these commands (you can view them as events) and react accordingly.

The API of listeners will be discussed in the deep dive section below.

Listeners are interview-optional but production-essential
Listeners are typically beyond the scope of rich text editors in the context of a system design interview, but they are essential for making powerful, extensible, and customizable rich text editors.

Transforms
Transforms are a mechanism to respond to changes within the editor before reconciliation happens. Multiple transforms still lead to a single DOM reconciliation pass.

The API of transforms will be discussed in the deep dive section below.

Transforms are interview-optional but production-essential
Transforms are beyond the scope of rich text editors in the context of a system design interview, but they are essential for making powerful, extensible, and customizable rich text editors.

Input UI
Important elements of a rich text editor UI.

Content element
The contenteditable DOM element. The editor adds event listeners (e.g. input, click, keydown) to this element and responds to these events by mapping them to its internal commands (e.g. FORMAT_TEXT, SELECTION_CHANGE, INSERT_LINE_BREAK) and dispatching them so that the state can be updated.

Toolbar element
The toolbar at the top of almost every rich text editor, containing the formatting buttons. It directly tells the editor to dispatch commands. Note that the toolbar UI is left entirely to the developer to customize. The rich text editor simply provides commands for the toolbar to dispatch programmatically, which is independent of the appearance.

Headless mode
The proposed architecture allows the rich text editor to be used in a headless fashion. The editor can be initialized with some existing content, and commands can be dispatched programmatically to modify the editor state. The only difference is that there's no need for a reconciliation step because there's no DOM to update.

Headless mode is useful for writing tests for the internals or manipulating the editor state on the server.

## Data model

There are a few core entities within rich text editors – the editor instance, which contains editor states, content states (nodes), and selection state. All the Editor-related interfaces/classes will be prefixed with Editor to disambiguate them from DOM types with the same name.

Editor and EditorState
The core data model for a rich text editor is the Editor. There can be multiple rich text editors on the page and each contenteditable DOM element is associated with one Editor instance, where each instance contains its own content state, selection state, transforms, listeners, customizations, etc.

In TypeScript, the core entities resemble:


// https://github.com/facebook/lexical/blob/71880d22f63a8f2631352a7032a22f639238b306/packages/lexical/src/LexicalEditor.ts#L562
interface Editor {
// Reference to contenteditable DOM element.
rootElement: HTMLElement | null;
// Both current and pending EditorStates.
editorState: EditorState;
pendingEditorState: EditorState | null;
// Non-core fields are omitted.
}

interface EditorState {
// Primarily contains content (nodes) and selection state.
nodes: EditorNodes;
selection: EditorSelection | null;
}

interface EditorNodes {
// Exact structure to be discussed.
}

interface EditorSelection {
// Exact structure to be discussed.
}
Content state (nodes structure)
The most important entity to discuss is the content state – how to design the state such that it can store rich text formatting consisting of block and inline elements, styling, custom nodes, etc.

The most flexible format to use is a tree-like structure. Since the DOM is a tree, a tree structure definitely works. However, let's discuss the possible formats and their tradeoffs.

Nodes
Notice that most text-centric documents (like the current document you're reading) consist of blocks of headings and paragraphs, and within each block-level element, some contain custom inline elements for formatting, like bold, italics, underline, etc. Nodes are discrete pieces of state that hold contents and have a hierarchical relationship with each other. The content state contains heading nodes, paragraph nodes, text nodes, etc.

#### Let's first test your understanding of the DOM. In the following HTML, how many DOM Nodes are being created?


<p>Hello <strong>World</strong></p>
If you answered 2, you are mistaken. Text in the browser has to be rendered using Text nodes, so there are actually 4 nodes:

Paragraph element (<p>)
Text node containing "Hello "
Strong element (<strong>)
Text node containing "World"
Note that the DOM Element interface extends from the DOM Node interface. Elements are also Nodes.

More accurately, the created DOM looks like this:


<HTMLParagraphElement>
<Text>Hello </Text>
<HTMLStrongElement>
<Text>World</Text>
</HTMLStrongElement>
</HTMLParagraphElement>
One reasonable data structure for content state that closely resembles the DOM is to have ElementNodes and TextNodes that inherit from EditorNode:

ElementNodes can contain children, which can be other ElementNodes or TextNodes.
TextNodes can only contain text. Since they cannot contain any other nodes, they are leaf nodes.
These nodes can be subclassed, e.g. HeadingNode and ParagraphNode inherit from ElementNode.

Rich text editor tree

Editor node types are not DOM Element and Text
These are custom interfaces/classes implemented within the rich text editor, not to be confused with the DOM Element and Text.


interface EditorNode {}

interface ElementNode extends EditorNode {
children: Array<EditorNode>;
}

interface TextNode extends EditorNode {
text: string;
}
With a structure resembling the DOM, we are able to represent all sorts of possible rich text content and formatting.

Tree representation: How are these nodes represented within EditorState? An intuitive way is to mirror the resulting DOM in JavaScript – a 1:1 tree representation using an ElementNode subclass as the root node.


{
type: "root",
children: [
{
type: "heading",
children: [
{ type: "text", text: "Hello world" },
]
},
{
type: "paragraph",
children: [
{ type: "text", text: "Lorem " },
{ type: "text", text: "ipsum", format: "bold" }
]
}
]
}
Many rich text editors (e.g. Slate.js, Draft.js) use this tree-like structure. It works, but trees aren't the most efficient data structure for frequent updates. For one, trees do not provide efficient access to their nodes; accessing deep leaf nodes would require traversal of the tree starting from the root.

#### Map with child pointers: How about using a Map of EditorNodes that contains pointers to other nodes?


```typescript
type NodeKey = string;

type NodeMap = Record<NodeKey, EditorNode>;

interface ElementNode extends EditorNode {
```
children: Array<NodeKey>;
// Can contain other fields, depending on the node type.
}

interface TextNode extends EditorNode {
text: string;
}
Rich text editor treemap

This NodeMap gives the same tree as before:

Rich text editor treemap

Using this structure, obtaining a reference to an EditorNode is a matter of doing nodeMap.get(nodeKey).

Map with children as linked list: Lexical started out using a Map of EditorNodes with a children pointer array. Eventually it moved to doubly linked lists (instead of arrays) for its child node pointers as a performance optimization, along with parent pointers. All EditorNodes can have siblings and parents. ElementNodes have additional references to their first and last child, if they exist.


```typescript
type NodeKey = string;

type NodeMap = Record<NodeKey, EditorNode>;

interface EditorNode {
```
parent: NodeKey | null;
prev: NodeKey | null;
next: NodeKey | null;
// Can contain other fields, depending on the node type.
}

interface ElementNode extends EditorNode {
firstChild: NodeKey | null;
lastChild: NodeKey | null;
// Can contain other fields, depending on the node type.
}

interface TextNode extends EditorNode {
text: string;
}
Rich text editor linked list

This NodeMap gives the same tree as before, but with more pointers between nodes.

Rich text editor linked list nodes

Let's look at the advantages and disadvantages of this map + linked list structure:

Advantages

Single-layer Map, extremely easy to clone both shallowly and deeply
O(1) access to any EditorNode if you know the NodeKey
Efficient removal and addition of child nodes because children are in a doubly linked list
Provides access to the parent node (not specific to the linked list implementation)
Reparenting of nodes is a matter of updating pointers
Disadvantages

Updates made to the content require careful updating of pointers on multiple EditorNodes, similar to linked list manipulation
Knowing the size of the children requires counting. This can be mitigated by maintaining a cached size field on ElementNodes that is updated whenever children are added/removed
Getting all the children of a node requires traversal from the first node to the last node
Formatting
One tricky issue with formatting text in the DOM is when characters have more than one format applied. E.g. to render the text Tarzan and Jane. Note that:

"Tarzan" is bolded
"and" is both bolded and underlined
"Jane" is underlined
Nested tags: In HTML/DOM, there are multiple ways to achieve this formatting. Here are some possible ways:

<strong>Tarzan </strong><u><strong>and</strong> Jane</u>
<strong>Tarzan <u>and</u></strong><u> Jane</u>
<strong>Tarzan </strong><strong><u>and</u></strong><u> Jane</u>
The main downside of this approach comes from there being many ways to render the HTML for the formatted text, and many possible ways to nest when the text has even more formatting, e.g. italics or strikethrough.

The first two ways use nested tags for formatting, and it can become a problem when editing already-formatted text to add/remove formatting: the editor needs to introduce more elements for the formatting, or remove the right elements (potentially even combining them) when formatting is removed. Given the number of possible combinations for composite formatting, it is quite complex to implement editing correctly.

contenteditable is able to handle the formatting correctly, but the resulting HTML is messy and not optimized.

Pick one canonical representation for overlapping formats
With multiple formats like bold plus underline, there are many valid HTML nestings and many ways for an editor to "drift" between them on edit. Both flat text runs with a format bitmask (Lexical, Slate) and explicit format ranges are fine choices, but the model must pick one and normalize on write; otherwise add/remove-format operations become a combinatorial mess and diffs look unstable.

Format ranges: Another way is to use format ranges (not to be confused with selection range), where each format is labeled with start/end indices. Tarzan and Jane would be:


[
{ start: 0, end: 10, format: 'bold' },
{ start: 7, end: null, format: 'underline' },
];
This list-like format is definitely convenient and more readable. But ultimately, to be rendered to the DOM, the list has to first be converted into element and text nodes. Updates to the text will require computation for updating indices correctly.

Lexical's formatting approach: Let's take a closer look at the last option in the HTML approach:


<strong>Tarzan </strong><strong><u>and</u></strong><u> Jane</u>
This approach refrains from having multiple tags of the same level within a tag – nested tags will never have sibling HTML tags or sibling text nodes. This is Lexical's and Slate.js's approach for formatting text.

Although this approach results in the most verbose DOM, the structure is flat and can be represented using a flat list of TextNodes:


<TextNode text="Tarzan " format={["bold"]} />
<TextNode text="and" format={["bold", "underline"]} />
<TextNode text=" Jane" format={["underline"]} />
When editing the formatting of the text, the following scenarios are possible:

Adding formatting to text: A TextNode is split into two or more TextNodes, with some of them having the new formatting.
Removing formatting from text: The editor simply has to look at the surrounding TextNodes, determine if the formatting is exactly the same, and combine them into a single TextNode if so.
Both Lexical and Slate.js use a similar content structure that supports block and inline styling, elements, and text nodes:

Content is broadly represented by ElementNodes and TextNodes
ElementNodes can contain children, which can be other ElementNodes or TextNodes
All visible text in the rich text editor is contained within TextNodes, which are leaf nodes
Only TextNodes have formatting, since formatting is only relevant to text
These only cover the core fields needed for basic functionality. More fields will be added as we dive deeper into specific topics below.

Selection state
In web browsers, the "selection state" refers to the portion of a web page that the user has highlighted or selected. This is usually done by clicking and dragging the mouse over text or other content, causing it to be highlighted.

Let's first get a better understanding of the Selection object in browsers as part of the DOM. Here are the key components:

Selection object: The Selection object is part of the DOM and provides access to the selection state. It can be accessed via window.getSelection() in JavaScript. The Selection object has methods and properties to manipulate and retrieve information about the selection.
Anchor and Focus: The Selection object has two primary points: the anchor and the focus. The anchor is the starting point of the selection. The focus is the endpoint of the selection. These points can be either a text node or an element node, along with an offset value, a number representing the offset of the selection within the node. Thus, the anchor and focus points are a combination of references to the anchorNode/anchorOffset and focusNode/focusOffset pairs.
The selection object has the following important properties:


interface EditorSelection {
anchorNode: EditorNode | null;
anchorOffset: number;
focusNode: EditorNode | null;
focusOffset: number;
type: 'caret' | 'range';
}
If one or more characters are selected, then the type will be Range; otherwise, it will be Caret and the anchorNode/anchorOffset values will be exactly the same as focusNode/focusOffset.

Selection wraps one or more Range objects under the hood
Strictly speaking, the Selection object contains one or more Range objects. A Range object represents a contiguous part of the document, starting and ending at specific points (similar to the anchor and focus points on the Selection object). There's a long history to the Range object if you're interested, but you don't have to bother with it for the sake of understanding selections.

Selection can be backwards
It's also important to note that selection direction matters! Selection can be backwards, and the focusNode can appear before the anchorNode. This happens when users highlight text from right to left, so we cannot assume the focusNode will always appear after the anchorNode.

This is important to remember because users who primarily navigate using the keyboard might experience difficulties if the selection direction is not taken into account. For instance, the Shift + Right Arrow keys should extend the selection if the selection order is forward, but shrink the selection if the selection order is backwards.

Anchor is not always "start" and focus is not always "end"
This is a surprisingly common source of bugs because offset math, shift-arrow handlers, and cursor-rendering code all tend to assume anchor <= focus in document order. Always derive an explicit (start, end) pair from the selection before running range logic, and preserve the original anchor/focus for keyboard extension to still feel correct.

How selection works in Lexical
The Editor maintains its own Selection state that mirrors document.getSelection() but points to its own EditorNodes (within the node map) instead of DOM nodes. It does so by listening to DOM events like selectionchange on the document or arrow keypresses within the content UI. However, since selectionchange events are global, the event handler is shared between all editor instances on the page, so it needs a way to know which contenteditable is being modified by tracing the parent chain of the selection nodes, or if none of the contenteditables are in focus.

Summary
Here is an overview of the data within Editor instances, closely following Lexical's internal state model. To render the content to the DOM, the editor renders recursively from the root node by rendering the node, then iterating through the children starting from the firstChild pointer and the next field of each subsequent child.

editorState

pendingEditorState


NodeKey -> EditorNode

parent / prev / next

firstChild

lastChild

is-a (type=element)

is-a (type=text)

anchorNode

focusNode


HTMLElement

rootElement


NodeKey

rootNodeKey


anchorOffset


focusOffset


NodeKey


NodeKey


NodeKey


NodeKey

firstChild

NodeKey

lastChild


Editor state entity relationships

interface Editor {
// Reference to contenteditable DOM element.
rootElement: HTMLElement | null;
// Both current and pending EditorStates.
editorState: EditorState;
pendingEditorState: EditorState | null;

// Non-core fields are omitted.
}

```typescript
type NodeKey = string;

interface EditorState {
```
// Primarily contains content (nodes) and selection state.
nodes: EditorNodeMap;
rootNodeKey: NodeKey;
selection: EditorSelection | null;
}

interface EditorNode {
parent: NodeKey | null;
prev: NodeKey | null;
next: NodeKey | null;
}

interface ElementNode extends EditorNode {
type: 'element';
firstChild: NodeKey | null;
lastChild: NodeKey | null;
// Can contain other fields, depending on the node type.
}

interface TextNode extends EditorNode {
type: 'text';
text: string;
format: string[];
}

```typescript
type EditorNodeMap = Record<NodeKey, EditorNode>;

interface EditorSelection {
```
anchorNode: EditorNode | null;
anchorOffset: number;
focusNode: EditorNode | null;
focusOffset: number;
type: 'caret' | 'range';
}
## Interface definition (API)

The API is organized by the core abstractions: the Editor instance that owns the state, the node types that compose a document, and the helpers for selection, commands, and serialization.

Editor APIs
The following APIs are for Editor instances and heavily reference Lexical's Editor API, with some tweaks. There are some minor differences, but they are not crucial.

Initialization
The first API to discuss is the initialization of Editor instances. The Editor constructor can expect the following parameters.

| Name | Type | Description |
| --- | --- | --- |
| rootElement | Element | HTML element to mount the rich text editor |
| --- | --- | --- |
| namespace | string | Unique identifier for each editor instance |
| --- | --- | --- |
| initialEditorState | EditorState | Initial editor state – content nodes and selection |
| --- | --- | --- |
| nodesConfig | Array of EditorNode classes and subclasses | Array of EditorNode classes and subclasses that are supported by this editor |
| --- | --- | --- |
| theme | ThemeObject | Mapping of node types to CSS classes. See section on theming below |
Since there can be multiple rich text editor instances on the page, it's useful to allow specifying a namespace so that developer-created listeners can identify which instance is being edited.

See the full list of editor constructor arguments here.

Update APIs
While most updates will happen because of users editing the content directly within the editor, it's useful to provide programmatic ways of triggering updates.

editor.update(callback: Function): A method that receives a function containing the current EditorState. Modifications to the EditorState will result in DOM updates. Lexical has a slightly different API, but the motivations are similar.
editor.setEditorState(editorState: EditorState): A method that replaces the current editor state with a new editor state, marks the root node as dirty, and invokes full reconciliation of all nodes.
Serialization and deserialization APIs
editor.getEditorState(): Get the current EditorState as a JavaScript object (not a string). This is useful for passing editor states to other instances.
editor.getEditorStateSerialized(): Get the serialized version of the EditorState as a JSON string. A string representation is necessary for long-term persistence (e.g. localStorage or sending over the network to save within databases).
editor.parseEditorState(serializedEditorState: string): Load editor state from a serialized string obtained from editor.getEditorStateSerialized().
Event-related APIs
editor.dispatchCommand(command: string, payload: unknown): Method to dispatch a command to update the editor, with an optional event-specific payload.
editor.registerCommand(command: string, callback: Function): Register a function that will run when a command is dispatched within the editor. The callbacks have access to the editor instance.
editor.registerUpdateListener(callback: Function): Register a function that will run whenever the editor is updated.
editor.registerNodeTransform(nodeType: string, callback: Function): Register a function that will run when an EditorNode of a specific type is modified during the update phase (happens before reconciliation).
Querying APIs
Notice that there are a number of APIs that accept callback functions which can read and modify the editor state. These callback functions will need a good way to query for EditorNodes, similar to how the DOM's document object has document.querySelectorAll(), document.getElementById(), document.getElementsByTagName(), etc.

editor.getNodeByKey(nodeKey: string): Get the EditorNode given its key.
editor.getElementByKey(nodeKey: string): Get the underlying HTML DOM Element for a specific EditorNode given its key.
EditorNode interface
There are actually more APIs on EditorNodes. Above, we presented a suggested interface for EditorNodes. In Lexical, these base nodes (LexicalNodes) are implemented as classes with properties and methods that do the following:

Tree structure: Properties for maintaining the tree structure and sibling order. E.g. parent, prev, next, etc.
Querying: Methods for querying parent/sibling nodes. These methods abstract the underlying "linked tree" implementation structure. E.g. getParent(), getNextSibling(), etc.
Reconciliation: Methods for creating and updating the DOM elements based on the node's properties during reconciliation (initial render and updates). E.g. createDOM(), updateDOM(), clone().
Serializing: Methods for serializing to a JSON string and deserializing from a string, used for importing/exporting to a persistence layer. E.g. exportJSON(), importJSON().
Since we are able to obtain references to EditorNode instances, it's useful to have APIs to access the surrounding nodes, similar to how the DOM Node interface has properties like childNodes, firstChild, lastChild, parentElement, etc.

These querying APIs on EditorNodes are useful because:

They can be used by developers who want to add custom behavior within editor.update() and the event-related API callbacks.
They are necessary for performing traversal and manipulation of the nodes during updates and reconciliation.
However, as a good software engineering practice, the underlying structure of the EditorNode should not be exposed; developers should not need to know that they are implemented as "linked trees". Hence the APIs should be implemented as methods instead of properties in order to conceal the implementation details.

Exposing the implementation details can result in tighter coupling between library code and developer code, resulting in breaking changes and a terrible migration experience. Lexical migrated from the map with child pointers to the map with linked list implementation with ease because its APIs did not make assumptions about how the internal states were implemented.

Expose node traversal as methods, not as raw pointer properties
getParent() and getNextSibling() let you swap between arrays, linked lists, or any other internal representation without breaking consumer code. Once callers start reading node.next or node.children[0] directly, the internal structure becomes part of your public API, and any future performance refactor turns into a painful major-version migration.

EditorNode APIs:
editorNode.getParent()
editorNode.getNextSibling()
editorNode.insertAfter()
editorNode.insertBefore()
editorNode.isSelected()
editorNode.getTextContent()
Only the important APIs are mentioned here. For more possible APIs, see LexicalNode API docs on Lexical website.

ElementNode APIs:
ElementNode can have child nodes, so additional APIs are provided to query the child nodes.

elementNode.getChildren()
elementNode.getFirstChild()
elementNode.getLastChild()
Only the important APIs are mentioned here. For more possible APIs, see ElementNode API docs on Lexical website.

TextNode APIs:
TextNodes are leaf nodes and they are primarily concerned with text formatting.

textNode.setTextContent()
textNode.setFormat()
textNode.getFormat()
Only the important APIs are mentioned here. For more possible APIs, see TextNode API docs on Lexical website.

## Optimizations and deep dive

The sections above establish the editor model and API. This section dives into the internal mechanisms and follow-up topics that make a modern rich text editor extensible, reliable, and usable:

Update loop in detail: This section explains what happens during editor.update(fn) and how state changes flow through the editor.
Keyboard shortcuts: This section covers input events, formatting shortcuts, command dispatch, and browser editing behavior.
Theming: This section discusses how editor output is styled while keeping document structure independent from presentation.
Custom nodes: This section explains how custom document nodes extend the editor's core model.
Listeners and commands: This section covers internal events, command registration, update listeners, and extension points.
Transforms: This section discusses efficient content normalization and reactions to node-level changes.
Plugins: This section explains how plugins combine commands, transforms, listeners, and custom nodes into reusable editor features.
Serialization & deserialization: This section covers exporting, importing, persistence, and API boundaries for editor state.
Undo/redo state: This section explains how history stacks, grouping, and reversible updates work in a rich text editor.
Real-time collaborative editing: This section discusses how collaborative editing builds on synchronization models such as OT or CRDTs.
Performance: This section covers the design choices that keep editing responsive as documents and plugins grow.
Accessibility: This section discusses keyboard behavior, screen-reader support, labels, and accessible editor chrome.
Internationalization (i18n): This section covers bidirectional text, localized UI strings, and multilingual content editing.
Update loop in detail
In this section, we will explain the entire update loop in detail, aka what happens when editor.update(fn) is called, either via a user DOM event (e.g. input, keypress) or a command dispatched programmatically (e.g. from the toolbar).

Clone state: The editor makes a shallow clone of the content state, which is the nodes map (Map<NodeKey, EditorNode>). A separate Map instance is created, but the values all point to the original EditorNodes.
Process updates: The callback passed to editor.update(fn) is called.
Whenever EditorNodes are modified, they are first cloned before modification.
The cloned content state will be updated to point to the new node instance.
The modified node and its chain of parent nodes are marked as dirty.
Since the callbacks can call editor.update(fn) again, the editor maintains a queue of update callbacks to go through, and the queue is processed in sequence.
Process transforms: All registered transforms are called.
Reconciliation:
If there are any dirty nodes, the reconcile function is called starting from the root node and recursively on all the child nodes.
For dirty nodes, the reconcile function updates the DOM based on the new EditorNodes.
As an optimization, reconciliation can be skipped if both new and old EditorNode instances for a NodeKey are the same and the node is not dirty. That means nothing in the node or its entire subtree was modified.
Process listeners: All registered listeners are called.
Listeners
contenteditable DOM
Reconciler
Transforms (dirty nodes)
PendingEditorState (cloned node map)
Editor (update queue)
Source (DOM event or toolbar)
Listeners
contenteditable DOM
Reconciler
Transforms (dirty nodes)
PendingEditorState (cloned node map)
Editor (update queue)
Source (DOM event or toolbar)
dispatchCommand or editor.update(fn)
Shallow clone node map
Run update callbacks, clone+mutate dirty nodes
Invoke transforms for dirty node types
Further mutations (still before reconcile)
Diff Active vs Pending from root
createDOM / updateDOM / reorder for dirty nodes
Notify registered update listeners


editor.update commit cycle from dispatch to listeners
Reconciliation
Above, we suggested an interface for EditorNodes. In reality, LexicalNodes, which are Lexical's version of EditorNodes, are base classes that are extended by other nodes with the following structure/interface.


// https://github.com/facebook/lexical/blob/main/packages/lexical/src/LexicalNode.ts
export class LexicalNode {
/**
* Called during the reconciliation process to determine which nodes
* to insert into the DOM for this Lexical Node.
*/
createDOM(_config: EditorConfig, _editor: LexicalEditor): HTMLElement {
invariant(false, 'createDOM: base method not extended');
}

/**
* Called when a node changes and should update the DOM
* in whatever way is necessary to make it align with any changes that might
* have happened during the update.
*
* Returning "true" here will cause lexical to unmount and recreate the DOM node
* (by calling createDOM). You would need to do this if the element tag changes,
* for instance.
*
* */
updateDOM(
_prevNode: unknown,
_dom: HTMLElement,
_config: EditorConfig,
): boolean {
invariant(false, 'updateDOM: base method not extended');
}
}
All the LexicalNode classes have to implement a createDOM() and updateDOM() method, used for reconciliation purposes:

createDOM(): Called during the initial render or if the root DOM node changes.
updateDOM(): Called during the reconciliation phase if the EditorNode instance has been modified. This method should contain reconciliation logic – the node should update the DOM's contents if the node's HTMLElement can be reused. If it returns true, createDOM() is called and the original DOM element is replaced by the newly returned DOM element.
They can be used as part of the reconciliation process.

Reconcile from the root: Start the reconciliation from the root of the new nodes.
Node comparison: Call node.updateDOM(). As mentioned, if it returns true, a new element should be created from scratch. Otherwise, the method should have modified the DOM element corresponding to that node.
Child reconciliation: If either the old or new node is an ElementNode, compare the children of the nodes by using the nodes' keys to match old and new children.
Insert: Add new children that do not have a corresponding old child.
Remove: Remove old children that do not have a corresponding new child.
Same position: Call the reconcile function on the old and new nodes.
Different positions: If the order of children has changed, reorder them within the parent DOM element, then call the reconcile function on the old and new nodes.
There are other nuances with regard to reconciliation, but the general idea is captured above.

Keyboard shortcuts
Out of the box, Lexical supports the common keyboard shortcuts like copying, pasting, cutting, bold formatting, etc. via the 'beforeinput' DOM event. Lexical core listens for that event on the contenteditable element and uses the event.inputType value to determine the user's intended action, intercept the event, and dispatch the relevant Lexical commands.

Another benefit of using normalized Lexical commands (e.g. DELETE_WORD_COMMAND, PASTE_COMMAND, etc.) is that developers can focus on the user's intentions and not worry about the different shortcuts on different operating systems, devices, or keyboard layouts.

The code looks something like:


// https://github.com/facebook/lexical/blob/24b58d8129a317f3467a6e81360fef0f042f04e4/packages/lexical/src/LexicalEvents.ts#L525
$contentEditableElement.addEventListener('beforeinput', (event) => {
// Prevent default behavior since Lexical will handle it by modifying the nodes. event.preventDefault();
switch (event.inputType) {
case 'deleteContentForward':
case 'deleteHardLineForward':
case 'deleteSoftLineForward': {
editor.dispatchCommand('DELETE_LINE_COMMAND');
break;
}

case 'formatUnderline': {
editor.dispatchCommand('FORMAT_TEXT_COMMAND', 'underline');
break;
}

case 'formatBold': {
editor.dispatchCommand('FORMAT_TEXT_COMMAND', 'bold');
break;
} // Handle other cases...
}
});
However, the beforeinput event is not standardized across browsers, so Lexical also listens for the keydown event to determine if there is any intention of formatting. Additionally, selection events such as navigation are handled.


// https://github.com/facebook/lexical/blob/24b58d8129a317f3467a6e81360fef0f042f04e4/packages/lexical/src/LexicalEvents.ts#L996
https: $contentEditableElement.addEventListener('keydown', (event) => {
const { key, ctrlKey, metaKey, altKey } = event;
event.preventDefault();

// All the isX() functions are utility functions to determine the
// key combinations pressed with support for
// different browsers and operating systems
if (isMoveForward(key, ctrlKey, altKey, metaKey)) {
dispatchCommand('KEY_ARROW_RIGHT_COMMAND');
} else if (isMoveBackward(key, ctrlKey, altKey, metaKey)) {
dispatchCommand('KEY_ARROW_LEFT_COMMAND');
} else if (isBold(key, altKey, metaKey, ctrlKey)) {
dispatchCommand('FORMAT_TEXT_COMMAND', 'bold');
} else if (isUnderline(key, altKey, metaKey, ctrlKey)) {
dispatchCommand('FORMAT_TEXT_COMMAND', 'underline');
} else {
// Other events
}
});
Developers are also free to attach their own keyboard event handlers using whichever means are suitable, outside of Lexical core. These custom keyboard event handlers can interact with the editor state by using editor.dispatchCommand() to dispatch commands.

Theming
Most rich text editors render vanilla HTML elements and leave the styling to the developer (except for formatting like bold and underlined text), Lexical included. This means that by modifying CSS, developers can style the rich text contents however they please.


<div contenteditable="true" data-lexical-editor="true">
<h1>Hello World</h1>
<p><span>Goodbye Earth</span></p>
</div>

/* Scoped only to Lexical output */
[data-lexical-editor='true'] h1 {
font-size: 32px;
}

[data-lexical-editor='true'] p {
font-size: 16px;
}
Lexical provides another way of theming, which is to accept a customizable theme object that maps EditorNode types to CSS class names.


// Editor constructor/initializer accepts this `theme` object
const theme = {
heading: 'editor-heading',
paragraph: 'editor-paragraph',
text: 'editor-text',
};
When a HeadingNode (subclass of ElementNode) is rendered, the DOM element will include the editor-heading class.

Further reading: https://lexical.dev/docs/getting-started/theming

Custom nodes
Creating custom nodes in Lexical is a matter of extending base nodes like ElementNode, TextNode, etc., and overriding the methods. Within Lexical's core, ParagraphNode already extends ElementNode.

The custom node should also be passed into the Editor constructor as nodesConfig, which is a list of supported EditorNodes in the Editor instance.

Custom node example
The following is an example taken from the Lexical docs on how to create a ColoredTextNode class that has a color property on TextNodes.


export class ColoredNode extends TextNode {
__color: string;

constructor(text: string, color: string, key?: NodeKey): void {
super(text, key);
this.__color = color;
}

static getType(): string {
return 'colored';
}

static clone(node: ColoredNode): ColoredNode {
return new ColoredNode(node.__text, node.__color, node.__key);
}

createDOM(config: EditorConfig): HTMLElement {
const element = super.createDOM(config);
element.style.color = this.__color;
return element;
}

updateDOM(
prevNode: ColoredNode,
dom: HTMLElement,
config: EditorConfig,
): boolean {
if (prevNode.__color !== this.__color) {
dom.style.color = this.__color;
}

return false;
}
}
Further reading: https://lexical.dev/docs/concepts/nodes#creating-custom-nodes

Listeners and commands
Listeners are callbacks triggered on internal Lexical events. One of the most common types is an "update" listener, which is code that gets run after an update is finished. They're useful if you want to do something whenever a change is made, for example, if you want to immediately persist the data or update your user interface to enable and disable the appropriate toolbar buttons. The code inside update listeners should be fairly lightweight, as they are run on every keystroke.


editor.registerUpdateListener(({ editorState }) => {
// Do something with editorState
});
Listeners are designed to be generic, and most of the time you'd want to listen for specific events. That's where commands come in handy. Commands are events that can be dispatched and listened to so that both the editor and external code can respond to them. If you are familiar with the concept of actions and reducers in Redux, Flux, or useReducer in React, the idea is similar.

Commands in Lexical are actually objects that can be imported from Lexical core, but in our command-related code examples, we've used strings for simplicity's sake.

The two main command-related APIs are:

editor.dispatchCommand(command: string, payload: unknown): Method to dispatch a command to update the editor, with an optional payload. It returns a teardown function that can be used to clean up the listener.
editor.registerCommand(command: string, callback: Function): Register a function that will run when a command is dispatched within the editor. The callbacks have access to the editor instance.
Stopping propagation
Something that wasn't mentioned is that the registerCommand's callback can return true to signal to the other listeners that the command has been handled and propagation will be stopped, just like event.propagation().

The following example demonstrates how to handle bold formatting in a custom fashion within the editor by registering a listener for the bold command.


editor.registerCommand('FORMAT_BOLD_COMMAND', (payload) => {
// Custom code to handle bold formatting.
return true;
});
Priority
The 3rd parameter of editor.registerCommand() is a numerical priority value that determines the order in which the command listeners respond to commands.


export const COMMAND_PRIORITY_EDITOR = 0;
export const COMMAND_PRIORITY_LOW = 1;
export const COMMAND_PRIORITY_NORMAL = 2;
export const COMMAND_PRIORITY_HIGH = 3;
export const COMMAND_PRIORITY_CRITICAL = 4;
Listeners registered by Lexical core or its first-party plugins use COMMAND_PRIORITY_EDITOR (the lowest priority) to give developers a chance to intercept the events and customize the behavior, potentially even stopping the propagation.

EDITOR (core default)
NORMAL / LOW listener
CRITICAL / HIGH listener
Editor.dispatchCommand
Caller (toolbar or plugin)
EDITOR (core default)
NORMAL / LOW listener
CRITICAL / HIGH listener
Editor.dispatchCommand
Caller (toolbar or plugin)
Lower-priority listeners and core default are skipped
[Returns true (handled)]
[Returns false]
dispatchCommand(FORMAT_TEXT_COMMAND, "bold")
Invoke highest priority first
true, stop propagation
Invoke next priority tier
Fall through to core default behavior


Command dispatch walks listeners from highest to lowest priority
Further reading: https://lexical.dev/docs/concepts/commands

Transforms
Commands provide a mechanism to respond to content changes via editor.registerCommand(), but listening to commands is not an efficient way to update the content in response to content updates.

While developing Lexical early on, the Lexical team noticed that it was common to want to do an update based on another update. For example, if you wanted to style a hashtag in a certain way, you could build a hashtag node. Then, when the user types a hash symbol, you want to turn it into a hashtag node. But in order to do that, we have to wait for Lexical to finish the update, and then in an update listener, we have to read the new state, go through all the nodes to find out which ones have hash symbols, and change them into hashtag nodes.

Transforms are registered with a certain node type that you care about, and you pass in a closure. Transforms run during an update, after the update has been run but before the reconciler. At this point, we have a list of all the dirty nodes. All the registered transforms that match the node type for the dirty node are called. Each transform then gets a chance to manipulate the editor state before the reconciler is run. So in the hashtag example, we could check for the presence of the hash symbol in the dirty nodes, which is much more efficient than having to check the entire node tree. If we find the hash symbol inside our transform, we can swap out that node for a hashtag node.

Here's an example that makes text bold if it contains the substring "congratulations".


const congratsTransform = editor.registerNodeTransform(TextNode, (textNode) => {
if (textNode.getTextContent().includes('congratulations')) {
textNode.setFormat('bold');
}
});
Transforms are also useful if you want to validate the content the user can put into the document or which formatting the user is allowed to use. For example, if you wanted to disallow bold text but didn't want to go to the trouble of subclassing a text node, then you could build a transform where you check to see if the dirty node is bold, and if it is, you just set it back to not bold.

Further reading: https://lexical.dev/docs/concepts/transforms

Plugins
Unlike many frameworks, Lexical does not prescribe an interface that plugins have to adhere to. In Lexical, plugins are a combination of commands, transforms, and custom nodes.

What might be surprising is that Lexical core does not support rich text formatting by default. Rich text editing is done through a plugin called lexical-rich-text. From the README:

This package provides a starting point for Lexical users by registering listeners for a set of basic commands that cover simple text-editing behavior such as entering text, deleting characters, copy + paste, or changing the selection with arrow keys. It also provides default behavior for rich text features, such as headings, formatted text, and blockquotes.

Lexical's rich text plugin code contains definitions of custom nodes like QuoteNode, HeadingNode, and many editor.registerCommand() calls to add the above-mentioned editing behaviors. Lexical Playground is built using this rich text plugin.

Further reading: https://lexical.dev/docs/react/plugins

Serialization & deserialization
In the API section, it was mentioned that Editor instances have methods to export and import the EditorState. How is this implemented in practice?

Recap of the APIs:

editor.getEditorStateSerialized(): Get the serialized version of the EditorState as a JSON string. A string representation is necessary for long-term persistence (e.g. localStorage or sending over the network to save within databases).
editor.parseEditorState(serializedEditorState: string): Load editor state from a serialized string obtained from editor.getEditorStateSerialized().
As there are many kinds of nodes (EditorNode and its subclasses) and even custom nodes, each node type should be responsible for implementing its own serialization and deserialization logic.


class EditorNode {
// Other properties and methods omitted.
exportJSON(): Object { … }

static importJSON(serializedNode: Object): EditorNode { … }
}
In editor.getEditorStateSerialized(), the exportJSON() method of the root node is called, and then recursively for all its children. At the end, the JavaScript object is serialized into a string.

In editor.parseEditorState(), the importJSON() static method of each node class is called, and then recursively for all its children. At the end, the root EditorNode is obtained.


class Editor {
// Pseudo-code for serialization of node
getEditorStateSerialized(): string {
function serialize(node: EditorNode): Object {
const nodeObject = node.exportJSON();
node.getChildren().forEach((childNode) => {
const childNodeObject = serialize(childNode);
nodeObject.children.push(childNodeObject);
});

return nodeObject;
}

const rootNodeObject = serialize(
this.editorState.nodes.get(this.editorState.rootNodeKey),
);
return JSON.stringify(rootNodeObject);
}

parseEditorState(stringifiedEditorState: string) {
function deserialize(nodeObject: Object): EditorNode {
// Converts a node type to an EditorNode class
const NodeClass = getNodeClass[nodeObject.type];
const node: EditorNode = NodeClass.importJSON(nodeObject);
nodeObject.children.forEach((childNodeObject) => {
const childNode = deserialize(childNodeObject);
node.appendChild(childNode);
});
return node;
}

const rootNodeObject = JSON.parse(stringifiedEditorState);
const rootNode = deserialize(rootNodeObject);
this.editorState.rootNodeKey = rootNode.key;
}
}
These JSON serialization methods are also useful for copying and pasting content between editors of the same namespace.

Lexical's serialization features go beyond JSON serialization:

EditorNodes can also implement exportDOM() and importDOM() methods for copying and pasting content between Lexical and non-Lexical editors.
Version numbers can be added to serialized states, and the deserialization logic can access that version to handle backwards-compatible parsing of nodes.
Undo/redo state
Implementing undo/redo functionality in web applications involves managing state changes in a way that allows the user to revert (undo) to a previous state or reapply (redo) a reverted state.

Typical stack-based approach
The typical way to implement undo/redo functionality in applications is to maintain a stack of states and a pointer to the current state. The pointer is initially null when the stack is empty.

Update operation (when stack is empty or pointer is at the top of the stack): Push the new state onto the stack, and set the pointer to the new top of the stack.
Undo operation: Decrease the pointer by one.
Redo operation: Increase the pointer by one.
Update operation (when pointer is not at the top of the stack): Remove all states that are above the pointer, push the new state onto the stack, and set the pointer to the new top of the stack.
This approach of a single state stack can be implemented using arrays or doubly linked lists.

Another approach is to have two state stacks – one for undo and another for redo. Undoing will shift items from the undo stack to the redo stack, and redoing will shift items from the redo stack to the undo stack. Updating with a non-empty redo stack will clear the redo stack.

Update, push state to undo stack

Update, push state (coalesce if continuous typing)

Undo, pop undo, push to redo

Undo, continue popping undo stack

Redo, pop redo, push to undo

Update, push state, clear redo stack

Clear history

Empty

Recording

Undone


Two-stack undo and redo lifecycle
Did you know that plain contenteditable has implicit undo/redo functionality by default? Try it out in the following example by editing the text and using Ctrl/Cmd + Z to undo and Ctrl/Cmd + Shift + Z to redo:

Undo-able and redo-able text.

However, it only works for input editing triggered directly by users, not external DOM updates. Because Lexical intercepts the user events, we cannot leverage this undo/redo functionality.

Lexical history plugin
Lexical provides the @lexical/history plugin to add undo/redo functionality to Lexical editors. It uses the two-stack approach where the stacks contain EditorStates. The plugin adds listeners for the UNDO_COMMAND and REDO_COMMAND commands. Within the listeners, states are shifted between the stacks where relevant, and editor.setEditorState() is called with the new current state.

Structural sharing
One implicit requirement of the values in the undo/redo states is that new states should not affect the old states. If the states are objects that share the same object references (either directly or within their contents), mutating the current state can affect previous states, affecting the undo/redo functionality.

There are a few general strategies regarding states in undo/redo functionality:

Immutable states: Store each state as an immutable snapshot. This ensures that when you undo or redo an action, you always have a reliable and unaltered version of the state to revert to.
Store states as mutable objects: This can be easier to implement, but it requires careful management to avoid unintended mutations. A common approach is to clone the current state to create a new state, then mutate it. Depending on whether the clone is a deep one, the memory usage can be very high due to the large number of duplicated objects.
Immutable states with structural sharing: When a modification is made to an immutable data structure, only the parts of the structure that are affected by the change are duplicated. The unchanged parts are shared between the old and new versions. This significantly reduces the amount of memory required and the time complexity of modifications. Immutable.js and Immer.js are libraries for creating immutable data structures that use structural sharing under the hood.
Draft.js by Meta, which is the predecessor to Lexical, uses Immutable.js to implement its EditorState. However, in recent years, Immutable.js has fallen out of favor due to sentiments that the library is huge and has performance and accessibility issues; hence Meta created Lexical to replace Draft.js.

Lexical's EditorState is immutable but does not use Immutable.js or any other immutable libraries. Structural sharing is implemented in a very clever way. Recall that the EditorState.nodes structure is implemented as a Map of NodeKey (strings) to EditorNode classes. When the editor is being updated, the Map structure is shallowly cloned. Any updated EditorNodes will be cloned and used in the new Map, leaving the nodes in the original Map untouched. This is how Lexical guarantees immutable EditorStates that are great for use in undo/redo stacks.

Undo/redo granularity
Undo/redo granularity refers to the level of detail at which user actions are captured and can be undone or redone. Achieving the right granularity is crucial for providing a good editing experience while maintaining performance and accuracy.

Action-level granularity: Users typically expect certain actions, like typing a word or applying a format, to be grouped together into a single undoable action. Complex actions, such as formatting a large block of text or inserting a table, should be captured as single actions for practicality.
Performance: Extremely fine granularity can lead to performance issues, especially if every single keystroke is recorded separately.
Some common granularity levels include:

Character level: Each individual keystroke is captured. This can lead to very fine-grained control but might overwhelm the undo stack and degrade performance.
Word level: Groups of characters (e.g., words or phrases) are captured. This is more user-friendly for text editing, as users typically think in terms of words rather than characters.
Action level: Groups related actions, such as applying a style or inserting an element, into single undoable actions.
Transaction level: Captures complex multi-step actions as a single unit, often used in collaborative editing or complex document manipulations.
Some ways to implement the various granularity levels:

Debouncing: Debouncing can be used to group closely related actions, such as consecutive keystrokes, into a single undoable action.
Batching by intention: Batching actions over a period or logical group can help manage granularity. For example, formatting one word bold and the next word bold could be batched as a single action of bolding two words.
Lexical calls this batching behavior "coalescing", and continuous typing actions will be coalesced into a single undo action. In Lexical, continuous typing events are limited to these three actions:

Forward typing characters: Normal key presses.
Deleting backwards: Backspace gestures, including backspace key presses.
Forward deletion: Forward delete gestures, including Delete key presses.
Further reading: https://lexical.dev/docs/concepts/history

Real-time collaborative editing
Real-time collaborative editing is complicated because of the following complexities:

Conflict resolution: When multiple users edit the same part of a document simultaneously, conflicts arise. The system must decide which changes to keep or how to merge them.
#### Undo/redo: Should undo and redo revert just your own changes or changes by others too?

Real-time collaborative editing is beyond the scope of rich text editor system design, but Lexical's core model + commands architecture allows for real-time collaboration functionality; the collaboration layer can be built beneath the existing editor core and dispatch commands for the editor core to execute.

For a deep dive into collaborative editing, we recommend reading our entire Collaborative Editor system design article.

Operational Transformations (OT) and Conflict-free Replicated Data Types (CRDTs) are ways of handling simultaneous conflicts in documents. OT resolves conflicts by transforming user operations against peer operations, accounting for the state of the document when the operation was made. CRDTs avoid conflicts altogether with specially designed data structures so that updates can be made in any order and clients will still eventually converge into a consistent state.

Default to CRDTs (Yjs) for a new collaborative rich-text editor
OT is battle-tested (Google Docs) but requires a central transform server and carefully ordered operation history. CRDTs let peers converge without a coordinating authority, have mature browser-ready implementations like Yjs and Automerge, and already ship with Lexical's collaboration plugin, making them the pragmatic default unless you have a specific reason to prefer OT.

Regardless of the conflict-handling approach, there needs to be a networking layer to inform clients about updates made by peers in real-time. Long polling, WebSockets, and Server-sent Events (SSE) are available options, and WebSockets are by far the most popular approach for real-time collaborative editing.

Lexical provides the LexicalCollaborationPlugin within the @lexical/react package to enable collaborative React-backed Lexical editors, which is built on top of Yjs, an implementation of Conflict-free Replicated Data Types (CRDTs). The core ideas implemented by the plugin:

Custom ElementNodes and TextNodes that contain special methods for syncing with Yjs events
Bindings with Yjs. These bindings convert Yjs events to Lexical model changes and vice versa
### Performance

Lexical stays performant through several key design choices and optimizations:

Lightweight core: The core Lexical package is only 22 kb in size (min+gzip), keeping the initial load time low.
Modular architecture: Lexical uses a plugin-based system, allowing developers to include only the features they need. This "pay for what you use" approach helps maintain performance as projects scale.
Lazy-loading support: Lexical plugins can be lazily loaded and lazily initialized, deferring the loading of additional features until the user interacts with the editor. This helps improve initial load performance.
Efficient updates: Lexical uses an immutable editor state model, which allows for efficient updates and comparisons. Updates are batched and processed through a single editor.update() call, preventing cascading updates.
Efficient reconciliation approach: Lexical uses a reconciliation approach that acts similarly to virtual DOM diffing in React, allowing for efficient updates and rendering.
### Accessibility

Here are some key accessibility considerations for rich text editors, including things such as the UI, which is outside of the editor core.

Keyboard accessibility
Focus management: Ensure the editor can receive focus and that users can navigate through the editor using the keyboard.
Keyboard shortcuts: Provide keyboard shortcuts for common actions (e.g., bold, italic, undo). Ensure these shortcuts are documented and do not conflict with existing browser or screen reader shortcuts. The implementation of keyboard shortcuts is discussed above.
Tab order: Maintain a logical tab order for navigating through toolbar options and the editor content.
Screen reader support
ARIA roles and properties: Use appropriate ARIA roles (e.g., role="textbox", role="toolbar") and properties to describe the editor and its controls. See how many ARIA attributes Lexical adds to the contenteditable element.
Live regions: Use ARIA live regions to announce dynamic changes, such as formatting changes or error messages.
Labels and instructions: Provide clear labels and instructions for all editor controls and ensure they are announced by screen readers. Icon-only buttons should use aria-label.
Visible focus indicators
Ensure that all interactive elements, including the editor itself and toolbar buttons, have visible focus indicators. This helps users who rely on the keyboard to see where the focus is.

Semantic HTML
Semantic markup: Use semantic HTML elements (e.g., <strong>, <em>, <ul>, <ol>, <li>) within the editor to convey meaning.
Content structure: Ensure that the content structure is maintained, such as headings and lists, to provide a logical reading order.
Speech-to-text software
Dragon NaturallySpeaking is popular speech recognition software. Lexical provides a lexical-dragon package that listens for the message event (dispatched by Dragon) on window and inserts the text into the editor.

### Internationalization (i18n)

An editor handles more languages than the chrome around it, so i18n spans both UI strings and the content itself, starting with bidirectional text.

Right-to-left text directions
The editor should support both left-to-right (LTR) and right-to-left (RTL) text directions. This includes not only the content area but also the UI components.

In Lexical, ElementNodes support a direction property. When rendering the list of children within the node, the direction property is taken into account.

Input method editors (IMEs)
Ensure the editor works seamlessly with IMEs, which are used for typing in languages that require complex character input, such as Chinese, Japanese, and Korean.

Summary
A Lexical-style rich text editor is best presented in the interview as a five-step authoring flow, where each step pins down one main decision before the next one makes sense.

Step 1. Design an editor-specific document model. Each Editor owns an immutable EditorState that bundles a Map<NodeKey, EditorNode> of ElementNodes and TextNodes with an EditorSelection. Child order is stored as parent, prev, next, firstChild, and lastChild pointers so node lookup and sibling traversal are O(1) and structural edits rewrite only a handful of keys. Overlapping styles collapse into flat TextNode runs tagged with a bitfield format, which keeps toggle, add-format, and remove-format operations well-defined instead of ambiguous over nested inline tags. EditorSelection mirrors the result of document.getSelection() against editor nodes and carries explicit handling for backwards selections so range math never has to second-guess direction.

Step 2. Map the model onto the DOM through createDOM and updateDOM. Nodes render themselves: createDOM produces the initial element for a node, and updateDOM returns whether an in-place patch is possible or a full rebuild is required. The editor mounts on a contenteditable host so caret positioning, line-height measurement, IME composition, and RTL layout stay the platform's responsibility. The rendered DOM is treated as output only, never the source of truth, so a stale or tampered DOM can always be reconciled back from ActiveEditorState without data loss.

Step 3. Define the command surface and the plugin contract. Every mutation flows through dispatchCommand, and features subscribe with registerCommand, registerNodeTransform, and registerUpdateListener at declared priorities. Public traversal is exposed as methods like getParent() and getNextSibling() rather than raw pointer fields so the linked-tree representation can evolve without breaking consumers, and persistence rides on getEditorStateSerialized and parseEditorState against a stable JSON shape. Rich text behavior is not baked into the core; it ships as the lexical-rich-text plugin, a bundle of custom nodes, commands, and transforms layered on a minimal kernel.

Step 4. Translate user intent into commands. Keyboard, pointer, paste, drag, and toolbar interactions are all captured at the contenteditable boundary, normalized, and dispatched as commands rather than mutating the DOM directly. Beforeinput and input events are intercepted so that native browser editing behavior is overridden by deterministic command handlers, which is what lets features like smart lists, Markdown shortcuts, and paste sanitization compose predictably. This indirection is also what makes programmatic edits, from the toolbar, from remote collaborators, or from tests, indistinguishable from human keystrokes at the model layer.

Step 5. Reconcile to the DOM with a single update loop. editor.update() shallow-clones the node map into a PendingEditorState, clones only dirty nodes, runs registered node transforms until the dirty set stabilizes, and then diffs ActiveEditorState against PendingEditorState so only changed subtrees touch the DOM. Structural sharing on the immutable node map makes undo and redo stacks in @lexical/history cheap to snapshot and coalesce, update listeners fire after the commit so the toolbar and side effects observe a consistent state, and collaboration slots in as Yjs CRDT bindings that emit editor commands instead of patching the DOM out-of-band.

Start with the document model and the update-loop reconciliation; custom nodes, transforms, listener priority, history coalescing, and Yjs collaboration are depth to reach for only after those two are on the board.

## References

The references below group editor engineering case studies, open-source editor frameworks, talks and deep-dives, and related system design write-ups on this site, so it is easier to map each source back to the part of the article it informed.

Editor engineering case studies
These posts describe how production teams have reasoned about contenteditable, document models, and the engineering tradeoffs behind shipping a rich text editor.

Why ContentEditable is Terrible for Nick Santos's walkthrough of why Medium moved away from naive contenteditable and how the Medium editor handles the resulting edge cases.
The data model behind Notion's flexibility for a block-structured document data model that informs editor state and serialization choices.
Creating the Notion API for how a block-based document model is exposed as a stable external API surface.
Facebook open sources rich text editor framework Draft.js for the engineering rationale behind a React-based rich text framework and its immutable model.
Open-source editor frameworks
These projects are concrete references for document models, plugin systems, and DOM reconciliation strategies in rich text editors.

Lexical for Meta's extensible editor framework with a separate model layer and reconciler.
Slate.js for a customizable editor framework built around an immutable document model and plugins.
Tiptap for a headless editor built on ProseMirror with a plugin and extension architecture.
Draft.js for an earlier React-based framework that popularized an immutable content-state model.
Talks and deep-dives
These talks go deeper into the architecture of modern editor frameworks than a written article usually can.

Rethinking Rich Text: A Deep Dive Into the Design of Lexical - Acy Watson for the design principles, node system, and reconciler behind Lexical.
001: Intro to Lexical iOS — Lexical iOS Tutorial Series for an example of reusing a shared editor model across platforms.
Related system design articles
These articles on this site cover adjacent editor and collaboration problems that share parts of the rich text editor architecture.

Collaborative Editor (Google Docs) for how the single-user editor surface extends into real-time collaboration with revisioning and conflict resolution.
Design a Drawing Tool (Figma / Canva) for adjacent editor-surface tradeoffs around canvas-based document models and command flows.