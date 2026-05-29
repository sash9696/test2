---
title: "Poll Widget"
slug: "poll-widget"
difficulty: "medium"
duration_mins: 30
tags:
  - system-design
  - ui-component
premium: true
summary: "Design a poll widget that can be embedded on websites."
free_preview: false
status: draft
---

# Poll Widget

## Question

Design a poll widget that can be easily embedded on websites, such as articles and blogs, to allow website viewers to vote on options.

Poll Widget Example

## Requirements

The widget displays the following information:
A list of options for users to vote on.
The latest number of votes for each option. This is only shown after the user has submitted a vote.
The following backend APIs are provided:
Fetching of the poll results.
Record a new vote on an existing poll option.
Remove a vote from an existing poll option.
It should not require much effort for a website owner to embed a poll widget within their website.
## Requirements exploration

These are questions you should be asking your interviewer to dive deeper into the problem and refine the requirements.

#### What are the most critical aspects of the component?

Ease of embedding the polling widget within websites.
### User experience as a voter.

#### Does the widget show details of people (e.g. thumbnail) who voted for an option?

Whether we show that level of detail will affect the data model and APIs. We assume a basic version where we only have to show the count.

#### Can a user vote on multiple options?

Yes, a user can vote on multiple options.

#### How do we determine the length/proportion of each option bar to render?

You are free to decide that.

#### In what order should the options be shown? By popularity/user has voted/random?

Popularity.

#### How are the options determined? Can users add more options?

The poll is created by the website owner in a separate admin portal, and the options are determined during poll creation and cannot be modified afterward.

#### Is there a maximum number of options shown in the widget?

The maximum number of options is 6.

#### Must users be logged in on the page in order to vote?

Anyone can vote regardless of whether they are logged in or out. Votes should be persisted for the same user.

## Architecture

The architecture depends on how the widget is delivered to third-party pages and how its components communicate with the host page and the backend.

### Rendering approach

We should first evaluate the possible rendering approaches because they will affect the architecture and subsequent discussions.

Generally speaking, there are two approaches for rendering external widgets/components on a page:

Render in an <iframe> (different browser context)
Render directly within the page (same browser context)
Note that we're talking about how to render the widget, which is different from the distribution approach, meaning how to run the code that renders the widget.

Render in an <iframe> (different browser context)
<iframe> (inline frame) is an HTML tag on a page that accepts a src attribute, which is a URL for a website you want to embed within the host website.

Popular embeddable widgets such as Facebook's Like button, Twitter's embedded tweets, YouTube's embedded videos, and Disqus' embedded comments are iframes. They are essentially websites that only render the content to be embedded.

Check out these examples to see for yourself:

Facebook Like Button


Twitter Embedded Tweets


Using iframes is a very common technique for embedding your content within third-party websites. In the case of a poll widget, the widget will be the only UI being rendered on the website that's defined as the iframe's src.

Pros

An iframe is a separate website and hence a separate browsing context. The contents of the iframe are isolated from the hosting site and vice versa.
The styling of the widget will not be affected by any CSS of the host website.
The widget's JavaScript environment is not affected by scripts running on the host website, which can contain polyfills or monkeypatching, affecting runtime behavior in an unpredictable manner.
Simple to set up because adding an <iframe> to the page only involves changing HTML. Users do not need much technical knowledge to achieve that.
Cons

Needs a web server to host a website that renders the widget. This is not a huge deal because a web server is needed to serve the poll results anyway. Depending on the type of website you build, this setup can range from simple to complex.
It is slower to load a separate website than to render it directly into the page.
Because of the isolation, the host website cannot use CSS to customize the component internals.
There are two common ways to get the <iframe> on the page, using Facebook's Like Button developer documentation as an example:

1. Run JavaScript code that dynamically adds an <iframe> into the DOM.

Facebook Like Button Plugin Docs JavaScript SDK

2. Directly embed the <iframe> code into the HTML.

Facebook Like Button Plugin Docs Inline Frame

The benefit of the JavaScript approach is that the script can customize the iframe rendering (e.g. theme, size) based on the environment if necessary. The direct <iframe> approach is simpler but less flexible.

Render within the page (same browser context)
Just like how a script can dynamically inject an <iframe> into the DOM, it can also directly add the DOM elements needed to render the poll and the poll results.

Pros

It is fast to render the poll widget within the same page.
Cons

The poll widget can be affected by the host website's JavaScript environment and global styling. There's no telling what kind of global styling is present, and it is very likely for the widget's appearance to be affected by the website's CSS.
The question is how to run the third-party JavaScript on the page to achieve the above.

1. Download the script via a CDN.

This approach is similar to Facebook's Like Button JavaScript SDK approach: add a <script> tag to the page that downloads and executes some external JavaScript.

2. Distribute the code via npm

npm is a package manager for JavaScript projects, and the widget code can be packaged into an npm project so that projects can add it as a dependency. Website owners will need to possess the technical knowledge of how to add a new npm package to their package.json.

Distributing only via npm is not a great idea because not all websites are built using a JavaScript-based build system. Low-code websites like WordPress, Webflow, and Blogger do not allow for including third-party JavaScript code on the page via npm.

#### Which rendering approach is better?

For embedding widgets, the <iframe> approach is clearly superior due to the encapsulation of styles and environment that an iframe provides.

Default to iframe embedding for third-party widget distribution
Same-context rendering exposes the widget to unpredictable host CSS and JavaScript (polyfills, monkeypatching, global styles), and there is no practical way to guard against every host environment. Iframe isolation is what makes the widget behave the same across every site that embeds it, which is the whole point of the product.

#### Which distribution approach is better for rendering <iframe>s?

Embedding <iframe>s directly in the HTML is the easiest, and the upsides of using a <script>-based approach are not that significant. That said, it'd be nice to give developers the option to choose between the two approaches. Facebook and YouTube provide both JavaScript SDKs and direct <iframe> embed options.

The discussion below will assume an <iframe> embed approach.

Diagram
Poll Widget Architecture

Component responsibilities
Host Website: Host website that embeds the widget as an <iframe>.
App Server: Renders the widget UI as a website by serving the HTML, CSS, and JavaScript needed.
API Server: Server that returns poll result JSON data for the widget and also accepts new polls.
Client Store: The module that interacts with the API server and stores data for the UI.
Polls UI: UI for the poll options.
Note: The App Server and API Server have been split into separate components for clarity, but they can be the same server with the same domain.

The relationships between these pieces can be summarized as a flow of responsibilities, from the host site embedding the iframe, to the app server serving the widget document, to the API server returning tallied poll data.


Host website
(third-party page)

Iframe
greatpollwidget.com/embed/:pollId

App server

API server

Redis cache
tabulated results

Votes DB

Client store

Polls UI


Iframe embed topology across host site, app server, and API server
## Data model

The data model for a polling widget is quite simple. Something like this will be sufficient:


const state = {
lastUpdated: 1628634891,
totalVotes: 421,
question: 'Which is your favorite JavaScript library/framework?',
options: [
{
id: 123,
label: 'React',
count: 234,
userVotedForOption: false,
},
{
id: 124,
label: 'Vue',
count: 183,
userVotedForOption: true,
},
{
id: 125,
label: 'Svelte',
count: 51,
userVotedForOption: true,
},
// ...
],
// Which option(s) the user has selected.
selectedOptions: [124, 125],
};
Only the selectedOptions field is client-only state; the rest of the fields are server-originated data.

On the server side, the same data is backed by a few related entities: the poll itself, its options, the votes cast by users, and the relationship between a viewer and the options they have selected.


totalVotes


lastUpdated


pollId


userId


optionId


createdAt


cookieFingerprint


Server-side poll entities and viewer-vote relationship
## Interface definition (API)

There are three kinds of APIs to discuss:

Embed API: How websites should embed the <iframe>.
Components API: How to build the poll widget and the props the components accept.
Server APIs: The HTTP APIs to fetch results, record new votes, and remove votes.
Embed API
Websites should be provided with some code to be copied and pasted to render the poll widget. The iframe's src attribute should be a unique URL specific to a poll instance.


<iframe
src="https://greatpollwidget.com/embed/{poll_id}"
style="border:none;overflow:hidden"
title="Poll widget for your favorite JavaScript framework"
frameborder="0"
scrolling="no"
style={{
height: 200,
width: '100%',
maxWidth: 450,
}}
/>
iframes by default are rendered with default styling, so attributes like frameborder="0", scrolling="no", and the inline styles help remove borders and scrollbars to make the widget appear to be part of the page and not obviously rendered by an iframe.

Components API
Poll
Server URL
PollOptionList
Maximum number of options displayed
PollOptionItem
Label
Number of votes for an option
Event handlers: onClick

// Example code in React
<Poll submitUrl="https://greatpollwidget.com/submit/{poll_id}">
<PollOptionList>
{options.map((option) => (
<PollOptionItem
key={option.id}
label={option.label}
count={option.count}
isSelected={option.userVotedForOption}
onVote={() => {
submitVote(option.id);
}}
onUnvote={() => {
removeVote(option.id);
}}
/>
))}
</PollOptionList>
</Poll>
Server APIs
The server APIs are ideally served on the same domain so that there is no need to mess with CORS, e.g. https://greatpollwidget.com/api/{poll_id}/results and https://greatpollwidget.com/api/{poll_id}/submit.

Fetching results
#### What format should the API return the results in?


1. Return the options and the counts for each: The server returns something like:


{
"totalVotes": 421,
"question": "Which is your favorite JavaScript library/framework?",
"options": [
{
"id": 123,
"label": "React",
"count": 234,
"userVotedForOption": false
},
{
"id": 124,
"label": "Vue",
"count": 183,
"userVotedForOption": false
},
{
"id": 125,
"label": "Svelte",
"count": 51,
"userVotedForOption": false
}
]
}
Pros
Client does not need to do tabulation of results, simply render the results.
Payload is small, contains exactly the data that needs to be displayed.
Cons
Server has to do the processing, but this is likely better because the server can cache/memoize the results, especially for popular polls, and return the cached results for different users.
The server will need to update the tabulation every now and then. This is a good use case for in-memory key/value stores like Redis/Memcached.
The tabulation pipeline that backs this API shape typically separates the write path (incoming votes) from the read path (cached tallies) so that hot-poll reads stay fast regardless of write volume.


stream changes

write tallies


JSON response

Widget
(iframe)

API server

Votes DB

Aggregation worker

Redis
per-poll counts


Vote tally aggregation pipeline
2. Raw list of poll responses: The server returns something like:


{
"question": "Which is your favorite JavaScript library/framework?",
"options": [
{
"id": 123,
"label": "React"
},
{
"id": 124,
"label": "Vue"
},
{
"id": 125,
"label": "Svelte"
}
],
"votes": [
{
"optionId": 123,
"createdAt": 1628634891
},
{
"optionId": 123,
"createdAt": 1628634892
},
{
"optionId": 124,
"createdAt": 1628634893
}
// ...
]
}
Pros
Can do all the tabulation entirely on the client side, and hence sorting/filtering does not incur any network latency.
Cons
Doesn't scale well when there are lots of responses; the network payload will be huge.
The client needs to do the tabulation of votes, which can be expensive on low-end devices and when there are many votes.
#### Which to choose?


Directly returning the counts is usually the superior option, as there is typically no need to do tabulation or manipulation of the data on the client, which does not scale for popular polls with tens of thousands of votes.

Submitting votes and unvoting
The vote submission/unvoting APIs can accept a list of option IDs and return the updated poll results in a similar format to the poll results fetching API.

## Optimizations and deep dive

This section covers the production details that make an embeddable poll reliable, understandable, fast, and accessible on third-party pages:

Persisting votes across sessions: This section explains how anonymous voters can be recognized across reloads and repeat visits.
Rendering the poll options: This section discusses ways to render result bars and option states without losing clarity or accessibility.
User experience: This section covers hidden results, vote changes, loading states, confirmation, and other poll lifecycle details.
Performance: This section explains how to keep an embedded widget lightweight, fast to initialize, and cheap to render.
Accessibility: This section covers text equivalents for visual results, keyboard interaction, and assistive-technology behavior.
Network: This section discusses out-of-order vote responses, retries, and request handling for quick vote or unvote actions.
Internationalization (i18n): This section covers localized widget strings, creator-provided content, and iframe language configuration.
Persisting votes across sessions
Because anyone can vote without having to log in first, we will need a way to identify users across sessions; otherwise, the user will not see that they have already voted after the browser tab is closed.

We can use cookies to uniquely identify each user by generating a unique string-based fingerprint (e.g. using uuid) to serve as the user ID cookie during the initial load if there's no existing user ID cookie present.

This helps the poll widget website identify users in order to track the options they have already voted on and prevent a user from voting for the same option multiple times.

Note that users can work around this by using different browsers or different devices. The only way to prevent this sort of abuse is to have user authentication.

Cookie fingerprinting is not a vote-stuffing defense
A UUID cookie only keeps the same browser session from double-voting. It is trivial to bypass by clearing cookies, switching browsers, or using incognito windows. Treat this as UX polish for returning voters, not as an integrity control; real abuse prevention requires authenticated users, rate limiting, and server-side checks.

Rendering the poll options
The polling results consist of multiple bars of various widths, and there are multiple ways to render such a result. It's worth discussing the various ways of rendering bars of varying widths.

What a full bar represents
A full bar can have two common representations:

Full width represents 100% of all responses. If an option is X% of the total, it will take up X% of the container width.
Accurate representation of the proportion of options.
Bad if the ratio is very even and there are many options with low %. Hard to discern because many bars will be very short.
The top-voted option is rendered with the full width, and other options are a proportion of it. E.g. the top-voted option is 40% of the total votes but will take up the entire container width. If another option is 20%, it will take up half the width.
Useful for highlighting the relative difference between the options.
Might give viewers a false impression that the top-voted option has a higher ratio than it actually does.
The first option is the more common one and is used by Reddit and Twitter.

Rendering bars of dynamic widths
The number of votes for an option over the total number of votes will be the proportion of the full width to render the bar. E.g. 400 votes for an option out of a total of 1000 votes will mean that the bar should be rendered at 40% of the full width. Since there are infinitely many possible values for the width, it is not practical to use static CSS classes to render a bar for a specific width. A better way would be to use inline styles that are dynamically generated during rendering.

1. Using CSS width inline style: This is the most common approach, and the only small downside is that if there's a need for animating the bars expanding/contracting, animating the width property is slower than transform. However, the widget is mostly static, so the animation issue is largely not present.


<div style="width: 40%">React</div>
<div style="width: 30%">Vue</div>
2. Using transform: scaleX() style: This method involves scaling the element horizontally.


<div style="transform: scaleX(40%)">React</div>
<div style="transform: scaleX(30%)">Vue</div>
Note that scaleX() will also transform the contents within it and make them appear horizontally squished.

We should use a % of the full width of the widget instead of hardcoded px values calculated once when the page loads, so that if the widget is resized, the bars' widths will be updated. % widths can be achieved using width and transform: scaleX().

If we are fine with some precision loss, then we could have 101 class names for percentages from 0 to 100. But in general, this is not a great approach, and inline styles are the preferred approach.

### User experience

The widget moves through a small set of visible states over its lifetime, and the UX guidelines below are best understood against that lifecycle: when results are shown, when they are hidden, and how the user can unvote, change their vote, or recover from errors.

results fetched

fetch failed


user picks option

"See responses"


server confirms

request failed


change / unvote


NotVoted

LoadError

Voting

ResultsPreview

Voted

VoteError


![Poll widget lifecycle and vote states](poll-widget-pdf-assets/figures/poll-widget-lifecycle-and-vote-states.png)

When the poll is still loading, instead of showing a spinner, use a shimmer loading effect in the shape of bars to hint that this is a poll and also to reduce layout thrash after the poll is loaded.
Poll results should be hidden before the user has voted, to reduce bias.
Consider having a "See responses" function for people who just want to see the results without voting.
### Performance

Because the widget is embedded on third-party pages, every kilobyte and millisecond matters. The subsections below cover fast initial rendering and keeping the bundle small.

Fast rendering
As mentioned earlier, to achieve fast rendering of the results, the server API should return the tabulated results, not raw results, so that the client does not need to do any tabulation of results.

Prefer to use server-side rendering for the initial load rather than fetching the poll results over AJAX, for a fast initial load.

Server-render the iframe document so the first paint already has results
Because the widget lives on third-party pages, it is competing with the host page's own network and CPU budget. Returning fully-rendered HTML from the embed URL avoids a render-blank-then-fetch-then-repaint sequence, and pairs well with server-side caching of tabulated results for popular polls.

Fast interactions via optimistic updates
Optimistic updating is a technique where the browser shows the new UI state after a server request is fired, even before a response is received from the server. Because the client has the current results during initial load, it is possible to increment the newly-voted option and compute the new ratio of all the bars on the client side.

However, there are a few caveats:

This optimization is more useful for polls with very few votes, where each new vote will make a noticeable difference to the visual results. For polls with many existing votes (>500), an extra vote will not result in a noticeable difference in the widths. Optimistic updates can be skipped for such polls.
The client-side computation might be stale if the poll is a popular one with people constantly voting on it, because the server response will contain many new votes since the widget was first loaded.
In most cases, the client can render an optimistic update first and then render again with the most up-to-date results from the server.

API server
Client (store + view)
API server
Client (store + view)
Apply optimistic update, increment count, recompute bar widths
Replace optimistic state with canonical tallies
Revert count, show inline error, offer retry
[Success]
[Failure]
Click option
Fresh tabulated results


![Cast-vote flow with optimistic update and reconciliation](poll-widget-pdf-assets/figures/cast-vote-flow-with-optimistic-update-and-reconcil.png)

Skip optimistic bar updates once a poll has many votes
Incrementing a single vote against a total of hundreds or thousands produces no perceptible change in bar width, so the added complexity of client-side tabulation buys nothing. Reserve optimistic rendering for low-volume polls where a single new vote actually shifts the visual, and fall back to waiting on the server response for popular polls.

Scalability issues
Given that there are only a maximum of 6 options, we will not run into the issue of rendering too many options and a large resulting DOM size. But if we do, we can use virtualized lists within a container with a max height to prevent the component from being too tall, and only render the on-screen options within the container.

### Accessibility

A poll is highly visual, so accessibility work centers on giving assistive tech a text-equivalent story and making the interactive elements keyboard operable.

Screen readers
Polling widgets are inherently very visual UI elements, and we need to take special care to ensure users relying on screen readers can still understand what is being shown on the screen.

Screen reader users will not know how long a bar is; hence aria-label, title, and aria-describedby need to be used for the poll options to indicate the option name, the number of votes, and the percentage if they are not in the rendered visual output.
Use aria-live to announce updates for any change in values of the results when the client receives a server response.
ARIA roles for options: role="radiogroup" and role="radio" for polls where only one option can be selected.
Match ARIA roles to the poll's selection model
Use radiogroup/radio only for single-select polls. Multi-select polls (like the one assumed in this design) should use checkbox semantics instead, because radio groups imply mutually exclusive selection and will confuse screen reader users when more than one option can be chosen.

Keyboard interaction
<button>s are preferred for rendering poll options, but if there's a reason to use <div>s, they should be made focusable by adding the tabindex="0" and role="button" attributes.
### Network

Request responses could be out of order if someone votes/unvotes in quick succession.
Keep track of the latest response and ignore stale responses.
Show errors in the UI if the API submission fails.
Out-of-order vote and unvote responses will corrupt the displayed tally
If a user toggles the same option quickly, the unvote response can arrive after the later vote response and silently revert the UI to the wrong state. Tag each in-flight request with a monotonically increasing sequence number (or track the latest request ID per option) and discard any response that is not the newest; do not just trust response arrival order.

The sequence below illustrates how a monotonically increasing request ID lets the client ignore a stale unvote response that arrives after a newer vote response.

API server
Client (store + view)
API server
Client (store + view)
UI reflects seq=2 only
Click option (unvote)
Click option again (vote)
Response (seq=2), latest, apply
Response (seq=1), older than latest, discard


Dropping stale responses during rapid vote and unvote toggling
### Internationalization (i18n)

If there's a need for i18n of strings in the poll, especially the strings not from the poll creator (e.g. aria-labels), the iframe embed URL can accept a query parameter for the language, and it'll be up to the website owner to provide the right language.

## Summary

The defining constraint is that the widget runs inside arbitrary third-party pages. The publisher's CSS, JavaScript, and analytics are outside the widget's control, so the first design decision is isolation. That is why the default distribution format is an <iframe> served from a poll-specific URL rather than a script that injects markup into the host DOM.

The <iframe> embed separates delivery from data. The App Server owns the iframe document and can server-render the initial bars for a fast first paint, while the API Server handles tabulated counts and vote mutations behind a Redis cache of per-poll totals. The two read and write paths then scale independently, and the component surface a Poll containing a PollOptionList of PollOptionItem children with onVote and onUnvote callbacks remains coherent whether a publisher picks the iframe or the scripted path.

Isolation keeps the client model and vote API flat. Since the iframe owns its own runtime, the client can keep question, totalVotes, options carrying count and userVotedForOption, and a client-only selectedOptions array for the in-flight multi-select UI. Server-side, that maps to POLL, OPTION, and VOTE entities keyed by a USER identified through a cookie fingerprint so anonymous voters keep continuity across sessions. Returning pre-tabulated counts, rather than raw vote lists, keeps popular polls cheap to read, and having the submit and unsubmit endpoints echo back the fresh tally gives the widget an authoritative number to reconcile against on every interaction.

The frame boundary dictates reconciliation and accessibility. Optimistic bar updates make a vote feel instantaneous, get skipped on high-volume polls where a single vote is imperceptible, and are protected by a monotonic sequence number on each vote and unvote so stale responses are dropped rather than clobbering newer state. Because nothing outside the iframe can be relied on, the accessibility story has to be carried entirely inside the widget frame: semantic structure, aria-live for result changes, and keyboard-operable <button> options so screen reader and keyboard voters reach the same outcome as pointer users. In an interview, start with the iframe isolation decision; every other choice here is a consequence of it.

## References

The references below group poll-widget sources by the topic they inform, so it is easier to map each source back to the part of the article it relates to: production embed examples, iframe and cross-origin messaging, security considerations, and related system design articles.

Production embed examples
This subsection collects real third-party embed products whose iframe and script patterns inform the poll widget's embedding model.

Facebook like button for a mature script-plus-iframe embed that hosts interactive state across origins.
Twitter's embedded tweets for an embed that progressively enhances a static fallback into a full iframe widget.
Disqus universal embed code for a widely deployed comments embed, useful as a model for configuration and theming of the poll widget.
Iframe embedding and cross-origin messaging
This subsection covers the browser mechanics behind embedding the widget into third-party sites and communicating with the host page.

iframe | MDN for the element that isolates the widget from the host page's DOM and scripts.
Window.postMessage() | MDN for the cross-origin messaging primitive used to coordinate height, theme, and events between the widget and its host.
Security considerations for embeds
This subsection covers the web platform features that keep third-party embeds safe for both the host page and the widget origin.

Content security policy (CSP) | MDN for controlling which origins can embed the widget and which resources it may load.
Cross-origin resource sharing (CORS) | MDN for the rules that govern API calls made from the embedded widget back to its backend.
SameSite cookies | MDN for how cookie scoping affects user identification and vote-dedup for an embedded widget.
Related system design articles
The articles below overlap with the poll widget on social interactions and third-party embedding.

News feed (Facebook) for the surrounding social surface where embeddable widgets like polls are often consumed and rendered at scale.