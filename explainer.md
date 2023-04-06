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

The table below shows that currently only same-document atomic navigations can be customized.

|   | Same-Document  |  Cross-Document Same-Origin | Cross-Origin  |
|---|---|---|---|
| Swipe Navigation | ❌ | ❌ | ❌ |
| Atomic Navigation  | ✅ | ❌ | ❌ |

Ideally all same-origin navigations should be customizable but the following API gaps remain to enable that:

* Gesture API: Most platforms have a swipe gesture which is used by the UA to trigger a back/forward navigation. On Android/iOS its swiping from the screen edge; on Mac, its multi-finger swipe. Site authors currently can not consistently intercept these events. Customizing swipe navigations for both same-document and cross-document navigations requires this[^1].

* Persisting Visual State: The transition requires showing visual state from both before/after the navigation. Site authors can do this for same-document navigations but not cross-document. [ViewTransition](https://github.com/WICG/view-transitions/blob/main/explainer.md#cross-document-same-origin-transitions) addresses this gap.

The exact details for the above APIs is out of scope for this proposal. These are meant to show that there are cases which are currently not customizable by site authors but will be going forward. The API proposals here should work well if/when one of these cases becomes customizable.

# Problem Statement
There is currently no way for a site to indicate that it has a custom transition which should override the default UA transition. For example, if the UA adds a visual transition for an atomic navigations (like clicking the back button) which the site has also customized.

## Same-Document Navigations
For same-document navigations the UA can not detect whether a site will execute a visual transition for a navigation. Also, while the site can not currently customize the swipe gesture, it may still do a visual transition when the navigation is eventually committed. Adding UA transitions for these navigations has a compat risk of "double transitions". This [site](https://darkened-relieved-azimuth.glitch.me) shows the issue on Chrome/Safari iOS and Safari on Mac. The site does a cross-fade when the user goes back:

* If the user navigates using the back button in the UI, a cross-fade defined by the site executes and is the only transition.

* If the user navigates using a swipe from the edge (iOS) or 2 finger swipe (mac), the UA does a transition showing content of the back entry. The Document then receives a `popState` event for the navigation and performs another visual transition.

## Cross-Document Navigations
This is not a problem for cross-document navigations since customization requires UA primitives. The UA can use this to detect if there are custom transitions and give them precedence. For example:

* Since atomic navigations will require a ViewTransition, the UA transition is suppressed only if the current Document has an opt-in for ViewTransitions for a navigation.

* Since swipe navigations will require a combination of a Gesture API + ViewTransition, the UA transition is suppressed only if the current Document has an opt-in for both. If there is an opt-in for only one of them and a swipe occurs, the opt-in is ignored.

# Proposals
## Choosing between UA and Custom Transition
This API provides authors control over whether a same-document navigation uses a UA vs custom visual transition. A global setting called `same-document-ua-transition` is introduced with the following options:

* `disable`: Suppresses UA transitions for all navigations.
* `enable`: Enables UA transitions for all navigations.
* `enable-atomic`: Enables UA transitions for atomic navigations only.
* `enable-swipe`: Enables UA transitions for swipe navigations only.
* `auto`: Allows the UA to decide which navigations get a UA transition.

`auto` is the default value for this setting. This allows UAs to make different decisions based on their current behaviour. For example, Safari already ships with transitions on swipe gestures; changing that would surprise users.

Question: It seems awkward to have this setting dictate behaviour for swipes. Especially since the Gesture API for swipes will need their own opt-in.

## Detecting UA Transition
For cases where the site is using `auto`, this tells the author whether a UA has already executed a visual transition for a navigation:

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

This proposal is also documented at [html/8782](https://github.com/whatwg/html/issues/8782).

[^1]: Note that on iOS, the touch event stream during a back/fwd swipe is dispatched to script: [example](https://scintillating-shadow-pastry.glitch.me/). But the author can not `preventDefault` to take over the gesture. This is different from Android where if the gesture results in a back swipe, the touch events are never dispatched.
