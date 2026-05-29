---
title: "Chat App (e.g. Messenger)"
slug: "chat-app"
difficulty: "hard"
duration_mins: 40
tags:
  - system-design
  - networking
premium: true
summary: "Design a chat application like Messenger and Slack."
free_preview: false
status: draft
---

# Chat App (e.g. Messenger)

## Question

Design a chat application that allows users to send messages to each other.

Chat Application Example

## Real-life examples

Messenger
WhatsApp Web
Slack
Discord
Telegram
## Requirements exploration

Before picking a transport or a storage model, clarify the core interactions, the delivery expectations, which message formats are in scope, and how the app behaves when the network is unavailable.

#### What are the core functionalities needed?

The must-have interactions are small and tightly scoped for an interview.

Sending a message to a user.
Receiving messages from a user.
Seeing one's chat history with a user.
#### Is message receiving real-time?

Yes, users should receive messages in real-time, as fast as possible without having to refresh the page.

#### What kind of message formats should be supported?

Support text, which can contain emojis. Image support can be discussed if there is time.

#### Does the application need to work offline?

Yes, where possible. Outgoing messages should be stored and sent out when the application goes online, and users should still be allowed to browse messages even if they are offline.

#### Are there group conversations?

Assume a 1:1 messaging service for the initial scope.

## Architecture / high-level design

The main difference between traditional applications and chat applications that can be used offline is that if the application loses network connectivity, some functionality like browsing on-device messages and searching should ideally still work. This greatly influences the application architecture, and it will differ a great deal from conventional web applications.

Treat the chat client as local-first, not as a thin view over server state
Unlike a typical web app where the UI reads directly from server responses, a chat client must remain usable when the network drops. That single constraint drives every major decision in this article: a persistent client database, optimistic writes, a scheduler that owns outgoing requests, and a sync model that reconciles with the server whenever connectivity returns. The posture is real-time first and offline capable, not offline first: the WebSocket is the default path for every message, and the local database exists so the UI never has to block on or degrade without it.

Tricky scenarios to be handled
A chat client has to survive a handful of messy real-world conditions. Naming them upfront makes the later architecture decisions easier to justify.

Using the application on different tabs in the same browser. The user might do this because they want to chat with different people at once and would rather switch between tabs than switch between conversations in the same tab.

Users should see the same messages for each conversation -> Rely on storage that is accessible across different tabs in the same browser.

Using the application on different devices/browsers. The same device, different browser case is rare, but it's not rare for users to be using multiple devices at once (work and personal device at the same time).

Users should see the same messages for each conversation -> Sync with the server on initial load and get the most updated data.

Application goes offline during usage. The user could have lost connectivity while on the move and passing through low connectivity areas (a common occurrence in the subway).

Outgoing messages have not all been fulfilled -> They should be retried when the application goes online again, or their statuses should be updated if they were written to the server but didn't receive the updates.
Messages are being sent while the application is offline -> These messages should be sent out when the application next goes online. However, this should only be done for messages sent recently. If these messages were sent too long ago, the conversation might have already progressed past the topic (possibly using other devices) and it no longer makes sense to retry sending them.
A combination of the above scenarios. In practice these cases compound: a commuter on a spotty train with a work laptop open on a second tab is the common case, not the edge case.

The chosen architecture must handle all of the above.

Client-side database
One way to store data on the client is to use a client-side database (henceforth referred to as the database). The UI reads data from the database as if it's a client-only application, as opposed to traditional applications where the UI directly makes HTTP queries and displays the fetched data. The UI is not and should not be aware of where the database got its data from. Where the database gets its data from should be an implementation detail of the data layer.

Different tabs in the same browser access the same client-side database. This ensures consistent data between tabs and helps to solve the UI consistency issue in the "using the application on different tabs in the same browser" scenario. Take care not to insert into the database twice when a "new message" event is received in multiple tabs.

Data syncer
The Data Syncer is the seam between the local database and the network. It owns the WebSocket connection, the real-time event stream, and the Message Scheduler that flushes outgoing writes. The UI never talks to the server directly; it reads and writes the local database, and the Syncer handles everything from optimistic writes through reconnect replay.

Keeping this responsibility in a single module means the reconnect, retry, and dedupe logic lives in one place rather than scattered across UI components.

Sending messages
When the user sends out a message (or any update made by the user in general), the UI should reflect the change immediately. Waiting for the server's confirmation before showing the sent message is a poor user experience.

Hence, outgoing chat messages and user actions are first inserted into the database, marked as pending, and reflected immediately in the UI. Chat applications typically surface the various message delivery statuses through small indicators.

Write to the local database first, not the network
Waiting for a server acknowledgment before rendering the sent message makes the app feel laggy on anything but perfect connectivity, and it breaks entirely offline. The database is the source of truth for the UI; the Message Scheduler reconciles pending rows with the server asynchronously and updates their status as acknowledgments arrive.

| Message status | Description | Messenger | WhatsApp |
| --- | --- | --- | --- |
| Sending | Application is attempting to send the message | Empty circle | Clock icon |
| --- | --- | --- | --- |
| Sent | Message was sent to the server successfully | Checkmark in outline circle | Single gray checkmark |
| --- | --- | --- | --- |
| Delivered | Message delivered to the recipient | Checkmark in filled circle | Double gray checkmark |
| --- | --- | --- | --- |
| Read | Recipient has read the message | Tiny version of user profile picture | Double blue checkmark (or ticks) |
| --- | --- | --- | --- |
| Failed | Message failed to send | Exclamation icon in circle | Exclamation icon in circle |
Source: Messenger Help Center and WhatsApp Help Center

Each outgoing message moves through a small, deterministic lifecycle driven by scheduler events and server acknowledgments.

user hits Send

message_sent ack

retries exhausted

message_delivered

message_read

user retries


Message status lifecycle from composer to read receipt
During database syncing, after the server has received and acknowledged the action, it sends back a response to the application, and those messages can be marked as "Sent".

As there can be multiple messages being sent in parallel across different conversations (and in real applications, even more actions like reactions and deleting messages), there is a need for a scheduler to ensure that actions are sent to the server in the right order, request statuses are tracked, request failures are retried, etc.

Receiving real-time updates
Real-time delivery means the application must be informed about new messages from the back end as soon as they arrive, with no manual refresh. On connect, the client opens a WebSocket, authenticates (typically with a short-lived token minted from its session cookie), and sends a subscribe frame carrying the latest event cursor. The server replies with any missed events in sequence (see the sync event below), then streams new ones as they happen.

The connection is expected to drop often, including tab suspension, network switches, and timeouts, so the Data Syncer transparently reconnects with exponential backoff and resumes from the last applied cursor. The specific transport options (short polling, long polling, SSE, WebSockets) are compared in the "Real-time updates" subsection under Optimizations.

#### Server-side rendering or client-side rendering?

Chat applications have the following characteristics:

Highly interactive in nature due to high frequency of sending and receiving messages. The page will probably need a fair amount of JavaScript.
Messages can only be accessed when logged in.
Messages do not have to (and should not!) be indexed by search engines.
Fast initial loading speed is desired but not the most crucial.
A pure client-side rendering, single-page application architecture works well here. Server-side rendering (SSR) with client-side hydration, as used in News Feed system design and Photo Sharing application system design, is an option for a faster initial load, but its benefits are limited to performance gains because chat surfaces don't need to be SEO-friendly. The additional engineering complexity of enabling SSR is rarely worth it for a signed-in, non-SEO surface.

Default to CSR for signed-in chat surfaces
Messages are private, unindexable, and only meaningful behind auth, so SSR's usual payoffs (SEO, shareable public pages) don't apply. The interactive, long-lived session also favors a single-page shell that boots once and then streams updates over a real-time channel.

Component responsibilities
The diagram below maps the Chat UI, Store/Controller, Data Syncer, and server into a single data flow. The UI reads from the client-side database and never directly from the network; the Data Syncer is the only component that talks to the server.

Chat Application Architecture

Each box in the diagram maps to one of the following responsibilities.

Chat UI: Contains a list of conversations and the currently selected conversation
Conversations List: Presents a list of conversations (user, last message, last message timestamp).
Selected Conversation: A list of messages in the conversation and an input box to type new messages.
Store/Controller: Controls the flow of data within the application. Fetches data from the database to show in the UI. Writes data to the database.
Data Syncer: Module that contains the database and also manages the outgoing messages. Also receives updates from the server and updates the database accordingly.
Client-side Database: Database to store all the data needed to be shown in the UI.
Message Scheduler: Monitors the outgoing messages, schedules them for sending and manages their statuses.
Server: Provides HTTP APIs and real-time APIs to fetch conversation and message data and to send new messages.
## Data model

For simplicity, this model focuses on chat functionality only.

Client-side database
Most of the data needed by the application will be stored in the client-side database. Any data that is needed for offline functionality should go into the database. Here's an entity-relationship diagram of the database tables.

Chat Application Data Model
| Table/Entity | Sync to Server | Used by | Description |
| --- | --- | --- | --- |
| Conversation | Yes | Conversations List | Conversation between users (only two users for now) |
| --- | --- | --- | --- |
| Message | Yes | Conversation | Text message sent by user. status is one of sending, sent, delivered, read, failed |
| --- | --- | --- | --- |
| User | Yes | All | User identity |
| --- | --- | --- | --- |
| ConversationUser | Yes | - | Associates users and conversations to allow a many-to-many relationship. A Conversation only has a maximum of two Users for now, but with this design it can support as many as required |
| --- | --- | --- | --- |
| DraftMessage | No | Conversation | Store half-written, unsent messages |
| --- | --- | --- | --- |
| SendMessageRequest | No | Message Scheduler | Tracks the status of messages to be sent |
The Message entity carries a handful of fields beyond the status shown above.

| Field | Description |
| --- | --- |
| id | Server-assigned durable id, used for ordering, sync, and deduplication. Absent on a pending row until message_sent arrives. |
| --- | --- |
| client_id | Client-generated id assigned at compose time, used to reconcile the optimistic row with the server's id on ack. |
| --- | --- |
| conversation_id | Foreign key into Conversation, indexed for paginated reads. |
| --- | --- |
| author_id | Foreign key into User. |
| --- | --- |
| content | Message body. |
| --- | --- |
| created_at | Wall-clock timestamp from the sender, shown in the UI but never used for ordering. |
| --- | --- |
| server_seq | Server-assigned monotonic sequence used to order the conversation. Absent on pending rows. |
| --- | --- |
| status | One of sending, sent, delivered, read, failed. |
The client_id / id pair is how optimistic writes reconcile: the composer inserts a pending row keyed by client_id, the message_sent event populates id and server_seq while leaving client_id intact. This preserves the identity of the row so React keys stay stable and the bubble doesn't flicker through a remount, and it keeps client_id available as an idempotency key for replayed events.

The Conversation row carries at minimum an id, a type (to distinguish direct chats from future group threads), last_message_at and last_message_id for the Conversations List sort, and created_at. SendMessageRequest carries an id (auto-increment for ordered scheduler reads), a client_id pointing to the pending Message, a conversation_id, the status enum, a fail_count, and a last_sent_at timestamp used by the scheduler's timeout check.

Note that the DraftMessage and SendMessageRequest tables are not synced to the server and are client-side only. However, they should still be in the database rather than client-only state, as they should be persisted across sessions.

DraftMessage: This table stores messages the user has typed in a conversation's message input box but hasn't sent. This has to be persisted in the database (as opposed to client-only state) so that if the user quits the application and opens it again, they don't lose their unsent message. Each (conversation_id, author_id) pair can have at most one draft, enforced by a unique index rather than by convention. Its fields are:

| Field | Description |
| --- | --- |
| conversation_id | Foreign key into Conversation, part of the unique key. |
| --- | --- |
| author_id | Foreign key into User, part of the unique key. Required because a single browser profile can be used by multiple accounts over time, and each account's draft must stay isolated. |
| --- | --- |
| content | Draft body. |
| --- | --- |
| updated_at | Client timestamp of the most recent keystroke flush, used to evict stale drafts on reset. |
Draft messages are not synced with the server, so they stay within the current device. Syncing drafts across devices is feasible, but it's a product decision that is out of scope for this core design.

SendMessageRequest: This table stores data related to each message that the user has sent but which hasn't been acknowledged by the server. status is an enum and can be one of the following:

pending: The default state of new messages to be sent.
in_flight: The application has sent the message to the server but hasn't received a response.
fail: When the server returns an error or the send request has timed out. The number of failures is tracked in fail_count so the scheduler knows whether to keep retrying (with exponential backoff) or stop retrying after a certain number of failures.
success: Indicates that the message has been received and acknowledged by the server. Strictly speaking, this enum value is not needed because when the client receives the server acknowledgment, this row can be removed from the table.
Database indexes
IndexedDB stores are fast only with the right indexes, and the hot queries in this design are predictable enough to enumerate upfront.

Message by (conversation_id, server_seq): The primary read path for rendering a conversation and for paginating history. Keying by server_seq rather than created_at avoids the clock-skew issue called out later in Optimizations.
Message by client_id: Used by the message_sent and message_failed events to locate the pending row and finalize it.
Conversation by last_message_at: Drives the Conversations List sort.
SendMessageRequest by status: The Message Scheduler polls this index, so a scan over the full table is avoided.
Client-only state
These are state fields that do not need to be persisted across sessions, i.e., it's ok to lose this data if the user exits the application by closing the browser tab/window.

Selected conversation: The currently selected conversation.
Conversation scroll position: Scroll position within each conversation. It's useful to restore the scroll position whenever the user switches between conversations.
Conversation outgoing message: This is whatever the user is typing in a specific conversation. It is almost the same as the DraftMessage, except that writes to the database should not happen on every keystroke. Persist to the DraftMessage table after the user has stopped typing (via blur/debounce) or throttle the write to every X ms.
## Interface definition (API)

The following APIs are needed:

Send message
Sync outgoing messages
Server events
Fetch conversations
Fetch conversation messages
Send message
The user-facing handoff is fully synchronous against the local database; network work is deferred to the Message Scheduler and reconciled later on the real-time channel.

Add a row to the Message table with a sending status.
Add a row to the SendMessageRequest table with a pending status.
Conversation UI reads from the Message table and shows the new message with a "Sending" indicator.
Any DraftMessage rows for the current conversation are deleted.
At this point, there are no synchronous steps left to be done. The Message Scheduler will be in charge of syncing the pending messages with the server.
The sequence below shows the full path from tap-to-send through the message_sent acknowledgment, with synchronous writes to the client database on the left and asynchronous scheduler work on the right.

Server (WebSocket)
Message Scheduler
Client DB (IndexedDB)
Conversation view
Server (WebSocket)
Message Scheduler
Client DB (IndexedDB)
Conversation view
Tap Send
Insert Message (status=sending)
Insert SendMessageRequest (pending)
Delete DraftMessage for conversation
Render message with "Sending" indicator
Pick up pending request
Send message over WebSocket
Update request to in_flight
message_sent ack
Update Message.status=sent, remove request row
Re-render with "Sent" indicator


Send message with optimistic UI and delivery acknowledgment
Sync outgoing messages
The Message Scheduler will be in charge of syncing outgoing messages with the server. It will maintain its own task queue and watch the SendMessageRequest table. Because the task queue needs to be in sync across tabs, it should not be stored in browser memory and can also use another table.

Whenever the table is not empty, it will fetch the first X rows (ordered by id) and try to process them by adding tasks to its own task queue. X is a configurable value. Depending on the row's status column:

pending: Enqueue a task to send the message to the server via the real-time channel. Update the row's status to in_flight.
in_flight: Check the last_sent_at timestamp. If it has exceeded the timeout threshold, update the row's status to fail and increment the fail_count by 1.
fail: Enqueue a task to retry sending this message sometime in the future. The delay duration depends on the fail_count. With an exponential backoff retry strategy, the delay duration will increase exponentially with the fail_count.
The "task queue" here is a logical queue, not a thread pool. Its consumer is a single loop inside the leader tab's Data Syncer (elected via the Web Locks API, covered under "Syncing across tabs"): it polls SendMessageRequest by status, writes send_message frames to the WebSocket, and updates rows as acks arrive. Web Workers are not needed because the work is I/O-bound, not CPU-bound, and a single JavaScript task already serializes writes to the socket correctly. Concurrency is logical rather than thread-based: a configurable in-flight cap (for example, four parallel sends) bounds how many in_flight rows can exist at once so a slow server does not stall the whole queue, but every acknowledgment still runs through the same event loop.

Each SendMessageRequest row is a small state machine, which makes the valid transitions easy to reason about when failures, retries, and acknowledgments interleave.

enqueue outgoing message

send over WebSocket

message_sent ack

timeout or message_failed

backoff elapsed, retry

fail_count exceeds cap

row removed


in_flight


SendMessageRequest lifecycle inside the message scheduler
Fetch conversations
The Conversations List is driven by a paginated read of the user's threads. On first load the client requests the top N conversations ordered by the latest message's timestamp and stores them in the Conversation table. Subsequent loads reuse the cached rows and only fetch newer conversations through the sync event described below.

Request: GET /conversations?limit=N&cursor=<opaque>.
Response: An array of conversations plus a next_cursor for pagination. Each item contains the conversation id, the participants, the most recent message (for preview rendering), and an unread count.
The client upserts each returned conversation and its participants into the Conversation, ConversationUser, and User tables.

Fetch conversation messages
When the user opens a conversation, the client reads whatever messages are already in the local database immediately, then fetches any missing older messages cursor-paginated from the server.

Request: GET /conversations/:id/messages?limit=N&before=<message_cursor>.
Response: An array of messages ordered newest first, plus a next_cursor for loading older history when the user scrolls up.
The client upserts returned rows into the Message table. Incoming messages newer than the local cursor still arrive through the incoming_message real-time event, so this endpoint is only for history loads.

Create a new conversation
The article so far assumes the user is chatting in an existing thread. Creating a new one needs two small additions: a way to find the other user and a way to mint the conversation row on the server before any message flows.

User search: GET /users/search?q=<query> returns a page of users the current user is allowed to message. Results render in a transient picker surface and are not persisted to User unless the user actually starts a conversation.
Create conversation: POST /conversations with the participant ids. The server returns the full Conversation row, including its assigned id and initial last_message_at. The client upserts into Conversation, ConversationUser, and any missing participants into User, then navigates into the new thread.
Conversation creation is server-first, not optimistic. A Conversation without a server-assigned id cannot route messages, cannot participate in the real-time channel, and cannot be addressed by other devices in a sync event, so deferring the UI transition until POST /conversations resolves is the correct tradeoff. If the user types into the composer before the response arrives, buffer the text locally and enqueue it into SendMessageRequest once the conversation id is known, then let the regular send path take over.

Adding or removing participants mid-conversation is out of scope for this design, since that requires group conversation semantics that the initial scope explicitly excludes.

Server events
The Data Syncer will receive real-time updates from the server in the form of events. Every event shares the same envelope: a type discriminator, a monotonic server_seq the client uses to order and dedupe events (and advance the sync cursor), and a payload whose shape depends on the type.


// Example envelope pushed from the server.
{
"type": "incoming_message",
"server_seq": 48213,
"payload": {
"message_id": "m_01HW...",
"conversation_id": "c_01HW...",
"sender_id": "u_01HW...",
"content": "See you at 7!",
"created_at": "2026-04-22T19:10:42.812Z"
}
}
The envelope's server_seq is the single source of ordering truth. For events that create or finalize a Message row (incoming_message, message_sent), the client copies the envelope's server_seq into the row's Message.server_seq column, and that becomes the message's canonical position in its conversation. Status-change events (message_delivered, message_read, message_failed) do not carry a per-message sequence because they target an already-positioned row by message_id or client_id.

The events the client must handle are listed below. Each subsection describes when the event fires, the shape of its payload, and the side effects the Data Syncer applies.

message_sent event
Fired by the server when it has accepted and persisted an outgoing message. The client marks the message as sent and clears the corresponding scheduler bookkeeping.


{
"client_id": "tmp_01HW...",
"message_id": "m_01HW...",
"conversation_id": "c_01HW...",
"created_at": "2026-04-22T19:10:42.812Z"
}
The client_id is the optimistic id the client generated when composing; the message_id is the durable id assigned by the server. The client uses client_id to locate the pending row and message_id to finalize it. The envelope's server_seq is copied into Message.server_seq at the same time.

Locate the pending row by client_id, populate its id (from message_id) and server_seq (from the envelope), and set status to sent. Keep client_id on the row so replayed acks are idempotent.
Clean up this message in the Message Scheduler:
Remove the row corresponding to this message in the SendMessageRequest table. This message has been received by the server and is no longer pending or in_flight.
Remove any tasks in the task queue that are related to this message.
Update UI
Notify the Conversation UI to be updated if that message's conversation is currently shown.
message_delivered event
Fired when the server has confirmed the message reached at least one of the recipient's devices (the product may choose to wait for all, or just the most recently active), which may be well after the message_sent acknowledgment.


{
"message_id": "m_01HW...",
"conversation_id": "c_01HW...",
"delivered_at": "2026-04-22T19:10:43.001Z"
}
Update the Message's status to delivered.
Update UI
Notify the Conversation UI to be updated if that message's conversation is currently shown.
message_read event
Fired when the server has confirmed the recipient has actually seen the message. The trigger lives on the recipient's client: when a message bubble enters the viewport of a focused conversation tab, the recipient emits a mark_read frame to the server, the server persists the read receipt, and fans out a message_read event to the sender.


{
"message_id": "m_01HW...",
"conversation_id": "c_01HW...",
"reader_id": "u_01HW...",
"read_at": "2026-04-22T19:11:02.410Z"
}
reader_id is redundant for strict 1:1 chat but is cheap to include and makes the event extension-ready for group conversations.

Update the Message's status to read.
Update UI
Notify the Conversation UI to be updated if that message's conversation is currently shown.
message_failed event
Fired when the server rejects an outgoing message. The scheduler increments the retry counter so the Message Scheduler can decide whether to back off and retry or give up.


{
"client_id": "tmp_01HW...",
"conversation_id": "c_01HW...",
"reason": "rate_limited",
"retryable": true
}
reason is an enum the client uses to decide between retrying, prompting the user, or giving up immediately (for example, validation_failed is terminal, rate_limited is retryable).

Update the row corresponding to this message in the SendMessageRequest table, change its status to fail, and increment fail_count by 1.
Do not modify the status of the row in the Message table to fail yet. The message is not deemed failed until the scheduler has exhausted its retries.
Update UI
Notify the Conversation UI to be updated if that message's conversation is currently shown.
incoming_message event
Fired whenever another user sends a message to the current user. The client appends the message locally and refreshes the affected surfaces.


{
"message_id": "m_01HW...",
"conversation_id": "c_01HW...",
"sender_id": "u_01HW...",
"content": "See you at 7!",
"created_at": "2026-04-22T19:10:42.812Z"
}
Append the new message to the Message table with server_seq copied from the envelope. If an entry with the same message_id already exists (for example, replayed on reconnect), ignore the event.
Create a new row in the Conversation table if it doesn't exist.
Create a new row for the message's sender in the User table if it doesn't already exist.
Update UI
Notify the Conversations List UI to be updated. Update the UI to surface this conversation to the top. If the Conversations List is sorted by decreasing timestamp of each conversation's latest message, it will automatically be surfaced to the top.
Notify the Conversation UI to be updated if that message's conversation is currently shown.
Because the UI reads from the database rather than the network, a new message becomes a database write first and a render second, which keeps behavior identical whether the event arrives live or is replayed after reconnect.

Chat UI (other tabs)
BroadcastChannel
Chat UI (leader tab)
Client DB (IndexedDB)
Data Syncer (leader tab)
Server (WebSocket)
Chat UI (other tabs)
BroadcastChannel
Chat UI (leader tab)
Client DB (IndexedDB)
Data Syncer (leader tab)
Server (WebSocket)
incoming_message event
Upsert User if new sender
Upsert Conversation if new thread
Insert Message row
Conversations List surfaces thread to top
Conversation view appends message if open
Broadcast db-change
Other tabs re-read from DB


Incoming message delivery over the real-time channel
sync event
Fired when a client connects or reconnects to the server, typically after an offline period or a fresh tab load. Its job is to catch the client up with everything that happened while it was away, so the local database is a complete projection of the server state for the user's conversations.

The client has to signal where it left off. Two approaches are common.

Client's last update timestamp: The client sends the timestamp of the last server-originated entity it applied. The server returns all conversations and messages created after that timestamp.
Each conversation's cursor: The client sends a per-conversation cursor (for example, the id of the latest message it has locally). The server returns the messages that come after each cursor, plus any conversations the client doesn't yet know about.
The server's response is a batched payload containing new or updated User, Conversation, and Message rows. The client applies the inserts in a single transaction, advances the cursor, and broadcasts a db-change message over the BroadcastChannel so other tabs re-read.

Prefer cursor-based sync over last-update timestamps
Client clocks can drift, and "messages newer than T" is ambiguous when two messages share a timestamp. A per-conversation cursor (such as the last applied message id or a server-assigned sequence number) gives the server an unambiguous starting point and makes the sync robust to clock skew and out-of-order delivery. This matches the cursor-based model used by Airbnb's messaging sync, linked in the References.

## Optimizations and deep dive

With the core architecture in place, this section dives into the production concerns that make chat reliable across long-running sessions, flaky networks, and fast-changing conversations:

Client database: This section explains how local message persistence supports offline browsing, reload recovery, and synchronization with the server.
Real-time updates: This section covers push-based delivery, transport choices, and how incoming messages reach the active conversation without manual refresh.
Typing indicator: This section discusses ephemeral typing events and how they differ from durable message events in storage and delivery.
Network: This section explains how the client handles failed sends, retries, deduplication, reconnects, and changing connectivity.
Performance: This section covers initial boot, message list rendering, steady-state updates, and storage costs as conversations grow.
Accessibility: This section discusses screen-reader announcements, keyboard navigation, focus behavior, and readable status updates in an active chat.
Offline support: This section explains how the local database, service worker, and browser APIs work together to keep chat usable offline.
User experience: This section covers scroll position, timestamps, message grouping, composer behavior, notifications, and other high-impact chat details.
Stale clients: This section explains how old devices recover when their sync cursors are too far behind to replay efficiently.
Further topics: This section lists additional chat concerns that are useful follow-up discussion points when time permits.
Client database
A chat client must persist messages locally so conversations stay browsable offline and survive reloads. The choice of storage mechanism and schema shapes how the app syncs with the server.

Deciding on client-side storage
There are a few ways of storing data on the client: Cookies, Web Storage, and IndexedDB. Refer to comparison between cookies and Web Storage mechanisms quiz question for a more detailed-comparison.

Cookies are out of the question due to their extremely small capacity (roughly 4 KB per cookie) and the fact that they are attached to every HTTP request to the origin. localStorage (one of the Web Storage mechanisms) caps out at roughly 5–10 MB per origin, blocks the main thread on every read and write because the API is synchronous, and offers no indexes or transactions, all of which make it a poor fit for the large, structured, queryable store this design needs.

IndexedDB is the best choice here. It is a low-level API for client-side storage of significant amounts of structured data, including files/blobs. Other useful features include database indexes, tables, cursors, and transactions, mostly over asynchronous APIs.

Default to IndexedDB for the chat client database
Cookies are capped at roughly 4 KB per cookie and localStorage is a synchronous ~5–10 MB key-value store with no indexes, so neither scales to thousands of messages with structured fields, indexes, and blob attachments. IndexedDB is the only browser-native option that supports the schema, indexes, and transactions this design relies on.

Syncing across tabs
Since IndexedDB is a client-side storage mechanism, the data is accessible across tabs, and it solves the "users should see the same messages in the application on different tabs in the same browser" scenario outlined at the top.

However, browser tabs are not aware of IndexedDB data changes in other tabs. To inform the other tabs about changes in the database, use the BroadcastChannel, which allows communication between different windows/tabs/frames of the same origin.

Shared storage alone also does not prevent the tabs from fighting each other. Every open tab will happily open its own WebSocket, receive the same incoming_message event, and try to insert the same row. Two coordination layers solve this cleanly:

Leader election via the Web Locks API: all tabs attempt to acquire a single named lock (for example, chat:data-syncer). Exactly one tab wins and becomes the leader; it owns the WebSocket, applies server events to IndexedDB, and is the only writer. The losers hold no WebSocket, read from IndexedDB, and wait for change notifications.
Change broadcast via BroadcastChannel: when the leader finishes applying a batch of writes, it broadcasts a db-change message so follower tabs know to re-read the affected stores.
If the leader tab closes, the Web Locks API releases its lock and one of the surviving tabs wins the next election. The transition is transparent to the user: there is only ever one writer to the local database and one live WebSocket per browser.

IndexedDB writes do not notify other tabs on their own
Shared storage does not imply shared change notifications. Without an explicit cross-tab signal such as BroadcastChannel, a second tab will only see new messages on its next read, causing stale conversations to sit unchanged until the user interacts with that tab.

Syncing client database with the server
Bidirectional syncing is the most subtle part of the architecture. A few concerns recur.

Out-of-order messages. Independent in-flight sends can reach the server in any order, and a reconnect replay can deliver older history after newer incoming messages already arrived. Order the conversation by the server-assigned server_seq (or an equivalent monotonic message id), never by client created_at. On insert, find the position by server_seq and splice into place rather than append; a message that arrives with an earlier server_seq should land where it belongs, not at the bottom. The UI still displays created_at to the reader, but the ordering key is always the server sequence.

Fetching new messages. The sync event described in the API section is the primary path: the client advances a per-conversation cursor on every applied event and replays from that cursor on reconnect. GET /conversations/:id/messages is reserved for history scrolls, not for "catch up since I was offline".

Sending pending messages. Outgoing messages composed offline land in SendMessageRequest and flush when the WebSocket reconnects. The scheduler should drop requests older than a threshold (for example, 24 hours) rather than send them blind; see the "Tricky scenarios" note earlier about messages that have aged past relevance.

Deletions and edits. If the product supports either, they must flow through the same durable event channel as sends, carry the same server_seq, and appear in the sync replay. Deleting a message on one device and not seeing the deletion on another is a common bug; explicit tombstones on the server side prevent it.

Other issues
A few practical gotchas are worth naming even when they don't change the architecture. The chat client needs graceful degradation paths for each.

Unsupported environments like private/incognito mode on Firefox and Safari, where IndexedDB may be disabled or memory-only. Detect this at boot and fall back to in-memory state with a clear "messages won't persist" banner.
Storage limits. Browsers apply origin-level quotas; long-running users can hit them. Evict oldest conversations or require re-sync when storage pressure is detected.
Errors initializing or opening the database, for example after a schema upgrade. Surface a recoverable error state instead of a white screen and retry with a clean open.
IndexedDB comes with a number of problems, bugs, and oddities; see also Best Practices for Using IndexedDB.
Real-time updates
Real-time messaging means that recipients of the messages receive new messages instantly (or near instantly) without having to reboot the application/refresh the page or manually trigger a button to fetch new messages.

A few transports can deliver real-time updates. Each trades off latency, efficiency, and implementation complexity differently.

Short polling (regular polling): The client re-requests the server on a fixed interval. Simple to implement on top of any HTTP server, but wasteful on both sides — most polls return nothing, and unfetched messages can sit on the server for the full poll interval before they're seen. Latency and load scale with the inverse of the poll interval.
Long polling: The client opens a request that the server holds open until it has data to send, then immediately reopens another. Latency is much better than short polling, but each delivery still pays the cost of tearing down and re-establishing an HTTP request, and the server has to hold one connection per client.
Server-sent events (SSE): A persistent one-way stream from server to client over HTTP. Good for incoming messages but cannot carry outgoing messages, typing indicators, or presence updates without a second channel. On the upside, EventSource has built-in auto-reconnect and Last-Event-ID resume, which a WebSocket client has to implement by hand.
WebSockets: A single persistent, full-duplex TCP connection established via an HTTP Upgrade handshake. One connection carries inbound messages, outbound sends, delivery and read receipts, typing indicators, and presence without per-event HTTP overhead.
Default to WebSockets for the chat real-time transport
Short polling adds avoidable latency and load, and long polling ties up a connection per request without supporting efficient server push. WebSockets give you a single persistent, bidirectional channel that is ideal for pushing incoming messages, delivery and read receipts, typing indicators, and presence updates without per-event HTTP overhead.

Reference: WebSockets vs Long Polling

Typing indicator
Typing indicators are the cleanest example of an ephemeral event riding on the WebSocket, and they teach a useful contrast with the message flow: durable events are written to IndexedDB and reconciled through the sync event, while ephemeral events bypass the database entirely and expire on a timer.

The delivery contract for these events is deliberately relaxed. Typing indicators are a low-value UX hint, not a correctness-critical signal, so the design can skip the guarantees that the message flow relies on: no acknowledgments, no retries, no ordering guarantees, no at-least-once delivery, and no cross-device consistency. If a typing_start is dropped the indicator simply never appears; if typing_start and typing_stop arrive out of order the receiver's expiry timer cleans it up within seconds. The worst outcome is a brief flicker or a stale indicator that resolves itself, which is a fine tradeoff for avoiding the complexity of acks, sequence numbers, or replay.

Both events share a minimal payload, just enough for the receiver to route and display the indicator. No server_seq, since ephemeral events do not participate in the event cursor.


// type: "typing_start" or "typing_stop"
{
"conversation_id": "c_01HW...",
"sender_id": "u_01HW..."
}
On the sender side, the composer fires a typing_start event the first time the user types in a conversation, then throttles so subsequent keystrokes send at most one event every few seconds. A short idle timeout (for example, three seconds with no new keystroke) fires typing_stop. Sending on blur or on message send also fires typing_stop to avoid stuck indicators.

On the receiver side, the Data Syncer routes typing events to an in-memory map keyed by conversation id, never to IndexedDB. Each entry carries an expiry that the receiver advances on new typing_start events and clears on typing_stop. A short timer on the receiver auto-expires stale indicators, which is essential when the sender goes offline mid-typing and typing_stop never arrives.

Ephemeral events are not persisted, synced, or retried
Typing indicators must not hit IndexedDB, must not appear in the sync event replay after reconnect, and must not be retried by the Message Scheduler. Conflating ephemeral events with durable ones leaks stale state; a typing indicator replayed on reconnect reads as "this person is typing a message they sent five minutes ago".

### Network

Connection failure is extremely common, as users could be using chat applications while on transport and going in and out of poor connectivity areas. Messages could fail to send out, along with other issues:

Offline usage: The application should detect the offline state and pause flushes rather than letting sends fail blind. Queue the user's composed messages into SendMessageRequest and resume when the WebSocket reconnects.
Failures: Retry failed outgoing messages with exponential backoff, capped at a max attempt count. After the cap, mark the Message as failed and surface a retry affordance in the UI so the user can decide whether to try again.
Out-of-order: A single WebSocket is a TCP stream, so frames the client writes arrive at the server in order. The real ordering concern lives on the server: two concurrent send_message frames still race each other through the server's handlers before persistence. The server assigns a monotonic server_seq on accept, and the client orders the conversation by server_seq on reconciliation — never by client created_at or arrival order.
Disconnection: Automatically attempt reconnection on drop with exponential backoff, resuming from the last applied event cursor so no user action is required.
Modeling the WebSocket connection as a small state machine makes the reconnect, backoff, and offline-detection behaviors explicit, so the Message Scheduler and UI both know when it is safe to flush pending requests versus queue them.

handshake ok

handshake failed

socket dropped or heartbeat missed

backoff elapsed

retries capped or device offline

online hint + heartbeat ok

tab suspended

app closed


reconnecting


Real-time connection lifecycle with automatic reconnect
Flush priorities, not uniform batching
Different outgoing event types have different latency tolerances, so the Message Scheduler should not treat them uniformly. A blanket "batch everything in quick succession" policy is the wrong default for chat: messages are the user's primary intent, and adding even 100 ms of debounce to a send makes the app feel laggy without a meaningful payoff, because a single WebSocket already coalesces small writes at the TCP layer.

A useful policy is to classify outgoing frames by priority:

| Priority | Examples | Flush policy |
| --- | --- | --- |
| Flush now | send_message, message edit/delete, call signaling | Write to the socket as soon as the row is enqueued. Latency matters far more than a few bytes of framing overhead. |
| --- | --- | --- |
| Batch softly | delivery_ack, read_receipt | High-volume, low-urgency. Coalesce on a short timer (for example, 100–200 ms) or a size threshold, whichever fires first. |
| --- | --- | --- |
| Throttle / coalesce | typing_start/typing_stop, presence | Keep only the latest state per conversation and send on a cadence. Covered in the Typing Indicator section. |
The Message Scheduler owns priorities 1 and 2; the typing indicator's throttling (priority 3) lives in the composer because it is purely ephemeral and bypasses SendMessageRequest entirely.

Do not rely on client timestamps or arrival order for message ordering
Client clocks drift and arrival order at the server is not deterministic under concurrent sends, so neither is a safe ordering key. Order messages by the server-assigned monotonic server_seq on reconciliation. Batching outgoing messages does not fix this; it just hides it and actively hurts latency on the one event type users notice most.

### Performance

Chat performance has three fronts: initial boot, steady-state render cost as messages stream in, and the storage and network layers beneath the UI.

Initial load and code splitting
The main bundle should contain only the shell, the currently selected conversation, and the WebSocket client. Route-split so that opening a new conversation does not require loading anything the user already sees. Heavy but rarely used surfaces like the emoji picker, attachment composer, media viewer, and settings should be dynamic imports that load on first interaction. Tree-shake the shared components library aggressively; chat apps typically pull in more UI than they use, because the surface looks small but the composer carries a long tail of formatting tools.

Rendering long lists
Virtualize the message list so only the visible window plus a small overscan is mounted. Memoize the bubble component with a key that combines message_id and the fields that affect rendering (status, content, reaction count), so re-renders on sibling changes do not ripple through the entire list. When the incoming_message event fires, touch only the Conversations List row and the bubble that was appended, not the entire conversation view.

Conversations List rows are equally sensitive: a new incoming message causes one row to move to the top, but the list should not re-render every row. Keying by conversation_id and reordering in place is preferable to replacing the array.

Media and attachments
Inline images and avatars should ship responsive srcsets, use loading="lazy" below the fold, and render into a fixed-height placeholder so scroll position does not jump when the image resolves. Video and audio previews should lazy-load their players on click, not on scroll. If the app supports file attachments, the upload path should use a separate HTTP endpoint rather than the WebSocket, since binary frames over a shared connection stall durable message delivery behind the upload.

Storage performance
IndexedDB reads are cheap in isolation but expensive if the UI reads one row at a time. Batch reads through a single transaction against the (conversation_id, server_seq) index when loading a conversation; avoid N+1 reads for message authors by joining against the User table in memory once. Reserve dedicated indexes for the Message Scheduler's polling read on SendMessageRequest.status so the scheduler does not scan the full table on every tick.

Wire format
Default to JSON over WebSocket for the control plane because it is legible, debuggable, and gzip-compresses well. Reserve binary frames for payloads that would otherwise bloat the text channel: message attachments, batched history dumps in the sync event, or presence fan-out. At high scale, a compact format like MessagePack or protobuf on the hot path can cut bandwidth meaningfully, but it's rarely worth the interview-time complexity of proposing unless the prompt emphasizes scale.

### Accessibility

Chat is an unusually hostile surface for accessibility because content streams in asynchronously while the user is reading, typing, or navigating elsewhere. A few concerns matter disproportionately.

Keyboard
Keyboard parity with mouse interactions is the first win; wire these shortcuts end to end.

Message composer
Enter to send a message.
Shift + Enter to add new lines within a message.
Between conversations
Shortcut to focus on the search bar.
Ctrl/Cmd + Up/Down to traverse between conversations.
On certain desktop clients, Ctrl/Cmd + number brings you to the nth conversation in your list of conversations.
Screen readers and live regions
Incoming messages arrive without the user initiating the interaction, so they need to be announced without hijacking focus. Wrap the message list in an aria-live="polite" region so new messages are announced after the screen reader finishes its current utterance. Status changes on sent messages (sending → sent → delivered → read) should update the message's accessible name rather than emit a separate announcement, otherwise the reader narrates every state transition and becomes unusable.

Avoid aria-live="assertive" for messages because it interrupts the user mid-sentence and is disproportionately loud for a low-urgency event. Reserve assertive announcements for error states (send failed, connection lost) where interrupting is the right behavior.

Focus management
Switching conversations is a navigation event; focus should land on the composer input by default, or on the newest unread message if the conversation has any. Opening modals (emoji picker, attachment previewer) must trap focus and restore it to the triggering element on close. When the WebSocket drops and a "Reconnecting…" banner appears, focus should not move to it because the user may be mid-sentence, but the banner should still be reachable via the Tab order.

Motion and color
Honor prefers-reduced-motion for the send-off animation, the typing-indicator pulse, and any scroll-to-bottom tween. Message status icons must be distinguishable by more than color; a double checkmark, a clock, and an exclamation mark already differ in shape, so long as the design system does not strip those shapes in favor of pure color swatches.

Touch targets and RTL
On touch devices, tap targets on message status, reaction pickers, and conversation rows should meet the 44×44 pt minimum. For RTL locales, the entire conversation flow mirrors: sender bubbles align right, recipient bubbles align left, and the composer's send button flips to the leading edge. Use logical CSS properties (margin-inline-start, padding-inline-end) so the layout inverts cleanly without a second stylesheet.

Offline support
Offline is not one feature. It is a collection of capabilities that the client-side database, service worker, and the browser's background APIs combine to deliver. The local-first architecture already handles the data side; the service worker handles code and delivery.

Service worker caching strategy
Install the app as a Progressive Web Application (PWA) so the service worker can cache the app shell on first visit and serve it instantly on subsequent loads. Use different caching strategies per resource class:

App shell (HTML, core JS, CSS): cache-first with a versioned URL so a new deploy invalidates the cache without shipping a stale shell.
Static assets (icons, fonts, component chunks): stale-while-revalidate so users see a cached response immediately while the worker updates the cache in the background.
API responses and real-time endpoints: network-only. These must never be intercepted or replayed — stale /conversations responses would corrupt the local database, and intercepting the WebSocket handshake is nonsensical.
Background sync for outgoing messages
The Message Scheduler described earlier keeps pending messages in SendMessageRequest, but that scheduler only runs while a tab is open. The Background Sync API lets the service worker pick up where the tab left off: register a sync tag when an outgoing message is queued, and the worker gets an opportunity to flush the queue once connectivity returns, even after the user has closed the tab. Availability is uneven across browsers (Safari notably omits it), so treat it as an optimization, not a correctness requirement; the scheduler still retries on next app launch.

Web Push notifications
Browser notifications via the Notifications API are only half the story; they fire while the tab is open. The Web Push API is what makes notifications possible while the app is closed: the client subscribes for a push endpoint, sends the subscription to the server, and the server pushes encrypted messages through the browser's push service, which wakes the service worker to display the notification. VAPID keys authenticate the server, and the service worker handles the push event to localize the notification text and set the badge. Respect permission prompts by asking in context (for example, on the first incoming message while the tab is backgrounded), not on first load.

Detecting online and offline
navigator.onLine reports whether the device thinks it has a network interface, not whether the server is reachable; a captive portal, a DNS failure, or a dead WebSocket server all return true. Treat it as a hint, not a source of truth. The authoritative signal is the WebSocket itself: a working connection means the client is online for chat purposes, a failed or timed-out heartbeat means it is offline regardless of what navigator.onLine says. Pair the online/offline events with a ping frame every 30 seconds or so, and treat the user as offline once a ping goes unanswered past the heartbeat window.

### User experience

A handful of UX details matter disproportionately in a chat app: preserving scroll position as new messages arrive, clear timestamp formatting, message grouping to reduce visual noise, composer affordances that speed up typing, notification badging for off-tab awareness, and new-message indicators when the conversation is off-screen.

Maintaining scroll position
Scroll position is a tricky issue to deal with in messaging apps due to the possibility of new messages being added to the list both above and below.

Scroll position should be:

Kept to the bottom of the message list when new messages come in and the scroll position is already at the bottom of the list. This is the default scenario for most situations.
Maintained when the scroll position is not at the bottom, such that the currently visible contents remain visible. When scrolling up to read older chat messages, the scroll position should be maintained and currently visible elements should not move even though more DOM elements will be added to the top. The application can calculate the current scroll offset and the height of the new elements to be appended, then modify the scroll height to accommodate the new elements' height.
The following are events that can potentially change the (scroll/client) height:

New messages inserted below (when receiving new messages).
New messages inserted above (when searching for history).
Window resizing.
Media finishing loading with a different height from the loading placeholder.
Avoid this issue by using a fixed-height placeholder and rendering the media within that element.
Page zoom changes.
The scroll position should be maintained (either at the bottom or showing the same contents) depending on the scenarios listed above.

Appending newer messages and prepending older history have different semantics
Auto-scroll to the bottom is correct only when the user is already pinned to the latest message; otherwise, yanking the viewport mid-read is jarring. When prepending older history, compensate for the added DOM height so the currently visible messages do not shift under the user's eyes.

Scroll and unread affordances
A few small affordances make long conversations much easier to navigate.

Add a "Scroll to Bottom" button that's visible when the user has scrolled upwards within the conversation messages, with an unread badge when new messages have arrived out of view.
Render an "Unread messages" divider at the boundary between read and unread messages when the user opens a conversation, so the eye can jump directly to new content.
Offer a "Jump to first unread" shortcut that scrolls the divider into view without losing the user's existing scroll context.
Timestamps and day dividers
Raw timestamps on every message overwhelm the eye and fight the conversation's flow. Show timestamps sparingly, at the start of a new day, after an extended gap (for example, more than 15 minutes since the previous message), and on hover for the rest. Format relative for recent messages ("2 min ago", "10 min ago"), switch to time-only for messages earlier the same day ("09:42"), then to weekday for messages this week ("Mon"), and finally to full date for older messages.

All formatting should route through Intl.DateTimeFormat for absolute dates and times, and Intl.RelativeTimeFormat for relative strings, so locale, calendar, and 12h/24h preferences are honored without a custom fork per region. Intl.RelativeTimeFormat itself only formats a (value, unit) pair; pair it with a small helper that picks the right unit from the timestamp diff (seconds → minutes → hours → days). Day dividers between messages sent on different dates give the reader an anchor when scrolling through history.

Message grouping
Consecutive messages from the same sender within a short window (typically two or three minutes) should collapse visually: show the avatar and name only on the first message in the group, tighten the vertical spacing between bubbles, and apply a single rounded corner on the first and last bubbles to form a cluster. Grouping trims vertical noise dramatically in active conversations and makes it easier to follow who said what.

Grouping breaks on sender change, on a long pause, and on a new day divider. The grouping decision is purely presentational and derived at render time from already-ordered messages; do not persist a "group id" in the database.

Composer affordances
The composer is where most of the active interaction happens; small wins here compound. Autocomplete for emoji shortcodes (:smile) and mentions (@name) should trigger inline as the user types, with arrow-key navigation and Enter to confirm. Paste-to-attach handles the common case of dropping an image from the clipboard. A visible character or attachment-size cap prevents the user from composing a message the server will reject. Local draft autosave writes the composer value to the DraftMessage table on blur or on throttle so the user never loses typed text across a reload.

If the product supports rich formatting, the toolbar should be keyboard-reachable and survive a paste from external sources without importing the external styles. This is a subtle but common source of formatting rot in long-lived chat threads.

Notification badging
An off-focus user needs to know that something happened without switching tabs. Three complementary surfaces:

Document title: prefix the title with a parenthesized unread count (for example, (3) Messenger) that updates as messages arrive and clears when the tab regains focus via the visibilitychange event.
Favicon: swap in a badged favicon (a red dot is standard) when the unread count is non-zero. Draw on a <canvas> to layer the dot onto the base icon, then feed the data URL into the <link rel="icon">.
OS badge: on supported browsers, call navigator.setAppBadge(count) when the app is installed as a PWA so the OS-level icon shows the count.
All three should clear together when the user reads the affected messages, whether they did so in this tab, another tab, or another device.

Stale clients
A device that has been offline for weeks, such as a second phone or an old browser profile, is a hard case for the cursor-based sync. Replaying every event since the cursor can mean tens of thousands of rows, a slow startup, and a UI that feels broken long before it renders the first conversation.

Detecting staleness
The server's response to the reconnect handshake includes the freshest server_seq for the user's conversations. If the distance between the client's cursor and the server's is past a threshold (for example, 5,000 events or a month of wall-clock time), treat the client as stale and opt into a fresh-load flow instead of streaming the full delta through the sync event.

![Fresh-load flow](chat-app-pdf-assets/figures/fresh-load-flow.png)

Treat the existing database as invalid for the purposes of this session and bootstrap as if it were a new install: fetch the latest N conversations, the latest M messages per conversation, and discard everything below that waterline. The user sees recent history immediately and the rest loads lazily as they scroll or open older threads.

Two details are worth being explicit about:

Pinned and active conversations first. Order the prefetch by last-read or last-message-at so the conversation the user most likely cares about renders first. A hard cap on total bytes avoids a surprise multi-megabyte download on a cold mobile connection.
Preserve the outgoing queue. Wiping the local database must not drop rows in SendMessageRequest. Those are the user's in-flight messages, and losing them silently is far worse than a slow sync. Move the queue to a holding area, wipe the rest, then restore the queue to be flushed once the WebSocket is healthy.
Progress UX
The fresh-load flow is still slow enough that the user will notice. Show a skeleton of the Conversations List with a clear "Catching up…" label, render conversations as they arrive (do not wait for the full batch), and unblock the composer as soon as the currently visible conversation's messages are loaded so the user can start typing before their sidebar finishes populating.

Further topics
These topics won't be discussed in this solution, but if time permits, you might want to discuss them with your interviewer.

Searching
A realistic search feature splits work between the client and the server rather than choosing one. The client builds IndexedDB indexes on message text, scoped per conversation, so recent history is searchable instantly and offline. Older history that does not live on the device, which is most of it for a multi-device user, falls through to a server endpoint that searches the server-side store.

Merging results is where it gets interesting: dedupe by message id, prefer local hits for anything that exists locally, and paginate both sides with cursors so the "load more" flow can pull from whichever source still has results. End-to-end encryption breaks the server path, forcing pure client-side search and raising the staleness cost on low-storage clients.

End-to-end encryption (E2EE)
E2EE is not just a crypto concern; it reshapes the sync and search layers described earlier. Per-device keys and a key exchange run whenever a new device joins a conversation, and prior history is typically not backfilled to the new device because the server cannot re-encrypt content it never had in the clear.

The sync event can only ship ciphertext plus metadata (sender id, timestamps, cursor info). The server cannot preview message content, build search indexes over it, or generate push notification previews. Decryption runs in a worker so the UI thread stays responsive while streaming large history batches after reconnect. Messenger historically kept E2EE conversations device-scoped rather than multi-device for exactly these reasons; see Challenges of E2E Encryption in Facebook Messenger.

Reactions
Reactions behave like lightweight durable events that reuse the message pipeline rather than introduce a new one. A Reaction table with an m:n relationship to Message via message_id, user_id, and emoji keeps the data model clean and supports per-user toggling.

New reaction_added and reaction_removed server events flow through the Data Syncer and participate in the sync replay on reconnect, so a reaction added offline by another device eventually appears. Optimistic writes mirror the outgoing-message flow: insert locally on user action, reconcile on the server acknowledgment, revert on failure. On the UI side, expect per-emoji aggregate counts, a self-vs-other indicator, and care to animate only the first add rather than every render.

Disappearing messages
TTL must hold whether or not the client is online, which makes this more than a client-side timer. Message gains an expires_at field, and both client and server evaluate it so expiry is enforced when the client is offline and re-enforced on reconnect. A client-side sweep evicts expired rows from IndexedDB on a timer and on every app boot.

The sync event must then omit already-expired messages and may carry explicit delete tombstones for messages that expired while the client was away, so stale rows do not linger locally. Edge cases worth flagging in an interview: a message composed offline with a short TTL and delivered hours later, "disappears N minutes after being read" rules that depend on multi-device read state, and the web platform's inability to detect screenshots. Name this as a real limitation rather than pretend it is solvable.

## Summary

The design rests on a single commitment: the chat client is local-first, real-time, and offline-capable, and the UI reads from a client-side database rather than directly from the wire. Once that commitment is made, almost every other decision in the article stops being a judgment call and starts being a consequence.

Split components around the local-first boundary. If the UI must render from local state, it cannot also own networking, so the Chat UI reads from IndexedDB and never speaks to the server. The Store/Controller mediates reads and writes against that local database. The Data Syncer owns the WebSocket, the incoming event stream, and the Message Scheduler that flushes outgoing writes, which concentrates reconnect, retry, and dedupe logic in one place instead of leaking it into views.

Separate server-owned chat records from client-only durability records. Conversation, Message, User, and ConversationUser are server-owned and synced both ways, while DraftMessage and SendMessageRequest are client-only but still persisted so the composer survives reloads and the scheduler survives restarts. Because the UI needs a stable ordering key that does not depend on client clocks, every Message carries a client_id generated at compose time alongside the server-assigned id and server_seq, letting optimistic rows reconcile without remounting and giving the store an unambiguous sort field.

Make WebSocket recovery cursor-based and single-owner per browser. WebSockets are the default because the UI expects a live event stream, and reconnect uses exponential backoff so a flaky network does not become a reconnect storm. The sync event resumes from a per-conversation cursor rather than a last-update timestamp, which is the reliable way to catch up a client that has been away for minutes or weeks without replaying the world. A single tab owns the socket through the Web Locks API, and a BroadcastChannel fan-out keeps the other tabs reading the same local state instead of racing the leader.

Persist the edge cases that users would notice losing. IndexedDB provides the structured store the Chat UI renders from, and the stale-client fresh-load path deliberately preserves SendMessageRequest rows when the rest of the database is rebuilt, so unsent messages are never silently dropped by a schema reset.

Start with the local-first commitment and name server_seq as the ordering contract; the sync, persistence, and recovery choices all depend on that commitment.

## References

The references below group chat and messaging sources by product area, so it is easier to map each source back to the part of the article it informed.

Messenger architecture and scale
These references cover how Facebook built and scaled Messenger's client and real-time infrastructure, which is directly relevant to the sync, transport, and mobile-first concerns discussed above.

Building Facebook Messenger for the original product and architecture framing.
Reverse engineering the Facebook Messenger API for a concrete look at the client-server protocol surface.
Facebook Messenger Engineering with Mohsen Agsen for engineering-leadership context on Messenger rewrites and tradeoffs.
F8 2019: Lighter, Faster, Simpler Messenger for the rationale behind the client-side simplification effort.
Project LightSpeed: Rewriting the Messenger codebase for a deep dive into the rewrite that prioritized size, speed, and simplicity.
Building Mobile-First Infrastructure for Messenger for how server infrastructure was shaped by mobile client constraints.
Building Real Time Infrastructure at Facebook - SRECon2017 for how the underlying real-time transport layer scales across Facebook products.
Facebook Messenger RTC – The Challenges and Opportunities of Scale for scale-oriented insights that apply beyond pure RTC.
Challenges of E2E Encryption in Facebook Messenger for how end-to-end encryption interacts with multi-device sync and history.
MySQL for Message - @Scale 2014 for the message storage layer that backs threads at scale.
Launching Instagram Messaging on desktop for how a messaging surface is extended to a new web platform.
Slack front-end performance and offline
These references cover how Slack makes a large messaging web app boot quickly, stay responsive, and work offline, which maps directly to the rendering, caching, and service worker discussions in this article.

Gantry: Slack's fast-booting frontend framework for how Slack structures its front-end for fast startup.
Making Slack faster by being lazy for how lazy loading is applied to reduce initial work.
Making Slack faster by being lazy: part 2 for the follow-up iteration on the lazy-loading strategy.
Getting to Slack faster with incremental boot for how the client incrementally hydrates channels and features.
Service workers at Slack: faster boot times and offline support for how service workers underpin both boot speed and offline behavior.
### Accessibility and localization

These references cover how a mature messaging product handles focus management and multilingual UI, which are easy to underweight in an initial design pass.

Managing focus transitions in Slack for keyboard and screen reader focus handling in a complex chat UI.
Localizing Slack for how a messaging product is internationalized end to end.
Adjacent messaging case studies
These references cover messaging work outside the Messenger and Slack ecosystems, which is useful for comparing sync models across products.

Messaging sync — scaling mobile messaging at Airbnb for a cursor-based message sync protocol designed for flaky mobile networks.
Related system design articles
The articles below overlap with chat on real-time transport, synchronization, and consistency.

Collaborative Editor (e.g. Google Docs) for how real-time sync, conflict handling, and presence are modeled in a collaborative editing context.
Video Conferencing (e.g. Zoom) for how real-time transport, reconnect flows, and presence are modeled for live media.
The "undefined is not a function" Tee — A debugging classic