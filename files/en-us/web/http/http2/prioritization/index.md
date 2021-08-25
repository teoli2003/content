---
title: HTTP Prioritization
slug: Web/HTTP/HTTP2/Prioritization
tags:
  - Guide
  - HTTP
  - WebMechanics
---
{{HTTPSidebar}}

The order a browser downloads the assets used by a web page significantly affects its performance, as perceived by the user.
Not all assets are equally important for the reader.
Furthermore, prioritizing resources needed to display the most helpful content leads to a much better experience.

Each browser implements a specific algorithm, using hints from the page. Browsers determine each resource's importance and request them to the server. The server sends the data back, and the browsers build the page as quickly as possible to display it to the users. Determining the order of download for the assets and the amount of multiplexing to use has been an active topic of research and still is. It leads to improvement in browser algorithms, HTTP and transport protocols, and servers.

## Signalling priorities and dependencies to the server

In HTTP/1.1, in the absence of multiplexing, there was no prioritization done in the HTTP layer;
each connection serializes resources to download.
Moreover, it was the sole responsibility of the browser to request them in the optimal order.

HTTP/2, with its introduction of _frames_, introduced multiplexing.
It is now possible to download multiple resources in parallel.
How much bandwidth has to be allocated to each request needs to be determined.
So HTTP Prioritization has been introduced.

The browser informs the server about the relative priority of requests
then this one sends back the answers in a multiplexed way, matching the priority.

It uses three pieces of information to transmit the priority of each HTTP stream:

- A _weight_ byte indicates the **relative priority** of this request relative to the other on the same connection. The higher it is, the higher the priority.
- An _exclusive_ boolean flag indicates if this request can be **multiplexed or not**. If set, it means that no other request should be multiplexed with it when the server transmits this request.
- A _stream ID_ represents its _parent stream_ that it depends on, allowing the server to build a **dependency** tree of resources. In other words, this ID indicates that the server must finish transferring the parent stream before it starts sending content for this stream.

## Bandwidth allocation

The server is responsible for allocating the bandwidth according to the priority of each stream. A logical tree represents the set of requested resources. A node is a resource associated with a _weight_ and an _exclusive_ flag. It is also eventually linked to the stream it depends upon or to the tree's root if none.

**TODO:** IMG of resources vs. tree.

The server calculates a relative priority for each node. It is the combined priority of its parent multiplied by its weight. A request is downloadable only if the server has finished transmitted its parent. The server allocates the bandwidth available proportionally to the sum of the priorities of all downloadable requests. The priorities are recalculated each time the set of downloadable requests changes, either because of a request completely transmitted or because of a new request made.

**TODO:** IMG of allocation of resources proportionally of the overall priority.

When an _exclusive_ resource is to be transmitted, the server first transmits all resources with higher priority, then the exclusive resource, then the algorithm resumes with all the remaining resources.

**TODO:** IMG with exclusive resources

## Determining priorities

The browser transmits the importance of each request to the server. Neither the user nor the webpage does control it directly; the browser does. Still, the web developer can influence the priorities of the requests using the ordering of the declaration in the HTML source (starting with the most important ones for better performance), and by providing _hints_ in the HTML.

As they implement the prioritization algorithm, servers also affect how browsers receive the different resources, and they may abide more or less by the specification.

## Browsers prioritization algorithms

Browsers are the leading actor in defining the prioritization of the download of the different assets. Different browsers use different algorithms, and this has a dramatic effect on page load.

### Internet Explorer and old Edge browsers

The crudest algorithm is used by Internet Explorer 11 and Edge (before version 79??????): there is no priority: All requests have the same weight (16) and therefore the same priority. It is a poor algorithm as the page load appears to stall after the HTML is loaded and often gives the impression to wait for the download of all other resources before displaying them, regardless of their importance.

**TODO:** IMG of the waterfall

As the server allocates the same bandwidth to everybody, such algorithms are named _Naive round robin_ (Naive RR).

### Safari and WebKit-based browsers

Safari and WebKit-based browsers, including _all_ browsers on iOS, signal relative priorities using five predefined weights, attributed to a resource according to its type:

- The HTML document, of highest priority, gets a weight of 255.
- CSS and JavaScript resources, get a weight of 24.
- Fonts get a weight of 16.
- Images get a weight of 8.

Here again, the server allocates the bandwidth equitably between resources with the same priority; this algorithm is called _Weighted round robin_ (Weighted RR).

**TODO:** IMG of the waterfall

### Chrome and Chromium-based browsers

Chrome creates an entirely linear dependency tree. It uses the _exclusive_ flag to load one resource at a time, sequentially. Resources are put in buckets, each having a specific _weight_:

- HTML, CSS and fonts are in the _highest_ priority bucket, with a weight of 255.
- XHR requests and JavaScript declared before the first image are in the _high_ priority bucket, with a weight of 220.
- JavaScript requests declared after the first image are in the _normal_ priority bucket, with a weight of 183.
- Images, and JavaScript files declared with the `defer` or `async` attributes, are in the _low_ priority bucket, with a weight of 147.
- All other assets are in the _lowest_ priority bucket, with a weight of 110.

As the tree is wholly linear, the actual value of the weight is not important. Only the ordering it defines on the buckets matter. Such an algorithm is called _Dynamic first come-first-serve_ (Dynamic FCFS).

**TODO:** IMG of the waterfall

### Firefox

Firefox is using a completely different algorithm.

**TODO:** IMG of the waterfall

## Page hints affecting priority

The page cannot directly modify the priority of the resources. It has a few way to make sure that all important assets are discovered on time, as well as to prevent the addition of some resources to the list of needed assets using mechanisms like lazy loading.

Most of the techniques described here are not specific to HTTP/2 or later. They were already effective with HTTP/1 and are part of the good practices of web development.

### Preventing JS resources to block the parsing of the HTML

When the HTML parser encounter a {{HTMLElement("script")}} element linking to an external script, it stops its parsing until the script is downloaded and executed. The reason is that the script can modify the HTML.

A lot of scripts do not modify the HTML and their is no reason to block the parsing. The parser has no way of knowing this, unless either the `defer` or the `async` attributes are used.

Blocking the HTML parser will give priority to the download of the blocking script, which is often not what is wanted. By using these attributes, the browser will be more efficient at downloading the important resources first.

### Link preloading attribute

A web page can use [`<link rel="preload">`](/en-US/docs/Web/HTML/Link_types/preload) for signaling to the browser resources that will be used on the page. This doesn't change the prioritization algorithm used by browsers but makes the browser discovering these assets earlier, especially if they are hidden in files downloaded after the HTML, like CSS or JavaScript files. This may have a significant impact, especially on browsers using first come first serve algorithms, like Chromium-based ones.

> **Note:** `<link rel="prefetch">` doesn't affect the priorities of the page. It indicates to the browsers a resource that is likely to be requested by another page. The browser is invited to download it during an idle time, that is after the current page has been displayed, and to populate its cache in prevision of a future request.

### Images and iframes lazy loading

Another way for the web developer to alter the resources to load is to remove them from the download list. Images and inline frames can be loaded lazily, only when they are close to the viewport and about to be displayed.

Lazy loading increases the website's visual performance, which does not waste bandwidth for resources that will not be displayed. It also lowers the overall amount of data transferred, which is helpful for a user that is on a metered connection.

Different [techniques](/en-US/docs/Web/Performance/Lazy_loading#images_and_iframes) are available to implement lazy loading:

- the [Intersection Observer API](/en-US/docs/Web/API/Intersection_Observer_API) allows [lazy loading of images](https://javascript.plainenglish.io/how-to-lazy-load-images-with-intersection-observer-beced03e4a43),
- a combination of {{domxref("Document/scroll_event", "scroll")}}, {{domxref("Window/resize_event", "resize")}}, and {{domxref("Window/orientationchange_event", "orientationchange")}} event handlers leads to [the same effect](https://imagekit.io/blog/lazy-loading-images-complete-guide/#trigger-image-load-using-javascript-events),
- the use of the `loading` attribute with the value `lazy`on {{HTMLElement("img")}} and {{HTMLElement("iframe")}} elements.

**TODO:** In case loading attr guide not good, link to blog: https://web.dev/browser-level-image-lazy-loading/

## Server implementations of the allocation algorithm

On the server side, there is not much to do, either. It is important to note that not all servers are  adhering to the HTTP prioritization specifications and some have bugs. Especially, not all server are adapting the prioriity when they receives higher priority requests after lower priority requests.

Also some companies are overriding the algorithm with one of their choices, that may be more efficient in some case, and less in others.

More than with HTTP/1.1, the choice of a hosting company for a website can affect its performance. Knowing what the server will do is important in choosing the right one.

## The Server Push mechanism

A last way to alter the order of resource download, is to work-around the decision of the browser at the server side. HTTP/2 defined a way for the _server_ to start sending resources it knows it will be requested even before the browser requests it. This technique is called [HTTP Server Push](/en-US/docs/Web/HTTP/HTTP2/Server_push).

## Conclusion

The web developer doesn't significantly control the prioritization algorithm. To get an optimal performance, it only can give direct and indirect hints to the browser. This is achieves by using the traditional performance best practices, preventing the HTML parser to stop, listing critical resources buried in other files using the {{HTMLElement("link")}} element, ordering the most important resources near the top of the file or using advance techniques like lazy loading.

That way, browsers will be able to request resources as efficiently as they can and provide what each of them think to be the best user experience.

## See also

- [HTTP/2 Prioritization](https://datatracker.ietf.org/doc/html/rfc7540#section-5.3), in RFC 7540, Hypertext Transfer Protocol Version 2 (HTTP/2).
- [Better HTTP/2 Prioritization for a Faster Web](https://blog.cloudflare.com/better-http-2-prioritization-for-a-faster-web/), by Patrick Meenan.
- [HTTP/2 Prioritization](https://calendar.perfplanet.com/2018/http2-prioritization), by Patrick Meenan.
- [Stream prioritization](https://hpbn.co/http2/#stream-prioritization), in High-Performance Networking, Chapter 12: HTTP/2, by Ilya Grigorik.
- [HTTP/2: Discover the Performance Impacts of Effective Prioritization](https://developer.akamai.com/blog/2019/01/31/http2-discover-performance-impacts-effective-prioritization), by Andy Davies.
- [Prioritize resources](https://web.dev/prioritize-resources/), by Sérgio Gomez.
- [Preload, Prefetch and Priorities in Chrome](https://medium.com/reloading/preload-prefetch-and-priorities-in-chrome-776165961bbf), by Addy Osmani.
