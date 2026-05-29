---
title: "Travel Booking (e.g. Airbnb)"
slug: "travel-booking"
difficulty: "medium"
duration_mins: 40
tags:
  - system-design
  - performance
  - seo
premium: true
summary: "Design a travel booking website like Airbnb and Expedia."
free_preview: false
status: draft
---

# Travel Booking (e.g. Airbnb)

## Question

Design a travel booking website that allows users to search for accommodations and make a reservation.

Travel Booking Application Product Listing Page

Travel Booking Application Product Details Page

## Real-life examples

Airbnb
Booking.com
Expedia
TripAdvisor
## Requirements exploration

We scope the problem by clarifying which user journeys are in play, which devices to target, and which global concerns, including SEO, localization, and currency, shape the design.

#### What are the core features to be supported?

Search and browse accommodation listings.
Viewing accommodation details such as price, location, photos, and amenities.
Make reservations for accommodations.
#### What do the user demographics look like?

International users of a wide age range: US, Asia, Europe, etc.

#### What are the non-functional requirements?

Each page should load under 2 seconds. Interactions with page elements should respond quickly.

#### What devices will the website be used on?

All possible devices: laptops, tablets, mobile, etc.

#### Do users have to be signed in?

Anyone can search for listings and browse details, but users need to be logged in to make a booking.

## Summary

From the above, we can gather that the most important aspects of the website are:

Search engine optimization: Travel sites should aim to rank high on search engines since organic search is one of the primary discovery mechanisms.
Performance: Performance is known to affect conversions. For websites where the aim is to get customers to make purchases, performance is very important.
Internationalization: The users are from many different countries and from various age groups. To capture a larger market, the website should be translated and localized.
Device support: The site should work well on a variety of devices since the user demographics are very broad.
The requirements and nature of a travel booking website are very similar to the E-commerce website system design, so there is a high degree of overlap between the solutions. In this solution, we will cover the unique aspects of travel booking websites that aren't found on e-commerce websites.

SEO and performance are the thesis, not nice-to-haves
Travel sites depend on organic search for discovery and on conversion-sensitive load times to turn browsers into bookers. Every design choice below, including rendering strategy, URL shape, and image loading, should be evaluated against whether it helps public pages get indexed and whether it lets results render quickly on first load.

## Architecture / high-level design

The key architectural decisions for a travel booking site, including where to render, how to navigate, and how to structure the client, all flow from two realities: public pages must rank on search engines, and users compare many listings in parallel.

#### Server-side rendering (SSR) or Client-side rendering?

As with the E-commerce website system design, performance and SEO are critical. Hence, server-side rendering is a must because of the performance and SEO benefits.

HTTP request

Initial HTML + data


Proxied geocoding

Pre-generated pages

User or crawler

Server - SSR + hydration

Browser

View layer
search, listing, map, checkout

Client store
filters, results, viewport

Data access layer

Backend APIs

Geocoding provider

Popular search landing pages
/san-francisco-ca/stays


![Travel booking rendering and navigation topology](travel-booking-pdf-assets/figures/travel-booking-rendering-and-navigation-topology.png)

Default to SSR with hydration for public listing and search pages
Pure client-side rendering is a weak answer for a travel site because the pages that matter most, search results and listing details, need to be crawlable by search engines and fast on the first paint. Render the initial HTML on the server and hydrate for interactivity rather than shipping an empty shell that fills in via JavaScript.

#### Single-page application (SPA) or Multi-page application (MPA)?

Many travel sites intentionally open the listing details in a new tab/window so as to preserve convenient access to the search results page. Because new pages are often opened, the initial load performance matters more than subsequent navigation performance. The website also doesn't benefit as much from a single-page app architecture because the most commonly navigated pages cannot reuse the app shell and existing client state.

Hence, the most important thing is that SSR is being used; whether the site is an SPA or MPA doesn't matter as much, and both can be viable depending on how important the subsequent navigation experience is. In the modern web ecosystem, travel websites are one of the few products where an MPA architecture is still viable.

However, travel websites have a fair bit of interaction (e.g. changing search filters, interacting with the map, expanding accommodation details, etc.). Using UI frameworks is crucial for writing maintainable client-side code. React, Vue, and Angular are top choices for doing so.

For an SSR + interaction-heavy use case, we can leverage universal/isomorphic rendering (or SSR with hydration). In universal rendering, the server renders the full initial HTML, but after that, rendering and navigation become client-side. JavaScript event handlers are attached to the interactive elements within the HTML (hydration).

Airbnb is one of the pioneers of building universal/isomorphic apps with the invention of Rendr, a library that renders Backbone.js apps on the client and the server. Rendr is no longer used these days with the rise of React and Next.js. Next.js is the most popular choice for building universal React-powered web apps that desire server-side rendering with hydration.

What top travel booking sites use
Let's take a look at the top travel booking sites in the wild and their rendering choices:

| App Type | Rendering | UI Framework |
| --- | --- | --- |
| Airbnb | SPA (some routes) | SSR | React |
| --- | --- | --- | --- |
| Booking.com | MPA | SSR | React |
| --- | --- | --- | --- |
| Expedia | MPA | SSR | React |
| --- | --- | --- | --- |
| TripAdvisor | SPA (some routes) | SSR | React |
All the top travel booking sites in the world use SSR and React! This shows the importance of SSR for travel sites and the dominance of React in the industry. Half of these sites use MPA, probably because an SPA + SSR combination adds complexity that might not be worth dealing with.

For simplicity, and since SPAs are covered commonly in other questions, the discussion below assumes that we'll be using an MPA architecture with server-side rendering + hydration due to the need for interactivity after initial load.

## References


Rearchitecting Airbnb's Frontend | Airbnb Engineering Blog
Server Rendering, Code Splitting, and Lazy Loading with React Router v4 | Airbnb Engineering Blog
Rendr: Run your Backbone apps in the browser and Node | Airbnb Engineering Blog
Isomorphic JavaScript: The Future of Web Apps | Airbnb Engineering Blog
Breaking the Monolith — Modular redesign of Agoda.com | Agoda Engineering & Design
## Data model

| Entity | Source | Belongs to | Fields |
| --- | --- | --- | --- |
| Search Params | User input (Client) | Search/Listing Page | City/Geolocation/Radius Date Range, Number of People, Amenities, etc. |
| --- | --- | --- | --- |
| ListingResults | Server | Search/Listing Page | results (list of ListingItems), pagination (pagination metadata) |
| --- | --- | --- | --- |
| ListingItem | Server | Search/Listing Page, Details Page | title, price, currency, image_urls, amenities (flexible format) |
We have omitted the entities related to payments and addresses since they are covered in E-commerce website system design. If the interviewer wants you to focus on the checkout flow as well, do mention those entities.

Across search, map, detail, and checkout surfaces, the client store holds a compact set of related entities. Listings reference hosts and locations, and a booking ties together a user, a listing, and a date range.


owned by

located at


reserved as


imageUrls


isSuperhost


listingId


bookedDates


minNights


listingId


userId


checkIn


checkOut


totalPrice


Client-side entity relationships for listings and bookings
## Interface definition (API)

We will need the following HTTP APIs:

Search: Fetch accommodation listings given some search/filter parameters. The results are rendered on the map and/or in a list view.
Listing details: Fetch the details for an accommodation listing.
Reserve: Make a reservation for an accommodation.
Search accommodations
| Property | Value |
| --- | --- |
| HTTP Method | GET |
| --- | --- |
| Path | /search |
| --- | --- |
| Description | Returns a list of accommodations that match the search query. |
Parameters
| Parameter | Type | Description |
| --- | --- | --- |
| Size | number | Number of results per page |
| --- | --- | --- |
| Page | number | Page number to fetch |
| --- | --- | --- |
| Guests | number | Number of staying guests |
| --- | --- | --- |
| Country | string | Country for the user, determines the currency |
| --- | --- | --- |
| Location | mixed (see below) | Location of the search |
| --- | --- | --- |
| Date range | mixed (see below) | Date range to occupy the accommodation |
| --- | --- | --- |
| Amenities | mixed (see below) | Amenities criteria |
Location: The exact format of the location field depends on the business requirements, but in general, we need to tell the server the desired boundary so that results within the boundary will be shown. There are two common ways to represent boundaries: (1) center position + radius, and (2) boundary coordinates.

Center position + radius: The radius can be in miles/km and the center position can be either of:
Free text search: This is any string containing a location, e.g. "San Francisco", "New York City". An external location service will be needed to convert the string into geographic coordinates (via geocoding). Geocoding is preferably done on the server to prevent API abuse.
Geolocation/Coordinates: This format is useful for map-based UIs where users can pan/zoom the map to search for listings within the area, or when users allow the app to access their current location to search around them.
Do not call the geocoding provider directly from the client
Shipping the geocoding API key to the browser exposes it to abuse and quota exhaustion, and makes it trivial for third parties to scrape the provider on your bill. Proxy geocoding through your own server so the key stays private and the service can be rate limited and cached per user.

Boundary coordinates: For map-based UIs, we can use the coordinates of the 4 corners of the presented map as the boundary area.
It's likely that the search API will have to support both formats if the website has different search UIs:

Landing pages for travel sites usually have a search bar with compulsory parameters like date range and number of guests. Such a UI will need the free text search location format.
Results pages with maps will use the geolocation/boundary coordinates format.
Date range: There are several date range formats to choose from, and none are clear winners:

| Format | Example | Pros | Cons |
| --- | --- | --- | --- |
| Array/tuple | ['2022-12-24', '2022-12-27'] | Short | Unclear. Has to be encoded |
| --- | --- | --- | --- |
| Object | { check_in: '2022-12-24', check_out: '2022-12-27' } | Clear | Long. Has to be encoded |
| --- | --- | --- | --- |
| Separate query parameters | check_in and check_out | Don't need encoding | Two query params vs one |
The array format is the least clear of the three, and the object requires encoding to be used in GET, so separate query parameters would be the most recommended.

Amenities: Structuring the query parameters for amenities has the same issues as the date range.

| Format | Example | Pros | Cons |
| --- | --- | --- | --- |
| Object | { breakfast: true, washer: true } | Clear | Long. Has to be encoded |
| --- | --- | --- | --- |
| Separate query parameters | breakfast and washer | More readable | N query params vs One |
In this case, due to the sheer number of query parameters if each amenity criterion is a separate query parameter, the object format is better because:

Putting each criterion in its own query parameter makes the query string really long, and the hierarchy is lost.
There's a possibility of query parameter name collisions with the non-amenities fields. This can be solved by prefixing all the amenity fields with amenities_ (e.g. amenities_breakfast and amenities_washer).
Moreover, some amenities criteria have non-primitive values, which will require URL encoding and decoding anyway.
There's no fixed standard on how to encode arrays and objects into the query string of URLs, and the format tends to be implementation-dependent. The most important thing is to maintain a consistent format between the server and the client.

Sample response

{
// Pagination metadata.
"pagination": {
"size": 5,
"page": 2,
"total_pages": 15,
"total": 74
},
"results": [
{
"id": 561602, // Accommodation ID.
"title": "Great view in the Mission, 15 mins by bus downtown",
"images": [
"https://www.greathotels.com/img/1.jpg",
"https://www.greathotels.com/img/2.jpg",
"https://www.greathotels.com/img/3.jpg",
"https://www.greathotels.com/img/4.jpg"
],
"rating": 4.82,
"coordinates": {
"latitude": 37.74403,
"longitude": -122.41755
},
"price": 200,
"currency": "USD"
}
// ... More accommodation results.
]
}
We use offset-based pagination here as opposed to cursor-based pagination because:

Having page numbers is useful for navigating between search results and jumping to specific pages.
Accommodation results do not suffer from the stale results issue that much because new listings are not added that quickly/frequently.
It's useful to know how many total results there are.
For a more in-depth comparison between offset-based pagination and cursor-based pagination, refer to the News Feed system design article.

If infinite scroll is desired, then a cursor-based pagination approach might be required. Offset-based pagination can still be used, but the client will need to go through the trouble of filtering out duplicate results.

On a typical search-with-map surface, filter changes, map panning, and free-text input all converge on the same /search call, with geocoding proxied through the server and the map + list views re-rendered from the same result set.

Geocoding provider
View (filters + map + list)
Geocoding provider
View (filters + map + list)
[Free-text location]
[Map viewport]
Update filters / viewport in store
Geocode (server-side, API key hidden)
lat/lng
{ results, pagination }
Render markers on map + list items
Hover list item
Highlight corresponding map marker


![Search with map and filters flow](travel-booking-pdf-assets/figures/search-with-map-and-filters-flow.png)

Default to offset pagination for travel search, cursor only if infinite scroll
Accommodation inventory churns slowly and users benefit from page numbers, total counts, and the ability to deep-link to a specific page of results. Switch to cursor-based pagination only when the UI is a continuous infinite scroll that must tolerate list mutations without showing duplicates.

Fetch accommodation details
Strictly speaking, if the accommodation details page is always opened in a new tab, then the details data will only ever need to be fetched on the server to create the initial HTML; it will not be fetched from the client, so a standalone HTTP API is not needed.

| Property | Value |
| --- | --- |
| HTTP Method | GET |
| --- | --- |
| Path | /accommodation/{accommodationId} |
| --- | --- |
| Description | Fetches the details of an accommodation listing. |
Parameters
| Parameter | Type | Description |
| --- | --- | --- |
| accommodationId | number | ID of accommodation to be fetched |
| --- | --- | --- |
| country | string | Country for the user, determines the currency |
Sample response

{
"id": 561602, // Accommodation ID.
"title": "Great view of Brannan Street, 15 mins by bus downtown. Bed and Breakfast provided!",
"images": [
"https://www.greathotels.com/img/1.jpg",
"https://www.greathotels.com/img/2.jpg",
"https://www.greathotels.com/img/3.jpg",
"https://www.greathotels.com/img/4.jpg"
],
"rating": 4.82,
"coordinates": {
"latitude": 37.74403,
"longitude": -122.41755
},
"price": 200,
"currency": "USD",
"amenities": {
"breakfast_provided": true,
"internet": true,
"washer": true,
"dryer": false
// Any additional amenities details.
},
"house_rules": "...",
"contact_email": "..."
// Any additional details.
}
Make reservation
| Property | Value |
| --- | --- |
| HTTP Method | POST |
| --- | --- |
| Path | /reserve |
| --- | --- |
| Description | Reserve an accommodation. |
Parameters
| Parameter | Type | Description |
| --- | --- | --- |
| accommodation_id | number | ID of accommodation to be reserved |
| --- | --- | --- |
| dates | mixed* | Dates to reserve the accommodation |
| --- | --- | --- |
| payment_details | object | Object containing payment method fields (credit card) |
* The format of the dates field should be consistent with the selected date range format in the search API.

The reservation flow spans availability checks, payment, and server confirmation. The client re-validates availability before sending payment details so the user is not charged for dates that were just booked by someone else.

Payment processor
Client (checkout view + store)
Payment processor
Client (checkout view + store)
[Dates still available]
[Dates no longer available]
Pick dates and confirm booking
Re-validate availability for selected dates
{ availableDates, price }
Submit reservation + payment details
Authorize card
Auth result
{ reservation } on success
Store reservation, navigate to confirmation
Conflict
Show "no longer available", prompt new dates


Reservation flow with availability re-check and payment
A reservation moves through a small number of states between inquiry and completion, and the client UI (confirmation page, trip dashboard, cancellation controls) is driven by this status field.

Submit booking

Payment authorized

Payment failed or abandoned

Guest or host cancels

Check-out date passes

Inquiry

PendingPayment

Confirmed

Cancelled

Completed


Reservation lifecycle states
Sample response
A reservation object is returned upon successful reservation.


{
"id": 456, // Reservation ID.
"total_price": 400,
"currency": "USD",
"dates": {
"check_in": "2022-12-24",
"check_out": "2022-12-27"
},
"accommodation": {
"id": 561602,
"address": {
"country": "US",
"street_address": "888 Brannan Street",
"city": "San Francisco",
"zip": "94103",
"state": "CA",
// ... Other address fields.
},
}
"payment_details": {
// Only show the last 4 digits.
// We shouldn't be storing the credit card number
// unencrypted anyway.
"card_last_four_digits": "1234"
}
}
## Optimizations and deep dive

To recap, the most important aspects of a travel booking website are search discovery, global usage, performance, device support, booking UX, accessibility, and forms:

Search engine optimization (SEO): This section explains how search and listing pages stay crawlable, shareable, and useful for organic discovery.
Internationalization (i18n): This section covers locale, language, currency, dates, addresses, and region-specific travel expectations.
Performance: This section discusses loading speed, image delivery, Core Web Vitals, and performance tradeoffs that affect conversion.
Device support: This section covers responsive images, mobile layouts, and cross-device booking behavior.
User experience: This section explains search context preservation, date-range selection, optimistic reservation flows, and high-friction travel UX details.
Accessibility: This section covers image alternatives, forms, controls, and keyboard access across search and booking flows.
Form optimizations: This section discusses checkout-style form improvements for guest details, payment, validation, and autofill.
Other topics: This section lists adjacent travel-product concerns that are useful follow-up discussion points.
Search engine optimization (SEO)
Travel sites rely heavily on organic traffic, so search result pages and listing pages both need to be crawlable and shareable, starting with making search URLs bookmarkable.

Bookmarkable search results
When searching on these travel sites, notice that the search query and filters are reflected in the URL. If you load the URL in a new window, the same results and search query are shown. Syncing the search query with the URL is an important feature because:

Bookmarking: Allows the particular search to be bookmarked, which is a common action users take when doing research for travel.
Deep linking: Other sites like travel blogs can link to the results page of specific search filters (e.g. vacation rentals in San Francisco). This helps improve the site's SEO and discoverability.
Treat the URL as the source of truth for search filters
Location, dates, guests, and amenity filters all belong in the query string or path so the same state can be bookmarked, shared, indexed, and restored on reload. Keeping filters only in client memory breaks back/forward navigation, loses deep-link SEO value, and forces users to re-enter their criteria whenever they open a listing in a new tab.

Pre-generate pages for popular searches
When you search for "Vacation in San Francisco" in a search engine, you will most likely see results from popular travel sites like Expedia, Airbnb, and TripAdvisor, among many others. These pages aren't content articles; they're the site's search results page with some pre-filled filters. How is this accomplished?

To improve SEO by appearing relevant to what users are searching for, travel sites generate tons of pages for popular search terms that show relevant listings for these terms. For example, Airbnb has generated dedicated pages for "Vacation rental apartments in San Francisco" and "Vacation rental apartments in New York". To let search engines know about these dedicated pages, Airbnb has a sitemap page just for listing links to their dedicated listing pages. There are thousands or more listing pages.

URLs are commonly implemented in one of two ways:

Readable URLs: https://www.airbnb.com/san-francisco-ca/stays
"Ugly" URLs: https://www.airbnb.com/search?location=123 (where 123 is an internal ID mapping to San Francisco city)
Airbnb uses readable URLs because they help with SEO rankings, especially when the URL contains the search term itself. Using this technique, search engines will think that travel sites have many different content pages when in reality they are all rendering the same results page but with different listings. It's great for SEO.

Prefer readable, keyword-bearing URLs for popular search landing pages
A path like /san-francisco-ca/stays ranks and clicks through better than /search?location=123 because the target keyword is visible in the URL itself. Reserve opaque query-parameter URLs for internal or rarely-indexed routes where SEO does not matter.

Dynamic rendering – serve crawlers different pages
Sites can also leverage dynamic rendering. Dynamic rendering involves using a web server to differentiate between requests made by crawlers and those made by users. When a request is made by a crawler, it is routed to a renderer that generates a version of the content that is optimized for the crawler, such as a static HTML version. Requests made by real users are handled as normal.

Dynamic rendering can be done in a different manner as well. For Expedia Group's Vrbo, crawlers are served a full page while real users are served only a lightweight page with content above the fold; the rest of the content is loaded asynchronously on the client.

### Internationalization (i18n)

Some of the following is extracted from our Quiz Questions on internationalization:

Have dedicated pages translated in the supported languages.
Make the language and country selector very prominent (e.g. in the navbar).
For listing details that are contributed by users, add an automatic translation feature so that users visiting country X who don't speak country X's language can understand the listing in their native language.
Set the lang attribute on the html tag (e.g. <html lang="zh-cn">) to tell browsers and search engines the language of the page, which helps browsers offer a translation of the page. This is also important for SEO.
Enable locale-specific UI:
Using the :lang() CSS pseudo-class to change display
Right-to-left languages
Using CSS Logical Properties
HTML's dir attribute
CSS direction: rtl
Consider differences in the length of text in different languages.
Do not concatenate translated strings.
Do not put text in images.
Be mindful of how colors are perceived in different cultures.
### Performance

Performance is known to affect conversions, so for websites where the aim is to get customers to make bookings, performance is important. Performance optimizations have been covered in great detail in earlier questions, so in this question we will list the relevant performance optimizations along with blog articles by companies should you want more details.

Image optimizations
Image carousel: Photos are widely used on travel websites to showcase how attractive the destination/accommodation is. We have covered the implementation of image carousels in the Image Carousel system design article.
Image preloading/lazy loading: One super useful technique is to use JavaScript to have fine-grained control over when images load.
Airbnb optimized the image carousel experience in their room listings with a combination of lazy loading and preloading:
Initially, only the first image is loaded (the remaining images will be lazily loaded).
The second image is preloaded when the user shows possible intent of viewing more images:
Cursor hovers over the image carousel.
Focuses on the "Next" button via tabbing.
Image carousel comes into view (on mobile devices).
If the user does view the second image (which signals high intent to browse even more images), the next three images (3rd to 5th) are preloaded.
As the user clicks "Next" again to browse more images, the (n + 3)th image is preloaded.
Airbnb Image Carousel Lazy Loading
Airbnb image carousel lazy loading example on mobile
Responsive images: Serving the most suitable image for the device.
Images on the Web: Part 1 — Responsive Images | Expedia Group Technology
Images on the Web: Part 2 — Implementing responsive images | Expedia Group Technology
Image formats: Use webp format for photos and SVGs for icons where possible.
Choose the right image format| web.dev
Code splitting
Expedia's Vrbo prioritizes above-the-fold content and loads code for it first.
Page/route-level code splitting to prevent huge JavaScript bundles if using a single-page app architecture.
Lazy load UI components that are not shown on initial render: (1) below-the-fold elements (e.g. footer), and (2) elements that only appear after interaction (e.g. popover, modal).
### Performance monitoring

Use tools like Lighthouse and Web Vitals to profile websites and measure performance.
Airbnb came up with their own Page Performance Score and defined the metrics that mattered for web performance.
Set up performance budgets that run on CI.
React-specific tips
React-specific performance optimizations done by Airbnb: React Performance Fixes on Airbnb Listing Pages | The Airbnb Tech Blog.
12 Tips to Improve Client Side Page Performance | Expedia Group Technology
Bundling optimizations
Module federation: Using Webpack Module Federation to Create an App Shell | Expedia Group Technology
Optimizing a Page: Resource Hints, Critical CSS, and Webpack | Expedia Group Technology
Virtual list/windowing for long lists with infinite scrolling
Use virtual list/windowing for long lists with infinite scrolling. Read more about List Virtualization on patterns.dev.

Progressive web apps
Progressive Web Apps with Service Workers | Booking.com Engineering
Other performance tips
When using client-side rendering, split up the listing response into multiple payloads so as to show results faster. To quickly show results, the response payload can be split into a few chunks, e.g. returning the first 5 results (or however many are above the fold), then loading the rest of the results on the page after that.
Web Applications: Analyzing Client-Side Performance | Expedia Group Technology
Go Fast or Go Home: The Process of Optimizing for Client Performance
Further reading
#### What kind of things must you be wary of when designing or developing for multilingual sites?

#### How do you serve a page with content in multiple languages?

Building Airbnb's Internationalization Platform
Adding support for Arabic and Hebrew languages on Airbnb
Device support
Responsive images: Serving the most suitable image for the device, as mentioned in the image optimization section above.
Images on the Web: Part 1 — Responsive Images | Expedia Group Technology
Images on the Web: Part 2 — Implementing responsive images | Expedia Group Technology
Device-specific UI: Showing different UI for different devices, which can involve using a totally different information architecture.
Dynamic number of result items in a row and per page depending on device width and height.
Since maps are hard to use on smaller devices due to the limited screen real estate, consider not having a map UI at all. Without the map, clients can avoid loading map code and textures entirely on mobile devices.
Airbnb's listings are more app-like on mobile, with a floating bottom bar. The listing page also does not open listings on a new page, presumably because it is harder to manage many open tabs on mobile.
Support swiping on image carousels on mobile.
A less aggressive preloading strategy on mobile since mobile data can be more expensive.
Interactive elements should be larger on mobile.
### User experience

A few travel-specific UX details matter disproportionately: preserving the user's search context when they dive into a listing, date-range pickers, and optimistic reservation flows.

Preserving search context
For most travel sites, clicking on a listing will open the listing details in a new tab. The reason for doing so is that users tend to browse multiple listings after narrowing down the search results, and opening the listing details on a new page allows users to easily dive into the details of a listing and go back to where they left off if the listing didn't meet their expectations or if they want to see other listings.

The alternative is to open the listing details within a full-screen modal on the same page and shallowly update the URL (via History.pushState()), similar to Zillow.com and Instagram.com. Dismissing the modal will reveal the search results behind it, and users can continue browsing the results. The downside of this approach is that it is more complicated to implement and will require more JavaScript on the client since everything is on one single page and the listing details need to be fetched via AJAX.

Default to opening listing details in a new tab on desktop
Travel shoppers compare many listings in parallel, and a new tab keeps the scroll position, filters, and map viewport of the results page intact while they dig into candidates. The shallow-modal alternative is only worth the extra client complexity when the product is committed to a single-page flow, and most travel sites fall back to full-page navigation on mobile where tab management is harder anyway.

### Accessibility

Image alt tags: If there are no descriptions provided for images uploaded by users, it's OK to have empty alt tags.
The Expedia Accessibility Guidelines cover topics like Color, Controls, Keyboard and Input Modalities, Forms, Images, Content Structure, Reading Order, Navigation Order, etc.
Form optimizations
Form optimizations have been covered in detail in E-commerce website system design. To recap:

Country-specific address/payment forms.
Optimize the autofill experience.
Show error messages.
Clear focus states.
Read more about building good Payment forms and general form best practices on web.dev.
Other topics
Map
Markers that are in close proximity can be clustered together.
Whitelabelling
Building and scaling different travel websites with one codebase | Agoda Engineering & Design
Managing and scaling different white label development and testing environments | Agoda Engineering & Design
Summary
A travel booking product like Airbnb has to be evaluated under two lenses that pull in different directions: an SEO lens that governs the crawlable, shareable public surfaces, and a conversion lens that governs the signed-in, transactional flow from search through reservation. Travel is unusual in that the same product has to win organic traffic on long-tail city and neighborhood queries and then move a logged-in guest through dates, guests, price, and payment without losing them. A design that optimizes only one lens either leaks acquisition or loses revenue at the checkout step, so the interview answer has to hold both simultaneously.

Through the SEO lens. Public search results and listing detail pages default to server-side rendering with hydration so crawlers see complete HTML, the first contentful paint arrives before hydration, and subsequent interactions run on the client. Popular queries are served from pre-generated landing pages under readable, keyword-bearing URLs like /san-francisco-ca/stays, which are surfaced to search engines through sitemap.xml and updated on a rebuild cadence tied to inventory freshness. Dynamic rendering sits behind user-agent detection as a crawler escape hatch for routes where client-side execution would otherwise block indexing, without regressing human-visible performance. The URL is treated as the source of truth for search filters, including location, check_in/check_out date range, guest count, and amenities, so any filtered result set is bookmarkable, shareable, and crawlable rather than hidden behind client-only state. Canonical tags, locale-aware hreflang, and lang/dir attributes round out the discoverability story across translated markets.

Through the conversion lens. The /search endpoint accepts either free-text location resolved through a server-side geocoding proxy or explicit bounding-box coordinates, along with separate check_in and check_out parameters rather than a single range string so date validation and timezone handling stay explicit. The client-side data model keeps Listing referencing Host and Location, binding to an AvailabilityCalendar, and participating in Booking records so the reservation lifecycle can be reasoned about without refetching. POST /reserve re-validates availability against the calendar immediately before charging, which prevents the failure mode where a guest is billed for dates another user just confirmed. Airbnb-style image carousels load the first image eagerly, preload the next on hover or focus intent, and lazily request the remainder, keeping the gallery feeling instant without blowing the image budget. Map and list stay synchronized through a shared client store of results, with marker clustering at low zoom levels and viewport-driven /search calls as the user pans; i18n, RTL, and currency handling follow locale, and device-specific UI removes the map on small screens, swaps in a floating bottom action bar, and enlarges touch targets for thumb reach.

Where the two lenses agree is the URL-backed client store that fronts /search, /accommodation/:id, and /reserve: the same encoded state that lets a crawler index a filtered result page also lets a guest share, bookmark, or reload mid-booking without losing context. Rendering, API shape, and data model line up behind that shared contract, which is what makes the design hold up under both SEO and conversion pressure.

References
The references below group sources by topic, so it is easier to map each source back to the part of the article it informed.

Airbnb frontend architecture and performance
These references inform the rendering strategy, code-splitting, and client performance choices discussed throughout the architecture and optimizations sections.

Rearchitecting Airbnb's frontend for the overall frontend platform direction behind airbnb.com and the tradeoffs that shaped its client architecture.
Building a faster web experience with the postTask scheduler for scheduling long-running work off the main thread on interactive booking pages.
Server rendering, code splitting, and lazy loading with React Router v4 for the SSR and route-based code-splitting approach relevant to listing and search pages.
Isomorphic JavaScript: the future of web apps for the isomorphic rendering background used in the rendering approach discussion.
Creating Airbnb's page performance score for how a large travel site turns performance into a shared internal metric.
Measuring web performance at Airbnb for the RUM and lab measurement practices behind the performance work referenced in the article.
React performance fixes on Airbnb listing pages for concrete listing-page performance wins that map directly onto the listing detail design here.
### Internationalization and localization

These references inform the i18n, currency, and locale considerations called out for a global travel product.

Building Airbnb's internationalization platform for the platform-level approach to translations, message management, and locale routing.
Adding support for Arabic and Hebrew languages on Airbnb for the RTL layout, typography, and bidirectional text considerations relevant to global travel sites.
Expedia and other travel-site case studies
These references provide cross-company context for images, bundling, and resource prioritization on travel sites with a similar listing-heavy shape.

Images on the web: Part 1 — responsive images for the responsive image fundamentals behind listing media.
Images on the web: Part 2 — implementing responsive images for the implementation details of srcset, sizes, and art direction on large travel pages.
Expedia's Vrbo prioritizes above-the-fold contents for concrete above-the-fold prioritization techniques on a travel homepage.
12 tips to improve client side page performance for a practical checklist of client performance fixes applicable to listing and search pages.
Using Webpack module federation to create an app shell for a micro-frontend style app-shell architecture on a multi-team travel codebase.
Optimizing a page: resource hints, critical CSS, and Webpack for resource hints and critical CSS techniques referenced in the loading discussion.
Progressive web apps with service workers | Booking.com for a booking-focused view on service workers, offline behavior, and caching.
Breaking the monolith — modular redesign of Agoda.com for how another travel site evolved from a monolithic frontend into modular surfaces.
Yes I'm lazy | TripAdvisor Engineering for lazy-loading patterns on image-heavy travel listing pages.
Form best practices for booking flows
These references back the payment, address, and autofill guidance in the form optimizations section.

Payment and address form best practices | web.dev for end-to-end guidance on payment and address fields in a checkout-style flow.
Address forms | web.dev for the address-specific field structure and locale-aware layout recommendations.
Autofill on browsers: a deep dive | eBay for the browser autofill behavior and attribute choices that directly affect booking form completion rates.
Related system design articles
The articles below dive deeper into mechanics that are touched on but not fully explored here.

E-commerce (Amazon) for the broader checkout, form, and catalog patterns that a booking flow reuses and adapts.
Photo Sharing (Instagram) for the image loading, lazy loading, and media-heavy feed techniques that apply to listing galleries.