# Security & Privacy Questionnaire: Speculative Load Measurement API

## 2.1. What information does this feature expose, and for what purposes?

The feature exposes, **only to the document that initiated the speculations**, information about speculative loads that the same document has already triggered:

- **`preloads` array** (one entry per `<link rel=preload>` initiated by the page, including ones delivered via Early Hints):
  - `url`: the URL the developer asked the user agent to preload.
  - `as`: the `as` attribute value (e.g. `"script"`, `"style"`, `"image"`).
  - `crossorigin`: `"none"`, `"anonymous"`, or `"use-credentials"`.
  - `used`: a `DOMHighResTimeStamp` for when the preload was actually consumed by a subsequent fetch, or `null` if it was never consumed.
  - `earlyhint`: a boolean indicating that load was initiated by an early hint.
- **`navigations` array** (one entry per Speculation Rules candidate the page sent to the browser):
  - `type`: `"prefetch"`, `"prerender"`, or `"prerender-until-script"`.
  - `url`: the destination URL the page asked the browser to speculate on.
  - `tags`: developer‑supplied tags from the rule, if any.
  - `eagerness`: `"conservative"`, `"moderate"`, `"eager"`, or `"immediate"`.
- **`navigationDestinationURL`** (USVString or null): the URL of the most recent non-same-document, same-origin outgoing navigation, captured at the time the Navigation API's navigate event fires (i.e. the same point at which NavigationDestination.url becomes observable to script). null if no qualifying navigation has been observed.

**Purpose:** Developers currently have no way to tell whether their speculations were actually used, or what navigations were speculated.
Without this, they cannot tune `<link rel=preload>` lists or speculation rules, which means they either default to the most conservative settings,
or ignore wasted resources by the user on or their servers.
The navigationDestinationURL lets a page collecting analytics on pagehide correlate the speculations it had outstanding with the URL the user actually navigated to — without requiring hacks to obtain that information.

The API gives them just enough signal to remove unused speculations and to safely raise eagerness when it pays off.

### 2.1.1. Information exposed to the first party that it could not previously determine

A first party can already determine most of this today.
For preloads, it knows the URLs, `as`, `crossorigin`, `tags`, `eagerness`, and `type`.
One new bit of information there is the `used` timestamp on preloads, which today can be roughly guessed through DOM inspection.

For navigations, the first party doesn't know speculated navigations unless they were actually useful, in which case the information is available to
it in the following same-site navigation. At the same time, it can guess unused speculated navigations by carefully immitating the browser heuristics around clicks,
hovers and intersection with the viewport.

The same applies to navigationDestinationURL: the page can already obtain the exact same URL synchronously, with the same timing and same same-origin scope, via event.destination.url inside a navigate event handler on navigation. navigationDestinationURL is just a convenience surface that retains that value past the event handler's stack so it remains available inside pagehide/visibilitychange=hidden handlers, where developers actually want to ship telemetry. No information is exposed that wasn't already on the navigate event.

### 2.1.2. Information exposed to third parties that they could not previously determine

None. A third‑party iframe sees only the speculations initiated **by that iframe's own document**, not those of the embedder.

### 2.1.3. Potentially identifying information already accessible to the first party that is duplicated

None.

### 2.1.4. Potentially identifying information already accessible to third parties that is duplicated

None.

---

## 2.2. Do features in your specification expose the minimum amount of information necessary?

Yes. Only information required for the feature's functionality is exposed.


---

## 2.3. Do the features in your specification expose personal information, PII, or information derived from either?

No. All values surfaced by the API originate from the same document's own markup (`<link>` tags and `<script type=speculationrules>` blocks). It does not surface user identity, contact information, or any cross‑site signal about the user.

The `used` timestamp is high‑resolution time of a *first‑party* event the page already participated in initiating; it is bounded by the page's own timeline and is not derived from the user.

---

## 2.4. How do the features in your specification deal with sensitive information?

The API does not handle sensitive information.

While URLs can be sensitive in some contexts (e.g. they can contain personal identifiers in query strings), `navigationDestinationURL` only ever contains a same-origin URL the page itself constructed or followed.


---

## 2.5. Does data exposed by your specification carry related but distinct information that may not be obvious to users?

No.

---

## 2.6. Do the features in your specification introduce state that persists across browsing sessions?

No.

---

## 2.7. Do the features in your specification expose information about the underlying platform to origins?

No.

---

## 2.8. Does this specification allow an origin to send data to the underlying platform?

No.

---

## 2.9. Do features in this specification enable access to device sensors?

No.

---

## 2.10. Do features in this specification enable new script execution / loading mechanisms?

No.

---

## 2.11. Do features in this specification allow an origin to access other devices?

No.

---

## 2.12. Do features in this specification allow an origin some measure of control over a user agent's native UI?

No.

---

## 2.13. What temporary identifiers do the features in this specification create or expose to the web?

None.

---

## 2.14. How does this specification distinguish between behavior in first‑party and third‑party contexts?

The API is available in any browsing context that has a `Window.performance` — first or third party. 
However, the data it returns is **scoped to the current document only**. 
A third‑party iframe will only see the preloads and speculation rules its own document initiated; 
it cannot observe what the top‑level document or other frames have speculated.

---

## 2.15. How do the features in this specification work in the context of a browser's Private Browsing or Incognito mode?

Behaviour is identical to normal mode. The API exposes only same‑document, in‑memory information that is destroyed when the document is destroyed.
It does not leak the existence of incognito mode, and it does not provide any persistent surface that could correlate normal and private sessions.

---

## 2.16. Does this specification have both "Security Considerations" and "Privacy Considerations" sections?

The current explainer at <https://github.com/WICG/speculative_load_measurement> contains a combined "Security and Privacy Considerations" section.

---

## 2.17. Do features in your specification enable origins to downgrade default security protections?

No.

---

## 2.18. What happens when a document that uses your feature is kept alive in BFCache after navigation, and potentially gets reused on future navigations back to the document?

`SpeculationData` is generated on demand each time `performance.getSpeculations()` is called, from state living on the `Document`/`ResourceFetcher`/`DocumentSpeculationRules`. 
The state is not updated while the document is non‑fully‑active in BFCache. 
When the document becomes fully active again, the state resumes accumulating from whatever the new navigation triggers. 
No additional retention or replay of speculative data occurs because of BFCache.

`navigationDestinationURL` is written by the navigate event hook just before the page is navigated away — i.e. immediately before the page may enter BFCache. If the page is BFCached and later restored, `navigationDestinationURL` will still hold the URL that triggered the BFCache entry (the page's own intended destination at the time it left). On subsequent navigations from the restored document, that value is overwritten as usual.

---

## 2.19. What happens when a document that uses your feature gets disconnected?

When the document becomes non‑fully‑active because its containing iframe is disconnected, the document is gone from the user's perspective. 
The state backing `SpeculationData` is owned by that document and will be discarded together with it. 
No callbacks or events are dispatched from this API, so there is nothing to suppress.

---

## 2.20. Does your spec define when and how new kinds of errors should be raised?

No. `getSpeculations()` never throws and never rejects. If there are no speculations (or the calling window has no document), it returns a `SpeculationData` with empty arrays.

---

## 2.21. Does your feature allow sites to learn about the user's use of assistive technology?

No. The API surfaces no information about user input modality, AT, or interaction patterns.

---

## 2.22. What should this questionnaire have asked?

Nothing additional surfaced during this review.

---
