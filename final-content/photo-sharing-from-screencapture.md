---
title: "Photo Sharing (e.g. Instagram)"
slug: "photo-sharing"
difficulty: "medium"
duration_mins: 40
tags:
  - system-design
  - accessibility
  - networking
  - performance
premium: true
summary: "Design a photo sharing application like Instagram."
free_preview: false
status: draft
---

# Photo Sharing (e.g. Instagram)

## Question

Design a photo sharing application that contains a list of photo posts created by users. Users can create new posts containing photos.

Photo Sharing Example

## Real-life examples

- https://www.instagram.com
- https://www.flickr.com
Read the News Feed and Image Carousel articles first
This application shares many similarities with the News Feed system design application, and the image carousels within the feed posts will benefit from the Image Carousel system design. Please have a read of those questions before starting on this one. For this question, we will focus on things that haven't been covered, and the discussion will be centered around the photos and images.

Treat this article as the photo-specific delta on News Feed
The feed shape, pagination, rendering strategy, and store design are inherited from the News Feed write-up, and the in-post media UI is inherited from the Image Carousel write-up. In an interview, anchor on those defaults and spend your time on what is actually Instagram-specific: the image pipeline, upload flow, in-browser editing, and media-heavy rendering concerns.

## Requirements exploration

We scope the exercise by clarifying which user journeys are in play (browsing, uploading, editing), which pagination pattern the feed uses, and which image-specific concerns belong in the interview.

#### What are the core features to be supported?

Browse a feed containing image and video posts by people a user follows.
Upload photos, add captions, and apply filters to them before posting.
#### What pagination UX should be used for the feed?

Infinite scrolling, meaning more posts will be added when the user reaches the end of their feed.

#### Is the user able to add captions to individual photos/images or only one overall caption to the post?

Only one overall caption to the post.

#### What devices will the application be used on?

Primarily mobile, but should be usable on desktop and tablet as well.

## Architecture / high-level design

Because posts must be reachable by logged-out users and crawlable by search engines, the architecture depends on the rendering model and how the client talks to the image and feed APIs.

### Rendering approach

Photo sharing applications have the following characteristics:

Posts are viewable by non-logged-in users.
Posts are searchable via search engines.
The application is interaction-heavy due to post composing and liking/commenting functionality on posts.
Fast initial loading speed is desired.
These are also characteristics of a News Feed system design, where there's a good amount of static content that also requires interaction. Hence, a combination of server-side rendering with hydration and subsequent client-side rendering would be ideal. In reality, instagram.com is built using the same technology stack as facebook.com, which uses React server-side rendering with hydration.

![Architecture diagram](photo-sharing-pdf-assets/figures/architecture-diagram.png)

Photo Sharing Architecture Diagram

Server: Provides HTTP APIs to fetch feed posts, upload images, and to create new image posts.
Controller: Controls the flow of data within the application and makes network requests to the server.
Client store: Stores data needed across the whole application. In the context of a photo sharing application, most of the data in the store will be server-originated data needed by the feed UI.
Feed UI: Contains a list of image posts and the UI for creating new posts.
Image post: A post containing a list of one or more images.
Post composer: UI for uploading images, applying filters to them, and adding a caption before submitting to the server.
## Data model

A photo feed shows a list of image posts fetched from the server; hence, most of the data involved in the application will be server-originated data. The only client-side data needed is for user-uploaded images and user-written captions in the post composer.

| Entity | Source | Belongs to | Fields |
| --- | --- | --- | --- |
| Feed | Server | Feed UI | posts (list of Posts), pagination (pagination metadata) |
| --- | --- | --- | --- |
| Post | Server | Feed Post | id, created_time, caption, image, author (a User), images |
| --- | --- | --- | --- |
| User | Server | Client store | id, name, profile_photo_url |
| --- | --- | --- | --- |
| NewPost | User input (Client) | Post composer UI | caption, images (Image) |
| --- | --- | --- | --- |
| Image | Server/client | Multiple | url, alt, width, height |
Just like in the News Feed system design, a normalized client-side store will also be useful for a Photo Sharing application.

postIds[]

authorId

images[]

images[] (pending upload)


postIds


nextCursor


authorId


createdAt


profilePhotoUrl


uploadState


Normalized client store for a photo feed with composer drafts
Ship width and height with every image entity, not just the URL
Knowing the intrinsic dimensions up front lets the client reserve the correct aspect-ratio box before the image decodes, which is the main defense against cumulative layout shift in an image-heavy feed. It also lets the server or client pick the right srcset candidate without waiting for the bytes to arrive.

## Interface definition (API)

The client talks to three API surfaces: the feed list for browsing, the post creation flow for uploads, and the image upload endpoint that accepts large binaries.

Feed list API
Like News Feed's list API, the photo sharing application's feed list API should also use a cursor-based pagination approach.

Post creation API
All posts in a photo sharing application have attached images. Hence, the post creation process in such apps is more complex.

Let's break down the post creation into steps:

User selects photos from their device.
User is able to perform light edits on their photos: resize/crop/filter.
User adds a caption to the post.
User submits post.
There are two common ways to implement post creation functionality involving images:

Create a single API for both image uploading and post creation.
Create separate APIs for image uploading and post creation.
1. Create a single API for both image uploading and post creation
Photo Sharing API Single Diagram

Pros:

Simple to implement, upload all required data in one request.
Cons:

API logic is more complex because it has to do multiple things on the back end (though that's not really a concern of the front end).
Upload will take longer because the post images have to be uploaded within one request.
The API has to take in both text-based data and media data.
2. Create separate APIs for image uploading and post creation
Photo Sharing API Separate Diagram

Pros:

The image upload stage can be done in an async fashion. Images can be uploaded in the background after the image editing stage while the user is drafting the post caption.
Images can be uploaded in parallel across multiple HTTP requests, which is faster than uploading all the images in a single HTTP request.
Can leverage existing general image uploading APIs, if any.
Cons:

More coordination is needed on the client. The client needs to wait for all the images to be uploaded, get references to the uploaded images, and use them when creating the post.
#### Which approach is better?

Comparing the two approaches, the separate APIs approach would be better.

Reusable: The image uploading API can be a general one that is used across the entire website and not just for the post creation flow (e.g., profile image upload).
Reduced bandwidth: Practically, blob storage solutions like Amazon S3 and Cloudflare R2 are used for storing static assets like images. Clients can request a presigned URL and upload directly to the storage buckets without going through the app servers, which reduces bandwidth, leading to lower server costs.
Upload photos via presigned URLs, then create the post by media ID
The client asks the server for a presigned URL, PUTs the image bytes straight to blob storage, and then calls the post creation endpoint with just the returned media IDs. This keeps large binaries off application servers, lets uploads run in parallel and in the background while the user is still writing the caption, and makes the final post creation a small JSON request.

Blob storage (S3/R2)
App server
Client (store + view)
User (Composer)
Blob storage (S3/R2)
App server
Client (store + view)
User (Composer)
[Upload image 1]
[Upload image 2]
[User drafts caption]
Finish editing photos
Request presigned URLs (N images)
N presigned URLs + media IDs
200 OK
200 OK
Submit post
Create post with mediaIds + caption
Normalized { post, images }
Prepend to feed, clear composer


Parallel presigned-URL uploads while the user drafts the caption
## Optimizations and deep dive

The deeper discussion builds on the News Feed optimizations but pays extra attention to image-heavy concerns, upload flow, and media accessibility:

Feed optimizations: This section explains which News Feed performance and state-management techniques also apply to an Instagram-style feed.
Image carousel optimizations: This section covers the carousel behavior used when a post contains multiple images or videos.
Rendering images: This section discusses image formats, sizing, CDN delivery, lazy loading, and responsive rendering.
Image editing: This section explains in-browser cropping, resizing, filters, stickers, text overlays, and upload preparation.
Accessibility (a11y): This section covers text alternatives, keyboard interaction, and screen-reader support for an image-first product.
Feed optimizations
The optimizations covered in the News Feed system design also apply to the feed of photo sharing applications.

### Rendering approach

Infinite scrolling implementation
Virtualized lists
Code splitting JavaScript
Loading indicators
Preserving feed scroll position
Lazy load code
Optimistic updates
Timestamp rendering
Icon rendering
Image carousel optimizations
The images within a post are shown as a carousel, so the Image Carousel system design article is also useful for this question and the same optimizations apply.

Rendering images
Use modern image formats such as WebP, which provides superior lossless and lossy image compression.
<img>s should use proper alt text.
Instagram allows users to provide alt text for each image. If the user doesn't specify the alt text, Machine Learning and Computer Vision techniques can be used to process the images and generate a description.
Image loading based on device screen properties
Send the browser dimensions in the initial request (or in a subsequent one) and the server can decide what image size to return.
Use srcset if there are image processing (resizing) capabilities to load the most suitable image file for the current viewport.
Adaptive image loading based on network speed
Devices with good internet connectivity/on WiFi: Prefetch offscreen images that are not in the viewport yet but are about to enter the viewport.
Poor internet connection: Render a low-resolution placeholder image and require users to explicitly click on it to load the high-resolution image.
Serve multiple sizes and modern formats, and let the browser choose
Produce WebP (or AVIF) variants at several widths during the upload pipeline and expose them via srcset plus sizes. The browser will then pick the right candidate for the current viewport and device pixel ratio, which matters far more in a photo-first feed than in a text-heavy one because images dominate both byte budget and perceived load time.

Image editing
In-browser editing covers three capabilities before upload: cropping and resizing, applying filters, and drawing overlays like stickers or text.

User picks file

Open editor

Crop / resize / filter / overlay

Confirm edits

Blob ready


### Network / storage error


Retry

Discard

Bound to NewPost mediaIds

Post created


Editing

Encoding

Uploading

Uploaded

Failed

Attached


Per-image lifecycle from selection to attached media
Cropping and resizing
Cropping and resizing of the images can be done via HTML5's <canvas>.


const canvas = document.getElementById('image-editor');
const context = canvas.getContext('2d');
const image = new Image();
image.src = 'https://greatimages.com/example.jpg';
context.drawImage(image /* Other parameters */);
The Instagram website allows editing by simulating the result with inline styles to modify the transform styling and positioning of the image.

Filters
CSS provides a filter property which allows applying filter functions like blur, contrast, hue, and sepia, to name a few. By using a combination of these filter functions, you can achieve Instagram-like filtering functionalities right within your browser.

Here are examples of filter functions to achieve the Clarendon and Gingham filter effects from Instagram, taken from the awesome Instagram.css.


.filter-1977 {
filter: sepia(0.5) hue-rotate(-30deg) saturate(1.4);
}

.filter-brannan {
filter: sepia(0.4) contrast(1.25) brightness(1.1) saturate(0.9) hue-rotate(-2deg);
}
The following images are enhanced right within your browser using these CSS filters.

| Original | 1977 | Brannan |
| --- | --- | --- |
| Air Balloon | Air Balloon with 1977 filter effect | Air Balloon with Brannan filter effect |
Ultimately, the final image still has to be converted into an image blob before being sent to the server, and this conversion can be done with libraries like html2canvas.

No

Yes

File input / drag-drop

Decode into HTMLImageElement

Canvas: crop and resize

CSS filter preview in DOM

#### User confirms?


html2canvas / canvas.toBlob

WebP or JPEG blob


Media ID attached to NewPost


In-browser editing pipeline from file picker to uploaded blob
### Accessibility (a11y)

An image-first product leans heavily on text alternatives and keyboard operability, which are covered in the screen reader and keyboard interaction subsections below.

Screen readers
<img> tags should have a meaningful alt description specified or use an empty string.
Add aria-labels to the prev/next buttons within an image carousel.
Icon-only buttons should have the appropriate aria-labels if there are no accompanying labels (e.g., heart, share, bookmark).
Keyboard support
Use the <button> HTML tag for the Prev/Next buttons where possible so that the buttons are focusable.
Add <div role="region" aria-label="Image Carousel" tabindex="0"> to make the carousel focusable and attach Left/Right keydown handlers to allow scrolling through the images with the keyboard.
## Summary

Treat Instagram as News Feed plus Image Carousel plus a media delta. Once the inherited feed and carousel mechanics are named, focus on the Instagram-specific image pipeline, upload flow, in-browser editing, and media-heavy rendering concerns.

Most of the scaffolding comes from News Feed and Image Carousel. The Server, Controller, and Client store shape the feed exactly as in News Feed, with Feed, Post, and User held in a normalized Client store and the Feed UI reading from it. Cursor-paginated requests drive a virtualized, infinite-scroll Feed UI so memory stays bounded as the user scrolls. Each post hosts the Image Carousel patterns for multi-image navigation, with srcset and sizes picking the right asset per viewport and accessibility defaults (alt text on <img>, aria-labels on icon-only buttons, focusable role="region" wrappers with keyboard handlers) carried through unchanged.

The media pipeline is the Instagram-specific design. The Post composer asks the Server for a presigned URL per image, then PUTs the bytes directly to blob storage in parallel with the user composing the caption, so upload latency is hidden behind typing time. Post creation is a second call that references media IDs returned by that upload, keeping the NewPost API small and the Image entities immutable once stored. In-browser cropping and resizing run on <canvas>, with CSS filter driving cheap live previews before the edited bitmap is encoded back to a blob for upload. Every Image ships with width and height metadata so the Feed UI can reserve aspect-ratio boxes and avoid layout shift as posts hydrate. The upload, edit, and submit flow is distinctly Instagram's and deserves explicit walkthrough.

Spend interview time on the presigned-URL upload pipeline, the <canvas>-based editor, and the layout-stability contract around width/height; the inherited feed mechanics can be named quickly and moved past.

## References

The references below group sources by topic, so it is easier to map each source back to the part of the article it informed.

Instagram engineering case studies
These references inform the overall client architecture, caching, and bundle-size choices discussed throughout the optimizations section.

Making Instagram.com faster: Part 1 for baseline performance work and the measurement approach behind the redesign.
Making Instagram.com faster: Part 2 for additional client-side loading and rendering strategies relevant to a photo feed.
Making Instagram.com faster: Part 3 — cache first for the cache-first data access pattern referenced in the client caching discussion.
Making Instagram.com faster: Code size and execution optimizations (Part 4) for code-splitting and execution tuning on a long-lived feed surface.
### Accessibility and messaging

These references inform the accessibility recommendations for the feed and the scope notes around direct messaging on the web.

Crafting an accessible Instagram feed for feed-specific accessibility patterns, focus management, and assistive technology considerations.
Launching Instagram Messaging on desktop for how a large photo product layered real-time messaging onto an existing web client.
Image loading and performance APIs
These references back the image lazy-loading, infinite scroll, and rendering-strategy guidance covered in the optimizations section.

Intersection Observer API | MDN for infinite scroll triggers and prefetch boundaries around the viewport.
Lazy loading images and iframe elements | web.dev for browser-native image lazy loading guidance used in the media optimization discussion.
Rendering on the Web | web.dev for the SSR, CSR, and hybrid rendering background that informs the rendering approach for feed and profile pages.
Instagram.css for a community CSS reference that mirrors Instagram-style visual treatments when prototyping a photo feed.
Related system design articles
The articles below dive deeper into mechanics that are touched on but not fully explored here.

News Feed (Facebook) for the shared feed architecture, pagination, and client store patterns that a photo feed builds on top of.
Pinterest for masonry-style image layouts and additional photo-heavy performance considerations.
The Mobile Fear Factor Tee — The ultimate horror story