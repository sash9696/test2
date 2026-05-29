---
title: "Google Sheets"
slug: "google-sheets"
difficulty: "hard"
duration_mins: 45
tags:
  - system-design
premium: true
summary: "Design a collaborative spreadsheet like Google Sheets and Excel."
free_preview: false
status: draft
---

# Google Sheets

## Question

Design a collaborative spreadsheet application that allows multiple users to view and edit the same spreadsheet simultaneously in real time.

Focus on the browser experience for an active sheet: editing literal values and formulas, selecting ranges, formatting cells, rows, and columns, resizing rows and columns, and reflecting collaborators' updates with low latency. You can assume the spreadsheet is persisted by a server and omit charts, pivot tables, comments, macros, and full offline editing unless the interviewer asks about them.

This article is spreadsheet-specific, not a repeat of Google Docs
This article builds on the collaboration ideas from the Google Docs system design article, but the main discussion here is spreadsheet-specific: grid rendering, formula recalculation, selections, and structured cell/range operations.

## Real-life examples

Google Sheets
Microsoft Excel
Airtable
Zoho Sheet
## Requirements exploration

Collaborative spreadsheets combine several hard front-end problems in one product: rendering a large 2D grid efficiently, evaluating formulas with dependency tracking, layering formatting rules, and reflecting multi-user edits in real time.

#### What are the core features to be supported?

Users can work on one active sheet at a time within a collaborative spreadsheet.
Cells support literal values and formulas, editable either directly in the grid or through a formula bar.
Users can select single cells and rectangular ranges using mouse and keyboard.
Formatting can be applied at the cell, row, and column levels.
Rows and columns can be resized.
#### Should the spreadsheet support real-time collaboration?

Yes. Multiple users can view and edit the same spreadsheet simultaneously. Collaborators should see each other's presence, current selection, and active editing cell, and updates from peers should appear automatically in near real time.

#### What is the expected grid size?

The spreadsheet should support large sheets with thousands of rows and columns. Most cells will be empty, so both the rendering layer and the data model should assume sparse data instead of a fully populated grid.

#### What features are out of scope?

Charts, pivot tables, macros, comments, scripting, and full offline-first conflict resolution are out of scope. Full multi-sheet coordination is also not part of the first-pass design and can be discussed later as an extension.

#### What devices should be supported?

Desktop browsers are the priority. The design should not prioritize a polished mobile spreadsheet experience.

#### What are the non-functional requirements?

Local edits should feel instant; users should not wait for a server round-trip before seeing their own changes.
Peer updates should typically appear within about 500ms.
Scrolling, navigation, and selection should remain smooth on large sheets.
Formula recalculation should touch only affected cells instead of recomputing the entire sheet.
The application should recover cleanly from reconnects and stale client state.
Background
Spreadsheets overlap with document editors on real-time sync but diverge sharply on data model, rendering, and computation, which is worth unpacking before jumping into the design.

#### What makes collaborative spreadsheets different from collaborative documents?

A collaborative spreadsheet shares some of the same real-time synchronization problems as a collaborative document editor, but the surface area is quite different. Instead of a linear document, the user interacts with a large 2D grid where each cell can contain either a literal value or a formula that depends on other cells.

That heavily influences decisions for both the rendering and state management. The browser has to render and navigate a potentially massive table efficiently, while the data layer has to track dependencies, recalculate affected cells after edits, and keep formatting and structural row/column changes consistent across collaborators.

In other words, the unit of interaction is not just text. It can be a cell, a rectangular range, a row, a column, or an entire sheet-level operation such as resizing or moving structure.

Spreadsheet concepts relevant to this design
Raw input vs displayed value: A cell might store =A1+B1 as its raw input while displaying 42 as its computed value.
Formulas and references: Formulas can depend on individual cells or ranges, so edits can trigger dependent-cell recalculation.
Ranges: Many spreadsheet interactions operate on rectangular ranges rather than single cells, including selection, formatting, and formulas such as SUM(A1:A10).
Structural row/column edits: Inserting, deleting, moving, or resizing rows and columns affects rendering, references, and collaboration semantics.
Formatting layers: Formatting is not just at the cell level; it can be applied at the cell, row, column, or sheet-default level, so the rendered result often comes from merging multiple layers of formatting.
Approach
The design is anchored by three decisions that reinforce each other: how to render the grid, how to model the cell graph and its formulas, and how to synchronize edits across clients.

Rendering the grid
The core rendering tradeoff is whether to represent visible cells as DOM nodes or to draw the grid into a canvas. This is similar to the tradeoff discussed in the Rich Text Editor system design article, but spreadsheets put much more pressure on layout density and scrolling performance.

| Approach | Strengths | Weaknesses |
| --- | --- | --- |
| DOM grid + editor overlay | Native keyboard input, IME support, accessibility, copy/paste behavior, and easier styling/debugging | Too many mounted cells will hurt layout, paint, and memory without aggressive virtualization |
| --- | --- | --- |
| Canvas grid | Lower per-cell DOM cost, fine-grained drawing control, and better scaling for extremely dense surfaces | Must reimplement hit testing, text measurement, editor overlays, selection visuals, clipboard behavior, and accessibility |
The recommended starting choice in the context of interviews is DOM grid:

Spreadsheet cells are mostly lightweight display surfaces.
Only one active cell needs a real editor at a time.
Browser primitives already handle a lot of hard UX details around focus, input, and accessibility.
Canvas is a valid production optimization if the interviewer pushes on very large sheets. For reference, Google Sheets itself uses a canvas-based renderer for its grid. However, canvas is better framed as an upgrade path than as the starting point in an interview because it requires reimplementing many browser primitives from scratch. A hybrid model is also practical: canvas for the display layer plus a DOM overlay for the active editor, selection UI, and accessible interactions.

Default to a DOM grid with virtualization, not canvas
Even though production Google Sheets uses canvas, DOM is the stronger interview default because it gets native focus, IME, clipboard, and accessibility for free. Only bring up canvas or a hybrid renderer as a scaling upgrade path if the interviewer explicitly pushes on extreme sheet density.

Request payload contents
The client should send structured operations, not full-sheet snapshots. Each update should include the sheetId, a baseRevision, the operation type, and only the range or metadata needed for that change.

That shape keeps the protocol small and makes spreadsheet-specific behaviors explicit. For example, SetCellInput, SetRangeFormat, and InsertRow are fundamentally different operations and should not be flattened into one generic patch format.

Collaboration model for spreadsheets
A spreadsheet can reuse the same broad client-server sync model as a collaborative document editor, but the conflict surface is usually smaller. Independent cell or range edits can merge naturally, while same-cell conflicts can use server ordering with last-write-wins at the cell-input level. There's usually no need for operational transforms to handle same-sentence or same-word conflicts.

Structural row and column operations are the main exception. They should be serialized and rebased on the server because they can shift coordinates, selections, formatting ranges, and formula references for other users at the same time. In practice, the client should not apply structural operations optimistically. Instead, it sends the operation to the server, which assigns it a revision, rewrites affected references, and broadcasts the rebased result back to all clients as the authoritative update.

Do not apply row or column structural edits optimistically
Insertion, deletion, movement, and resizing of rows and columns shift addresses and rewrite formula references across the entire sheet. If each client rebases those independently, collaborators will diverge. Let the server serialize the operation, rebase references authoritatively, and broadcast the rebased delta to every client.

Formula recalculation strategy
The client should parse formulas into an Abstract Syntax Tree (AST), maintain dependency edges for hydrated cells, and recalculate only the dirty subgraph it has loaded after an edit. For optimistic feedback, the browser should eagerly hydrate any offscreen precedents or dependents touched by visible formulas or the actively edited range. If part of the affected subgraph is still missing, the client can keep the local edit responsive but treat the recomputation as tentative until the server returns the authoritative result for that revision.

The server should repeat the same logic against the full sheet for the authoritative revision that is broadcast back to collaborators. Range-heavy formulas need special care because naively expanding every range into individual edges can blow up the dependency graph. Structural operations also feed into this subsystem because inserting or deleting rows and columns may require formula references to be rewritten before recalculation.

Overall approach summary
The overall design is a DOM grid with optimistic local edits, structured spreadsheet operations, server-ordered synchronization, and selective recalculation based on a dependency graph. That gives us a responsive browser experience without treating the whole spreadsheet as one giant mutable blob.

## Architecture / high-level design

Collaborative spreadsheets should use a client-server model with HTTP for bootstrapping initial state and WebSocket for live collaboration.

The client keeps a local replica of the active sheet so that edits can apply optimistically and the UI can remain responsive even before the server responds. The server orders operations, persists revisions, recalculates affected formulas, and broadcasts the authoritative result back to connected clients. This is the same broad collaboration model used in the Google Docs system design article, but the unit of change here is a cell, range, row, or column operation rather than a text operation.

Component responsibilities
Server: Stores spreadsheet and sheet data, assigns monotonically increasing revisions, recalculates affected formulas after committed operations, and broadcasts authoritative updates and presence to connected clients.
Collaboration adapter: Maintains the WebSocket connection, sends local operations and presence updates, and receives remote operations, acknowledgements, and recalculation results.
Spreadsheet store: Holds the active spreadsheet snapshot, visible sheet window, row and column metadata, pending local operations, selections, and collaborator presence. Serves as the source of truth for the UI.
Formula engine: Parses formulas, tracks dependency edges, marks dirty cells, and recalculates affected values. Runs on the client for fast optimistic feedback and on the server for authoritative results.
Grid UI: Renders the visible cell window, sticky headers, selection borders, fill handles, and collaborator highlights. Uses lightweight display cells instead of mounting a full editor in every cell.
Cell editor overlay: A single positioned editor mounted over the active cell. The editor overlay and formula bar share the same draft value so users can edit from either surface.
Toolbar and formula bar: Dispatch structured formatting, sizing, and editing commands rather than mutating DOM state directly.
![Update flow](google-sheets-pdf-assets/figures/update-flow.png)

The page loads spreadsheet metadata and the initial visible region of the active sheet over HTTP.
The client renders only the visible rows and columns for the active sheet, along with sticky headers and selection overlays.
When the user commits an edit, the client updates its local replica immediately, recalculates the hydrated portion of the affected dependency subgraph, and sends a structured operation with a base revision.
The server orders the operation, persists it as a new revision, recalculates authoritative values, and produces an authoritative delta for the affected cells and any related structural updates.
The originating client receives acknowledgement for the accepted operations, every client applies the same authoritative delta for that revision, and the UI clears or rebases pending local operations as needed.
The server should be especially authoritative for row and column insertion, deletion, movement, and resizing. Those operations can invalidate addresses and references across a large part of the sheet, so the safest model is for the server to serialize them, rewrite affected coordinates and formulas, and then broadcast the rebased result back to clients.


Client

HTTP bootstrap: GET window

WebSocket: ops + presence

authoritative delta + ack

Grid UI + editor overlay

Spreadsheet store
sparse cells + pending ops

Formula engine
AST + dep graph

Collaboration adapter

Operation orderer

Authoritative formula engine

Persistent store
op log + snapshots


Client-server architecture for one active sheet
## Data model

Use a sparse sheet store keyed by cell address, not a dense 2D array. A large sheet may expose millions of possible coordinates, but only a small fraction will be non-empty, formatted, or currently visible.

Core entities
| Entity | Purpose | Important fields |
| --- | --- | --- |
| Spreadsheet | Top-level state for the current spreadsheet | id: string, revision: number, activeSheetId: string, sheetOrder: string[] |
| --- | --- | --- |
| Sheet | State for one tab in the spreadsheet | id: string, name: string, cells: Map<string, Cell>, rowMeta: Map<number, RowMeta>, columnMeta: Map<number, ColumnMeta>, defaultFormat: FormatPatch, frozenRows: number, frozenColumns: number |
| --- | --- | --- |
| Cell | Raw input plus derived output | address: string, rawInput: string, computedValue: unknown, displayValue: string, formulaAst: ASTNode | null, error: string | null, formatOverride: FormatPatch | null |
| --- | --- | --- | --- | --- | --- |
| RowMeta | Row-level presentation state | height: number, hidden: boolean, formatPatch: FormatPatch | null |
| --- | --- | --- | --- |
| ColumnMeta | Column-level presentation state | width: number, hidden: boolean, formatPatch: FormatPatch | null |
| --- | --- | --- | --- |
| DependencyNode | Recalculation metadata | precedents: Set<string>, dependents: Set<string>, dirty: boolean |
| --- | --- | --- |
| CollaboratorPresence | Ephemeral peer state | userId: string, color: string, sheetId: string, selection: Range | null, editingCell: string | null, lastSeenRevision: number |
| --- | --- | --- | --- | --- |
| PendingOperation | Optimistic local mutation waiting for acknowledgement | opId: string, type: OperationType, target: string | Range, payload: object, baseRevision: number |
Sparse cell storage
Sheet.cells should be a map keyed by cell address such as A1 or a numeric row/column tuple. Only cells that contain raw input, computed errors, or explicit formatting overrides need entries. An absent entry means "empty cell with inherited defaults".

This matters for both memory and update cost. Formatting an entire column should update one ColumnMeta record rather than materializing thousands of blank Cell objects. It also keeps viewport fetching simple because the client can request only the visible rectangular window plus a small buffer.

Do not model the sheet as a dense 2D array of cells
A sheet can expose millions of coordinates, but only a small fraction hold raw input, computed errors, or explicit formatting. Store cells in a map keyed by address and let absent entries mean "empty with inherited defaults" so that row-wide, column-wide, and sheet-wide operations stay O(1) instead of O(cells).

Formatting precedence
When rendering a cell, resolve each formatting property independently in this order:

Sheet defaults
Column-level metadata
Row-level metadata
Cell-level override
If both the row and column define the same property, store per-property update timestamps or sequence numbers and let the most recently applied formatting operation win. Cell-level overrides still take final precedence.

Derived state and recalculation metadata
A cell's rawInput is the source of truth. computedValue is the evaluated result of a formula before display formatting, while displayValue is the final rendered value shown in the grid, including formatting or error presentation. formulaAst and dependency edges are also derived state that can be cached and invalidated when precedents change. In practice, keep a dependency graph keyed by cell address so that an edit can mark only the affected subgraph as dirty.

The client store should also track the current authoritative revision and any PendingOperations waiting for acknowledgement. That gives the UI instant local feedback while still allowing the server to rebase, accept, or correct the final result.

sheetOrder[]

cells (sparse)

rowMeta

columnMeta


activeSheetId


frozenRows


frozenColumns


rawInput


computedValue


displayValue


opId


baseRevision


userId


editingCell


Spreadsheet client store entity relationships
## Interface definition (API)

The most important interface is not a large REST surface area. It is a structured operation layer shared by the grid, formula bar, toolbar, and collaboration adapter. UI components should dispatch operations against the spreadsheet store, then let the collaboration layer send them to the server, rather than mutating DOM state directly.

Initialization API
Use HTTP to bootstrap the page and fetch the initial sheet window:

The second endpoint is important because the client should not fetch an entire large sheet up front. As the user scrolls, the browser can request additional windows and merge them into the local sparse store. The layout metadata should be modeled sparsely too: most rows and columns can inherit defaults, while only resized or hidden items need explicit overrides. The dependency-slice endpoint keeps the formula engine accurate without forcing the client to preload the full sheet just to recalculate a visible formula.

Update API
Send edits as batches of typed operations over WebSocket:


sendOperations({
spreadsheetId,
sheetId,
baseRevision,
operations: [
{ type: 'SetCellInput', address: 'B4', rawInput: '=SUM(B1:B3)' },
{ type: 'SetRangeFormat', range: 'B1:B10', formatPatch: { fontWeight: 'bold' } },
{ type: 'ResizeColumn', column: 2, width: 160 },
],
});
The server responds asynchronously over the same WebSocket. A simple model is to send a lightweight acknowledgement to the originating client and broadcast the same authoritative delta to every connected client, including the sender:


// Sent to the originating client
ack({ revision: 1842, acceptedOpIds: ['op-91', 'op-92'] });

// Broadcast to every connected client, including the sender
operationsApplied({
revision: 1842,
changedCells: [{ address: 'B4', computedValue: 42, displayValue: '42' }],
changedRows: [],
changedColumns: [{ column: 2, width: 160 }],
});
This authoritative delta is what all clients reconcile against after optimistic edits. It is especially important for structural operations and formula recalculation because the server may have rebased addresses, rewritten references, or corrected a tentative local result.

Peer client
Server (orderer + engine)
Client (store + view)
User (editor)
Peer client
Server (orderer + engine)
Client (store + view)
User (editor)
Commit edit (=SUM(B1:B3))
Apply optimistically, queue PendingOperation
Recalc hydrated dirty subgraph
sendOperations({ baseRevision, ops })
Assign revision, recalc authoritative values
ack({ revision, acceptedOpIds })
operationsApplied({ revision, changedCells })
operationsApplied({ revision, changedCells })
Drop pending op, reconcile against delta
Merge delta into local store


Collaborative cell edit with optimistic apply and rebroadcast
Useful operation types for this article are:

SetCellInput
ClearRange
SetRangeFormat
ResizeRow
ResizeColumn
InsertRow
DeleteRow
InsertColumn
DeleteColumn
This keeps the protocol aligned with how spreadsheets actually behave. A row insertion is not just a batch of cell edits; it is a structural operation that can shift addresses, formatting, and formula references.

Presence updates
Presence should use a separate, lightweight event stream because it is ephemeral and should not be persisted like sheet edits.

sendPresence({ sheetId, selection, editingCell }): Sent by the active client when selection or editing focus changes.
presenceUpdated: Broadcast by the server to other collaborators so they can render colored ranges and active-cell indicators.
Presence updates can be throttled because they do not need the same durability guarantees as cell edits. WebSockets are a good choice for the transport mechanism. If small delays can be tolerated, both HTTP short and long polling can be used as well, which could be cheaper in terms of infrastructure costs.

Client C
Client B
Server (presence relay)
Client A (active editor)
Client C
Client B
Server (presence relay)
Client A (active editor)
Selection changes (throttled)
sendPresence({ sheetId, selection, editingCell })
presenceUpdated({ userId: A, selection })
presenceUpdated({ userId: A, selection })
Render colored range + avatar for A
Render colored range + avatar for A


![Presence and cursor broadcast flow](google-sheets-pdf-assets/figures/presence-and-cursor-broadcast-flow.png)

Acknowledgements and reconciliation
Clients remove acknowledged pending operations after applying the authoritative delta for that revision, then rebase any newer local edits on top of the latest revision.

If two users edit the same cell at nearly the same time, a practical first-pass policy is server ordering / last-write-wins at the cell-input level. Structural operations such as row and column insertion need server-side coordinate rebasing before the final update is broadcast.

Though rare, conflicting operations that cannot be resolved by the server should be discarded by the client.

## Optimizations and deep dive

This section expands the spreadsheet-specific tradeoffs behind the recommended design and covers deeper concerns that interviewers commonly ask about after the first architecture sketch:

Grid virtualization: This section explains how the client renders large row and column ranges without mounting the entire spreadsheet DOM.
Formula engine: This section covers parsing, dependency tracking, recalculation, error handling, and the split between raw input and derived output.
Formatting: This section discusses cell presentation, styles, number formats, and how formatting stays separate from cell values.
Accessibility: This section explains ARIA grid semantics, keyboard navigation, focus management, and screen-reader output for cells.
Clipboard and copy-paste: This section covers browser clipboard integration, range serialization, paste parsing, and spreadsheet-specific edge cases.
History and versioning: This section explains audit history, accidental overwrite recovery, and how spreadsheet changes can be replayed or inspected.
Reliability and reconnection: This section covers reconnect behavior, late acknowledgements, conflicting edits, and resilient collaborative sync.
Multi-sheet support: This section discusses how the model extends from one active sheet to multiple sheets in the same workbook.
Offline editing: This section explains the follow-up design for local spreadsheet edits while the client is disconnected.
Grid virtualization
By now, you should be familiar with the concept of DOM virtualization. Because spreadsheets can become potentially large (lots of columns and rows), virtualization is essential in a DOM-based implementation. Virtualizing a spreadsheet is harder than virtualizing a unidirectional feed because we need to window on two axes instead of one.

The basic approach is:

Keep a scroll container sized to the total logical dimensions of the sheet.
Render only the rows and columns that intersect the viewport, plus a small overscan buffer.
Position visible cells absolutely within that window.
Keep row headers, column headers, selections, and collaborator highlights in separate overlay layers.
Mount a single editor overlay instead of an <input> inside every visible cell.
This keeps scrolling smooth and prevents the DOM tree from exploding as the sheet grows. It also makes frozen rows and columns easier to implement because they can live in fixed layers while the main grid scrolls underneath.

Spreadsheet virtualization must window on both axes
Unlike a feed that only virtualizes vertically, a spreadsheet has potentially large row and column counts, so the visible window is a rectangle and both axes need independent cumulative-size indexes. Variable row heights and column widths make naive index math wrong, so maintain sparse size overrides plus a cumulative offset structure that supports binary search.

To make this work with variable row heights and column widths, bootstrap the total logical row and column counts plus sparse layout overrides for non-default sizes and hidden items. The client can then size the full scroll surface without materializing every row and column, and patch those layout indexes incrementally as resize or hide operations arrive.

There are a few spreadsheet-specific optimizations worth noting:

Maintain cumulative row-height and column-width offsets so the viewport can binary-search to the first visible row and column.
Prerender boundary row and column windows so fast scrolling does not immediately hit empty states.
Render selection borders and autofill handles as overlays instead of styling every selected cell individually.
Preserve scroll position when remote updates arrive by patching only affected cells rather than remounting the whole window.
Formula engine
The formula engine is in charge of parsing and recalculation. It should store raw input and derived output separately:

rawInput: exactly what the user typed, such as =SUM(B1:B3)
computedValue: the evaluated result before display formatting is applied.
displayValue: the final rendered value shown in the grid, including formatting or an error such as #CYCLE!
When a cell changes:

Remove the old dependency edges for that cell.
Parse the new input into an AST if it starts with =.
Resolve referenced cells or ranges.
Update the dependency graph.
Mark the edited cell and its transitive dependents as dirty.
Recalculate only the affected subgraph in topological order.
The engine should not recompute the entire sheet after every edit. This is the key optimization. If a cell has no dependents, the recalculation cost may be effectively zero beyond the edited cell itself. In the browser, this optimistic path only works for the dependency slice the client has hydrated locally. If an active formula reaches beyond the loaded slice, fetch the missing precedents or dependents, or treat the result as tentative until the server pushes the authoritative recomputation.

Recalculate only the dirty subgraph, never the whole sheet
Recomputing every cell on every edit does not scale. Maintain a dependency graph keyed by cell address, mark the edited cell and its transitive dependents dirty, and recalculate only that subgraph in topological order. Large repeated ranges such as SUM(A1:A10000) should be represented as reusable range nodes rather than expanded into per-cell edges, or the graph itself will become the bottleneck.

Range formulas need an extra layer of care. A formula such as SUM(A1:A10000) should not force the engine to rebuild an enormous dependency structure from scratch on every edit if the same large ranges appear repeatedly. In practice, range-aware caching or reusable range nodes help keep the dependency graph from growing unnecessarily.

Structural row and column edits also feed back into recalculation. If a user inserts a row above A10, formulas that reference A10 or A1:A20 may need their references rewritten before the dirty subgraph is recalculated. That is one reason these operations are safer when the server applies and rebases them authoritatively.

Cycle handling is important. If the affected subgraph cannot be topologically sorted, the cycle participants should surface an error state such as #CYCLE! instead of hanging or recursively evaluating forever.

Always detect cycles before evaluating a formula subgraph
A user can easily author A1 = B1 + 1 and B1 = A1 + 1 by accident. Without cycle detection, the recursive evaluator will blow the call stack, hang the main thread, or freeze the tab entirely. Run a topological sort on the dirty subgraph first and mark every cycle participant with #CYCLE! instead of attempting evaluation.

The client and server can share the same mental model here:

The client recalculates immediately for optimistic feedback.
The server recalculates again for the authoritative revision that is broadcast to all collaborators.
For sheets with large formula graphs, move recalculation onto a Web Worker so the main thread remains responsive during scrolling, selection, and typing.

Offload heavy recalculation to a Web Worker
Large dependency graphs and range-heavy formulas can easily spend tens of milliseconds inside a single recalculation pass, which is enough to drop frames during scroll or typing. Running the formula engine inside a Web Worker and posting back a delta of changed cells keeps the grid interactive even when recalculation is expensive.

For a smaller version of the same problem (in object-oriented design / coding format), look at the Spreadsheet, Spreadsheet II, and Spreadsheet III questions.

View
Recalculator
Dependency graph
Parser (AST)
Editor
View
Recalculator
Dependency graph
Parser (AST)
Editor
[Cycle detected]
[Acyclic]
rawInput = "=SUM(B1:B3)"
New precedents { B1, B2, B3 }
Remove old edges, add new edges
Mark edited cell + transitive dependents dirty
Emit dirty subgraph (topologically sorted)
displayValue = "
Evaluate each dirty node in order
Patch changedCells { address, computedValue }


Formula dependency cascade and recompute pipeline
Formatting
In a spreadsheet, formatting is the presentation layer that sits on top of the underlying cell contents. It controls how values look and how they are displayed, including styles such as font weight, text color, borders, and alignment, as well as display-oriented rules such as currency, percentage, and date/time formats.

Because formatting is separate from the underlying rawInput and computedValue, it should stay layered instead of being copied into every affected cell.

Use separate style layers (from lowest to highest priority):

Sheet-level (sheet.defaultFormat): Holds the base formatting for the sheet.
Row-level (rowMeta[rowIndex].formatPatch): Stores row-level formatting overrides.
Column-level (columnMeta[columnIndex].formatPatch): Stores column-level formatting overrides.
Cell-level (cell.formatOverride): Stores the highest-priority cell-specific overrides.
At render time, resolve the final format for a cell by merging those layers in precedence order. This keeps row-wide and column-wide formatting cheap, avoids duplicating style objects across thousands of cells, and keeps range-formatting operations compact on the wire. Strictly speaking, row and column are at the same priority level.

If both row and column metadata define the same property, a practical first-pass policy is last applied formatting operation wins for that property. Cell-level overrides still take final precedence.

Google Sheets actually lets row and column styles clear cell overrides
In reality, Google Sheets works a bit differently: a cell-specific override can be removed if a sheet/row/column-level style operation affects the same cell-level property. E.g. if a cell has a red background color and the entire row is changed to green, that cell is changed to green as well. If the entire column is set to blue, that cell will turn blue. There are pros and cons to both approaches, and they are similar in terms of technical complexity, so which approach to take hinges more on the desired product UX than on technical reasons.

Value formatting works in the same way too. The formula engine computes the underlying computedValue, while the formatting layer turns that into the final displayValue shown as plain text, currency, percentage, or date/time.

### Accessibility

A spreadsheet grid maps naturally to the ARIA grid pattern, which gives screen readers a structured way to navigate rows, columns, and cells.

Key concerns for a DOM-based grid:

Use role="grid" on the grid container, role="row" on each rendered row, and role="gridcell" on each cell. Row and column headers should use role="rowheader" and role="columnheader" respectively.
Manage focus with aria-activedescendant on the grid container rather than moving DOM focus to each cell. This avoids expensive focus shifts during keyboard navigation.
Announce the active cell's address, display value, and formula (if any) via a live region so screen reader users get the same information sighted users see in the formula bar.
Support full keyboard navigation: arrow keys move the active cell, Tab moves to the next cell in the row, Enter confirms an edit and moves down, and Escape cancels editing. Ctrl/Cmd + arrow keys should jump to the edge of a contiguous data region, matching native spreadsheet behavior.
When the user enters edit mode on a cell, shift focus to the editor overlay and announce the raw input. When they exit, return focus semantics to the grid.
Collaborator presence indicators (colored borders, avatar labels) should include aria-label annotations so screen reader users know which cells other collaborators are selecting.
If the design switches to a canvas renderer later, all of these behaviors need to be reimplemented through a hidden accessible DOM overlay, which is one of the main costs of the canvas approach.

Clipboard and copy-paste
Copy-paste is a core spreadsheet interaction and one of the more nuanced browser-integration surfaces.

When the user copies a range:

Write the selection to the clipboard in at least two formats: plain text as tab-separated values (TSV) for interoperability with other spreadsheets and external applications, and a richer internal format (JSON or a custom MIME type via the Clipboard API) that preserves formulas, formatting, and range structure.
Store the copied range reference in client state so that an internal paste can use the richer format while an external paste falls back to TSV.
When the user pastes:

If the clipboard contains the internal format, paste the full cell data including raw formulas, formatting overrides, and relative reference adjustments based on the paste target offset.
If the clipboard contains only plain text or TSV, parse it into a grid of literal values and write them as SetCellInput operations.
Support paste-special operations such as "paste values only" (discard formulas, keep computed values) and "paste formatting only" (apply format patches without changing cell input).
Use the browser Clipboard API (navigator.clipboard.read/write) for async clipboard access. For broader browser support, also handle trusted copy, cut, and paste events on the grid or editor surface and read or write event.clipboardData there.

Paste operations should go through the same structured operation path as manual edits. This means they are batched, revisioned, and broadcast to collaborators, rather than applied as a silent local-only DOM mutation.

History and versioning
Spreadsheets benefit from stronger history features than many ordinary CRUD interfaces because users often need to audit who changed a formula, recover accidental overwrites, or compare the current state against an earlier version of the sheet.

At the storage layer, the simplest model is:

Assign a monotonically increasing revision number to every accepted operation batch.
Persist an append-only operation log for cell edits, formatting changes, and structural row or column operations.
Periodically write sheet snapshots so recovery does not require replaying the entire history from the beginning.
This gives us two useful product-level capabilities. First, connected clients can synchronize against revision numbers and ask for updates since their last known state. Second, the product can expose version history later without redesigning the collaboration model from scratch.

It is also useful to distinguish between full revision history and cell-level edit history. Full revision history can restore an earlier spreadsheet state, while cell-level history is a narrower audit view for a specific cell. For example, "restore the sheet to yesterday's 4pm version" depends on full revision history, while "who last changed B12?" depends on cell-level history. In practice, cell-level history may omit some structural, formatting-only, or formula-driven changes, so the article should treat it as a helpful extension rather than the primary durability model.

Reliability and reconnection
Real-time collaboration only feels good if the client handles imperfect networks gracefully. A spreadsheet should remain responsive during brief disconnects, late acknowledgements, and overlapping local and remote edits.

A few behaviors matter most in practice:

Batching operations: Batch rapid operations such as drag-fill, paste, or repeated formatting into small operation groups before sending them to the server.
Reconnect by revision: After reconnecting, the client sends its last seen revision and either receives the missing operations or refetches visible windows if the gap is too large.
Maintain a local operations queue: Keep a queue of unacknowledged local operations so the client can drop acknowledged ones and rebase the rest on top of the newest revision.
Inform about potential conflicts/overwriting: If a remote update lands on the cell the user is actively editing, preserve the local draft in the editor but surface that the visible sheet state has advanced.
If the client falls too far behind, loses too many operations, or detects that local rebasing is no longer trustworthy after structural changes, it should abandon replay, disable any further local operations, and refetch the visible sheet window from the server.

User focuses cell

Escape (discard draft)

Enter / Tab (commit edit)

Authoritative delta acked

Remote edit landed on same cell

Keep local draft, preserve editor

Accept remote, discard draft

Disconnected

Reconnect + replay by revision

Gap too large, refetch window

Idle

Editing

Committing

Synced

Conflict

Queued

Invalidated


Cell edit lifecycle with sync and conflict states
These behaviors let the interface feel local-first without pretending the transport is perfectly reliable.

Multi-sheet support
The main design focused on one active sheet because that is where most of the front-end complexity lives. Extending the system to support multiple sheets does not require a new collaboration model, but it does add another layer of state and coordination at the spreadsheet level.

The main extensions are:

Keep sheet metadata, ordering, and the active sheet id at the spreadsheet level.
Track one sparse cell store, one formatting layer stack, and one dependency graph per sheet.
Load inactive sheets lazily instead of fetching every sheet up front.
Include sheetId in operations, presence updates, and revisioned updates so clients can route changes to the right sheet.
Cross-sheet formulas are the biggest extra complexity. Once formulas can reference Sheet2!A1, recalculation and structural operations can no longer stay isolated to a single sheet. That is why multi-sheet support is a good late deep-dive topic rather than part of the first-pass scoped design.

Offline editing
Full offline editing is out of scope for the main design, but it is a natural follow-up once the online collaboration model is stable.

At a high level, offline support would require the client to persist local sheet state and any pending operations so the user can continue editing while disconnected. After reconnecting, the client would replay those operations against the latest server revision and re-run reconciliation.

The hard part is conflict handling. Same-cell edits are already manageable with server ordering, but structural row and column operations become much harder to merge if remote changes happened while the client was offline. That is why offline editing is better positioned as a follow-up discussion than as part of the first-pass recommended architecture.

## Summary

The design covers one active sheet of a collaborative spreadsheet with a dependency-aware formula engine, a virtualized 2D grid, and a revisioned sync path over WebSocket. If only three things survive from this write-up into an interview answer, make them these.

Virtualize both axes with cumulative offsets, not just rows. A spreadsheet is a 2D grid where columns can be resized independently, so the usual list-style row virtualization is not enough. Maintain cumulative offset indexes over RowMeta and ColumnMeta heights and widths, then binary-search those indexes on every scroll to resolve the visible row and column windows. Render only that window plus a small overscan, mount a single cell editor overlay over the active cell, and leave everything outside the viewport out of the DOM entirely. This is what keeps scrolling at 60 fps on sheets with tens of thousands of rows and hundreds of columns without reaching for canvas on day one.

Recalculate the dirty subgraph, not the sheet. Formulas parse into an AST, and the engine wires precedents and dependents into a graph of DependencyNodes keyed by cell address. When a SetCellInput edit lands, walk dependents transitively, topologically sort the dirty set, evaluate in order, and short-circuit on cycle detection with a #CYCLE error rather than looping. For pathological fan-out the evaluator can move to a Web Worker so the main thread stays free for input and scroll. A full-sheet recompute on every keystroke is the obvious wrong answer, and saying so out loud signals that you understand what actually makes a spreadsheet feel responsive.

Use server-authoritative ordering with selective optimistic updates. The client sends operations over WebSocket, the server assigns a monotonically increasing revision, runs authoritative recalculation, and broadcasts deltas back. Cell and formatting edits apply optimistically through PendingOperations so typing feels instant, and on reconnect the client replays its queue from the last acknowledged revision. Structural operations — InsertRow, DeleteRow, InsertColumn, DeleteColumn — are deliberately not optimistic, because only the server can rebase every formula reference across every client consistently. Treating the two classes of operations differently is the distinguishing move.

Together these three decisions buy a grid that stays responsive under load, a formula engine that scales with edit size rather than sheet size, and a sync model that converges without the client ever having to rewrite its peers' formulas.

## References

The references below group Google Sheets product and API docs, browser rendering and storage APIs, and related system design write-ups on this site.

Spreadsheet product docs and APIs
These docs and APIs describe the product-level collaboration, structural, calculation, and rendering behaviors referenced throughout the article.

Google Workspace: Share & collaborate on a spreadsheet for product-level collaboration constraints.
Google for Developers: Row & column operations and Named & protected ranges for concrete spreadsheet-specific structural and range operations.
Google Docs Editors Help: Set a spreadsheet's location & calculation settings and Find what's changed in a file for calculation policy and history behavior.
Microsoft Learn: Excel recalculation for dependency trees, dirty propagation, and calculation ordering.
HyperFormula: Dependency graph and Performance for practical spreadsheet-engine modeling and recalculation tradeoffs.
Handsontable: Row virtualization and Column virtualization for concrete two-axis viewport-rendering patterns.
Related system design articles
These articles on this site cover adjacent problems and share parts of the collaboration and editor-surface backbone.

Collaborative Editor (Google Docs) for the shared collaboration backbone: optimistic local state, revisioned synchronization, and real-time client-server updates.
Rich Text Editor for adjacent rendering and editor-surface tradeoffs, especially around DOM-based editing surfaces.
Related practice questions
These practice questions drill down into focused implementation problems that underlie the larger spreadsheet design.

Spreadsheet, Spreadsheet II, and Spreadsheet III for smaller-scale formula parsing, dependency tracking, and cycle-detection problems.
The useEffect() Tee — Mount, Update, Rerender, Repeat