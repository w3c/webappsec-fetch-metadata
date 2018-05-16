# `Sec-Metadata` (TODO: Bikeshed the name)

## A Problem

Interesting web applications generally end up with a large number of web-exposed endpoints that might reveal sensitive data about a user, or take action on a user's behalf. Since users' browsers can be easily convinced to make requests to those endpoints, and to include the users' ambient credentials (cookies, privileged position on an intranet, etc), applications need to be very careful about the way those endpoints work in order to avoid abuse.

Being careful turns out to be hard in some cases ("simple" CSRF), and practically impossible in others (cross-site search, timing attacks, etc). The latter category includes timing attacks based on the server-side processing necessary to generate certain responses, and length measurements (both via web-facing timing attacks and passive network attackers).

It would be helpful if servers could make more intelligent decisions about whether or not to respond to a given request based on the way that it's made in order to mitigate the latter category. For example, it seems pretty unlikely that a "Transfer all my money" endpoint on a bank's server would expect to be referenced from an `<img>` tag, and likewise unlikely that `evil.com` is going to be making any legitimate requests whatsoever. Ideally, the server could reject these requests _a priori_ rather than delivering them to the application backend.

## A Proposal

Browsers could give servers the ability to make better decisions earlier if they added metadata to requests. We could, for example, add a [structured header](https://tools.ietf.org/html/draft-ietf-httpbis-header-structure) [dictionary](https://tools.ietf.org/html/draft-ietf-httpbis-header-structure#section-4.1) containing a set of interesting labels and values to all requests made to an HTTPS endpoint. For example:

```
// <picture>
Sec-Metadata: initiator=imageset, destination=image, site=cross-site, target=subresource

// Top-level navigation
Sec-Metadata: initiator="", destination=document, site=cross-site, target=top-level, cause=user-activation

// <iframe> navigation
Sec-Metadata: initiator="", destination=document, site=same-site, target=nested, cause=forced
```

So, what labels and values are interesting and valuable enough that we'd want to include them? Unsurprisingly, I have some suggestions:

* Enums representing requests' [`initiator`](https://fetch.spec.whatwg.org/#concept-request-initiator) and [`destination`](https://fetch.spec.whatwg.org/#concept-request-destination) values would be helpful, as they enable granular decision-making in a way that the Accept header does not. If a frontend server notices that a particular document is being requested from a `<script>` or `<img>`, for instance, it can respond with a 400 or 406 rather than forwarding the request on to the backend, generating a response, and delivering it back to the client. A number of potential attacks can be mitigated using this technique.

* It might also be helpful to categorize the request broadly into an enum of {`top-level`, `nested`, `subresource`}. This high-level distinction enables interesting security decisions: API endpoints do not expect to be navigation targets, and listing pages do not expect to be subresources. (Note that we almost get this data from the enums above, though Fetch currently doesn't distinguish between top-level and nested navigations, giving both a destination of `document`. Perhaps we could extract the "reserved client is either null or an environment whose target browsing context is a nested browsing context" into a request property we could expose, or add granularity to initiator, split `document`?)

* The relationship between the context initiating the request, and the request's target, perhaps an enum of {`same-origin`, `same-site`, `cross-site`} (where "site" boils down to eTLD+1). This gives developers the ability to construct a low-cost CSRF defense for endpoints that don't expect cross-origin or cross-site callers, and has low-enough granularity that it seems reasonable to include on every request. If you squint a bit, you can consider it something of an imperative counterpart to the declarative `SameSite` attribute on cookies.

* For navigations, an indication of whether the navigation was [triggered by user activation](https://html.spec.whatwg.org/multipage/interaction.html#triggered-by-user-activation), or forced via `window.location`/`<meta>`, etc. would give servers the ability to build models that heuristicly differentiate between an attacker poking at `window.opener.location` and a user's intent to navigate to a search result page.

## FAQ

### Isn't [`From-Origin`](https://github.com/whatwg/fetch/issues/687) enough to handle the threats discussed above?

No. The `From-Origin` proposal is an _a posteriori_ mitigation against a response's content falling directly into the hands of unsavory actors. That is, it takes effect client-side, once a response has been generated server-side and served across the network to the client. CSRF attacks will still reach the application and do damage. Timing attacks based both on the work the server does when responding to requests, and on the length of the delivered content itself, will still be possible client-side and through passive network observation.

This proposal is valuable above and beyond `From-Origin` insofar as it gives servers the opportunity to more actively filter incoming requests _a priori_, before doing any work. Applications can quickly reject requests based on testing a set of preconditions, and that work can even be lifted up above the application layer (to reverse proxies, CDNs, etc) if desired. Also, the fact that the server is capable of rejecting the request itself means that we don't need to come up with a reporting story: the server can do its own reporting to itself.

### What about including the request's origin?

Based on discussion in [whatwg/fetch#700](https://github.com/whatwg/fetch/issues/700), we seem to have reasonable agreement about the value of the above items. There are other bits and pieces which seem more controversial, in particular including the request's [origin](https://fetch.spec.whatwg.org/#concept-request-origin). This would enable more granular decision-making for applications with more complicated relationships than `same-site` can express (consider `docs.google.com` and `mail.google.com`, which both wish to talk to `accounts.google.com`, but aren't interested in talking to each other), but there's a reasonable worry that creates a new mechanism for referrer leakage ([whatwg/fetch#700 (comment)](https://github.com/whatwg/fetch/issues/700#issuecomment-382762249) spells out some of the controversy). This proposal punts on that question for the moment. Maybe we can extend the applicability of the `Origin` header, maybe we can add origin information to this header at a later date, or maybe implementation experience will teach us that we don't really need the additional granularity.

### Do we need both `initiator` and `destination`?

The table in Fetch (just under https://fetch.spec.whatwg.org/#request-destination-script-like) gives some examples of categories of request and their respective `initiator` and `destination` values. Some interesting kinds of requests are ambiguous if we relied purely on their `destination` (consider `fetch()` vs `<a download>`, for instance). As things are specified today, but I think we'd want to cover both, since it's somewhat unclear what might come in the future, and where we'd record it.

It's possible that pursuing this proposal might result in rethinking those values, however, as only `destination` is currently exposed to script (in service workers). If we're going to expose them more broadly, perhaps we can also make them more granular in certain ways.

### Why bundle these up into one header? Why not have one header for each distinct value, like [client hints](http://httpwg.org/http-extensions/client-hints.html#rfc.section.4)?

I like the idea of having everything that developers need to make an authentication decision for a given request in one place. But I also don't have terribly strong feelings about the way we spell the feature. If splitting it into a zillion distinct headers is the right thing to do, I'm happy to run with that.

### Doesn't CORS take care of this somehow?

No. CORS allows a server to opt-out of the same-origin policy for a given response, sharing a resource's content with a cross-origin requestor. There is some conceptual similarity, insofar as `Origin` headers are sent along with CORS-enabled requests in order to allow the server to make a reasonable decision about them, but this proposal has very little to do with explicit sharing, aiming instead to address the unintentional leakage that happens as a side-effect of a server's activity. 
