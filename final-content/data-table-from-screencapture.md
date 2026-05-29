---
title: "Data Table"
slug: "data-table"
difficulty: "medium"
duration_mins: 40
tags:
  - system-design
  - accessibility
  - ui-component
premium: true
summary: "Design a reusable data table component with fixed headers, flexible cell rendering, and good performance on large datasets."
free_preview: false
status: draft
---

# Data Table

## Question

Design a reusable, client-rendered data table component for a large web application. The table should support a fixed header, scrollable rows and columns, declarative column definitions, and cells that can render different kinds of data such as strings, numbers, dates, and images.

Assume the table is used primarily for reading and scanning tabular data rather than spreadsheet-style editing. Unlike the Google Sheets system design article, this question is about designing a reusable table component for a shared component library, not formulas, collaboration, or spreadsheet-style cell editing semantics.

## Real-life examples

TanStack Table
MUI Data Grid
AG Grid
Ant Design Table
## Requirements

The component should be generic enough to reuse across multiple product surfaces.
The header should remain visible while rows scroll vertically.
The table should support horizontal scrolling when there are many columns or when columns cannot shrink further.
Columns should be configurable declaratively and cells should support custom rendering.
The table should accept an optional outer viewport/container width while still producing sensible default column widths.
The component should perform well with large datasets such as thousands of rows.
Focus on the base table design first. Features such as sorting, filtering, column resizing, and fixed columns can be discussed as extensions if time permits.
## Requirements exploration

Data tables show up everywhere in modern web applications: admin dashboards, analytics views, internal tools, moderation queues, billing pages, and CRMs. A good solution to this question should balance component API design with practical frontend concerns such as layout, semantics, sizing, and rendering performance.

#### What is the primary usage model?

This table is meant for scanning tabular data across internal tools, admin dashboards, moderation queues, billing pages, and CRM-style surfaces. The basic design should optimize for display, readability, and reuse across product surfaces rather than spreadsheet-style editing or highly interactive cell-by-cell workflows.

#### What kinds of data and customization should the table support?

Columns should be defined declaratively by the consumer.
Each column should be able to render a consistent cell type across the whole column.
The base component should handle common data such as strings, numbers, dates, and images.
Consumers should also be able to supply custom renderers for richer content such as links, badges, avatars, and status pills.
Column-level configuration should include at least the header label, accessor logic, alignment, and sizing constraints.
This is an important clarification because a reusable table component is more naturally configured per column than per individual cell. That keeps the API predictable and makes the component easier to reuse across a shared component library.

#### What scrolling and sizing behavior is expected?

The header should remain visible while rows scroll vertically.
The table should support horizontal scrolling when the total column width exceeds the available viewport.
By default, columns should use sensible best-fit widths instead of requiring every width to be hard-coded up front.
The component should also accept an optional overall width, which means the design needs a deterministic strategy for shrinking, expanding, and truncating columns when space is constrained.
For large-data mode, assume the table renders inside a bounded-height viewport rather than growing to fit every row.
The width behavior is one of the main design challenges in this problem. Cover both the unconstrained case and the constrained-width case, rather than just leaving sizing to “whatever the browser does”.

#### What scale should the basic design support?

The basic design should comfortably handle thousands of rows and dozens of columns. It is reasonable to start the architecture by assuming the caller already has the row data in memory, then discuss how the rendering model should change once the dataset becomes too large to mount all rows at once, or when rows are incrementally loaded.

That means smooth scrolling and predictable layout are non-functional requirements, not nice-to-haves. The component should be designed so that large datasets do not explode the DOM tree or make scrolling janky.

#### What devices and accessibility expectations should guide the first version?

Desktop browsers should be the priority because dense tabular data is most commonly consumed there. The design should still be responsive enough to avoid breaking on smaller screens, but a mobile-first dense table experience is not the main target.

Accessibility should be treated as a base requirement, not an extension. Because this is a read-heavy tabular surface, the proposed design should preserve semantic table behavior and clear header-to-cell relationships.

#### What is explicitly out of scope for the basic design?

Inline editing and spreadsheet-like keyboard navigation.
Formulas, computed cell dependencies, and collaboration semantics.
Frozen columns, tree rows, grouped rows, and merged-cell layouts.
Server-driven sorting, filtering, and pagination protocols.
Canvas rendering as the default implementation.
Common table features such as sorting, filtering, column resizing, row selection, and summary footers are worth acknowledging, but they should be treated as follow-up extensions unless the interviewer asks to prioritize them.

## Architecture / high-level design

For the scoped version of this problem, the architecture should stay mostly on the client. The consumer passes row data and declarative column definitions into a reusable DataTable component, and the table is responsible for sizing columns, rendering cells, managing scroll state, and switching between full rendering and a virtualized body when the dataset is large enough to justify it.

The most important architectural decision is the rendering model. For a read-heavy table, the best default is a semantic table-based design rather than a spreadsheet-style grid. Native table semantics already match how assistive technologies expect tabular data to be exposed, and they align with the problem statement better than a fully custom div-based layout.

Default to semantic <table> markup over a <div>-based grid
For a read-heavy tabular surface, native <table>, <thead>, <tbody>, <tr>, and <td> elements give correct accessibility semantics and header-to-cell relationships for free. A <div>-based grid has to recreate all of that behavior manually with ARIA, and is only worth the cost once the component becomes interactive enough to need managed focus or spreadsheet-style keyboard navigation.

An example usage of the component in React could look like this:


<DataTable
rows={users}
columns={[
{ key: 'name', header: 'Name', accessor: (row) => row.name },
{
key: 'joinedAt',
header: 'Joined',
accessor: (row) => row.joinedAt,
renderCell: (value) => formatDate(value),
},
{
key: 'avatar',
header: 'Avatar',
accessor: (row) => row.avatarUrl,
renderCell: (value, row) => (
<img src={value} alt={`${row.name} avatar`} />
),
},
]}
getRowId={(row) => row.id}
aria-label="Users"
height={600}
width={960}
/>
This shape keeps the component reusable across product surfaces. Rows are plain data, columns define how that data should be interpreted, and custom rendering remains a per-column concern rather than an ad hoc per-cell escape hatch.

There are many other ways to define the API for a data table, and each comes with tradeoffs.

Markup approach and tradeoffs
There are two realistic markup directions:

Native <table>: Better default semantics, simpler header-to-cell relationships, and a more natural fit for read-heavy tabular content.
Custom div-based layout with ARIA roles: More control over custom layout and virtualization, but significantly more accessibility and focus behavior has to be recreated manually.
The recommended starting point is native table semantics. In the scoped version of this problem, that recommendation should stay true even when row virtualization is enabled: render the visible rows plus top and bottom spacer rows inside the same <tbody> so the component keeps one real table, one sticky header, and one scroll container.

A div or grid-style implementation becomes more appropriate only once the component needs absolutely positioned rows, spreadsheet-like keyboard navigation, cell editing, or aggressive two-axis virtualization. In an interview setting, those tradeoffs are follow-up material rather than the default architecture.

A simplified native structure for the basic design might look like this:


<div class="table-scroll-container" style="overflow: auto;">
<table>
<caption>Users</caption>
<thead>
<tr>
<th scope="col">Name</th>
<th scope="col">Joined</th>
<th scope="col">Status</th>
</tr>
</thead>
<tbody>
<tr>
<td>Ada Lovelace</td>
<td>Jan 3, 2024</td>
<td>Active</td>
</tr>
</tbody>
</table>
</div>
This example is intentionally simple, but it makes the structure explicit: the scrollbars belong to the outer scroll container, while the inner table preserves the semantic relationship between headers, rows, and cells.

The div route is still worth acknowledging as a fallback path. role="table" fits a mostly read-only surface that needs more layout control than native tables provide, while role="grid" becomes appropriate only once the component is interactive enough to require managed focus and directional keyboard navigation. If the internals ever move away from native table flow, metadata such as aria-rowcount, aria-colcount, aria-rowindex, and aria-colindex may be needed to preserve accessibility. That is a later escalation point, not the recommended starting architecture.

Scroll and sticky-header strategy
The table should render inside one bounded scroll container that owns both vertical and horizontal overflow. The header row stays visible by using sticky positioning relative to that container, while the body rows scroll underneath it.

Architecturally, the important choice is to keep one shared scroll surface rather than splitting the header and body into separate scroll surfaces. That avoids scroll-sync bugs, duplicated width calculations, and alignment drift.

Do not split the header and body into separate scroll surfaces
A shared scroll container with a sticky <thead> is the simpler and more robust default. Two separate scroll surfaces require manual scroll synchronization on every scroll event, and any hiccup shows up as visibly drifting column alignment, exactly the kind of bug that is hard to debug in production and that you can avoid by letting one container own both axes.

That bounded-height detail should be explicit in the API contract. Sticky headers and row virtualization only make sense when the component has a real vertical viewport, so height should be treated as required for scrollable mode. If height is omitted, the table can still render as a plain non-virtualized table that grows naturally, but fixed-header behavior is inactive because there is no containing scroll surface for position: sticky to latch onto.

It is also important to distinguish the scroll viewport width from the resolved table content width. The optional width prop should size the outer scroll container, while the inner table may still resolve to a wider width and overflow horizontally if its columns cannot shrink any further.

Component responsibilities
| Component | Role |
| --- | --- |
| DataTable root | Accepts external props, resolves static versus virtualized-body layout mode, owns derived scroll and sizing state, and coordinates the subcomponents. |
| --- | --- |
| Scroll container | Defines the bounded viewport and handles both horizontal and vertical scrolling. |
| --- | --- |
| Header row / header cells | Render column labels, expose header semantics, and remain visible with sticky positioning. |
| --- | --- |
| Row renderer | Renders the visible rows using stable row identity. |
| --- | --- |
| Cell renderer | Applies the default or custom renderer for each column while preserving alignment and overflow rules. |
| --- | --- |
| Virtualization controller | Determines which rows should be mounted based on the viewport and overscan window, and inserts spacer rows when native-table virtualization is active. |
| --- | --- |
| Column sizing logic | Computes default widths, applies explicit min/max constraints, handles constrained-width redistribution, and exposes one shared width map to the header and body. |
The result is a relatively simple architecture: a semantic table surface, a shared column sizing model, one scroll container, and a virtualization layer that can reduce DOM cost without changing the table's external API.

rows, columns, getRowId

computedColumnWidths

visibleRowStart / visibleRowEnd

computedColumnWidths

Consumer app

DataTable root

Column sizing logic

Virtualization controller

Scroll container

tbody / row renderer

table element

thead / sticky header cells

Cell renderer


DataTable component hierarchy and shared state
## Data model

For a reusable data table component, the data model is a mix of consumer-provided inputs and internally derived layout state. The table should not mutate the caller's row data. Instead, it should treat the input rows and column definitions as the source of truth, then derive sizing, scrolling, and virtualization state from them.

Core external data structures / config / props
The two most important external structures are the row collection and the column definition model.


```typescript
type ColumnDef<Row> = {
```
key: string;
header: ReactNode | ((column: ColumnDef<Row>) => ReactNode);
accessor: (row: Row) => unknown;
renderCell?: (value: unknown, row: Row) => ReactNode;
width?: number;
minWidth?: number;
maxWidth?: number;
align?: 'start' | 'center' | 'end';
overflow?: 'truncate' | 'clip' | 'wrap';
};
This shape reflects an important design decision: columns are first-class configuration objects, not something inferred ad hoc from the first row of data. That makes the component much easier to reason about across a large codebase.

Core internal state
| State | Type | Description |
| --- | --- | --- |
| rowIds | string[] | Stable row identities derived from getRowId, used for rendering and virtualization. |
| --- | --- | --- |
| computedColumnWidths | Map<string, number> | Final resolved width for each column after applying defaults and constraints. |
| --- | --- | --- |
| scrollLeft | number | Horizontal scroll offset of the scroll container. |
| --- | --- | --- |
| scrollTop | number | Vertical scroll offset of the scroll container. |
| --- | --- | --- |
| viewportWidth | number | Visible width of the scroll container. |
| --- | --- | --- |
| viewportHeight | number | Visible height of the scroll container. |
| --- | --- | --- |
| visibleRowStart | number | Index of the first mounted row in the current viewport window. |
| --- | --- | --- |
| visibleRowEnd | number | Inclusive index of the last mounted row in the current viewport window. |
The table may also keep lightweight UI state for loading, empty, and error presentation, but the key state for this design problem is layout-oriented rather than data-mutating.

Derived state and layout metadata
Not every value needs to be stored explicitly. Several important values can be derived from the external inputs and the core state above:

Total logical row count
Total scrollable body height
Natural table width before any constrained-width redistribution
Total resolved table width after grow/shrink logic
Whether horizontal overflow is active
Whether virtualization should be enabled for the current dataset size
Resolved overflow behavior for each column after applying defaults and virtualization constraints
Treating these as derived values keeps the component easier to reason about and reduces the risk of duplicated state getting out of sync.

resolved per key

getRowId


sums into


minWidth


maxWidth


scrollTop


scrollLeft


visibleRowStart


visibleRowEnd


viewportWidth


viewportHeight


rowHeight


naturalTableWidth


resolvedTableWidth


hasHorizontalOverflow


Relationship between consumer inputs, core state, and derived layout
Row height model
The basic design should assume a fixed row height. That keeps the viewport math straightforward because the visible window can be calculated from scrollTop, rowHeight, viewportHeight, and overscan without measuring every mounted row.

Variable-height rows are possible, but they push the design into a different complexity tier. Once row heights depend on rendered content, the table needs measurement logic, cached row metrics, and more complicated virtualization behavior.

For these reasons, variable row heights are better treated as an extension rather than part of the base architecture.

That assumption also affects content overflow behavior. In the fixed-row-height base design, virtualized rows should default to single-line truncation or clipping. Multi-line wrapping fits better in non-virtualized mode or in a variable-row-height extension. The base design should not claim to support wrap together with fixed-height virtualization unless it is also willing to measure row heights dynamically.

## Interface definition (API)

The API should make the happy path obvious: pass in rows, declare columns, provide a stable row identity, and optionally configure the viewport and sizing behavior. A good table API should feel reusable across many product surfaces without requiring every consumer to understand the component's internal virtualization or layout machinery.

Core API
| Prop | Type | Description |
| --- | --- | --- |
| rows | Row[] | Source data for the table. |
| --- | --- | --- |
| columns | ColumnDef<Row>[] | Declarative column definitions that control headers, accessors, renderers, and sizing. |
| --- | --- | --- |
| getRowId | (row: Row) => string | Returns a stable identity for each row. Important for rendering, virtualization, and any row-level interactions. It serves a similar purpose to keys in React. |
| --- | --- | --- |
| height | number | string | Bounded viewport height. Required when the caller wants vertical scrolling, a sticky header, or row virtualization. If omitted, the table grows naturally and those behaviors stay off. |
| --- | --- | --- | --- |
| width | number | string | Optional width of the outer scroll viewport. The resolved inner table can still be wider than this value, in which case horizontal scrolling remains enabled. |
| --- | --- | --- | --- |
| rowHeight | number | Fixed row height used by the basic design. |
| --- | --- | --- |
| overscan | number | Number of extra rows to render above and below the viewport when virtualization is enabled. |
| --- | --- | --- |
| caption | ReactNode | Optional visible table caption. Preferred when the design supports a visible accessible name. |
| --- | --- | --- |
| aria-label / aria-labelledby | string | Accessible naming hooks when a visible caption is not appropriate or when the name should come from surrounding UI. |
| --- | --- | --- |
| loading | boolean | Whether the table is currently in a loading state. |
| --- | --- | --- |
| emptyState | ReactNode | Content shown when there are no rows to display. |
| --- | --- | --- |
| errorState | ReactNode | Content shown when the table cannot render normally because of an error or failed load. |
The base table should accept a plain list of row records. That mirrors how product code usually models server data already, so the component can drop into many surfaces without forcing consumers to reshape everything first.

The key sizing distinction is that width controls the viewport, not the sum of the column widths. The component still computes a separate resolved table width from the column model, then compares that content width against the viewport to decide whether to grow columns, shrink flexible ones, or keep horizontal overflow.

An example consumer-facing API could look like this:


<DataTable
rows={users}
columns={[
{
key: 'name',
header: 'Name',
accessor: (row) => row.name,
minWidth: 180,
},
{
key: 'joinedAt',
header: 'Joined',
accessor: (row) => row.joinedAt,
renderCell: (value) => formatDate(value),
width: 140,
align: 'end',
},
]}
getRowId={(row) => row.id}
aria-label="Users"
height={600}
width={960}
rowHeight={44}
overscan={8}
/>
Columns as a single array make reordering trivial
One of the greatest strengths of this design is that columns can be added or removed easily; you only need to modify the contents of the array passed into the columns prop and ensure that the row contains data accessed by that column. Other approaches that separate header and body cell definitions do not enjoy this benefit.

Column API details
Each column definition should expose a small but expressive set of fields:

key: Stable identifier for the column.
header: Header label or custom header renderer.
accessor: Function that extracts the value for that column from a row.
renderCell: Optional custom renderer for richer presentation.
width: Preferred width when the consumer wants to opt into explicit sizing. This is especially useful for non-text columns such as avatars, thumbnails, or action cells whose ideal width is hard to infer from raw data alone.
minWidth / maxWidth: Constraints for the sizing algorithm.
align: Horizontal alignment for the cell content.
overflow: Overflow policy for the column. In the fixed-row-height virtualized base design, truncate or clip should be the default. wrap is a better fit for non-virtualized or variable-height mode, and the simplest base policy is to disable virtualization when wrapping is required.
This keeps the API aligned with how engineers actually think about tables. Most customization happens per column, not at the level of individual cell instances.

Optional extension points
It is reasonable to leave room for common extensions without making them mandatory in the base API:

onRowClick
sortField
onSortChange
renderFooter
className or slot-level styling hooks for the root, header, rows, and cells
These are intentionally optional. The base component should not force every consumer to think about sorting, row actions, or footers if all they need is a performant read-heavy table.

API design notes
Prefer a small, composable API over a huge prop surface. In particular:

Keep the data contract centered on rows, columns, and getRowId.
Keep sizing explicit enough to be predictable, but not so verbose that every column requires manual tuning.
Make the accessibility contract explicit by supporting a visible caption and pass-through naming props such as aria-label and aria-labelledby.
Keep advanced behavior as additive extension points instead of making the base API feel like a kitchen sink.
There is also a real API tradeoff worth calling out. A component library could expose a more markup-like composition API such as Table, Table.Head, Table.Body, Table.Row, and Table.Cell. That can feel nicely declarative, but it is usually more verbose and often pushes every consumer into building the row-mapping layer themselves. For a reusable product table, a rows plus columns contract is usually the more pragmatic default.

Designing good customization surfaces for reusable UI primitives follows the same general principles discussed in the Front End Interview Guidebook's UI Components API Design Principles Section.

We recommend checking out TanStack Table, which is a very good example of a headless table library API.

## Optimizations and deep dive

This section discusses production extensions that commonly follow the base table design, especially when datasets become large and cells become more customized:

Width allocation and layout: This section explains column sizing modes, overflow behavior, and layout constraints for predictable table rendering.
Sticky header implementation details: This section covers sticky positioning, scroll containers, z-index layering, and header behavior during scroll.
Row virtualization: This section discusses rendering only the visible row window to keep large datasets responsive.
Column virtualization: This section explains when horizontal virtualization is useful and why it is usually a follow-up optimization.
Accessibility: This section covers semantic table structure, keyboard support, screen-reader behavior, and accessibility implications of virtualization.
Rich cell content: This section discusses renderer APIs, mixed content types, truncation, alignment, and default layout rules for complex cells.
Further extensions: This section lists practical table features that teams often add but are usually beyond the core interview scope.
Width allocation and layout
Column sizing is one of the hardest parts of this problem because the component has to support two different modes:

Natural width mode when the table width is unconstrained.
Constrained width mode when the scroll viewport is bounded and the column model may need to redistribute space.
To keep the behavior predictable, the component should track two separate values:

viewportWidth: the visible width of the scroll container, derived from layout or the optional width prop.
naturalTableWidth / resolvedTableWidth: the width implied by the column model before and after redistribution.
A practical sizing algorithm is:

Start with any explicit width values supplied by the consumer.
For columns without explicit widths, estimate a preferred width from the header text and a representative sample of cell contents.
Clamp every preferred width by minWidth and maxWidth.
Sum the preferred widths to get the natural table width.
In practice, step 2 should avoid scanning the full dataset. A better policy is to combine header width with a small sample of representative rows, or to measure cell width in a hidden sizing layer (e.g., an offscreen DOM element). Measuring every rendered value would be too costly, and constantly recalculating and adjusting widths during scroll is definitely undesirable.

Custom renderers need an extra rule. For text-like content, sampling the accessor output is often good enough. For richer cells such as avatars, badges, links with icons, or action menus, the raw value is not a reliable proxy for rendered width. In those cases, the component should either calculate an estimated width during the initial sizing pass or expect the consumer to provide width or minWidth. That keeps rich cells predictable without requiring the table to inspect every mounted instance.

Consider Pretext for cheap multiline text measurement
Pretext is a modern library for multiline text measurement and layout. It sidesteps the need for DOM measurements by implementing its own text measurement logic, using the browser's own font engine as ground truth. It can be used to cheaply calculate width for contents.

Once the natural width is known, compare it against the viewport width:

If the viewport is larger, distribute the extra space across flexible columns.
If the viewport is smaller, shrink flexible columns proportionally until they hit minWidth.
If the table is still wider than the viewport after all shrinkable columns reach minWidth, keep horizontal scrolling instead of over-compressing the content.
That last point is important. A table should not pretend it can always fit by making every column unreadably narrow. It is usually better to preserve legibility and allow horizontal overflow than to crush the layout.

Yes

No

viewport larger

viewport smaller


Yes

No

Column definitions

#### Has explicit width?


Use consumer width

Estimate from header plus sample rows

Clamp by minWidth and maxWidth

Sum to naturalTableWidth

Compare with viewportWidth

Distribute slack to flexible columns

Shrink flexible columns down to minWidth

Keep preferred widths

computedColumnWidths

#### Still wider than viewport?


Allow horizontal overflow

Publish via colgroup shared by thead and tbody


Column width resolution pipeline
Late-arriving rows introduce another subtle tradeoff. If the table keeps recomputing widths whenever new data appears, the columns will visibly jump around. A steadier default is to size from headers plus an initial sample, then let unusual outliers follow the column's overflow policy instead of immediately forcing a remeasure. In the base design, remeasurement can stay an internal policy that runs only when the column model changes, the viewport changes materially, or the implementation deliberately performs a new sizing pass.

Once the component supports explicit constraints and virtualization, application-managed sizing is easier to reason about than relying entirely on the browser's auto table layout. The browser is still useful for basic table behavior, but the sizing policy should live in the component's column model.

After the widths are resolved, they should be applied through one shared mechanism rather than allowing the header and body to size themselves independently. In native table mode, a <colgroup> is a clean way to publish the computedColumnWidths map so that both <thead> and <tbody> follow the same column model. In that mode, table-layout: fixed is usually the better companion once the component owns the width map, because it makes truncation, sticky-header alignment, and virtualization behavior more deterministic. Browser auto layout is still reasonable for a simpler non-virtualized table that mostly defers sizing to content, but it should not be the primary strategy once the component is doing explicit width allocation. If the implementation later switches to a div + ARIA table, that same width map can drive a shared grid-template-columns or explicit inline widths. The key idea is that header and body alignment come from one authoritative width source.

Pair the column width map with table-layout: fixed
Once the component computes explicit column widths, table-layout: fixed makes rendering O(columns) per row instead of letting the browser scan content to decide widths. That is what keeps sticky-header alignment, virtualized row insertion, and truncation behavior consistent at scale; auto layout can silently diverge from the shared width map as new rows mount.

Sticky header implementation details
The architecture already chose one bounded scroll container. At implementation time, the main concerns are sticky positioning and visual layering. Header cells can use position: sticky with top: 0, plus an explicit background color and z-index so scrolling rows do not bleed through visually. If the design uses borders or shadows to separate the header from the body, those should stay attached to the sticky header cells so the boundary remains visible during scroll.

Resize handling matters here as well. When the viewport changes, the component should run one sizing pass and publish the updated widths through the shared column model so the sticky header and scrolling body remain aligned. This is another reason to avoid separate header and body scrollers unless the design has already outgrown the native-table architecture.

Row virtualization
For datasets with thousands of rows, row virtualization is the main optimization. The core idea is the same as in Autocomplete: render only the visible window plus a small overscan buffer instead of mounting the whole dataset.

With fixed row height, the visible window can be computed cheaply:

visibleRowStart = max(0, floor(scrollTop / rowHeight) - overscan)
visibleRowEnd = min(rowCount - 1, ceil((scrollTop + viewportHeight) / rowHeight) - 1 + overscan)
In other words, visibleRowEnd should be treated as an inclusive bound. The mounted window is the clamped range [visibleRowStart, visibleRowEnd], not a half-open interval.

In the scoped design, the DOM only contains that row window plus top and bottom spacer rows inside <tbody>, so the scrollbar still reflects the full logical height while the table keeps native header and cell relationships. Those spacer rows should be treated as implementation detail rather than real data rows: render them with one empty cell spanning all columns, keep them free of focusable content, and ensure actual data rows still carry the correct logical ordering. If a later version needs translated layers or absolutely positioned rows for more aggressive windowing, that is the point where switching the internals to a div + ARIA table becomes more reasonable.

tbody (window + spacers)
Virtualization controller
Scroll container
tbody (window + spacers)
Virtualization controller
Scroll container
[Window unchanged]
[Window changed]
Scroll (update scrollTop)
Report scrollTop, viewportHeight
Compute [visibleRowStart, visibleRowEnd] with overscan
No-op
Update top spacer height (visibleRowStart * rowHeight)
Mount rows in new window (keyed by getRowId)
Update bottom spacer height ((rowCount - visibleRowEnd - 1) * rowHeight)


Row virtualization on scroll with spacer rows
Spacer rows must never carry focusable content or readable text
If spacer rows contain interactive elements or readable text, assistive technologies treat them as real rows and keyboard users can tab into empty space. Keep them as a single empty cell with colSpan={columns.length}, no focusable descendants, and ideally aria-hidden="true", so they stay a pure layout artifact of the virtualization window.

This keeps the number of mounted row nodes proportional to viewport size instead of dataset size. It also makes the performance discussion concrete:

Stable row identities prevent unnecessary remounts.
Fixed row height keeps the math cheap.
Overscan prevents flashes of empty content during fast scrolling.
Lightweight cell renderers keep scrolling smooth.
Virtualization reduces DOM size, but it does not automatically prevent expensive rerenders. In a React implementation, scroll state and viewport calculations should stay localized to the table controller, while visible rows and cells receive stable inputs whenever possible. Stable column definitions, lightweight formatter functions, and memoized row or cell subtrees can all help keep scroll work proportional to the visible window rather than redoing unnecessary work across every mounted cell.

Do not key rows by array index in a virtualized table
Use the stable identity returned by getRowId as the React key. Index-based keys cause rows to get reconciled to a different underlying record as the window shifts during scroll, which breaks row-local state, defeats memoization, and can produce subtle visual glitches where cell content lags behind the intended row.

In interviews, it is usually fine to assume the data has already been fetched and is handed directly to the component. If a real product later needs side-loading, server pagination, or infinite loading, the same rows and columns contract can remain intact while fetching stays outside the table itself. That keeps the primitive reusable instead of binding it to one transport pattern.

Column virtualization
Column virtualization is a valid follow-up optimization, but it should not be the base recommendation. In most product tables, row count becomes a problem much earlier than column count.

Column virtualization also adds cost:

Header and body alignment get trickier.
Keyboard navigation and accessibility metadata become harder to maintain.
Sticky regions and future fixed-column support become more complex.
If the interviewer pushes on extremely wide surfaces with hundreds of columns, column virtualization becomes more compelling. That is closer to the tradeoff space in the Google Sheets system design article, where both axes are large enough to justify heavier viewport-related computation logic.

### Accessibility

The basic design should keep semantic table behavior:

Header cells should use proper table header semantics such as <th scope="col">.
The table should expose a clear accessible name through a visible caption, aria-labelledby, or aria-label.
When loading is true, the table region or scroll container should expose aria-busy="true" so assistive technologies know the content is updating.
Empty and error states should be announced in a way that makes sense for assistive technology users. If they replace the body, rendering them as a full-width row with colSpan={columns.length} preserves table context; if they are transient status messages, an adjacent live region may be more appropriate.
mount with loading=true

rows.length === 0

rows.length > 0

errorState provided

loading=true with rows present

new rows arrive

fetch fails


Empty

Ready


Refreshing

aria-busy="true" on scroll container
while existing rows stay visible


![Table presentation lifecycle](data-table-pdf-assets/figures/table-presentation-lifecycle.png)

Virtualization adds one important nuance: the full dataset may no longer be present in the DOM. If the implementation stays in native-table mode by using spacer rows inside <tbody>, header associations remain straightforward, but the spacer rows must not behave like meaningful data rows to assistive technologies. They should be empty, non-interactive layout artifacts, while the mounted data rows preserve the expected reading order and, if necessary, expose logical indices. If browser and screen-reader testing shows that spacer rows still create confusing row navigation, that is a good reason to switch the internal implementation to a div + ARIA table or grid structure. In that mode, metadata such as aria-rowcount, aria-colcount, aria-rowindex, and aria-colindex becomes more important so that assistive technologies still understand the logical table size and the position of each visible row.

The article should also be explicit that role="grid" is not the default. It becomes appropriate only when the table is meaningfully interactive and needs composite-widget keyboard behavior.

Rich cell content
Column renderers add support for mixed content types, but a few default layout rules should be established:

Text-heavy columns should use an explicit overflow policy instead of overflowing unpredictably. In the fixed-row-height virtualized base design, truncation or clipping is the safer default, while multi-line wrapping fits better in non-virtualized or variable-height mode.
Numeric and date columns often read better when aligned consistently.
Image cells should use constrained thumbnails or avatars so they do not blow up row height.
Rich non-text cells usually need either offscreen DOM measurements or explicit width constraints so the sizing algorithm does not guess from raw values.
Custom renderers should be lightweight and predictable because they sit in the hot rendering path during scrolling.
Further extensions
Here are other practical follow-up features that teams usually want but are typically out of scope for interviews:

Sorting
Filtering
Paginated data
Column resizing
Sticky/frozen columns
Sticky footers
Row actions or row selection
Those features are compatible with the overall architecture, but they should layer on top of the same core design: declarative columns, one shared sizing model, one scroll container, and virtualization that is primarily row-oriented.

## Summary

Scope DataTable as a reusable, read-heavy component. It turns rows, declarative columns, and a stable getRowId into a semantic, accessible table capable of handling thousands of rows and dozens of columns. Sorting, filtering, pagination, selection, and inline editing stay outside the core contract and attach as additive extensions, keeping the default surface small and the performance envelope predictable.

Keep semantics and column configuration in the core contract. Native <table>, <thead>, and <tbody> markup lives inside a single bounded scroll container so the browser handles semantics and alignment for free, while ColumnDef<Row> concentrates per-column concerns like accessor, renderCell, width constraints, and overflow into one configuration object.

Share one width contract between header and body. The column model feeds a computedColumnWidths pipeline that resolves explicit widths, estimates the rest from headers plus a small sample, clamps against min and max bounds, and distributes remaining space. Publishing the result through a <colgroup> paired with table-layout: fixed keeps header and body widths from drifting.

Use row virtualization as the main scale lever. A fixed rowHeight and overscan derive an inclusive [visibleRowStart, visibleRowEnd] window, with spacer rows preserving scroll height and getRowId keying real rows so React reconciliation stays correct.

Let native table semantics carry accessibility until the component becomes interactive. <th scope="col"> on headers, aria-busy during loading, full-width colSpan rows for empty and error states, and role="grid" held in reserve for the interactive escalation path keep the baseline accessible without overcomplicating the default component.

Start with the semantic table structure plus a shared width contract, then present row virtualization as the single performance lever that makes the scale target realistic.

## References

The references below group data-table sources by the topic they inform, so it is easier to map each source back to the part of the article it relates to: accessibility patterns, layout and CSS primitives, virtualization, production data-grid libraries, and related articles and coding questions.

### Accessibility patterns

This subsection links the canonical ARIA authoring guidance for read-heavy tables versus interactive grids, which underpins the semantics choices in the article.

WAI-ARIA APG: table pattern for the accessibility model of read-heavy tabular content.
WAI-ARIA APG: grid pattern for the tradeoffs when a table evolves into a more interactive composite widget.
React Aria: useTable for accessible table structure and reusable component patterns.
Layout and CSS primitives
This subsection covers the CSS features that sizing, sticky headers, and column widths rely on.

MDN: position for sticky positioning behavior within a scroll container.
MDN: table-layout for the tradeoffs between automatic and fixed table layout.
Virtualization and production data grids
This subsection links the windowing model used by row virtualization along with practical guidance from mature production data-grid libraries.

web.dev: virtualize large lists with react-window for the core windowing model behind row virtualization.
MUI X: data grid - layout for the relationship between bounded container sizing and virtualization.
AG Grid: DOM virtualisation for practical production tradeoffs around viewport rendering.
AG Grid: row height for the implications of fixed versus variable row heights on virtualization.
Related system design articles
The articles below overlap with the data table on virtualization and interactive grid surfaces.

Collaborative spreadsheet (Google Sheets) for the point where a read-heavy table evolves into a more interactive grid with two-axis virtualization, cell-level semantics, and spreadsheet-style behavior.
Autocomplete for another example where viewport rendering and virtualization become important once the result set grows beyond what should be mounted all at once.
Related coding questions
The coding questions below build up the same column-definition and table-rendering ideas from this system design article in a hands-on format.

Data table for the basic rendering and pagination mechanics of a tabular UI.
Data table II for adding column sorting behavior on top of the base table.
Data table III for practicing the same column-definition and reusable API ideas discussed in this system design article.
Data table IV for extending the generalized table with per-column filtering.