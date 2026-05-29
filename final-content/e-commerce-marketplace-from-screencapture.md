---
title: "E-commerce Marketplace (e.g. Amazon)"
slug: "e-commerce-marketplace"
difficulty: "medium"
duration_mins: 40
tags:
  - system-design
  - seo
premium: true
summary: "Design an e-commerce marketplace website like Amazon and eBay."
free_preview: false
status: draft
---

# E-commerce Marketplace (e.g. Amazon)

## Question

Design an e-commerce website that allows users to browse products and purchase them.

Product Listing Page Example

Product Details Page Example

Cart Page Example

Checkout Page Example

## Real-life examples

- https://www.amazon.com
- https://www.ebay.com
- https://www.walmart.com
- https://www.flipkart.com
## Requirements exploration

Before designing the storefront, we scope the exercise by clarifying which user journeys are in play, which devices to target, and which commerce concerns (payments, localization, accessibility) fall inside the interview.

#### What are the core features to be supported?

Browsing products.
Adding products to cart.
Checking out successfully.
#### What are the pages in the website?

Product listing page (PLP)
Product details page (PDP)
Cart page
Checkout page
#### What product details will be shown on the PLPs and PDPs?

PLPs: Product name, Product image, Price.
PDPs: Product name, Product images (multiple), Product description, Price.
#### What do the user demographics look like?

International users of a wide age range: US, Asia, Europe, etc.

#### What are the non-functional requirements?

Each page should load under 2 seconds. Interactions with page elements should respond quickly.

#### What devices will the application be used on?

All possible devices: laptop, tablets, mobile, etc.

#### Do users have to be signed in to purchase?

Users can purchase as a guest without being signed in.

## Architecture / high-level design

E-commerce Website Architecture Diagram

Component responsibilities
Server: Provides HTTP APIs to fetch product data and cart items, modify the cart, and create orders.
Controller: Controls the flow of data within the application and makes network requests to the server.
Client Store: Stores data needed across the whole application. Since there are many pages with some amount of overlapping data, a client store is useful for sharing data across sections of a page and across pages.
Pages:
Product List: Displays a list of product items that can be added to the cart.
Product Details: Displays details of a single product along with additional details.
Cart: Displays added cart items and allows changing of quantity and deleting added items.
Checkout: Displays address and payment forms the user has to complete in order to place the order.
#### Server-side rendering or client-side rendering?

Firstly, let's understand what the two terms mean:

Server-side rendering (SSR): The traditional way of building websites where the server fetches all necessary data, uses it to create the final markup, and sends down the HTML needed every time a user visits a page. Most of the rendering work is done on the server.
Client-side rendering (CSR): The server sends down initial HTML which contains the JavaScript to bootstrap the application. The client then fetches the necessary data, combines it with templates, and creates the final page, all within the browser. CSR is typically used with Single-page Application models, and subsequent navigations don't require a full page refresh. Most of the rendering work is done on the client.
Benefits of SSR:

Performance is generally better, First Contentful Paint score is high, and SSR-ed pages appear faster than CSR.
Lower Cumulative Layout Shift score as the final HTML is already present.
SSR allows for personalization of pages (user-specific content) as opposed to static site generation. Personalization is an important factor for conversion as e-commerce platforms scale up.
Downsides of SSR:

Page transitions are slower because the entire page has to be constructed on the server for every request.
SEO is important for e-commerce websites, hence SSR should be a priority.

Default to SSR for PLPs and PDPs, not CSR
Unlike a personalized feed, product discovery on e-commerce sites is driven heavily by organic search, and shoppers on slow networks or mid-range phones bail when first paint is slow. Server-rendering PLPs and PDPs gets real content into the HTML for crawlers and cuts time-to-product-visible, both of which directly influence conversion.

#### Single-page application (SPA) or Multi-page application (MPA)?

SPAs by default use CSR and MPAs by default use SSR. While CSR and SSR are two ends of the rendering spectrum, somewhere in the middle lies a hybrid mode called universal rendering (or SSR with hydration) where the server renders the full HTML, but after that, rendering and navigation become client-side.

Most universally rendered sites use popular UI frameworks (e.g. React, Vue, and Angular), and the page will need to be hydrated (add event handlers) after initial load. Hydration also brings about the double data problem.

Which application architecture should be used? The most important factor is that SSR is being used; whether SPA or MPA doesn't matter as much. As seen above, both are viable, as long as your website has good performance.

Real-world technologies to implement the following rendering and architecture choices:

| SSR | CSR |
| --- | --- |
| SPA | Next.js, Remix, Nuxt | Create React App |
| --- | --- | --- |
| MPA | Ruby on Rails, Django* | This combination isn't practical |
* Refers to the default mode of server-side web frameworks, which render HTML, and routing is done on the server side.

What top e-commerce sites use
Let's take a look at e-commerce sites in the wild and their rendering choices:

| Architecture | Rendering | UI Framework |
| --- | --- | --- |
| Amazon | MPA | SSR | In-house |
| --- | --- | --- | --- |
| eBay | MPA | SSR | Marko |
| --- | --- | --- | --- |
| Walmart | SPA | SSR | React |
| --- | --- | --- | --- |
| Flipkart | SPA | SSR | React |
All these e-commerce sites use SSR! This suggests the importance of using SSR for e-commerce sites.

The discussion below assumes that we'll be using a universal rendering approach of SSR + SPA. Concretely, this means the catalog surfaces (PLP, PDP) prioritize SSR for SEO and first paint, while interactive transactional surfaces (Cart, Checkout) lean more on the client after hydration.

Browser after hydration

Request


Catalog data

Cart / order data

Initial HTML

Initial HTML

SPA nav

Add to cart

Proceed

Shopper

CDN / edge cache

SSR server

Product service

Cart / order service

PLP - SSR + hydrate

PDP - SSR + hydrate

Cart - CSR after hydrate

Checkout - CSR after hydrate

Client store - cart, user, session


Page-type rendering strategy across the storefront
## Data model

There are quite a number of entities involved in an e-commerce website due to the complexity of the user flow spanning multiple pages.

| Entity | Source | Belongs To | Fields |
| --- | --- | --- | --- |
| ProductList | Server | Product Listing Page | products (list of Products), pagination (pagination metadata) |
| --- | --- | --- | --- |
| Product | Server | Product Listing Page, Product Details Page | name, description, unit_price, currency, primary_image, image_urls |
| --- | --- | --- | --- |
| Cart | Server | Client Store | items (list of CartItems), total_price, currency |
| --- | --- | --- | --- |
| CartItem | Server | Client Store | quantity, product, price, currency |
| --- | --- | --- | --- |
| AddressDetails | User input (Client) | Checkout Page | name, country, street, city, etc. |
| --- | --- | --- | --- |
| PaymentDetails | User input (Client) | Checkout Page | card_number, card_expiry, card_cvv |
Few points for discussion:

The Cart entity belongs to the Client Store because some websites might want to show the number of cart items in the navbar or have a popup that allows users to quickly access the cart items and make modifications. If there's no such need, then it's acceptable for the cart to belong to the Cart page.
Cart and CartItem have total_price and price fields respectively fetched as part of the server response, instead of having the client compute the price (multiply the quantity by the unit_price), to give the flexibility of applying discounts due to bulk purchases or use of promo codes. The price computation logic is defined on the server, so the final price should be computed on the server, and the client should rely on the server to calculate the total price instead of making its own calculations.
Since our website has an international audience, we should have localized prices in the user's currency, hence the currency field.
The diagram below summarizes how these entities relate once the shopper moves from browsing a product to placing an order. Product is referenced by both CartItem and the snapshot stored on the Order, while AddressDetails and PaymentDetails are captured at checkout time and attached to the order record.


converts to


ships to

paid with


number[]

item_ids


total_price


product_id


primary_image


image_urls


unit_price


total_price


card_last_four_digits


Core e-commerce entities and their relationships
Treat the server as the source of truth for cart totals and prices
Client-side multiplication of quantity times unit_price silently breaks once bulk discounts, promo codes, tax, or regional pricing enter the picture. Always render the price and total_price returned by the server, and re-fetch the cart after any mutation so the user never sees a stale or inconsistent total at checkout.

## Interface definition (API)

We need the following HTTP APIs:

Product Information
Fetch products listing
Fetch a particular product's detail
Cart Modification
Add a product to the cart
Change quantity of a product in the cart
Remove a product from the cart
Complete the order
We will omit discussion of the APIs between the client components because the data format and functionalities are similar to the HTTP APIs.

We can also assume that a user only has a maximum of one cart and the user's current cart can be retrieved on the server. Hence, we can omit passing the cart ID as an argument for any APIs related to cart modification.

Fetch products listing
| Field | Value |
| --- | --- |
| HTTP Method | GET |
| --- | --- |
| Path | /products |
| --- | --- |
| Description | Fetches a list of products. |
Parameters
| Parameter | Type | Description |
| --- | --- | --- |
| size | number | Number of results per page |
| --- | --- | --- |
| page | number | Page number to fetch |
| --- | --- | --- |
| country | string | Country for the user, determines the currency |
Sample response

{
"pagination": {
"size": 5,
"page": 2,
"total_pages": 4,
"total": 20
},
"results": [
{
"id": 123, // Product ID.
"name": "Cotton T-shirt",
"primary_image": "https://www.greatcdn.com/img/t-shirt.jpg",
"unit_price": 12,
"currency": "USD"
}
// ... More products.
]
}
We use offset-based pagination here as opposed to cursor-based pagination because:

Having page numbers is useful for navigating between search results and jumping to specific pages.
Product results do not suffer from the stale results issue that much, as new products are not added that quickly/frequently.
It's useful to know how many total results there are.
For a more in-depth comparison between offset-based pagination and cursor-based pagination, refer to the News Feed system design article.

Fetch product details
| Field | Value |
| --- | --- |
| HTTP Method | GET |
| --- | --- |
| Path | /products/{productId} |
| --- | --- |
| Description | Fetches the details of a product. |
Parameters
| Parameter | Type | Description |
| --- | --- | --- |
| productId | number | ID of product to be fetched |
| --- | --- | --- |
| country | string | Country for the user, determines the currency |
Sample response

{
"id": 123, // Product ID.
"name": "Cotton T-shirt",
"primary_image": "https://www.greatcdn.com/img/t-shirt.jpg",
"image_urls": [
"https://www.greatcdn.com/img/t-shirt.jpg",
"https://www.greatcdn.com/img/t-shirt-black.jpg",
"https://www.greatcdn.com/img/t-shirt-red.jpg"
],
"unit_price": 12,
"currency": "USD"
}
Add a product to cart
| Field | Value |
| --- | --- |
| HTTP Method | POST |
| --- | --- |
| Path | /cart/products |
| --- | --- |
| Description | Add a product to the cart |
Parameters
| Parameter | Type | Description |
| --- | --- | --- |
| productId | number | ID of product to be added |
| --- | --- | --- |
| quantity | number | Number of items to be added |
Sample response
The updated cart object is returned.


{
"id": 789, // Cart ID.
"total_price": 24,
"currency": "USD",
"items": [
{
"quantity": 2,
"price": 24,
"currency": "USD",
"product": {
"id": 123, // Product ID.
"name": "Cotton T-shirt",
"primary_image": "https://www.greatcdn.com/img/t-shirt.jpg"
}
}
]
}
On the client, add-to-cart benefits from an optimistic update so the cart badge and drawer feel instant, but the server still owns the authoritative totals once discounts and tax kick in. The flow below shows the client applying a temporary update, hitting POST /cart/products, and then reconciling against the canonical cart returned by the server.

Client (store + view)
User (PDP)
Client (store + view)
User (PDP)
[Success]
[Failure]
Click "Add to cart"
Optimistic: bump badge count, drawer item
{ productId, quantity }
Updated cart { total_price, items[] }
Replace cart from server (authoritative totals)
Roll back optimistic update, show retry toast


Add-to-cart flow with optimistic update and server reconciliation
Change quantity of product in cart
| Field | Value |
| --- | --- |
| HTTP Method | PUT |
| --- | --- |
| Path | /cart/products/{productId}/ |
| --- | --- |
| Description | Change quantity of a product in the cart |
Parameters
| Parameter | Type | Description |
| --- | --- | --- |
| productId | number | ID of product to be modified |
| --- | --- | --- |
| quantity | number | New quantity of the product |
Sample response
The updated cart object is returned.


{
"id": 789, // Cart ID.
"total_price": 24,
"currency": "USD",
"items": [
{
"quantity": 3,
"price": 36,
"currency": "USD",
"product": {
"id": 123, // Product ID.
"name": "Cotton T-shirt",
"primary_image": "https://www.greatcdn.com/img/t-shirt.jpg"
}
}
]
}
Remove product from cart
| Field | Value |
| --- | --- |
| HTTP Method | DELETE |
| --- | --- |
| Path | /cart/products/{productId} |
| --- | --- |
| Description | Remove a product from the cart |
Parameters
| Parameter | Type | Description |
| --- | --- | --- |
| productId | number | ID of product to be removed |
Sample response
The updated cart object is returned.


{
"id": 789, // Cart ID.
"total_price": 0,
"currency": "USD",
"items": []
}
Place order
| Field | Value |
| --- | --- |
| HTTP Method | POST |
| --- | --- |
| Path | /order |
| --- | --- |
| Description | Create an order from a cart |
Parameters
| Parameter | Type | Description |
| --- | --- | --- |
| cartID | number | ID of the cart containing the items |
| --- | --- | --- |
| address_details | object | Object containing address fields |
| --- | --- | --- |
| payment_details | object | Object containing payment method fields (credit card) |
Sample response
The order object is returned upon successful order creation.


{
"id": 456, // Order ID.
"total_price": 36,
"currency": "USD",
"items": [
// ... Same items as per the cart.
],
"address_details": {
"name": "John Doe",
"country": "US",
"address": "1600 Market Street",
"city": "San Francisco"
// ... Other address fields.
},
"payment_details": {
// Only show the last 4 digits.
// We shouldn't be storing the credit card number
// unencrypted anyway.
"card_last_four_digits": "1234"
}
}
Never let raw card numbers touch your servers or client storage
Accepting full PANs through your own POST /order endpoint drags the entire stack into PCI-DSS scope and makes any XSS or log leak catastrophic. Tokenize card details in the browser using a provider SDK (Stripe Elements, Braintree Hosted Fields, Adyen Drop-in) and send only the resulting payment token; the API should never see card_number or card_cvv in plaintext.

Concretely, the checkout page hosts the address form directly but embeds the payment fields inside the provider's iframe-backed SDK. The sequence below captures how the client collects an address, exchanges card fields for a payment token with the provider, and then sends only that token to POST /order.

Payment provider SDK
Client (store + view)
User (Checkout)
Payment provider SDK
Client (store + view)
User (Checkout)
[Success]
[Payment declined]
Submit address form
Validate and stash AddressDetails
Enter card details in provider iframe
Client-side validation
Click "Place order"
Request payment token
paymentToken (no PAN ever leaves provider)
{ cartId, address_details, paymentToken }
Order { id, total_price, status: "placed" }
Clear cart, route to confirmation page
Error { code, message }
Stay on checkout, surface inline error


Checkout and place-order flow with tokenized payment
Notes
Depending on whether we want to optimize for returning users, we might want to save the address and payment details on the cart object so that people who abandoned the cart after filling up the checkout form but before placing the order can resume from where they left off without having to fill up the forms again.
The state machine below names the checkout steps explicitly so the UI and the persisted cart can share the same vocabulary. Each transition is driven by a user action or a server response, and any terminal state (placed, abandoned) is a useful analytics boundary for funnel instrumentation.

Add to cart

Continue shopping

Proceed to checkout

Address valid

Payment token obtained

Place order

Provider approved

Declined (retry)

Session ends

Session ends

Session ends

Returning user resumes

Browsing

CartOpen

AddressEntry

PaymentEntry

ReviewOrder

PaymentPending

Placed

Abandoned


Checkout and order lifecycle states
## Optimizations and deep dive

With the core flows in place, this section dives into production concerns that directly affect discovery, conversion, checkout reliability, and customer trust:

Performance: This section explains why page speed matters for commerce and how to prioritize loading work across the shopping flow.
Search engine optimization: This section covers crawlable product pages, metadata, structured data, and search discovery for product listings.
Images: This section discusses responsive product images, modern formats, lazy loading, CDN delivery, and layout stability.
Form optimizations: This section covers checkout form ergonomics, validation, autofill, payment flow, and reducing user friction.
Internationalization (i18n): This section discusses translations, currencies, addresses, dates, and locale-specific commerce expectations.
Accessibility: This section covers semantic HTML, forms, buttons, images, and keyboard support across shopping and checkout pages.
Security: This section explains client-facing security concerns around authentication, payment data, transport, and third-party scripts.
User experience: This section covers checkout focus, cart behavior, error recovery, and other details that reduce abandonment.
### Performance

Performance is absolutely critical for e-commerce websites. Seemingly small performance improvements can lead to significant revenue and conversion increases. A study by Google and Deloitte showed that even a 0.1-second improvement in load times can improve conversion rates across the purchase funnel. web.dev by Google has a long list of case studies of how improving site performance led to improved conversions.

General performance tips
Code split JavaScript by routes/pages.
Split content into separate sections and prioritize above-the-fold content while lazy loading below-the-fold content.
Defer loading of non-critical JavaScript (e.g. code needed to show modals, dialogs, etc.).
Prefetch JavaScript and data needed for the next page upon hovering over links/buttons.
Prefetch full product details needed by PDPs when users hover over items in PLPs.
Prefetch checkout page while on the cart page.
Optimize images with lazy loading and adaptive loading.
Prefetch top search results.
Core Web Vitals
Know the various Core Web Vital metrics, what they are, and how to improve them.

Largest Contentful Paint (LCP): The render time of the largest image or text block visible within the viewport, relative to when the page first started loading.
Optimize performance – loading of JavaScript, CSS, images, fonts, etc.
First Input Delay (FID): Measures load responsiveness because it quantifies the experience users feel when trying to interact with unresponsive pages. A low FID helps ensure that the page is usable.
Reduce the amount of JavaScript needed to be executed on page load.
Cumulative Layout Shift (CLS): Measures visual stability because it helps quantify how often users experience unexpected layout shifts. A low CLS helps ensure that the page is delightful.
Include size attributes on images and video elements or reserve space for these elements using CSS aspect-ratio to reserve the required space for images while the images are loading. Use CSS min-height to minimize layout shifts while elements are lazy loaded.
Search engine optimization
SEO is extremely important for e-commerce websites as organic search is the primary way people discover products.

PDPs should have proper <title> and <meta> tags for description, keywords, and open graph tags.
Generate a sitemap.xml to tell crawlers the available pages of the website.
Use JSON structured data to help search engines understand the kind of content on your page. For the e-commerce case, the Product type would be the most relevant.
Use semantic markup for elements on the page, which also helps accessibility.
Ensure fast loading times to help the website rank better in Google search.
Use SSR for better SEO because search engines can index the content without having to wait for the page to be rendered (in the case of CSR).
Pre-generate pages for popular searches or lists.
Emit Product JSON-LD on every PDP
Google's rich-result treatment for products (price, rating, availability in the search snippet) is one of the highest-leverage SEO wins on a PDP and measurably lifts click-through. Include schema.org/Product JSON-LD in the server-rendered HTML and keep the price, priceCurrency, and availability fields in sync with what the user actually sees on the page.

The Travel Booking (Airbnb) system design article goes into more details about SEO.

Images
Images are one of the largest contributors to page size, and serving optimized images is absolutely essential on e-commerce websites, which are image-heavy, with every product having at least one image.

Use the WebP image format, which is the most efficient image format that currently exists. eBay uses the WebP format across all their web, Android, and iOS apps. Ensure you are able to articulate on a high level why the WebP format is superior.
Images should be hosted on a CDN.
Define the priority of the images and divide them into critical and non-critical assets.
Lazy load below-the-fold images.
Use <img loading="lazy"> for non-critical images.
Load critical images early.
Inline the image within the HTML as a data blob so that there's no need to make a separate HTTP request to fetch the image.
Use <link rel="preload"> so they download as soon as possible.
Adaptive loading of images, loading high-quality images for devices on fast networks and using lower-quality images for devices on slow networks.
Form optimizations
Filling in forms is a huge part of the checkout flow, and a very troublesome one at that. It is the last step of the checkout process, and nailing a good checkout experience will greatly help the conversion rate.

Completing forms is especially painful on mobile devices, and extra attention has to be given to optimize forms for mobile. There are two kinds of forms to fill out during checkout: the shipping address form and the credit card form.

Country-specific address forms
Different countries have different address formats. To optimize for global shipping, having localized address forms helps greatly in improving conversions and ensures that users do not drop off when filling out the address forms because they do not understand certain fields. For example:

"ZIP Code"s are called "Postal Code"s in the United Kingdom.
There are no states in Japan, only prefectures.
Different countries have their own postal/zip code formats and require different validation.
It is a hassle to have to find out this country-specific knowledge and build these forms yourself. This is where services like Stripe Checkout come in handy by providing a localized checkout form. Users will complete the rest of the payment flow on Stripe's platform.

Examples of Stripe checkout forms for different countries

| US | UK |
| --- | --- |
| Stripe checkout for US | Stripe checkout for UK |
Further reading: Frank's Compulsive Guide to Postal Addresses provides useful links and extensive guidance for address formats in over 200 countries.

Optimize autofilling
Filling up forms, especially long ones, is prone to typos. Most modern browsers have a feature called autofill, which helps users enter data faster and avoid filling up the same form data again by using values from similar forms filled previously.

Help users autofill their address forms by specifying the right type and autocomplete values for the form <input>s for the shipping address forms and credit card forms.

Shipping Address Form

| Field | type | autocomplete | Others |
| --- | --- | --- | --- |
| Name | text | shipping name | autocorrect="off" spellcheck="false" |
| --- | --- | --- | --- |
| Country | Use <select> | shipping country | N/A |
| --- | --- | --- | --- |
| Address line 1 | text | shipping address-line1 | autocorrect="off" spellcheck="false |
| --- | --- | --- | --- |
| Address line 2 | text | shipping address-line1 | autocorrect="off" spellcheck="false |
| --- | --- | --- | --- |
| City | text | shipping address-level2 | autocorrect="off" spellcheck="false |
| --- | --- | --- | --- |
| State | Use <select> | shipping address-level1 | N/A |
| --- | --- | --- | --- |
| Postal Code | text | shipping postal-code | autocorrect="off" spellcheck="false inputmode="numeric" |
Credit Card Form

| Field | type | autocomplete | Others |
| --- | --- | --- | --- |
| Card Number | text | cc-number | autocorrect="off" spellcheck="false inputmode="numeric" |
| --- | --- | --- | --- |
| Card Expiry | text | cc-exp | autocorrect="off" spellcheck="false inputmode="numeric" |
| --- | --- | --- | --- |
| Card CVC | text | cc-csc | autocorrect="off" spellcheck="false inputmode="numeric" |
Notes

inputmode="numeric" provides a hint to browsers for devices with onscreen keyboards to help them decide which keyboard to display. Unlike <input type="number">, inputmode="numeric" does not prevent users from typing non-numeric values; it simply affects the keyboard being shown. As these numeric fields are not related to quantity, the chevrons which appear when using <input type="number"> are not too helpful. <input type="text" inputmode="numeric"> also works with the maxlength/minlength/pattern attributes, but these do not work with <input type="number">.
Read more about forms:

Autofilling | web.dev
Autofill on Browsers: A Deep Dive | eBay Engineering Blog.
Alternative ways of address input
Instead of making users fill up a form containing granular address fields, there are other approaches that may be easier, at the cost of engineering complexity or reliance on external services:

Address search/autocomplete: Allow users to search for an address by typing in their street number and selecting from a list of suggestions. This reduces typos and is generally faster. However, users should still be given the option to override some values in case none of the suggestions are correct, which can happen due to an outdated address database. The Google Maps JavaScript API provides this via the Place Autocomplete library.

Selecting address location from a map: Open up a map and allow users to pinpoint a location on the map. This is less common for checkout addresses and more common for ride-hailing applications.

Error messages
Leverage client-side validation and clearly communicate any form errors. Connect the error message to the <input> via aria-describedby and use aria-live="assertive" for the error message.


<form>
<div>
<label for="name">Name</label>
<input
minlength="6"
type="text"
id="name"
name="name"
aria-describedby="name-error-message" />
<span
id="name-error-message"
aria-live="assertive"
class="name-error-message">
Name must have at least 6 characters!
</span>
</div>
<button>Submit</button>
</form>
Source: Help users find the error message for a form control | web.dev

Focus states
Make the currently focused form control visually different from the other form inputs to help users identify which element is focused.

Best practices for payment and address forms
Read more about building good Payment forms, Address forms, and general form best practices on web.dev.

### Internationalization (i18n)

Have pages translated into the supported languages.
Set the lang attribute on the html tag (e.g. <html lang="zh-cn">) to tell browsers and search engines the language of the page, which helps browsers offer a translation of the page.
Provide support for RTL languages by using CSS Logical Properties.
Forms
Use a single form field for names.
Allow for various address formats. This is covered in the section above on address forms.
Read more:

### Internationalization and localization | Forms | web.dev

### Internationalization | Design | web.dev

### Accessibility

Use semantic elements where possible: headings, buttons, links, and inputs instead of styled <div>s.
<img> tags should have the alt attribute specified, or left empty if the merchant did not provide a description for them.
Building accessible forms has been covered in detail above. In summary:
<input>s should have associated <label>s.
<input>s are linked to their error messages via aria-describedby and error messages are announced with aria-live="assertive".
Use <input>s of the correct types and appropriate validation-related attributes like pattern, minlength, maxlength.
Visual order matches DOM order.
Make the currently focused form control obvious.
Reference: Accessibility | Forms | web.dev and WebAIM: Creating Accessible Forms

Security
Since payment details are highly sensitive, we have to make sure the website is secure:

Use HTTPS so that all communication with the server is encrypted and so that other users on the same Wi-Fi network cannot intercept and obtain any sensitive details.
The payment details submission API should not use HTTP GET because the sensitive details will be included as a query string in the request URL, which will get added to the browsing history, which is potentially unsafe if the browser is shared with other users. Use HTTP POST or PUT instead.
HTTPS-only, and keep payment fields out of URLs and analytics
Sensitive fields in query strings leak into browser history, referer headers, CDN access logs, and third-party analytics scripts, so GET is unsafe even over TLS. Serve the whole site over HTTPS with HSTS, submit payment forms via POST, and scrub any inputs in checkout iframes from session-replay and error-tracking tools.

Source: Security and privacy | web.dev

### User experience

Make the checkout page clean (e.g. minimal navbar and footer) and remove distractions to reduce bounce rate.
Allow persisting cart contents (either in a database or cookies), as some people spend time researching and considering, only making the purchase during subsequent sessions. You don't want them to have to add all of the items to their cart again.
Make promo code fields less prominent so that people without promo codes will not leave the page to search the web for a promo code. Those who have a promo code beforehand will take the effort to find the promo code input field.
Persist the cart server-side for signed-in users, cookie-backed for guests
Carts are effectively purchase intent, and losing them across sessions or devices is a direct revenue hit. Tie the cart to the user ID for authenticated shoppers so it survives device switches, and fall back to a long-lived cookie or localStorage for guests. Then merge the guest cart into the account cart on sign-in.

## Summary

E-commerce revenue comes from three levers a front-end design actually controls: how fast the first paint lands, how well catalog pages rank in organic search, and how little friction sits between an added item and a completed POST /order.

First paint? The PLP and PDP are server-rendered, so shoppers see product imagery, titles, and prices in the initial HTML response rather than after a hydration round trip. Critical above-the-fold assets are prioritized, images are served as WebP through a CDN with below-the-fold lazy loading, and route-level code splitting keeps the JavaScript budget lean. Prefetching PDP data on PLP hover collapses the perceived latency of the next click, and LCP, FID, and CLS are treated as first-class budgets rather than afterthoughts.

Organic search? Catalog surfaces must be crawlable, which is why SSR is non-negotiable for ProductList and Product pages while Cart and Checkout stay on CSR because they are authenticated, transactional, and irrelevant to search engines. Every PDP ships Product JSON-LD so rich results surface price, availability, and ratings directly in the SERP. A generated sitemap.xml exposes the full product graph to crawlers, and the Controller keeps <title> and <meta> tags accurate per route so deep links and share previews are coherent.

Checkout friction? The Client Store holds an optimistic Cart of CartItems so the navbar badge and drawer respond instantly, but every mutation is reconciled against the authoritative Cart returned by the Server; price and total_price are server-owned so promos, tax, and regional pricing never disagree between surfaces. GET /products uses offset pagination to keep PLP state shareable and restorable. The checkout page is prefetched from the Cart, AddressDetails adapts to country-specific forms with correct autocomplete and inputmode hints, and PaymentDetails is tokenized by a provider SDK so raw card data never hits POST /order.

Start with the conversion thesis, name SSR-for-catalog and CSR-for-transactional as the rendering split, and then spend the remaining time on server-owned pricing, optimistic-but-reconciled cart mutations, and the performance and form-UX work that closes the gap between a working prototype and a storefront people actually buy from.

## References

The references below group e-commerce sources by the topic they inform, so it is easier to map each source back to the part of the article it relates to: real-world performance case studies, form and address input patterns, rendering strategy, and related system design articles.

### Performance case studies

This subsection collects production postmortems from major e-commerce sites showing how page speed and Core Web Vitals improvements translated into concrete business metrics such as conversion and revenue.

Shopping for speed on eBay.com | web.dev for eBay's end-to-end story of optimizing a large product catalog experience.
Speed by a thousand cuts | eBay Engineering for the incremental wins that compound into a measurably faster shopping flow.
How Rakuten 24's investment in Core Web Vitals increased revenue per visitor by 53.37% | web.dev for a concrete link between Core Web Vitals improvements and revenue per visitor.
How focusing on web performance improved Tokopedia's click-through rate by 35% | web.dev for a large marketplace's mobile-first performance work.
Luxury retailer Farfetch sees higher conversion rates for better Core Web Vitals | web.dev for how performance gains shift conversion at a high-end retailer.
How Renault improved its bounce and conversion rates | web.dev for a long-funnel purchase flow where bounce and conversion were both sensitive to speed.
Lowe's website is among fastest performing e-commerce websites | web.dev for the kinds of architecture choices that keep a catalog-heavy retailer fast.
JD.ID improves their mobile conversion rate by 53% with caching strategies | web.dev for how caching and network strategy lift mobile conversion.
Milliseconds make millions | web.dev for the broader research backing latency-to-revenue sensitivity across retailers.
Forms, autofill, and address input
This subsection covers the specific UX and browser integrations that make checkout forms fast and correct, especially around autofill and international addresses.

Autofill on browsers: a deep dive | eBay Engineering for how browser autofill interacts with real-world checkout forms.
Autofilling | web.dev for the standard guidance on wiring forms so browsers can autofill them reliably.
Address forms | web.dev for conventions and pitfalls when designing address entry fields.
Payment and address form best practices | web.dev for end-to-end guidance on checkout form quality.
Frank's compulsive guide to postal addresses for background on the real-world variance in postal address formats across countries.
Rendering strategy
This subsection links the canonical overview that frames the rendering tradeoffs used throughout the architecture discussion.

Rendering on the web | web.dev for the vocabulary and tradeoffs behind CSR, SSR, SSG, and hybrid approaches applied to e-commerce pages.
Related system design articles
The articles below overlap with this one on product surfaces, media-heavy pages, or input UX, and are useful cross-references when extending the e-commerce design.

Travel booking (Airbnb) for another catalog-plus-detail-plus-checkout product with similar rendering and performance tradeoffs.
Photo sharing (Instagram) for image-heavy rendering and media pipeline patterns that also apply to product imagery.
Autocomplete for the interaction pattern powering product search suggestions on the storefront.