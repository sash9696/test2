---
title: "Music Streaming (e.g. Spotify)"
slug: "music-streaming"
difficulty: "hard"
duration_mins: 45
tags:
  - system-design
  - networking
premium: true
summary: "Design a music streaming application like Spotify with persistent playback, queueing, and offline support."
free_preview: false
status: draft
---

# Music Streaming (e.g. Spotify)

## Question

Design a music streaming application similar to Spotify, Apple Music, or YouTube Music that lets users browse and discover music, search for tracks and artists, open albums and playlists, and play audio with a persistent player and queue.

Focus on the front-end architecture and the client/server contract for browsing, playback, queue management, local persistence, and reconnect behavior. You do not need to go deep on recommendation ranking, music licensing, or the full audio ingestion and transcoding pipeline.

## Real-life examples

- https://open.spotify.com
- https://music.apple.com
- https://music.youtube.com
## Requirements

Users should be able to browse or discover music, search for tracks, artists, albums, and playlists, and open collection detail pages.
Users should be able to play music with a persistent player while navigating between different parts of the application.
Users should be able to manage a queue while moving between different playback contexts.
The design should consider offline and reconnect behavior where relevant.
You do not need to cover recommendation ranking, music licensing, or the full media ingestion and transcoding pipeline.
## Requirements exploration

Designing a music streaming application is a useful front-end system design exercise because playback must persist across navigation, queue state must remain consistent during long listening sessions, and offline or reconnect behavior changes how the client stores and syncs data.

#### What are the core features to be supported?

Browse and discover music on home or editorial surfaces.
Search for tracks, albums, artists, and playlists.
Open artist, album, and playlist detail pages and start playback from any track.
Keep a persistent player visible across navigation.
Support queue operations such as play now, add next, append, remove, reorder, shuffle, and repeat.
#### Is this a web-only player or should it also cover desktop?

While the problem sounds platform-neutral, many interviewers really mean "design the Spotify Web Player". The web player is the best baseline answer because it is the stricter environment, while the same shared frontend core can also power a desktop shell.

The web shell is the baseline target and owns browser playback, service workers, and browser storage constraints.
The desktop shell is the extension path and can offer stronger offline guarantees, filesystem-backed downloads, and OS media integrations.
If the interviewer wants a web-only answer, the desktop shell can be collapsed and the shared core plus the web-specific pieces can stand on their own.

#### What should offline support mean?

Offline support matters, but it should not dominate the baseline architecture.

On web, offline is best-effort because storage quotas, eviction, autoplay policies, and service worker lifecycle constraints vary by browser. The baseline promise should be cached metadata, artwork, local queue or session restore, and opportunistic cached playback when a previously fetched response is still available.
On desktop, explicit offline downloads can be treated as a richer first-class capability because the shell can use durable local storage and OS hooks.
Start by focusing on local restore and reconnect behavior in the baseline flow. Reliable downloadable offline listening for tracks, albums, and read-only playlists fits better as a desktop or offline-capable-shell extension. Writable personal playlists and offline playlist editing are useful, but they belong to a deeper extension rather than the baseline flow.

#### What are the non-functional requirements?

Fast time to first audio after the user presses play.
Smooth transitions between tracks with minimal rebuffering.
Queue state should stay consistent during long listening sessions and after refresh or restart.
The product should behave gracefully during connectivity loss, reconnects, and partial offline availability.
Playback controls should be accessible through keyboard, screen reader, and system media surfaces.
## Architecture / high-level design

For products that have both web and desktop clients, a common and practical app architecture is to use platform-specific app shells wrapping a shared core. That is the main design choice for this problem. The baseline answer targets a web player, where the player, queue, and playback controller stay mounted above route changes. A desktop client can then reuse the same core while swapping in stronger playback and storage adapters.

### Rendering approach

A hybrid rendering model fits well:

Use server-side rendering for browse and detail routes when fast first paint, link previews, or search discoverability matter.
Hydrate into a single-page application so navigation between browse surfaces does not recreate the player.
Keep the persistent player, queue, and playback controller inside the app shell rather than inside individual pages.
If the interviewer wants a simpler answer, this can be framed as "SSR for content-heavy entry pages, then SPA-style navigation after hydration". The important architectural point is that playback state does not live inside route-local UI.

Mount the player above the router, not inside a route
The persistent player, queue, and playback controller must live in the app shell so route changes never unmount the audio element. Tearing down and remounting audio on navigation causes audible gaps, lost buffers, and broken Media Session state, and any design that puts playback inside a page-level component is fighting the product's most basic expectation.

Shared core plus platform adapters
The cleanest architecture is a shared frontend core with thin platform-specific adapters around it:

The shared core owns domain models, queue rules, playback state transitions, checkpoint persistence, reconnect policy, and client-side interfaces such as startPlayback(), insertIntoQueue(), and restoreSession().
The web shell owns browser-specific playback and storage concerns such as HTMLAudioElement, service workers, IndexedDB, CacheStorage, MediaSession, and cross-tab coordination.
The backend and CDN layer provides browse/search metadata, playback bootstrap data, and media delivery, but it does not own the queue logic, which must remain responsive on the client.
For desktop, the same shared core can sit inside a desktop shell that swaps in native playback, filesystem-backed downloads, OS media controls, and stronger background behavior. That keeps the main interview answer web-first without giving up a clean extension path.

This keeps the hardest product logic in one place. Queue behavior, shuffle and repeat semantics, playback checkpoints, and reconnect handling should not diverge between web and desktop just because the playback and storage primitives differ.

State ownership should also be explicit:

Catalog state such as artists, albums, playlists, and search results is server-originated data cached on the client.
Playback session state such as the current track, queue snapshot, repeat mode, shuffle seed, and checkpoint position is client-owned state that must remain responsive even when the network is slow and should be restored locally first after refresh or restart.
Server continuity state such as the last synced track, position, repeat mode, shuffle seed, source context, and device metadata is a smaller shared subset used for signed-in cross-device continuity, not an authoritative copy of the mutable queue.
Local media state such as cached responses, downloaded files, and storage bookkeeping is adapter-owned local state exposed through stable interfaces to the shared core.
Component responsibilities
![Music streaming architecture diagram](music-streaming-pdf-assets/figures/music-streaming-architecture-diagram.png)


App shell: Owns top-level navigation, route transitions, and the persistent player surface that stays mounted across pages.
Catalog UI: Renders browse, search, artist, album, and playlist pages using server-originated metadata.
Playback controller: Runs the playback state machine, handles play, pause, seek, skip, and reacts to readiness, failure, connectivity, and media-key events from the platform adapter.
Queue and context service: Builds queue snapshots from search, artist, album, or playlist contexts, applies local queue mutations, and keeps shuffle and repeat behavior deterministic.
Persistence and sync service: Stores queue snapshots, playback checkpoints, cached media metadata, and recent session information locally; pushes lightweight continuity checkpoints to the server when online; and reconciles local state after refresh, restart, or reconnect.
Platform adapter: Hides environment-specific playback and storage APIs behind a stable interface used by the shared core.
API server: Provides HTTP APIs to fetch recommendations, search results, artists, albums, playlists, and audio metadata.
Backend

Local storage

Platform adapter

Shared core

App shell

Catalog UI

Persistent player UI

Playback controller

Queue and context service

Persistence and sync service

Web: HTMLAudioElement, MediaSession, Service Worker

Desktop: native audio, filesystem, OS media keys

IndexedDB

CacheStorage

Filesystem, desktop

API server

Media CDN


Shared core with platform adapters on web and desktop
The browser version of the persistence layer should usually split structured state from media caches:

IndexedDB for queue state, checkpoints, cache bookkeeping, continuity metadata, and entity metadata.
CacheStorage for artwork, manifests, and opportunistically reusable media responses.
The desktop version can keep the same logical interfaces while backing durable downloads with the filesystem. On web, the baseline remains cache-first rather than download-first. The storage responsibility breakdown is covered in the offline storage responsibilities section under interface definition.

![Playback flow](music-streaming-pdf-assets/figures/playback-flow.png)

The main playback path should stay event-driven from end to end:

A user starts playback from a track, album, artist, or playlist page.
The queue and context service creates a QueueSnapshot anchored to that source context.
The playback controller updates PlaybackSession immediately so the player reflects the requested track without waiting for the network round-trip.
The client requests playback bootstrap data for the active track, such as stream URLs, manifests, entitlement data, or capability flags.
The platform adapter loads and buffers the media, then emits readiness, progress, ended, and failure events back to the shared core.
The persistence and sync service writes a PlaybackCheckpoint in the background so refresh or restart can restore the session.
As the current track nears completion, the controller prefetches the next track's bootstrap data and warms the next playback handoff.
Media CDN
API server
Platform adapter
Queue service
Playback controller
Catalog view
Media CDN
API server
Platform adapter
Queue service
Playback controller
Catalog view
Click play on track
startPlayback(context, trackId)
Build QueueSnapshot
Set active session (optimistic)
Manifest URL + expiry + caps
load(manifest)
Fetch initial chunks
Media bytes
readyToPlay
play()
timeupdate events
Persist PlaybackCheckpoint (throttled)


Playback startup from user intent to first audio
This split keeps UI responsiveness local while still relying on server-authoritative metadata and media URLs.

Failure handling should go through the same state machine rather than through one-off UI code paths:

If playback bootstrap fails or a track is unavailable, mark the item as unavailable and either skip forward or show a recoverable error state.
If the user is offline but the next item is already cached, the platform adapter can continue playback from local media without changing the queue model.
If the user is offline and the next item is not cached, preserve the queue and checkpoint, but move the player into a blocked state that can recover automatically after reconnect.
On web, if another tab becomes the active playback owner, keep queue and UI state in sync while handing playback control to the active tab.
## Data model

The data model should follow the same ownership boundaries as the architecture. Catalog entities are server-originated and cached locally, playback session state is client-owned, and offline media bookkeeping stays in the local adapter layer.

| Entity | Source | Used by | Purpose / key fields |
| --- | --- | --- | --- |
| Track | Server | Catalog UI, queue, playback controller | Canonical playable unit. Key fields include id, title, durationMs, artwork references, artist or album references, and catalog-level streamability or availability flags. Stream URLs or manifests should come from playback bootstrap, not from the catalog entity itself. |
| --- | --- | --- | --- |
| Album | Server | Catalog UI, queue and context service | Read-only playback context with ordered trackIds, album metadata, and artwork. |
| --- | --- | --- | --- |
| Artist | Server | Catalog UI, queue and context service | Artist metadata plus entry points into top tracks, albums, and related playback contexts. |
| --- | --- | --- | --- |
| Playlist | Server | Catalog UI, queue and context service | Read-only playlist in the baseline article. Acts as a queue seed with ordered trackIds and playlist metadata, not as a writable user-owned model. |
| --- | --- | --- | --- |
| QueueSnapshot | Client, persisted locally | Queue and context service, playback controller, persistence and sync service | Materialized playback context created from search, album, artist, or playlist entry points. Key fields include id, source type, source id, ordered queueItemIds, and createdAt. This is the primary source of truth for exact queue restores after refresh or restart. |
| --- | --- | --- | --- |
| QueueItem | Client | Queue and context service, playback controller | Represents a track inside a queue snapshot. Key fields include trackId, queueSnapshotId, original source position, insertion mode, and session-level availability state. |
| --- | --- | --- | --- |
| PlaybackSession | Client, persisted locally | Playback controller, persistent player, persistence and sync service | The current listening session. Key fields include active queue snapshot id, current queue index, repeat mode, shuffle seed, playback status, and active device id. |
| --- | --- | --- | --- |
| PlaybackCheckpoint | Client, persisted locally and partially synced | Persistence and sync service, playback controller, cross-device resume UI | Restorable playback position for a session. Local restore can use sessionId, queueSnapshotId, currentQueueIndex, trackId, positionMs, and updatedAt. The server-synced subset can be smaller and focus on best-effort continuity across devices. |
| --- | --- | --- | --- |
| Device | Client and server metadata | Persistence and sync service, handoff or resume UI | Best-effort continuity target, not full remote orchestration. Key fields include deviceId, type, capability flags, and lastSeenAt. |
| --- | --- | --- | --- |
| DownloadJob | Local adapter state, mainly desktop or offline-capable shells | Persistence and sync service, platform adapter | Tracks explicit offline intent and progress. Key fields include target type, target id, state, progress, last validated time, and linked cached entries or downloaded files. |
| --- | --- | --- | --- |
| CachedMediaEntry | Local adapter state | Platform adapter, service worker, persistence layer | Tracks best-effort cached media, artwork, manifests, or opportunistically reusable responses. Key fields include cache key, media type, storage location, validation metadata, and size. |
Relationship notes
Album and Playlist should keep ordered track references rather than duplicating full Track payloads inside every collection entity.
QueueSnapshot materializes a playback context, while QueueItem stores the ordered entries inside that snapshot. Separating them lets each item carry its own session-level state such as availability, gives drag-to-reorder and insertion operations stable reference ids to work with, and in IndexedDB, avoids rewriting the entire snapshot blob on every queue mutation. That makes local reorder and insertion operations safe without mutating the original album or playlist.
Track should stop at catalog metadata. Media delivery data such as stream URLs, manifests, or download package references should only come from playback bootstrap.
PlaybackCheckpoint is the persisted subset of PlaybackSession. Local restore should reload the full session and queue snapshot first, then apply the most recent checkpoint. A smaller synced checkpoint can be used for best-effort continuity on another device.
Exact ad hoc queue edits belong to local restore. A server resume hint can remember source context, current track, repeat mode, and shuffle seed, but it should not be treated as an authoritative copy of the locally mutated queue.
A single DownloadJob can produce multiple CachedMediaEntry records such as artwork, manifests, and media chunks or files, which is why explicit offline intent should stay separate from cache bookkeeping.

trackIds[]

trackIds[]

queueItemIds[]

trackId


activeDeviceId


targetId


durationMs


albumId


artistId


sourceType


sourceId


createdAt


trackId


sourcePosition


availability


queueSnapshotId


currentIndex


repeatMode


shuffleSeed


sessionId


trackId


positionMs


updatedAt


targetType


Client store entities and their relationships
Keep stream URLs and manifests out of the Track entity
Catalog entities should carry only long-lived metadata. Stream URLs, manifests, and entitlement data are short-lived, device-specific, and issued per playback session, so they belong in the playback bootstrap response rather than on Track. Embedding them in the catalog leaks expiring tokens into every cached list response and couples browse APIs to DRM and device capability concerns they should never know about.

Queue semantics
Starting playback from search results, an album, an artist page, or a playlist should create a queue snapshot. The player should then operate on that snapshot instead of directly mutating the source collection.

Queue mutations such as "play now", "add next", append, remove, and reorder should only update QueueItems inside the current QueueSnapshot.
Shuffle should be modeled from a stable seed stored on PlaybackSession so refresh or restore can reproduce the same order.
Previous should usually restart the current track if playback is already a few seconds in; otherwise, it should move to the previous playable item in the active snapshot.
Turning shuffle off should restore the original queue order and move the current index back to the current track's position in that order without restarting playback.
Unavailable tracks should remain representable inside the queue with a session-level availability state, which keeps queue indexes stable even when a specific track cannot be played on the current device or under current offline conditions.
Local resume should restore both PlaybackSession and PlaybackCheckpoint, not just a raw trackId, because repeat mode, queue position, and source context also matter. A reduced server resume hint can restore the source context and current track on another device, but exact ad hoc queue edits are not guaranteed to transfer in the baseline design.
Client-only UI state
Some player UI state does not need to be persisted. Examples include whether the queue panel is expanded, hover state, a transient seek preview, and an in-flight volume slider drag.

That state can stay in route-local or component-local memory. Losing it on refresh is acceptable; losing the queue, session, or playback checkpoint is not.

## Interface definition (API)

The player needs two API layers. The shared core exposes local interfaces through the playback controller and the queue and context service, while the data layer talks to server APIs for catalog data, playback bootstrap, signed-in continuity hints, and cache or download revalidation.

Catalog and discovery APIs
These APIs serve browse and detail routes and should return normalized entities plus ordered ids for the current surface.

These APIs are straightforward metadata reads. They should not try to encode queue behavior, because the queue is client-owned.

Playback bootstrap APIs
The playback bootstrap API should return only the data needed to start playback for the current environment, not a second copy of the full catalog payload.


// POST /tracks/:id/playback request
{
"deviceType": "web",
"mode": "stream",
"quality": "high"
}

// POST /tracks/:id/playback response
{
"trackId": "trk_123",
"mode": "stream",
"playback": {
"manifestUrl": "https://cdn.example.com/playback/trk_123.m3u8",
"expiresAt": "2026-03-18T12:00:00Z",
"capabilities": {
"canSeek": true,
"canPlayOffline": false
}
}
}
For desktop, the same logical playback contract can point at a different adapter or packaged media path. Separate download-resolution APIs can return downloadable media references for explicit offline jobs. The shared core should not need to know which platform-specific primitive ultimately plays or stores the audio.

On web, this separation also leaves room for Media Source Extensions or similar browser playback layers without changing the queue or session model.

Spotify's own Web Playback SDK is an example of a higher-level browser playback adapter, but the frontend architecture here should not depend on a vendor-specific SDK.

Local playback and queue interfaces
Queue mutation should stay local in the baseline design. User interactions such as "play now" or "add next" should not require a round trip before the UI updates.


```typescript
type PlaybackContextInput = {
```
sourceType: 'search' | 'artist' | 'album' | 'playlist';
sourceId?: string;
startTrackId: string;
};

```typescript
type QueueInsertionInput = {
```
trackId: string;
queueSnapshotId?: string;
sourceType?: 'search' | 'artist' | 'album' | 'playlist';
sourceId?: string;
insertionMode: 'playNow' | 'addNext' | 'append';
};

interface PlaybackController {
startPlayback(input: PlaybackContextInput): Promise<void>;
insertIntoQueue(input: QueueInsertionInput): void;
removeQueueItem(queueItemId: string): void;
reorderQueue(queueItemId: string, beforeQueueItemId?: string): void;
restoreSession(
session: PlaybackSession,
checkpoint?: PlaybackCheckpoint,
): Promise<void>;
}
If full multi-device remote control were in scope (e.g. Spotify Connect), server-side queue mutation APIs would matter much more. For this baseline answer, the queue should remain a client-owned structure derived from playback contexts and persisted locally.

Resume and offline APIs
Local restore should happen before any network call. After refresh or restart, the client should rebuild the exact QueueSnapshot, PlaybackSession, and PlaybackCheckpoint from local storage first. GET /resume is a best-effort continuity API for signed-in recovery when there is no local session to restore, or when the user intentionally resumes on another device.


// GET /resume
{
"trackId": "trk_456",
"positionMs": 91342,
"repeatMode": "off",
"shuffleSeed": "seed_789",
"sourceType": "playlist",
"sourceId": "pl_42",
"updatedAt": "2026-03-18T11:58:00Z",
"device": {
"deviceId": "web_chrome",
"type": "web"
}
}

// POST /playback/checkpoint
{
"trackId": "trk_456",
"positionMs": 91342,
"repeatMode": "off",
"shuffleSeed": "seed_789",
"sourceType": "playlist",
"sourceId": "pl_42",
"deviceId": "web_chrome",
"updatedAt": "2026-03-18T11:58:00Z"
}
Exact queue mutations such as ad hoc inserts, removals, or reorders should be guaranteed only by local restore in the baseline design. Across devices, the continuity contract is intentionally smaller: it helps the product resume the right track in the right source context without pretending the server has an authoritative copy of every local queue edit.

These APIs should support reconnect and offline recovery without turning the baseline design into a full sync engine. On web, they back best-effort cache revalidation and signed-in continuity. On desktop, they can support stronger durable download workflows.

The offline bridge should stay batch-oriented on offline-capable shells:

The client can enqueue one DownloadJob from that response instead of making one follow-up bootstrap request per track.

// POST /downloads/collections/resolve
{
"collectionType": "playlist",
"collectionId": "pl_42",
"revisionToken": "rev_17",
"tracks": [
{
"trackId": "trk_100",
"artifacts": {
"packageUrl": "https://cdn.example.com/downloads/trk_100.pkg",
"expiresAt": "2026-03-19T12:00:00Z"
}
},
{
"trackId": "trk_101",
"artifacts": {
"packageUrl": "https://cdn.example.com/downloads/trk_101.pkg",
"expiresAt": "2026-03-19T12:00:00Z"
}
}
]
}
Offline storage responsibilities
Offline support spans multiple storage layers, and the boundaries between them should stay explicit.

| Layer | Stores | Why |
| --- | --- | --- |
| App state | Visible job status, progress, optimistic UI flags, and currently mounted errors | Fast reactive updates while the app is open |
| --- | --- | --- |
| IndexedDB | Queue snapshots, checkpoints, continuity hints, revision tokens, cache metadata, and download bookkeeping for offline-capable shells | Structured persistence across refreshes and restarts |
| --- | --- | --- |
| CacheStorage | Artwork, playback manifests, and previously fetched media responses that are still reusable | Efficient browser-native response caching |
| --- | --- | --- |
| Service worker | Fetch interception, offline lookup, cache versioning, and revalidation policy | Central place to serve offline hits without making the page own every fetch decision |
The service worker's fetch-interception strategy should vary by resource type:

Playback bootstrap and catalog API calls should use a network-first strategy because freshness and entitlement validity matter.
Artwork and other immutable metadata can use a cache-first or stale-while-revalidate strategy because these resources are versioned and safe to reuse.
Previously fetched media responses can be reused opportunistically when they are still present and still valid, but the web baseline should not promise a dedicated "download" flow that makes tracks durably available offline.
Streaming media chunks should generally bypass the service worker entirely, since they are short-lived, managed by the playback adapter, and usually not worth caching for offline use in the web baseline.
This split also makes failure handling easier to reason about. The queue and session should not depend on binary cache state being in memory, and the service worker should not become the source of truth for playback intent or queue order. It also makes the browser's storage quotas and eviction behavior easier to explain.

## Optimizations and deep dive

Once the baseline player model is clear, this section refines it for long listening sessions, changing network conditions, and the practical limits of web and desktop runtimes:

Browser audio primitives: This section explains the browser media APIs that a web music player wraps behind its playback abstraction.
Audio delivery and buffering: This section covers how tracks are delivered, buffered, cached, and kept playable across network changes.
Playback startup, seeking, and quality selection: This section discusses fast startup, seek behavior, and choosing an appropriate audio quality.
Event-driven playback and failure handling: This section explains how playback state is reconciled from audio events, queue state, network events, and user actions.
Platform tradeoffs: This section compares what changes between web, desktop, mobile, and offline-capable runtimes.
Performance: This section covers responsiveness during long sessions, control latency, progress updates, and queue rendering costs.
User experience: This section discusses startup latency, transitions, resumability, queue behavior, and other details that shape listening quality.
Accessibility and controls: This section covers keyboard behavior, screen-reader output, media-session integration, and repeated control usage.
Browser audio primitives
On web, music playback still starts from the browser media stack. A frontend music player will usually wrap the browser's HTMLMediaElement or HTMLAudioElement behavior behind the platform adapter, so route-level UI never talks to the media primitive directly and the shared playback controller can stay in charge of queue, session, and checkpoint behavior.

In the most basic form, audio can be played progressively once enough of the file has been fetched to start playback. More advanced delivery can use segmented manifests and browser layers such as Media Source Extensions, which is why the architecture separates catalog metadata from playback bootstrap instead of baking media URLs directly into track entities.

Commercial music streaming also requires content protection. On web, that means Encrypted Media Extensions (EME), which lets the browser negotiate with a Content Decryption Module to decrypt protected audio before playback. EME constrains what the client can do with the media stream and affects which playback paths are available, but those details should stay entirely inside the platform adapter. The shared core should not need to know whether the underlying audio is DRM-protected or not, which is another reason the adapter abstraction matters.

DRM and entitlement concerns must stay inside the platform adapter
EME, Widevine or FairPlay negotiation, license expiry, and CDM capability differ by browser and device, and any leakage of those concerns into queue, catalog, or UI code makes the client hard to port and easy to break. Keep decryption, key rotation, and protected-playback capability checks behind the adapter, and treat license failures as explicit playback-controller states rather than ad hoc error banners.

This is also why playback in the article is modeled as an event-driven state machine. Browser media element events such as play, pause, waiting, ended, error, and time updates become inputs to the playback controller, whether the web shell is wrapping native browser primitives directly or a higher-level adapter such as Spotify's own Web Playback SDK.

Audio delivery and buffering
Once browser playback is abstracted behind the platform adapter, the next question is how media should actually be delivered and buffered for a music product. The goal here is not to go deep into media protocols, but to explain the delivery and buffering choices that have the biggest impact on how responsive the player feels.

Same protocols, different priorities. Audio streaming can use the same delivery mechanisms as video, including CDN-backed chunked media, manifest files, and short-lived playback tokens. The difference is usually not the transport itself but the client behavior around it: a music player optimizes for instant startup, smooth track-to-track transitions, and quick recovery instead of long-form viewing, subtitles, and complex bitrate switching. Audio codec choice also matters: common codecs include AAC, Opus, and OGG Vorbis, and browser support varies across them. The playback bootstrap response can include codec and format information so the platform adapter selects a rendition the current environment can decode, keeping codec negotiation out of the shared core.

Playback bootstrap is the media boundary. Track entities should stop at catalog metadata, while playback bootstrap returns the short-lived contract that tells the player how to load the current track in the active environment. On web, that can still mean HLS or DASH-style audio manifests, just as a video player might use, but here the point is to keep browse APIs cacheable while letting the playback contract carry expiry, entitlement, and device capability information. Offline-capable shells can layer download packaging on top without polluting the catalog entities.

Buffering favors fast starts. The player only needs enough audio buffered to start confidently; it can then extend the buffer in the background as playback continues. It should also warm the next track before the current one ends by prefetching the next track's playback bootstrap data, and sometimes the first media chunks, so skip-forward actions feel immediate and normal track transitions avoid audible gaps.

Platform adapter (next track)
Media CDN
API server
Playback controller
Platform adapter (current track)
Platform adapter (next track)
Media CDN
API server
Playback controller
Platform adapter (current track)
timeupdate (nearing end)
Manifest + expiry for next
preload(manifest, initial bytes)
Fetch first chunks
Media bytes (warm buffer)
play()
timeupdate events


Next-track prefetch and gapless handoff
Warm the next track's bootstrap and first chunks before the current one ends
Starting the next track's playback bootstrap request and pulling a small amount of initial media while the current track is still playing is usually the single largest win for perceived responsiveness. It keeps skip-forward and normal transitions near-instant without committing to a large speculative buffer window that would be wasted on the next user skip.

Interrupts are normal. Music playback is far more skip-heavy than long-form video, so stale bootstrap work, manifest fetches, and partial media requests should be canceled aggressively when the user changes quality, skips tracks, or switches playback context. That makes request cancellation and next-track warmup more important here than large speculative prefetch windows.

Keep the protocol depth shallow. It is useful to mention HLS or DASH-style audio delivery because it explains why playback bootstrap and buffering contracts exist, and why audio can reuse the same core streaming ideas as video. Offline-capable shells may add separate download-resolution contracts, but you do not need to give a Netflix-style deep dive into bitrate ladders, subtitle tracks, or full adaptive-streaming internals, because the harder front-end problems in this system are queue continuity, resume behavior, and offline handling. It's still helpful to understand media streaming, which the Netflix system design article covers.

Playback startup, seeking, and quality selection
Playback bootstrap stays small only if the client has a clear policy for what to do with it once it arrives.

Initial quality selection should be conservative. The player should choose a starting rendition based on user preference, device capability, and current network conditions, but it should bias toward uninterrupted audio instead of immediately chasing the highest bitrate. For music, a slightly lower bitrate is usually less noticeable than a stalled start, and browser signals such as the Media Capabilities API can help avoid obviously bad choices.

Adaptive fallback should happen early. If buffering becomes unstable or the reported connection quality drops, the player should switch down before the user hears audible gaps. That keeps the listening experience smooth and avoids a retry loop where the same high-quality rendition keeps failing.

Seeking depends on the media format. For plain file playback, seeking can map directly to the media element's currentTime. For segmented playback, the platform adapter should translate the requested timestamp into the correct segment window and refill the buffer from there without rebuilding the queue or session, similar to how adaptive streaming media sources are managed on the web.

Startup warmup should stay selective. Prefetching the current track and the opening media for the next likely track is usually enough. Buffering large stretches of future audio too aggressively wastes bandwidth, memory, and offline storage without helping the most important user interactions.

Event-driven playback and failure handling
Playback lasts for minutes or hours, but the signals that drive it arrive piecemeal from the audio element, the queue, the network, and the user. Without a single place to reconcile them, the visible player state, persisted session state, and actual audio output can drift out of sync very quickly.

Playback should be state-machine-driven. The platform adapter will emit low-level events such as play, pause, timeupdate, waiting, ended, and error, but the UI should not react to those events directly. The Playback controller should normalize them into a small set of application states that update PlaybackSession, PlaybackCheckpoint, and the visible player controls in one place, just as browser media element events are normalized in more specialized players.

startPlayback

manifest ready

bootstrap failed

autoplay denied

user gesture

canPlay


waiting / stall

track ended

next in queue

queue empty

retry / skip

media error


Idle


Buffering


Blocked

Playing

Paused

Ended


![Playback controller lifecycle](music-streaming-pdf-assets/figures/playback-controller-lifecycle.png)

Failures should become explicit states. Autoplay denial, expired playback contracts, buffering stalls, and unavailable tracks should not all collapse into a generic error banner. Each one needs a distinct recovery path: autoplay denial waits for user interaction, an expired contract triggers a new playback bootstrap call, buffering stalls keep the current queue item active, and an unavailable track can be marked on the QueueItem and skipped without corrupting the rest of the queue.

Reconnect should resume the session, not rebuild it. When the network drops and later returns, the player should continue from the current local QueueSnapshot, PlaybackSession, and PlaybackCheckpoint instead of reconstructing playback from route data or search results. That keeps the queue stable across temporary failures and lets the Persistence and sync service decide whether the current track can resume immediately, needs a refreshed playback contract, or must fall back to cached media on web or downloaded media on an offline-capable shell.

Concurrency matters most on web. Multiple tabs can compete for media ownership, listen to the same checkpoint store, or issue playback commands against stale local state. The simplest model is to elect one active playback owner while other tabs remain read-only observers of queue and progress state, which keeps media output centralized without introducing full Spotify Connect-style device orchestration. The Web Locks API (navigator.locks.request) is a clean way to implement leader election: the tab that holds the lock owns playback, while the Broadcast Channel API synchronizes queue and progress state to observer tabs.

Two tabs playing audio at once is a visible product bug
Without explicit leader election, a second tab can start its own audio element while the first is still playing, producing overlapping output and conflicting checkpoint writes that corrupt resume state. Pick one active playback owner via Web Locks or a similar coordination primitive, and demote all other tabs to observers that render queue and progress from the shared store.

Platform tradeoffs
The core player model should stay the same across platforms, but the runtime still changes what guarantees are realistic. Grouping those tradeoffs in one place makes it easier to compare what the web baseline can safely promise and what a desktop shell can strengthen without forking the architecture.

Web baseline
The browser can support a capable music player, but it does not behave like a single long-lived app process. Autoplay policy, storage quotas, tab lifecycles, and cross-tab coordination all shape what the web player can promise and where it needs explicit fallbacks.

Startup is policy-bound. The web player cannot assume that calling play() will succeed as soon as a track is selected, because browsers may require a user gesture before audio can start. That means autoplay denial should be modeled as a normal blocked state in the playback controller, not as an exceptional failure, and any protected-playback capability checks should stay inside the platform adapter and playback bootstrap path instead of leaking into the catalog or queue model. The browser's autoplay rules are part of the platform contract here.

Autoplay denial is a normal state, not an error
Designing it as a first-class "waiting for user gesture" state in the playback controller keeps session restore, cross-tab handoff, and resume-on-reconnect paths all reusing the same recovery surface. If autoplay denial is treated as a thrown error instead, every one of those flows has to re-implement the same gesture-prompt fallback.

Storage, lifecycle, and ownership are best-effort. IndexedDB, CacheStorage, and service-worker-managed caches can support metadata caching, local session restore, and opportunistic cached playback, but they are still subject to quota pressure and eviction, and background execution is tied to the tab and browser lifecycle rather than a durable app process. The same account can also be open in multiple tabs or windows, so the player needs explicit media-ownership rules and should treat offline support on web as conditional rather than promising desktop-level durability.

System hooks are progressive enhancement. Browser APIs such as the Media Session API improve lock-screen metadata, hardware media keys, and notification-surface controls, but they should remain thin layers on top of the shared player state. That keeps the web player usable even when those integrations are missing and prevents browser-specific features from becoming a second control path. Cross-tab ownership can also use the Broadcast Channel API without changing the core player model.

Route Media Session and media-key input back through the playback controller
Treat lock-screen buttons, hardware media keys, and OS notification controls as another input device that dispatches the same play, pause, skip, and seek actions as the in-app UI. If those handlers mutate the audio element directly, the visible player state, Media Session metadata, and PlaybackCheckpoint drift apart within a single track.

Desktop extension
Desktop is not a different product so much as a richer runtime for the same player core. The goal is to take advantage of durable storage, background work, and operating-system integrations without splitting queue, session, and checkpoint ownership into a separate native-only design.

Offline can become durable. A desktop shell can store downloaded tracks, albums, and read-only playlists on the filesystem instead of relying on browser-managed storage that may be evicted later. That makes it reasonable to promise reliable offline playback, but downloads should still be modeled as explicit user intent with revalidation handled in the persistence layer rather than scattered across the UI.

user marks offline

worker picks up

network lost / user pause


all artifacts saved

fetch or write error


revision check

still valid

token or revision changed

re-resolve artifacts

user removes download

Queued

Downloading

Paused

Cached

Failed

Revalidating

Expired


Offline download job lifecycle
Background work and lifecycle are more predictable. Desktop applications can keep download jobs running when the window is hidden, survive longer listening sessions, and resume more predictably after restarts or transient disconnects. That makes sleep, wake, restart, and interrupted downloads product-visible behaviors, so the architecture should define when checkpoints are flushed, how download jobs resume, and whether the previous session is restored automatically. Desktop shells also have stronger hooks such as Electron's powerSaveBlocker when playback or downloads must survive device sleep policy.

Native controls should reuse the same control surface. Media keys, lock-screen controls, notification controls, and richer audio routing make a desktop player feel native, but they should still map back to the same play, pause, seek, skip, and queue actions used by the in-app UI. If native integrations bypass the shared playback flow, the operating system and the app will drift out of sync. On desktop, that usually means routing native triggers through shell APIs such as Electron's global shortcuts instead of inventing a second playback state machine.

### Performance

Performance in a music player is not just about initial load time. The more important question is whether playback, controls, progress updates, and queue interactions still feel smooth after the app has been open for a long time.

Playback-critical work should dominate.

Startup latency: Users notice the delay between pressing play and hearing sound much more than they notice small differences in page render metrics.
Priority: Playback bootstrap, small startup buffers, and next-track warmup deserve precedence over background catalog work.
Selective prefetching: Warm the current track, the likely next track, and nearby queue metadata, but avoid letting artwork loads, search prefetches, or metadata refreshes compete with active playback.
Rendering cost should track what is visible.

High-frequency state: Progress bars, current time, buffered state, and volume changes can update many times per second, but most of the page does not need to react at that cadence.
Isolation: Keep those updates local to the persistent player instead of re-rendering the whole app shell while music is playing.
Large lists: Search results, album track lists, and long queues should use incremental rendering or virtualization.
Artwork loading: Non-critical artwork should be lazy-loaded at the right size so browsing does not interfere with playback responsiveness.
Long sessions need disciplined background work.

Timing source: The media element should remain the source of truth for playback progress instead of spawning many independent timers.
Storage writes: Checkpoints should be throttled to roughly every 10–30 seconds during continuous playback, plus on significant events such as pause, skip, or queue mutation. Queue persistence and download progress should be similarly batched so storage churn does not compete with playback.
Cross-tab work: On web, only the active playback owner should do expensive sync, polling, or warmup work.
Cleanup: Across all platforms, the player needs explicit teardown for listeners, timers, stale buffers, and abandoned media state so performance stays stable over hours of use.
### User experience

A music player is judged less like a normal page and more like a tool people live in for hours. That is why seemingly small details such as startup latency, smooth transitions, reliable resume, and clear recovery states matter so much: they are part of the core product experience, not just polish.

Playback should feel immediate. Pressing play, pause, skip, or add next should produce visible feedback right away, even if some underlying work still depends on network or media readiness. That means the player should update control state, queue state, and loading indicators optimistically enough to feel responsive while still making it clear when playback is waiting on buffering or user interaction.

Continuity matters more than page transitions. Users expect the player to survive navigation, refresh, temporary disconnects, and device sleep without losing the queue or forgetting where playback left off. That expectation is what makes queue snapshots, checkpoints, and resume behavior central to the architecture rather than implementation details hidden deep in the data layer.

Failures should be recoverable and legible. A capable media player does not just stop working when autoplay is blocked, a track becomes unavailable, or connectivity drops. It should explain what happened in the language of playback, preserve as much context as possible, and give the user a clear next step such as retrying, resuming, or skipping forward.

Long sessions amplify rough edges. Small inconsistencies in progress display, volume behavior, shuffle order, queue editing, or track transitions become much more noticeable when the player runs for hours instead of minutes. A good front-end design therefore treats these as first-class UX concerns rather than as nice-to-have refinements for later.

### Accessibility and controls

Music players are unusually control-heavy interfaces because the same actions are used over and over again during long listening sessions. That makes keyboard behavior, screen-reader output, focus management, and persistent controls part of the core player design rather than a late-stage accessibility checklist.

The persistent player needs a stable interaction model. Play, pause, skip, seek, volume, queue access, and device or output controls should stay in predictable locations and expose the same semantics across routes. That is especially important in a player that persists across navigation, because users should not have to rediscover how to control playback every time the surrounding page changes.

Keyboard support should cover the common listening workflow. Space to play or pause, arrow-based seek adjustments, volume changes, and queue navigation are all high-value shortcuts for a product that people use for hours at a time. The important architectural point is that keyboard commands should dispatch into the same playback and queue actions as pointer input, so there is only one control path to test and reason about.

Screen-reader feedback should track meaningful playback changes. Track changes, play or pause state, buffering, and actionable failures such as autoplay denial or unavailable media should be surfaced as concise status updates instead of silent UI changes. Seek bars and volume sliders also need clear labels, value announcements, and predictable keyboard semantics so they behave like real media controls rather than generic draggable widgets, similar to the guidance for accessible multimedia controls.

Focus management has to respect a long-lived shell. The player should not steal focus on every progress update or track transition, but it should return focus sensibly when users open the queue, dismiss a dialog, or trigger controls from the keyboard. That matters even more when queue management, search, and playback all coexist inside one application shell, because poor focus handling can make a persistent player feel much harder to use than a static page.

## Summary

The player feels professional when a long listening session stays coherent across navigation, refreshes, lost connectivity, tab switches, and handoff to the desktop app. Four concrete questions drive the continuity design.

Playback survives route changes and refresh because the player is mounted above routing. The persistent player, queue, and playback controller sit above browse, search, artist, album, and playlist routes, so navigation never unmounts the audio element or resets transport state. PlaybackSession and PlaybackCheckpoint are client-owned and persisted locally, with throttled checkpoint writes every 10 to 30 seconds plus on significant events. On reload the controller rehydrates from the local QueueSnapshot, PlaybackSession, and PlaybackCheckpoint before it consults the network, so audio resumes at the last known position even when the catalog response is still in flight. A hybrid rendering model keeps first paint fast without pulling playback state into route data.

The queue stays coherent by materializing playback context separately from playlists. QueueSnapshot and QueueItem let reorder, "play now", and "add next" operations mutate a local queue structure instead of the source playlist. On web, one tab is elected the active playback owner via the Web Locks API, and other tabs become read-only observers through the Broadcast Channel API, which prevents two audio elements from fighting over the same session. Cross-device continuity rides on a smaller synced subset of session state plus GET /resume and POST /playback/checkpoint, treated as best effort so a slow sync never blocks local playback.

Skips and transitions feel instant because playback bootstrap is the only media boundary. POST /tracks/:id/playback returns short-lived manifest URLs, expiry, and capability flags without polluting catalog payloads, which keeps the shared core out of codec negotiation. Next-track warmup prefetches the upcoming track's playback bootstrap and initial media chunks before the current track ends, so the gap between songs is dominated by decode, not network. Aggressive cancellation tears down stale bootstrap work, manifest fetches, and partial media requests the moment the user skips or changes quality, so a fast finger never stacks competing streams against each other. High-frequency UI for progress, buffered state, and volume is isolated from the rest of the shell.

Offline works without forking the player by hiding runtime details behind an adapter. The platform adapter hides HTMLAudioElement, MediaSession, service worker, and storage primitives on web, and native audio, filesystem, and OS media keys on desktop, so the player core never branches on runtime. Web offline stays best-effort through IndexedDB, CacheStorage, and service worker revalidation, while desktop downloads become durable, filesystem-backed jobs with explicit revision tracking via POST /downloads/collections/resolve and POST /downloads/revalidate. Device, DownloadJob, and CachedMediaEntry live in the adapter layer so offline intent stays separate from cache bookkeeping, and Track, Album, Artist, and Playlist stay server-originated and catalog-only.

Start with client-owned, locally restorable session and queue state behind a single playback bootstrap boundary. Streaming protocol choice and offline durability become easier to explain once that boundary is clear.

Advanced optimizations and topics
Under typical interview circumstances, presenting the above would usually be sufficient. However, if you have time to spare or the interviewer wants to dive deeper into certain areas, here are some plausible topics and possible optimizations.

Search and discovery responsiveness
Search is one of the most interrupt-driven parts of the product, and the same browsing surfaces are often revisited repeatedly during a listening session. That makes result stability and cancellation behavior just as important as raw API speed.

Requests should be cancellable. Search input should be lightly debounced and stale requests should be aborted, for example with AbortController, so older responses do not overwrite newer ones when a user types quickly or changes filters mid-request.

Pagination should preserve ordering. Search and browse cursors should stay tied to the query and the server's current ranking or shelf version, so loading more results does not silently mix incompatible result orderings inside the same surface.

Local affordances can hide latency. Recent searches, cached top results, and lightweight suggestions can make the product feel much faster even before the full result set arrives. These are useful local enhancements because they improve perceived responsiveness without changing the core playback architecture.

Advanced audio features as extensions
The baseline player does not need gapless playback, crossfade, equalizers, or visualizers to be credible in an interview. They are still worth mentioning because they change how much low-level control the client needs over timing, mixing, and audio processing.

Track transitions are the hardest upgrade. Gapless playback and crossfade both require the next track to be prepared before the current one finishes, and they need more precise scheduling than simply waiting for one media element to end and starting another. That usually pushes the platform adapter toward the Web Audio API (AudioContext, GainNode, AnalyserNode), because HTMLAudioElement alone cannot mix multiple sources, apply volume ramps, or perform real-time audio processing. The same API is what enables equalizers and visualizers, which is why all of these features tend to arrive together as a richer audio pipeline upgrade.

Effects should layer on top of the same player core. Equalizers, loudness normalization, and visualizers should observe or process the active audio stream without becoming new owners of queue, checkpoint, or session state. The playback controller should still decide what is playing and when, while any audio-processing layer only changes how that playback is rendered.

These features should stay outside the baseline answer. Bringing them up is useful because it shows awareness that some products will eventually need deeper control over audio rendering on web or desktop. They should still be framed as extensions, so the main design remains centered on persistent playback, queue continuity, resume behavior, and offline support.

Lyrics and synchronized text as an extension
Lyrics are another playback-adjacent feature that benefits from the same shared player model without needing to reshape the baseline architecture. They sit on top of the current track and playback time, so the main queue, session, offline, and reconnect design can stay exactly the same whether lyrics exist or not.

Loading should stay separate from playback bootstrap. Lyrics availability, licensing, locale, and versioning can vary independently from audio playback, so the client should fetch them through a separate extension contract such as GET /tracks/:id/lyrics?locale=...&version=.... That keeps catalog metadata small, avoids coupling lyric availability to stream entitlement, and lets the product choose among original, translated, or region-specific lyric payloads without changing the player core.

Synchronization should follow the shared playback clock. The lyrics renderer should subscribe to the playback controller for current track identity, playback position, pause state, and seeks. That way, pausing freezes the active line, seeking jumps to the right cue, reconnects resume from the current checkpoint, and track changes swap the lyric document without the lyrics UI becoming a second timing source. Even when the product uses custom scrolling UI instead of native TextTrack rendering, the timing model is still conceptually the same: cues advance in response to the media clock.

Formats vary, so the client should normalize them. Music products often use simple line-timed LRC-style payloads, while richer timed text can arrive in formats such as WebVTT or TTML2. Regardless of source, the client should normalize lyrics into a small internal cue model with start time, optional end time, text, language, cue kind, and optional word-level timing. That normalized shape makes it easier to support plain line highlighting today and karaoke-style word highlighting later without tying the UI to one upstream format.

Edge cases are mostly product problems, not player problems. Instrumental sections, unsynced lyrics, missing lines, multiple languages, censored variants, and word-level highlighting all affect how the lyrics panel renders, but they do not require changes to queueing or playback continuity. Offline support can also stay lightweight: if a track is already cached on web or downloaded on an offline-capable shell, the client can cache the lyric payload and version token beside it so the lyrics panel still works offline when rights and storage policy allow.

Personal playlists and offline playlist listening as an extension
Writable personal playlists change the shape of the problem because playlists stop being read-only playback contexts and become user-owned data that can diverge across devices and offline sessions.

The main difficulty is not rendering the playlist page itself, but preserving a stable playback experience while playlist contents are being edited, downloaded, and later reconciled. This is essentially another form of conflict resolution that has been discussed in the Google Docs article.

Playlist editing should reuse the same sync architecture. Creating, renaming, adding tracks, removing tracks, and reordering tracks should be handled as optimistic client updates that flow through the same persistence and sync layer used for checkpoints and downloads. The important boundary is that the currently playing queue should remain a separate queue snapshot, so editing a playlist does not silently rewrite the order of the active session unless the user explicitly starts playback again from the updated playlist.

Offline playlist listening should pin a playlist revision. Downloading a playlist is more than downloading its tracks one by one, because the client also needs the ordered track list, playlist metadata, and a revision marker describing what version of the playlist was saved for offline use. That lets the player keep a stable offline snapshot for browsing and playback, while the download layer fans that snapshot out into track-level downloads and cached media entries underneath.

Reconciliation becomes an ordered-list problem. When the user comes back online, the client may need to merge local playlist edits with changes made on another device or by a server-side refresh of the playlist. Add, remove, and reorder operations are harder to reconcile than simple field edits, so the client should keep a log of pending playlist mutations, fetch the latest playlist revision on reconnect, and rebase those mutations where possible instead of blindly overwriting one side. Rebasing here means replaying each pending local operation (add, remove, reorder) on top of the latest server state, skipping or flagging any operation that conflicts with a remote change such as a track that was already removed by another device.

Conflicts should favor continuity over surprise. If the same track was removed remotely, reordered in two places, or the playlist changed while a user was listening offline, the product needs a deterministic policy for what happens next. In practice, that usually means preserving the current playback queue until the session ends, applying merged playlist results only to future playback starts, and surfacing clear conflict states when automatic reconciliation would otherwise produce a surprising playlist order.

Multi-device handoff as an extension
Spotify Connect-style multi-device control is one of the product's most distinctive features, and the baseline architecture already has some of the pieces to support it without a major redesign. The Device entity is already in the data model, and checkpoint sync already pushes lightweight continuity state to the server. The extension is primarily about adding a real-time command channel and an explicit playback-transfer path between devices.

Device discovery should be server-mediated. Each active client registers its Device with the server and periodically refreshes its lastSeenAt timestamp. The server maintains an active device list that any client can query, so a user can see available targets for playback transfer without devices needing to discover each other directly.

Playback transfer is a checkpoint handoff plus queue export. When the user transfers playback to another device, the source client flushes its current PlaybackCheckpoint and either uploads a serialized QueueSnapshot for that transfer or promotes the session into a short-lived server-owned transfer object. The target device then receives a transfer command through a real-time channel such as a WebSocket or server-sent events and restores playback using the same restoreSession() path that already handles refresh and restart recovery.

Remote commands should flow through the same playback controller. Play, pause, skip, and seek commands from a remote device should enter the local playback controller through the same interface as local UI actions. That keeps the state machine centralized and prevents remote commands from creating a second control path that can drift out of sync with the local player state.

The baseline design does not need to fully solve this. Multi-device control adds real complexity around conflict resolution when two devices issue commands simultaneously, and around latency when remote commands travel through the server. It is worth mentioning in an interview because it shows how the architecture extends, but the baseline answer should remain focused on single-device playback with best-effort resume across devices.

## References

The references below cover Spotify's own engineering writing, browser media and storage APIs, desktop shell integration, and related system design articles on this site.

Spotify and product architecture
These references cover Spotify's own engineering writing on how the client and player are structured across platforms.

Building the Future of Our Desktop Apps | Spotify Engineering — useful background for the shared-core-plus-desktop-shell framing.
Spotify Player API | Spotify Engineering — good reference for player orchestration and capability-aware control surfaces.
Web Playback SDK reference | Spotify for Developers — useful for understanding what a browser playback integration can expose.
Browser media playback and UX
These references cover the browser media APIs and timed-text formats the player UI integrates with.

Media Source Extensions API | MDN — background on manifest-driven media streaming and segmented playback.
Setting up adaptive streaming media sources | MDN — useful background for segmented playback, seeking, and buffering behavior.
Media Capabilities API | MDN — relevant for capability-aware quality selection.
HTMLMediaElement events | MDN — useful for playback state machines driven by browser media events.
Media Session API | MDN — relevant for lock-screen metadata, hardware media keys, and system media controls.
Encrypted Media Extensions API | MDN — relevant for DRM-protected audio playback and Content Decryption Module negotiation.
Web Audio API | MDN — relevant for crossfade, equalization, visualization, and advanced audio processing beyond basic HTMLAudioElement playback.
Autoplay guide for media and Web Audio APIs | MDN — useful for browser autoplay constraints and recovery behavior.
Accessible multimedia controls | MDN — useful for accessible seek bars, control labels, and playback announcements.
AbortController | MDN — useful for canceling stale search and metadata requests.
WebVTT format | MDN — useful background for line-timed and cue-based lyric formats.
TextTrack | MDN — useful background for browser timed-text concepts and cue synchronization.
Timed Text Markup Language 2 | W3C — useful background for richer timed-text payloads and metadata.
Offline storage and browser platform APIs
These references cover the storage and coordination APIs that support offline playback and cross-tab behavior in the browser.

Service Worker API | MDN — relevant for offline fetch interception and cache coordination on web.
IndexedDB API | MDN — useful for structured local persistence of queue state, checkpoints, and download metadata.
CacheStorage | MDN — useful for cached manifests, artwork, and media responses.
Storage quotas and eviction criteria | MDN — important background for why web offline remains best-effort.
Broadcast Channel API | MDN — relevant for cross-tab playback ownership and coordination.
Web Locks API | MDN — relevant for cross-tab leader election to determine the active playback owner.
Desktop shell and OS integration
These references cover the desktop shell hooks that the web client delegates to for native integration.

Process model | Electron docs — helpful when thinking about desktop shell boundaries and native integration points.
Global shortcuts | Electron docs — relevant for media-key handling on desktop.
powerSaveBlocker | Electron docs — relevant for sleep behavior during playback or downloads.
Related system design articles
The articles below overlap with music streaming on player state, persistence, and sync concerns.

Video Streaming (e.g. Netflix) — useful background for player state, buffering, manifests, and adaptive delivery.
Chat Application (e.g. Messenger) — useful for understanding client-owned persistence, offline recovery, and cross-tab coordination patterns.
News Feed (e.g. Facebook) — useful for understanding browse-surface pagination, caching, and rendering tradeoffs.
Google Docs — useful for understanding conflict resolution techniques.