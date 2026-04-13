[@yoavweiss](https://github.com/yoavweiss), [@tunetheweb](https://github.com/tunetheweb)

# Speculative load measurement API

## Introduction

Modern web applications can use speculative loading techniques to improve navigation performance.
However, developers currently lack visibility into whether these speculations were actually used, making it hard for them to deploy higher eagerness values than "conservative". Such values can result in wasted user bandwidth and server-side load, and currently developers have no way to weigh the trade-off between those and potential performance gains.

This proposal addresses this by exposing information about used and unused speculative loads on page dismissal, enabling developers to measure speculation effectiveness and optimize their speculative load strategies, to pick the trade-off that's right for them.

## Use-cases

When experimenting with speculative loads, developers need to be able to answer questions such as:
* Does preloading more/certain resources cause the site's performance metrics to get better or worse?
* How often do my preloaded resources go unused? Which preloaded resource should I stop preloading?
* Is more aggressive prefetching/prerendering worth the cost tradeoff? User performance improves by X%, but server-cost is increased by Y%.
* What links should I stop prefetching/prerendering, as they almost never get used?
* What's the benefit from moving preloads to use Early Hints?

This proposal aims to enable developers answering those questions.

## Goals

1. Enable developers to measure the effectiveness of their speculation rules and preload strategies. 
1. Expose relevant metadata ([speculation rules tags](https://html.spec.whatwg.org/C#prefetch-record-tags), `as` attribute values) to make the above measurement actionable.
1. Only expose accurate speculation data. "Used" status should not be included before we know it.
1. Only expose data on user actions in the current document (even for cross-origin speculative loads), without exposing cross-origin data.

## Non-Goals
1. Enable developers to detect and handle situations when speculations are unsuccessful.
1. Measurement of prefetched resources (as opposed to prefetched navigations).
   - Speculative resource fetches for the current page are better served with preload, which is covered.
   - For speculative resource fetches for future pages, it's very hard to say with certainty that they were not used, as they may still be used in a future navigation.

## API Design

We will add a `getSpeculations()` method to the `performance` object, which will return an object that holds information about unused preloads, speculation rules navigational prefetches and speculation rules prerenders.

That object will have the following shape:
```
{
  preloads: [
    { url: '...', as: '...', crossorigin: '...', earlyhint: true, used: true },
    { url: '...', as: '...', crossorigin: '...', earlyhint: true, used: false },
    // ...
  ],
  navigations: [
    { type: 'prefetch', url: '...', tags: '...', eagerness: '...' },
    { type: 'prerender', url: '...', tags: '...', eagerness: '...' },
    // ...
  ]
}
```

* `URL` - the URL of the resource.
* `as` - The reflected [`as` attribute](https://html.spec.whatwg.org/C#attr-link-as) value, as it can often lead to mismatches and unused preloads
* `crossorigin` - an enum representing the `crossorigin` attribute value, as it can similarly lead to mismatches
* `type` - the type of the speculative fetch. e.g. "prefetch", "prerender" or "prerender-until-script"
* `tags` - The relevant [tags](https://html.spec.whatwg.org/C#prefetch-record-tags).
* `eagerness` - The [eagerness](https://html.spec.whatwg.org/C#speculation-rule-eagerness) of the rule that lead to the speculative load.
* `earlyhint` - whether a preload was delivered using an early hint header.
* `used` - See definition below, under "key concepts".

### Example Usage

```javascript
window.addEventListener('pagehide', (event) => {
  const { preloads, navigations } = performance.getSpeculations();

  // Send unused speculation data to analytics endpoint
  fetch('/analytics/unused-speculations', {
    method: 'POST',
    keepalive: true,
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      preloads, navigations
    })
  });
});
```

TODO: We also need to expose the URL we're navigating to outside of the navigate event, to make it easier to collect. We need to figure out the desired shape for that.

## Key Concepts

### "Used" Definition

A preload is considered **unused** if no resource load during the page lifetime [consumed](https://html.spec.whatwg.org/multipage/links.html#consume-a-preloaded-resource) it from the [map of preloaded resources](https://html.spec.whatwg.org/multipage/links.html#map-of-preloaded-resources).

## Security and Privacy Considerations

This proposal can potentially make it easier to access same-origin data (e.g. "user navigated to a certain link", "user had certain sections go into the viewport" or "user hovered over something"). This data is already something developers can gather today.

## Considered Alternatives

### PerformanceObserver API

PerformanceObserver is the natural choice for exposing performance-oriented data.
At the same time, there are a couple of issues with PerformanceObserver in this particular case:
* There's no single point in time in which we know that a load went unused before the page dismissal events. So an observer feels like the wrong pattern.
* By default, observers are async. That makes it particularly hard to work with them inside dismissal events, after which the page unloads.

### Exposure on a dismissal event

An early iteration of this proposal suggested to augment the `pagehide` event with the `speculations` object.
We got feedback that this ties the solution too much to dismissal events, and may make it harder to change their behavior in the future, if desired.

### Reporting API

While a Reporting API can be most accurate in terms of the point in time it can send the information (e.g. after the page was already dismissed, and potentially after eviction from the bfcache), the Reporting API presents some challenges to developers:

It requires a dedicated backend collection point. Smaller sites in particular without dedicated endpoint solutions often use analytics solutions (e.g. Google Analytics) which need the payload in a certain format and do not support the Reporting API directly.

Once the data is collected, developers would frequently need to join the data reported by the API, with other performance data collected through JavaScript.

Finally, using the Reporting API would also give less control over how much of the information is sent over the network. Sites may only be interested in some of the information (e.g. just counts and not URLs, or only the `navigations` data without the without the `preload` data). This can of course be filtered server 
-side but if the data is not needed then it's preferable not to send it in the first place.

These reasons go some way to explaining why [the Reporting API usage](https://chromestatus.com/metrics/feature/timeline/popularity/56650) is considerably smaller than [performance API usage like LCP](https://chromestatus.com/metrics/feature/timeline/popularity/2927). In our experience, the Reporting API is a more advanced API for data that cannot be measured in JavaScript (e.g. browser crashes, CSP violations that may prevent JavaScript from even loading... etc.).

## Frequently Asked Questions

### Can't you get that information from server side logs?

It is tricky to correlate previous speculative requests with future page loads on the server.

Additionally, many resources or navigations may be served from caches (CDNs, browser caches...etc.) further making this difficult (impossible?) to measure server-side.

### What about `<link rel=prefetch>` and `<link rel=prerender>`?

As detailed in [the non-goals](#non-goals), resource prefetches (as opposed to navigation prefetches) are often for future page views so are explicitly excluded. `<link rel=preload>` should be used for this page view and so is included.

`<link rel=prerender>` is an older API which has been replaced with the Speculation Rules API, and may be deprecated in future, so is excluded. Usage of this API is also low, and likely will drop further once [`prerender_until_script` becomes available by default](https://developer.chrome.com/blog/prerender-until-script-origin-trial?hl=en).
