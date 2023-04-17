# Introduction
Smooth visual transitions as users navigate on the web can lower cognitive load by helping users stay in context. It can also provide a [visual cue](https://www.wired.com/2014/04/swipe-safari-ios-7/) about the destination before initiating the navigation. Both site authors and UAs add visual transitions to their navigations for these use-cases.

However, the user experience is bad if [both the site author and the UA](https://stackoverflow.com/questions/19662434/in-ios-7-safari-how-do-you-differentiate-popstate-events-via-edge-swipe-vs-the) add these transitions. The goal of this proposal is to avoid such cases to ensure only one visual transition is executed at a time.

The ethos guiding the API design are:
* If a visual transition can be defined by both the site author and the UA, the site author takes precendence.

* If site authors can not define the transition, they should be able to detect if a visual transition was executed by the UA (assuming no security/privacy concerns).

# Transition Cases
The matrix of navigation cases which define whether a visual transition could be customized by site authors depends on the following characteristics:

* Swipe (predictive) vs Atomic: An atomic navigation would be clicking a button in the browser UI or a programmatic navigation via [`history.pushState`](https://developer.mozilla.org/en-US/docs/Web/API/History/pushState). A swipe navigation would be gesture which shows the user a preview of the destination before initiating the navigation. This is also called predictive navigation since the user can choose to initiate the navigation based on the preview.

* Same vs Cross-Document (Same-Origin): Whether site authors can customize a transition today and which APIs are necessary to support customization depends on whether the navigation is same-document or cross-document.

* Same vs Cross-origin: Customization of visual transition when navigating between cross-origin sites should either not be supported or should be extremely limited for security/privacy concerns.

The table below shows the current state for the cross-section of these cases:

* ✅ are cases which are currently customizable.
* ☑ are cases which should be customizable but require new platform APIs.
* ❌ are cases which should not be customizable

|   | Same-Document  |  Cross-Document Same-Origin | Cross-Origin  |
|---|---|---|---|
| Swipe Navigation | ☑ | ☑ | ❌ |
| Atomic Navigation  | ✅ | ☑ | ❌ |

The platform APIs required for cases marked with ☑ are:

* Gesture API: Most platforms have a swipe gesture which is used by the UA to trigger a back/forward navigation. On Android/iOS its swiping from the screen edge; on Mac, its multi-finger swipe. Site authors currently can not consistently intercept these events. Customizing swipe navigations for both same-document and cross-document navigations requires this[^1].

* Persisting Visual State: The transition requires showing visual state from both before/after the navigation. Site authors can do this for same-document navigations but not cross-document. [ViewTransition](https://github.com/WICG/view-transitions/blob/main/explainer.md#cross-document-same-origin-transitions) addresses this gap.

The exact details for the above APIs is out of scope for this proposal. These are meant to show that there are cases which are currently not customizable by site authors but will be going forward. The API proposals here should work well if/when one of these cases becomes customizable.

# Problem Statement
There is currently no way for a site to indicate that it has a custom visual transition tied to a navigation. For example, if the UA adds a visual transition for an atomic navigation (like clicking the back button), there is no way for the UA and site-author to coordinate the 2 transitions. The results in 2 category of issues:

* Double transitions: Executing both the UA and author defined visual transitions looks like a visual glitch. This [site](https://darkened-relieved-azimuth.glitch.me) shows the issue on Chrome/Safari iOS and Safari on Mac. The site does a cross-fade when the user goes back:

   * If the user navigates using the back button in the UI, a cross-fade defined by the site executes and is the only transition.

   * If the user navigates using a swipe from the edge (iOS) or 2 finger swipe (mac), the UA does a transition showing content of the back entry. The Document then receives a `popState` event for the navigation and performs another visual transition.
   
   This has *compat risk* if the UA adds visual transitions for navigations which can be implicitly customized by the site author, i.e., same-document navigations. This is not a problem for cross-document navigations since customization requires UA primitives like ViewTransitions. The UA can use this to detect if there are custom transitions and give them precedence.
   
* Sub-optimal UX: It's possible that the transition UX defined by the UA is not ideal given the DOM change associated with a navigation. For example, scrolling to a different position in the same Document.

   This can be resolved by giving precedence to author defined transitions over UA transitions if the case is customizable. For cases which can't be customized yet (like swipe navigations), it's unclear whether no visual transition is better than the UA default.

# Proposals
## Choosing between UA and Custom Transition
This API provides authors control over whether a same-document navigation performs a UA transition. The base primitive is a setting called `same-doc-ua-transition` with the following options:

* `disable-atomic`: Disables UA transitions for atomic navigations.
* `disable-swipe`: Disables UA transitions for swipe navigations.

The value is a space seperated list of options.

### Default Value
The default value for this setting depends on whether UA transitions should be an opt-in or opt-out:

* Opt-out has the compat issue of causing double transitions for sites which already have custom transitions. But it allows browser UX to be consistent for same-document vs cross-document navigations. This will also be a behaviour change for Webkit based browsers which already ship this experience.

* Opt-in avoids the compat risk but conservatively disables UA transition on sites which don't have custom transitions.

Ideally we'd want consistent behaviour across browsers here but if its not feasible, the default value can also be left to the UA.

### Handling Swipes
In the absence of a Gesture API for customizing swipes, authors can use this API to handle swipes in the following ways:

1. Add `disable-swipe` which means no visual transition occurs during the gesture. The site instead does a visual transition when the navigation commits (using the `popState` or `navigate` event). This could be sub-optimal for users since they can't use the previous page's preview to decide whether the navigation should occur.
   However, it's unlikely that authors will do this unless a custom transition when the navigation commits is strictly better than the default UA transition.

2. Omit `disable-swipe` which allows the UA to do a visual transition during the gesture. The author can detect this using the API described in [Detecting UA Transition](#detecting-ua-transition).

### Global vs Navigation Based
`same-doc-ua-transition` could be a global setting that applies to all same-document navigations originating from the current Document, or configured based on the source/destination URL. This depends on whether authors have custom transitions for a subset of navigations, or whether the behavior for swipes is set different based on the destination page.

The author could simply update the value of the global setting based on what's in the navigation history, but the following cases are still infeasible:

* The setting needs to be different for the back and forward entries.
* Browser UI allows the user to traverse multiple entries, so back/fwd can jump to any entry in the navigation history.
* The user navigates to a new URL by pasting in the URL bar.

The proposal is to specify this using pair of URLs, which are the source/destination URLs for the navigation. The destination URL must be the value before redirects. This is because the setting is applied when the swipe gesture starts, before the navigation is initiated (which will resolve redirects).

### CSS API
The precise API builds on top of media queries to add a concept of source/destination URLs to CSS.

```css
/* Applies to all same-document navigations from this Document */
same-doc-ua-transition: disable-atomic;

/* Applies to same-document navigations from the current Document if the destination URL matches "to" */
@media (to: urlPattern("/articles/*")) {
  same-doc-ua-transition: disable-atomic disable-swipe;
}

/* Applies to all same-document navigations from the current Document if the current URL matches "from".
   This is syntactic sugar to avoid having to change rules based on what the current URL is. */
@media (from: urlPattern("/articles/*")) {
  same-doc-ua-transition: disable-atomic disable-swipe;
}

@media (from: urlPattern("/articles/*")) and (to: urlPattern("/index")) {
  same-doc-ua-transition: disable-atomic disable-swipe;
}
```

## Detecting UA Transition
For cases where the site is using `disable-atomic`, this tells the author whether a UA has already executed a visual transition. This is needed because whether there was a UA transition depends on whether the navigation was atomic or swipe.

```js
window.addEventListener("popstate", (event) => {
  if (event.hasUAVisualTransition)
    updateDOM(event);
  else
    updateDOMWithVisualTransition(event);
});

navigation.addEventListener("navigate", (event) => {
  if (event.hasUAVisualTransition)
    updateDOM(event);
  else
    updateDOMWithVisualTransition(event);
});
```

This proposal is also documented at [html/8782](https://github.com/whatwg/html/issues/8782). The UA transition may not necessarily finish when this event is dispatched.

[^1]: Note that on iOS, the touch event stream during a back/fwd swipe is dispatched to script: [example](https://scintillating-shadow-pastry.glitch.me/). But the author can not `preventDefault` to take over the gesture. This is different from Android where if the gesture results in a back swipe, the touch events are never dispatched.

# Remaining Use-Cases
## Cross-Document Navigations
TODO

## Subframe Navigations
TODO
