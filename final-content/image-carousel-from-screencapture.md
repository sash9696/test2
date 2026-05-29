---
title: "Image Carousel"
slug: "image-carousel"
difficulty: "medium"
duration_mins: 30
tags:
  - system-design
  - accessibility
  - performance
  - ui-component
premium: true
summary: "Design a horizontally-scrolling image carousel component."
free_preview: false
status: draft
---

# Image Carousel

## Question

Design an image carousel component that displays a list of images one at a time, allowing the user to browse through them with pagination buttons.

Image Carousel Example

## Requirements exploration

Before designing the API, we clarify how images are supplied, which devices the carousel must support, and which navigation and customization controls are in scope.

#### How are the images specified?

It will be a configuration option on the component, and the list of images has to be specified before initializing the component.

#### What devices should the component support?

Desktop, tablet, and mobile.

#### How will the pagination buttons behave when the user is at the start/end of the image list?

It should cycle through infinitely.

#### Will there be animation when transitioning between images?

Yes, the images should animate with horizontal translation.

## Architecture / high-level design

Since this component doesn't need any server data, our architecture will just consist of the client-side components.

Image Carousel Architecture

Component responsibilities
ViewModel + Model: The brain of the component; holds the configuration and state of the component, orchestrates events between the components, and informs the "Image" component which image to render.
Image: Displays the currently selected image.
Prev/Next Buttons: Tell the "ViewModel" to change to the prev/next image depending on which button is clicked.
Progress Dots: Tell the "ViewModel" which image to show when the respective dot/page is clicked/selected.
There's no need to separate the "Model" and the "ViewModel" in this component because it's a small component.

View

currentIndex

currentIndex

prevImage()

nextImage()

showImage(index)

nextImage()

prev/nextImage()

ViewModel + Model
(config, currentIndex, timer)

Image (current slide)

Prev button

Next button

Progress dots

Autoplay timer

Keyboard (Left/Right)


![Component hierarchy and action flow](image-carousel-pdf-assets/figures/component-hierarchy-and-action-flow.png)

![Flux architecture](image-carousel-pdf-assets/figures/flux-architecture.png)

A Flux/Redux/Reducer-like architecture is recommended to abstract away the action sources from the action logic/implementation. Some actions can be triggered by interacting with various UI elements, periodically by a timer, or by keypresses.

## Data model

Only the "ViewModel" will hold any state and data; the other components are part of the view and won't hold data. It can contain the following fields:

Configuration
List of images, which includes both the image URL and alt value, if provided.
Transition duration.
Size: Height and width of the image.
State
Index of current image. This value can be modified by the interactive elements (Prev/Next buttons, Progress Dots).
images[]

(if autoplay)


currentIndex


transitionDuration


ViewModel state and configuration relationships
## Interface definition (API)

Since we're talking about designing a UI component here, the API will focus on the external API of the components: what configuration options are provided so that developers can use the component in a customized fashion.

Basic API
List of images: An array of image URLs to be displayed within the carousel with any associated metadata (optional but good to have).
Transition duration (ms): Duration for the translation animation during image transitions.
Height (px): Height of the image.
Width (px): Width of the image.
An example of an ImageCarousel component defined in React:


<ImageCarousel
images={[
{ src: 'https://example.com/images/foo.jpg', alt: 'A foo' },
{ src: 'https://example.com/images/bar.jpg', alt: 'A bar' },
/* More images if desired. */
]}
transitionDuration={300}
height={500}
width={800}
/>
Advanced API
These are non-essential APIs but worth a discussion if there's time.

Autoplay: Whether the carousel will automatically progress to the next image after some time.
A timer state value will be needed to keep incrementing the image.
The timer should be cancelled/reset if the current image was manually changed by the user (either through Prev/Next buttons or Progress dots).
Delay: Delay between transitions. Only needed if the carousel is in autoplay mode.
Event listeners: It'd be useful to add event listeners to buttons of the component so that developers can implement their own extra functionality (e.g. logging user interactions).
onLoad: When the first image is done loading and shown in the carousel.
onPageSelect: When a page is selected.
onNextClick: When the next image button is triggered.
onPrevClick: When the prev image button is triggered.
Theming: See Front End Interview Guidebook's UI Components API Design Principles Section.
Loop: Enable a looping behavior where clicking on the "Next" button while the last image is presented returns to the first, and clicking on the "Prev" button while the first image is presented shows the last image.
Autoplay in particular needs a small state machine so that user interactions (hover, manual navigation) pause the timer without permanently disabling it.

autoplay enabled

timer tick

transition ends

pointer enters carousel

pointer leaves

prev / next / dot / swipe

delay elapses (or resume)

autoplay disabled

Idle

AutoplayRunning

Transitioning

PausedByHover

PausedByUser


Autoplay lifecycle with user interruption
Internal API
| API | Source | Description |
| --- | --- | --- |
| prevImage() | Prev Button / Keyboard events | Shows the previous image |
| --- | --- | --- |
| nextImage() | Next Button / Keyboard events / Timer (if autoplay) | Shows the next image |
| --- | --- | --- |
| showImage(index) | Progress Dots | Skips to a specific image |
These behaviors are encapsulated within APIs because they can be called from multiple places (UI elements, timers) and can contain a fair amount of logic depending on whether looping behavior is turned on.
prevImage() and nextImage() can call showImage(index) internally.
These internal APIs can be implemented as Flux/Redux actions if using a Flux/Redux architecture.
View (Image + dots)
Autoplay timer
ViewModel
Source (button / dot / key / timer)
View (Image + dots)
Autoplay timer
ViewModel
Source (button / dot / key / timer)
[Source is user
interaction]
prev / next / showImage(index)
reset / restart timer
resolve target index (with loop wrap)
update currentIndex
animate translate to slide
transition end
emit onPageSelect / onNextClick / onPrevClick


Navigation action flow from multiple sources
## Optimizations and deep dive

The deeper discussion centers on the details that make a carousel feel right across devices, media sizes, input methods, and languages:

Layout implementation: This section explains how to arrange slides, animate transitions, and position navigation controls.
User experience: This section covers scroll snapping, pagination behavior, touch gestures, autoplay expectations, and interaction details.
Performance: This section discusses image loading, preloading, formats, sizing, and bandwidth control for media-heavy carousels.
Internationalization (i18n): This section covers localized control labels and text-direction considerations for carousel navigation.
Accessibility (a11y): This section explains keyboard support, screen-reader behavior, focus handling, and safer carousel patterns.
Layout implementation
The core layout decisions are how to arrange the image strip, how to animate transitions between slides, and how to position the navigation controls.

Images
A simple version to achieve the layout is to use display: flex to make the images render in a horizontal row and programmatically change the horizontal scroll offset to show the various images.

You can draw out such a diagram on the whiteboard to illustrate the layout:

Image Carousel Layout

The box with the black border indicates the currently visible window.

Sample Code


<div class="carousel-images">
<img class="carousel-image" alt="..." src="..." />
<img class="carousel-image" alt="..." src="..." />
<img class="carousel-image" alt="..." src="..." />
<!-- More images -->
</div>

.carousel-images {
display: flex;
height: 400px;
width: 600px;
overflow: hidden;
}

.carousel-image {
height: 400px;
width: 600px;
}
To scroll to a particular image smoothly, we can do:


document.querySelector('.carousel-images').scrollTo({
left: selectedIndex * 600,
behavior: 'smooth',
});
Fitting Images
The layout above assumes that all the images are of the same size. However, it is unlikely that the user will provide images that are of that exact size.

Here are a few ways we can get around this:

CSS background-size: This requires a change in the HTML to render the image using CSS background/background-image instead of <img src="...">. The advantage of this is that we can use the background-size property that has these two modes:

contain: Scales the image as large as possible within its container without cropping or stretching the image. If the container is larger than the image, this will result in image tiling, unless the background-repeat property is set to no-repeat.
cover: Scales the image (while preserving its ratio) to the smallest possible size to fill the container (that is: both its height and width completely cover the container), leaving no empty space. If the proportions of the background differ from the element, the image is cropped either vertically or horizontally.
Source: background-size - CSS: Cascading Style Sheets | MDN

Both options have their merits, and which to use really depends on the provided images. One way is to allow the developer to customize whether to use contain or cover for all the images, or even allow customizing this setting for individual images.

CSS object-fit: CSS object-fit is a feature that is similar to how background-size works for background-images, but for <img> and <video> HTML tags. It has contain and cover properties as well, which work the same way as background-size's version.

This way is preferred since it's more semantic to use <img> tags than <div>s with background-images.

Prefer <img> with object-fit over <div> backgrounds for slides
<img> keeps the slide semantic for assistive tech, exposes alt text, and benefits from native lazy loading and srcset. CSS background-image should only be reached for when you genuinely need background-specific behavior that object-fit cannot replicate.

Vertically-center buttons
To vertically center the buttons, we can use position: absolute on the buttons along with transform: translateY(-50%) to shift them up by half their height.


<div class="carousel-image-container">
<div class="carousel-images">
<img class="carousel-image" alt="..." src="..." />
<img class="carousel-image" alt="..." src="..." />
<!-- More images -->
</div>
<button class="carousel-button carousel-button-prev">...</button>
<button class="carousel-button carousel-button-next">...</button>
</div>

.carousel-image-container {
height: 400px;
width: 600px;
position: relative; /* So that position: absolute will be relative to this element. */
}

.carousel-button {
position: absolute;
top: 50%;
transform: translateY(-50%); /* Shifts the button up by half its height. */
}

.carousel-button-prev {
left: 30px;
}

.carousel-button-next {
right: 30px;
}
### User experience

Scroll snapping: Use CSS scroll snapping and show the scrollbar so that users can still use native scrollbars to scroll through the images but the images will "snap" nicely and the scroll position will nicely align to show a full image once scrolling stops. Mobile users expect to be able to swipe through the images and CSS scroll snap helps you achieve this on mobile devices easily.
Interactive elements should be large enough. The Prev/Next buttons should be at least 44px tall and wide to be easy to tap on mobile devices. One trick is to increase the hit area of the button to the entire leftmost/rightmost section. The dots are probably too small on mobile for precise interactions and can just be hidden or made non-interactive.
Reposition Prev/Next buttons: It's more convenient to have the Prev/Next buttons close to each other. This goes against the example design but speeds up navigation because the distance between the buttons is minimized.
Default to CSS scroll snap for swipe and native scrolling on mobile
A horizontal scroller with scroll-snap-type: x mandatory and scroll-snap-align on each slide gives you touch swiping, momentum, and keyboard-accessible scrolling essentially for free. Reach for JavaScript-driven transforms only when you need behavior that scroll snap cannot express.

### Performance

Carousels can be bandwidth-heavy, so the main levers are deferring off-screen image loads and choosing sensible image formats and sizes.

Defer loading of images that aren't on-screen
Most images in the carousel are never shown to the user, especially those at the back. It'd be a waste of bandwidth to load all the images if they are not being shown.

Hence, the images can be loaded only when they are on-screen (or about to be shown). This can be implemented using JavaScript or just via HTML. A pure HTML method involves using the <img loading="lazy"> attribute, which defers loading images that aren't currently within the visible viewport.

Preloading of images
JavaScript can be used if fine-grained control over image loading is desired. To help minimize the scenario where an image needs to be shown but hasn't finished loading, the component can pre-emptively load the next image with the following code:


const preloadImage = new Image();
preloadImage.src = url;
Airbnb optimized the image carousel experience in their room listings with a combination of lazy loading and preloading behaviors:

Initially, only the first image is loaded (the remaining images will be lazily loaded).
The second image is preloaded when the user shows possible intent of viewing more images:
Cursor hovers over the image carousel.
Focuses on the "Next" button via tabbing.
Image carousel comes into view (on mobile devices).
If the user does view the second image (which signals high intent to browse even more images), the next three images (3rd to 5th) are preloaded.
As the user clicks "Next" again to browse more images, the (n + 3)th image is preloaded.
hover / focus Next / scroll into view


Carousel mounted

Load image 1 only

#### Intent signal?


Preload image 2

#### User advanced to image 2?


Preload images 3-5

#### User clicks Next again?


Preload image (n + 3)


Preload scheduling escalating with browse intent
Preload adjacent slides on intent, not the entire image list upfront
Eagerly fetching every carousel image wastes bandwidth because most images are never viewed. Load the first slide immediately, then preload the next one or two when the user hovers, focuses the Next button, or the carousel scrolls into view, escalating only when continued navigation signals real intent.

Airbnb Image Carousel Lazy Loading
Airbnb image carousel lazy loading example on mobile
Device-specific images
It'd be a waste to load high-resolution images on a mobile device where the screen size is too small to display the details anyway. A good feature to add would be to allow the user to provide images of different sizes to be displayed on different devices. This can be implemented using JavaScript, the srcset attribute on <img> tags, or the new <picture> tag.

Virtualized lists
If there are too many images, we can use virtualized lists to render only the visible images in the DOM.

### Internationalization (i18n)

i18n is not extremely relevant to this question, but the user should be able to customize the aria-label strings for the Prev/Next buttons for the current language of the page. These strings can be specified as configuration options.

### Accessibility (a11y)

Image carousels are notorious for having poor accessibility due to how much effort it takes to build them right. For this reason, you should probably not build your own custom image carousels without expecting to dedicate a significant amount of time to achieve a top-quality component. Some things to take note of when building accessible image carousels:

Rolling your own carousel almost always ships with a11y regressions
Custom carousels routinely miss keyboard traversal, focus management, screen reader labeling, and reduced-motion handling. Prefer a well-tested library unless you are committing real time to validating the component against assistive tech.

Mobile-friendliness
Interactive elements should be large enough for mobile (at least 44px x 44px).
Enable swiping to scroll through images (this is already the case with an overflow-x: scroll + CSS scroll snap implementation).
Progress dots should be spaced further apart or made non-interactive.
Screen readers
<img> tags should have a meaningful alt description specified or use an empty string.
Add aria-labels to the Prev/Next buttons since there's no visible label within them.
Keyboard support
Use the <button> HTML tag for the Prev/Next buttons where possible so that the buttons are focusable.
Add <div role="region" aria-label="Image Carousel" tabindex="0"> to make the carousel focusable and attach Left/Right keydown handlers to allow scrolling through the images with the keyboard.
## Summary

Keep the carousel simple by concentrating coordination in one controller. The ImageCarousel exposes a configuration-first surface (images, transitionDuration, height, width, plus opt-in autoplay, loop, and event listeners onLoad, onPageSelect, onNextClick, onPrevClick), and collapses the Model and ViewModel into a single controller that owns currentIndex and the autoplay timer. Every input source, including Prev/Next Buttons, Progress Dots, keyboard arrows, touch swipes, and the autoplay tick, converges on the same three actions, prevImage(), nextImage(), and showImage(index), so new triggers slot in without reshaping the view.

Keep layout close to the platform. A display: flex strip inside an overflow: hidden window and <img> tags sized via object-fit let arbitrary aspect ratios land cleanly. Navigation layers on top rather than replacing defaults: absolutely positioned Prev/Next Buttons and Progress Dots drive the controller, CSS scroll snap delivers free swipe and keyboard scrolling, and a role="region" wrapper with Left/Right key handlers makes the whole component operable without a mouse.

Treat accessibility and autoplay as interaction rules, not visual extras. Meaningful alt text, aria-labels on the buttons, and 44x44px hit targets make each control usable, while autoplay stays a small state machine that pauses on hover and resets on manual navigation so the timer never fights the user.

Tier image loading by user intent. The first slide ships immediately, loading="lazy" defers the rest, hover or Next-button focus preloads the adjacent slide, Airbnb's n+3 prefetch escalates as the user advances, and srcset right-sizes each request per device.

Start with the external API and the controller that funnels every input source through one action set; the rest of the design builds on that decision.

## References

The references below group image carousel sources by theme, so it is easier to map each source back to the part of the article it informed.

### Accessibility and UX patterns

This subsection covers the accessibility and user experience foundations for designing a usable carousel.

Creating an Accessible Image Carousel for a hands-on tutorial that ties markup, keyboard support, and ARIA together.
A Content Slider for an inclusive design walkthrough of slider semantics and interaction design.
Designing A Perfect Carousel UX for UX research and patterns that actually keep carousels useful.
Carousel pattern | W3C ARIA APG for the canonical ARIA authoring practices governing carousel semantics.
Production case studies
These case studies show how product teams ship carousels and image-heavy surfaces at scale.

Building a Faster Web Experience with the postTask Scheduler | Airbnb for how Airbnb uses scheduling primitives to keep carousel-heavy pages responsive.
### Performance and lazy loading

This subsection collects the browser APIs and techniques that keep carousels cheap to render and navigate.

Intersection Observer API | MDN for detecting when slides enter or leave the viewport to drive prefetching and autoplay pausing.
Lazy loading images and iframe elements | web.dev for native loading="lazy" guidance that avoids downloading off-screen slides.
Related system design articles
The articles below overlap with the image carousel on media-heavy product pages and media galleries.

Photo sharing (Instagram) for how a feed of media items uses carousels within a larger image-centric product.
E-commerce website (Amazon) for how product page image galleries and carousels integrate into a commerce surface.