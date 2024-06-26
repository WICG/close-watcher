# Close Watchers

_Read on for the full story, or check out [the demo](https://close-watcher-demo.glitch.me/)!_

## The problem

Various UI components have a "modal" or "popup" behavior. For example:

- a `<dialog>` element, especially the [`showModal()` API](https://developer.mozilla.org/en-US/docs/Web/API/HTMLDialogElement/showModal);
- a sidebar menu;
- a lightbox;
- a custom picker input (e.g. date picker);
- a custom context menu;
- fullscreen mode.

An important common feature of these components is that they are designed to be easy to close, with a uniform interaction mechanism for doing so. Typically, this is the <kbd>Esc</kbd> key on desktop platforms, and the back button on some mobile platforms (notably Android). Game consoles also tend to use a specific button as their "close/cancel/back" button. Finally, accessibility technology sometimes provides specific close requests for their users, e.g. iOS VoiceOver's "dismiss an alert or return to the previous screen" [gesture](https://support.apple.com/guide/iphone/learn-voiceover-gestures-iph3e2e2281/ios#:~:text=Dismiss%20an%20alert%20or%20return%20to%20the%20previous%20screen).

We define a **close request** as a _platform-mediated_ interaction that's intended to close an in-page component. This is distinct from _page-mediated_ interactions, such as clicking on an "x" or "Done" button, or clicking on the backdrop outside of the modal. See our [platform-specific notes](./platform-implementation-notes.md) for more details on what close requests look like on various platforms.

Currently, web developers have no good way to handle these close requests. This is especially problematic on Android devices, where the back button is the traditional close request. Imagine a user filling in a twenty-field form, with the last item being a custom date picker modal. The user might click the back button hoping to close the date picker, like they would in a native app. But instead, the back button navigates the web page's history tree, likely closing the whole form and losing the filled information. But the problem of how to handle close requests extends across all operating systems; implementing a good experience for users of different browsers, platforms, and accessibility technologies requires a lot of user agent sniffing today.

This explainer proposes a new API to enable web developers, especially component authors, to better handle these close requests.

## Goals

Our primary goals are as follows:

- Allow web developers to intercept close requests (e.g. <kbd>Esc</kbd> on desktop, back button on Android, "z" gesture on iOS using VoiceOver) to close custom components that they create.

- Discourage platform-specific code for handling modal close requests, by providing an API that abstracts away the differences.

- Allow future platforms to introduce new close requests (e.g., the introduction of the swipe-from-the-sides dismiss gesture in Android 10), such that code written against the API proposed here automatically works on such platforms.

- Prevent abuse that traps the user on a given history state by disabling the back button's ability to actually navigate backward in history.

- Be usable by component authors, in particular by avoiding any global state modifications that might interfere with the app or with other components.

- Specify uniform-per-platform behavior across existing platform-provided components, e.g. `<dialog>`, `<input>` pickers, and fullscreen. (For example: currently the back button on Android does not close modal `<dialog>`s, but instead navigates history. It does close `<input>` pickers and close fullscreen.)

- Allow the developer to confirm a close request with the user, to avoid potential data loss. (E.g., "are you sure you want to close this dialog?")

- Explain `<dialog>`'s existing `close` and `cancel` events in terms of this model.

The following is a goal we wish we could meet, but don't believe is possible to meet while also achieving our primary goals:

- Avoid an awkward transition period, where the Android back button closes components on sites that adopt this new API, but navigates history on sites that haven't adopted it. In particular, right now users generally know the Android back button fails to close modals in web apps and avoid it when modals are open; we worry about this API causing a state where users are no longer sure which action it performs.

## What developers are doing today

On desktop platforms, this problem is currently solved by listening for the `keydown` event, and closing the component when the <kbd>Esc</kbd> key is pressed. Built-in platform APIs, such as [`<dialog>`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLDialogElement/showModal), [fullscreen](https://developer.mozilla.org/en-US/docs/Web/API/Fullscreen_API), or [`<input type="date">`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/input/date), will do this automatically. Note that this can be easy to get wrong, e.g. by accidentally listening for `keyup` or `keypress`.

On mobile platforms, getting the right behavior is significantly harder. First, platform-specific code is required, to do nothing on iOS, capture the back button or swipe-from-the-sides gesture on Android, and listen for gamepad inputs on PlayStation browser.

Next, capturing the back button on Android requires manipulating the history list using the [history API](https://developer.mozilla.org/en-US/docs/Web/API/History_API). This is a poor fit for several reasons:

- This UI state is non-navigational, i.e., the URL doesn't and shouldn't change when opening a component. When the page is reloaded or shared, web developers generally don't want to automatically re-open the same component.

- It's very hard to determine when the component's state is popped out of history, because both moving forward and backward through history emit the same `popstate` event, and that event can be emitted for history navigation that traverses more than one entry at a time.

- The `history.state` API, used to store the component's state, is rather fragile: user-initiated fragment navigation, as well as other calls to `history.pushState()` and `history.replaceState()`, can override it.

- History navigation is not cancelable. Implementing "Are you sure you want to close this dialog without saving?" requires letting the history navigation complete, then potentially re-establishing the dialog's state if the user declines to close the dialog.

- A shared component that attempts to use the history API to implement these techniques can easily corrupt a web application's router.

## Proposal

The proposal is to introduce a new API, the `CloseWatcher` class, which has the following basic API:

```js
const watcher = new CloseWatcher();

// This fires when the user sends a close request, e.g. by pressing Esc on
// desktop or by pressing Android's back button.
watcher.onclose = () => {
  myModal.close();
};

// You should destroy watchers which are no longer needed, e.g. if the
// modal closes normally. This will prevent future events on this watcher.
myModalCloseButton.onclick = () => {
  watcher.destroy();
  myModal.close();
};
```

If more than one `CloseWatcher` is active at a given time, then only the most-recently-constructed one gets events delivered to it. ([Usually](#user-activation-grouping).) A watcher becomes inactive after a `close` event is delivered, or the watcher is explicitly `destroy()`ed.

Read on for more details and realistic usage examples.

### Requesting close yourself

The API has an additional convenience method, `watcher.requestClose()`, which acts _as if_ a close request had been sent by the user. The intended use case is to allow centralizing close-handling code. So the above example of

```js
watcher.onclose = () => myModal.close();

myModalCloseButton.onclick = () => {
  watcher.destroy();
  myModal.close();
};
```

could be replaced by

```js
watcher.onclose = () => myModal.close();

myModalCloseButton.onclick = () => watcher.requestClose();
```

deduplicating the `myModal.close()` call by having the developer put all their close-handling logic into the watcher's `close` event handler.

As usual, reaching the `close` event will inactivate the `CloseWatcher`, meaning it receives no further events in the future and it no longer occupies the "free `CloseWatcher` slot", if it was previously doing so.

### Asking for confirmation

There's a more advanced part of this API, which allows asking the user for confirmation if they really want to close the modal. This is useful in cases where, e.g., the modal contains a form with unsaved data. The usage looks like the following:

```js
watcher.oncancel = async (e) => {
  if (hasUnsavedData) {
    e.preventDefault();

    const userReallyWantsToClose = await askForConfirmation("Are you sure you want to close this dialog?");
    if (userReallyWantsToClose) {
      hasUnsavedData = false;
      watcher.close();
    }
  }
};
```

_Note: the name of this event, i.e. `cancel` instead of something like `beforeclose`, is chosen to match `<dialog>`, which has the same two-tier `cancel` + `close` event sequence._

For abuse prevention purposes, this event only fires if the page has received [transient user activation](https://html.spec.whatwg.org/multipage/interaction.html#transient-activation-gated-api). Furthermore, once it fires for one `CloseWatcher` instance, it will not fire again for any `CloseWatcher` instances until the page gets user activation again. This ensures that if the user sends a close request twice in a row without any intervening user activation, the request definitely goes through, destroying the `CloseWatcher`.

Note that the `cancel` event is not fired when the user navigates away from the page: i.e., it has no overlap with `beforeunload`. `beforeunload` remains the best way to confirm a page unload, with `cancel` only used for confirming a close request.

If called from within transient user activation, `watcher.requestClose()` also invokes `cancel` event handlers, which would trigger event listeners like the above example code. If called without user activation, then it skips straight to the `close` event.

### User activation grouping

In addition to the user activation restrictions for `cancel` events, mentioned [above](#asking-for-confirmation), there is a more subtle form of user activation gating for `CloseWatcher` construction. The basic idea is that if you have created more than one `CloseWatcher` without user activation, then the newly-created one will get grouped together with the most-recently-created close watcher, so that a single close request will close them both. This is meant to prevent [abuse](#abuse-analysis).

So for example:

```js
window.onload = () => {
  // This will work as normal: it is the first close watcher created without user activation.
  (new CloseWatcher()).onclose = () => { /* ... */ };
};

button1.onclick = () => {
  // This will work as normal: the button click counts as user activation.
  (new CloseWatcher()).onclose = () => { /* ... */ };
};

button2.onclick = () => {
  // These will be grouped together, and both will close in response to a singe close request.
  (new CloseWatcher()).onclose = () => { /* ... */ };
  (new CloseWatcher()).onclose = () => { /* ... */ };
};
```

Note that for developers, this means that calling `watcher.destroy()` properly is important. Doing so is the only way to get back the "free" ungrouped close watcher slot, that allows you to create an ungrouped `CloseWatcher` even without user activation. Such `CloseWatcher`s are useful for cases like session inactivity timeout dialogs, or urgent notifications of server-triggered events, that want to be closable with a close request but are not created via user activation.

### Using `AbortSignal`s to destroy `CloseWatcher`s

As discussed above, destroying a `CloseWatcher` using `watcher.destroy()` is helpful to avoid any future events and free up the free `CloseWatcher` slot.

Another way to destroy `CloseWatcher`s is by using `AbortSignal` objects, like so:

```js
const controller = new AbortController();
const watcher = new CloseWatcher({ signal: controller.signal });

// ... later ...
controller.abort();
```

If the `AbortSignal` is only being used for the `CloseWatcher`, this is not that helpful. But it works nicely when you are using the `AbortSignal` to abort many ongoing operations or dispose of many different resources.

### Abuse analysis

As discussed above, we have various forms of user activation gating. These are meant to prevent abuse for platforms like Android where the close request is to use the back button. There, we need to prevent abuse that traps the user on a page by effectively disabling their back button.

In detail, a malicious page which wants to trap the user would be able to do at most the following using the `CloseWatcher` API:

- If the user never interacts with the page:
  - Create its "free" (i.e., not user-activation-consuming, but also not grouped) `CloseWatcher`
  - The first Android back button press would destroy the `CloseWatcher`
  - The second Android back button press would navigate back through the joint session history, escaping the abusive page
- If the user activates the page once:
  - Create its free `CloseWatcher` at page start
  - Install a `cancel` event handler that calls `event.preventDefault()`
  - The first Android back button press would hit the `cancel` event, which calls `event.preventDefault()`, and thus does nothing
  - The second Android back button press would ignore the `cancel` event, and destroy the `CloseWatcher`
  - The third Android back button press would navigate back through the joint session history, escaping the abusive page

Other variations are possible; e.g. if the user activates the page once, the abusive page could create two `CloseWatcher`s instead of one, but then it wouldn't get `cancel` events, so three back button presses would still escape the abusive page.

In general, if the user activates the page <var>N</var> times, a maximally-abusive page could make it take <var>N</var> + 2 back button presses to escape.

Compare this to the protection in place today for the `history.pushState()` API, which is another means by which apps can attempt to trap the user on the page by making their session history list grow. In [the spec](https://html.spec.whatwg.org/#shared-history-push/replace-state-steps), there is an optional step that allows the user agent to ignore these method calls; in practice, this is only done as a throttling measure to avoid hundreds of calls per second overwhelming the history state storage implementations.

Another mitigation that browsers implement against `history.pushState()`-based trapping is to try to have the actual back button UI skip entries that were added without user activation, i.e., to have it behave differently from `history.back()` (which will not skip such entries). Such attempts [are a bit buggy](https://bugs.chromium.org/p/chromium/issues/detail?id=1248529), but in theory they woud mean an abusive page that is activated <var>N</var> times would require <var>N</var> + 1 back button presses to escape.

We believe that the additional capabilities allowed here are worth expanding this number from <var>N</var> + 1 to <var>N</var> + 2, especially given how unevenly these mitigations are currently implemented and how that hasn't led to significant user complaints. However, there are a number of alternatives discussed below which would allow us to lower this number:

- [No free close watcher](#no-free-close-watcher) would reduce the number of Android back button presses by 1.
- [Browser-mediated confirmation dialogs](#browser-mediated-confirmation-dialogs) would reduce the number of Android back button presses by 1, although it would replace it with another press.

Note that [no confirmation dialogs](#no-confirmation-dialogs) would not reduce the number, since in our current design the user activations are shared between `CloseWatcher` creation and the `cancel` event, so removing just the `cancel` event does not change the calculus.

For now we are sticking with the current proposal and its <var>N</var> + 2 number, but welcome discussion from the community as to whether this is the right tradeoff.

Finally, we note that in most browser UIs, the user has an escape hatch of holding down the back button and explicitly choosing a history step to navigate back to. Or, closing the tab entirely. These are never a close request.

### Realistic examples

The above sections give illustrative usage of the API. The following ones show how the API could be incorporated into realistic apps and UI components.

#### A sidebar

For a sidebar (e.g. behind a hamburger menu), which wants to hide itself on a user-provided close request, that could be hooked up as follows:

```js
const hamburgerMenuButton = document.querySelector('#hamburger-menu-button');
const sidebar = document.querySelector('#sidebar');

hamburgerMenuButton.addEventListener('click', () => {
  const watcher = new CloseWatcher();

  sidebar.animate([{ transform: 'translateX(-200px)' }, { transform: 'translateX(0)' }]);

  watcher.onclose = () => {
    sidebar.animate([{ transform: 'translateX(0)' }, { transform: 'translateX(-200px)' }]);
  };

  // Close on clicks outside the sidebar.
  document.body.addEventListener('click', e => {
    if (e.target.closest('#sidebar') === null) {
      watcher.close();
    }
  });
});
```

Note that it never really makes sense to use the `cancel` event for a sidebar.

#### A picker

For a "picker" control that wants to close itself on a user-provided close request, code like the following would work:

```js
class MyPicker extends HTMLElement {
  #button;
  #overlay;
  #watcher;

  constructor() {
    super();
    this.#button = /* ... */;

    this.#overlay = /* ... */;
    this.#overlay.hidden = true;
    this.#overlay.querySelector('.close-button').addEventListener('click', () => {
      this.#watcher.requestClose();
    });

    this.#button.onclick = () => {
      this.overlay.hidden = false;

      this.#watcher = new CloseWatcher();
      this.#watcher.onclose = () => this.overlay.hidden = true;
    }
  }
}
```

Similarly, picker UIs do not usually require confirmation on closing, so do not need `cancel` event handlers.

## Platform close requests

With `CloseWatcher` as a foundation, we can work to unify the web platform's existing and upcoming close requests:

### Explaining `<dialog>`

The `<dialog>` spec today states that "user agents may provide a user interface that, upon activation, \[cancels the dialog]". In particular, here canceling the dialog first fires a `cancel` event, and if the web developer does not call `event.preventDefault()`, it will close the dialog.

The existing `<dialog>` implementation in Chromium implements this, but only with with <kbd>Esc</kbd> key on desktop. That is, on Android Chromium, the system back button will not close `<dialog>`s. (Perhaps this is because of the fears about back button trapping mentioned above?)

Our proposal is to replace the vague specification sentence above with [text based on close watchers](https://whatpr.org/html/9462/interactive-elements.html#dom-dialog-showmodal). This has a number of benefits:

- It allows the Android back button to close dialogs, subject to anti-abuse restrictions. That is: the call to `dialogEl.showModal()` must be done with user activation or use up the free close watcher slot; and the `cancel` event will only fire if there's been an intervening user interaction; and if you open multiple `<dialog>`s within a single user activation, after using up your free one, a single close request will close both of them.

- It makes it clear how `<dialog>`s interact with `CloseWatcher` instances: they both live in the same per-`Window` close watcher stack.

- It drives interoperability in terms of user- and developer-facing `<dialog>` behavior by providing a more concrete specification, e.g. with regards to how the `<dialog>`'s `cancel` event is sequenced versus the `keydown` event for <kbd>Esc</kbd> key presses.

Note that the user activation "resource" is shared between everything that uses the general close watcher infrastructure: i.e., both `<dialog>` elements, and `CloseWatcher`s. So, for example, creating a `<dialog>` without user activation uses up the free close watcher slot, so that if you then proceed to create another `<dialog>` and construct a `CloseWatcher`, all without user activation, that `CloseWatcher` and the second `<dialog>` will be grouped together and both be closed by a single close request.

### Integration with Fullscreen

The Fullscreen spec today states "If the end user instructs the user agent to end a fullscreen session initiated via `requestFullscreen()`, fully exit fullscreen". Existing Fullscreen implementations implement this using the <kbd>Esc</kbd> key on desktop, the back button on Android, and a floating software "x" button on iPadOS. (iOS on iPhones does not appear to implement the fullscreen API.)

We propose replacing this with [explicit integration into the close request steps](https://whatpr.org/html/9462/interaction.html#close-request-steps). Again, this gives interoperability benefits by using a shared primitive, and a clear specification for how it interacts with `<dialog>`s, `CloseWatcher`s, and key events.

### Integration with `popover=""`

The [`popover=""` attribute](https://html.spec.whatwg.org/#the-popover-attribute) can benefit from similar integration as `<dialog>`. See our [proposed spec text](https://whatpr.org/html/9462/popover.html#popover-close-watcher).

### Integration with `<input>`?

We could update the specification for `<input>` to mention that it should use close watchers when the user opens the input's picker UI. This kind of unification fits well with the goals of this project, but it might be tricky since the existing `<input>` specification is intentionally vague on how UI is presented.

## Alternatives considered

### Integration with the history API

Because of how the back button on Android is a close request, one might think it natural to make handling close requests part of either the existing history API, or a revised history API. This idea has some precedent in mobile application frameworks that integrate modals into their "back stack".

However, on the web the history API is intimately tied to navigations, the URL bar, and application state. Using it for UI state is generally not great. See also the [above discussion](#what-developers-are-doing-today) of how developers are forced to use the history API today for this purpose, and how poorly it works. In fact, we're hopeful that by tackling this proposal separately from the history API, other efforts to improve the history API will be able to focus on actual navigations, instead of on close requests.

Note that the line between "UI state" and "a navigation" can be blurry in single-page applications. For example, [Twitter.com](https://twitter.com/)'s logged-in view lets you type directly into a "What's happening?" text box in order to tweet, which we classify as UI state. But if you click the "Tweet" button on the sidebar, it [navigates to a new URL](https://twitter.com/compose/tweet) which displays a lightbox into which you can input your tweet. In our taxonomy, this new-URL lightbox is a navigation, and it would not be suitable to use the `CloseWatcher` API for it, because closing it needs to update the URL back to what it was originally (i.e., navigate backwards in the history list).

### Automatically translating all close requests to <kbd>Esc</kbd>

If we assume that developers already know to handle the <kbd>Esc</kbd> key to close their components, then we could potentially translate other close requests, like the Android back button, into <kbd>Esc</kbd> key presses. The hope is then that application and component developers wouldn't have to update their code at all: if they're doing the right thing for that common desktop close request, they would suddenly start doing the right thing on other platforms as well. This is especially attractive as it could help avoid the awkward transition period mentioned in the [goals](#goals) section.

However, upon reflection, such a solution doesn't really solve the general problem. Given an Android back button press, or a PlayStation circle button press, or any other gesture which might serve multiple context-dependent purposes, the browser needs to know: should perform its usual action, or should it be translated to an <kbd>Esc</kbd> key press? For custom components, the only way to know is for the web developer to tell the browser that a close-request-consuming component is open. So our goal of requiring no code modifications, or awkward transition period, is impossible. We'd still need some API, a counterpart to our `new CloseWatcher()`, which tells the browser that the next close request should be turned into an <kbd>Esc</kbd> keypress. Given this, the strangeness of synthesizing fake <kbd>Esc</kbd> key presses in response to some other setup API does not have much to recommend it.

### A single event

Why do we need the `CloseWatcher` class? Why couldn't we just fire a global `close` event or similar, which abstracts over platform differences in close requests?

The problem here is similar to the previous idea of translating all close requests into an <kbd>Esc</kbd> key press. On some platforms, a close request has an important fallback behavior. (Notably, on Android, where the fallback behavior is to perform a history navigation.) This means developers need some way of signaling to the browser that the next instance of such a close request should be directed to their code, instead of performing that fallback action. And, we need to [gate](#user-activation-grouping) the ability to redirect close requests in that way behind user activation.

A less fundamental benefit is for developer ergonomics. By having essentially one event per close watcher, we get a kind of stack for free. For example, if various modal or popup components (including `<dialog>` elements) use `CloseWatcher`s, then we can be sure to always route the event to the "topmost" (i.e., most-recently-created) modal. If we had just one global event, then such components would need to coordinate with each other to ensure that only the topmost acts on the event, and the others ignore it. In other words, the stack of `CloseWatcher` instances, which is pushed onto by the constructor and popped off of by `CloseWatcher` destruction, is a nice bonus for web developers. Our experience implementing the close watcher concept in Chromium, and sharing the infrastructure between `CloseWatcher`, `<dialog>`, and `popover=""`, supports this ergonomic argument; the close watcher stack infrastructure is somewhat complex, but can be nicely encapsulated away from its usage sites.

### Browser-mediated confirmation dialogs

[Previous iterations of this proposal](https://github.com/WICG/close-watcher/blob/5233b324f3b45e867d7e0a9fd02566d532fec850/confirmation.md) had a different semantic for the `cancel` event, where calling `event.preventDefault()` would show non-configurable browser UI asking to confirm closing the modal.

The benefit of this approach is that, because the browser directly gets a signal from the user when the user says "Yes, close anyway", we can reduce the number of Android back button clicks that [abusive pages](#abuse-analysis) can trap from <var>N</var> + 2 to <var>N</var> + 1. However, on balance this isn't actually a big win, because the user still has to perform <var>N</var> + 2 actions to escape: <var>N</var> + 1 back button presses, and one "Yes, close anyway" press.

Additionally, some early feedback we got was that custom in-page confirmation UI was very desirable for web developers, instead of non-configurable browser UI.

### No confirmation dialogs

It's also possible to start with a version of this proposal that does not fire a `cancel` event at all. This means that pressing the Android back button will always destroy the close watcher.

However, by itself this this doesn't change what it takes to escape an [an abusive page](#abuse-analysis): it still takes <var>N</var> + 2 Android back button presses, since a maximally-abusive page will just take advantage of user activation to create new `CloseWatcher` instances, instead of bothering with `cancel` events.

In talking with web developers, this variant of the proposal was not as preferable:

- By default, it would mostly be usable for cases like simple pickers or sidebars. It would not be suitable for anything that involved more complex data entry, like a custom dialog.

- Developers of non-abusive pages that still wanted to use the `CloseWatcher` API would end up hacking around this limitation by trying to create new spare `CloseWatcher`s on user activation, so that they could use up one of them for a confirmation and another for actually closing the dialog. Or, they would ignore the `CloseWatcher` API and go back to [hacky history manipulations](#what-developers-are-doing-today). Thus mainline use scenarios become much more awkward.

### No free close watcher

As discussed [above](#user-activation-grouping), this proposal gives pages the ability to create one "free" `CloseWatcher`, which doesn't get grouped with others, even without user activation. This is meant for cases like session inactivity timeout dialogs or urgent notifications of server-triggered events that want a presentation that is closable with a close request. However, it comes with a cost in terms of allowing [abusive pages](#abuse-analysis) to add 1 to the number of Android back button presses necessary to escape.

We could remove this ability from the proposal, always grouping together `CloseWatcher`s created without user activation. This would reduce the number of Android back button presses by 1, from <var>N</var> + 2 to <var>N</var> + 1.

### Bundling this with high-level APIs

The proposal here exposes `CloseWatcher` as a primitive. However, watching for close requests is only a small part of what makes UI components difficult. Some UI components that watch for close requests also need to deal with [top layer interaction](https://github.com/whatwg/html/issues/4633), [blocking interaction with the main document](https://github.com/whatwg/html/issues/897) (including trapping focus within the component while it is open), providing appropriate accessibility semantics, and determining the appropriate screen position.

Instead of providing the individual building blocks for all of these pieces, it may be better to bundle them together into high-level semantic elements. We already have `<dialog>`; we could imagine others such as `<toast>`, `<tooltip>`, `<sidebar>`, etc. These would then bundle the appropriate features, e.g. while all of them would benefit from top layer interaction, `<toast>` and `<tooltip>` do not need to handle close requests.

Our current thinking is that we should produce both paths: we should work on bundled high-level APIs, such as the existing `<dialog>`, but we should also work on lower-level components, such as close requests, `popover=""`, top layer management, or focus trapping. And we should, as this document [tries to do](#platform-close-requests), ensure these build on top of each other. This gives authors more flexibility for creating their own novel components, without waiting to convince implementers of the value of baking their high-level component into the platform, while still providing an evolutionary path for evolving the built-in control set over time.

## Security and privacy considerations

Much of these issues were discussed above. See the [W3C TAG Security and Privacy Questionnaire answers](./security-privacy-questionnaire.md) for more. To summarize:

### Security considerations

The main security consideration with this API is preventing abusive pages from hijacking the fallback behavior in the last part of the [close request steps](https://whatpr.org/html/9462/interaction.html#close-request-steps). A concrete example is on Android, where the close request is the software back button, and this fallback behavior is to traverse the history by a delta of &minus;1. If developers could always intercept Android back button presses via `CloseWatcher` instances and `<dialog>` elements, then they could effectively break the back button by never letting it pass through to the fallback behavior.

Much of the complexity of this specification is designed around preventing such abuse. Without it, the API could consist of a single event. (Although that would make it ergonomically difficult for developers to coordinate on which component of their application should handle the close request.) But with this constraint, we need an API surface such as the `new CloseWatcher()` constructor which can note whether transient activation was present at construction time, as well as the close watcher stack to ensure that we remove at least one close watcher per close request.

Concretely, the mechanism of creating a close watcher ensures that web developers can only create `CloseWatcher` instances, or call `preventDefault()` on `cancel` events, by attempting to consume user activation. If a `CloseWatcher` instance is created without transient activation, then after one "free" close watcher, any such close watcher is grouped together with the currently-top close watcher in the stack, so that both of them are closed by a single close request. This gives similar protections to what browsers have in place today, where back button UI skips entries that were added without user activation. Similarly, if there hasn't been any transient activation, the `cancel` event is not fired.

We do allow one "free" ungrouped `CloseWatcher` to be created, even without transient activation, to handle cases like session inactivity timeout dialogs, or urgent notifications of server-triggered events. The end result is that this specification expands the number of Android back button presses that a maximally-abusive page could require to escape from <var>number of user activations</var> + 1 to <var>number of user activations</var> + 2. (See [above](#abuse-analysis) for a full analysis.) We believe this tradeoff is worthwhile.

### Privacy considerations

We believe the privacy impact of this API is minimal. The only information it gives about the user to the web developer is that a close request has occurred, which is a very infrequent and coarse piece of user input.

In all cases we're aware of today, such close requests are already detectable by web developers (e.g., by using `keydown` listeners on desktop or `popstate` listeners on Android). In theory, by correlating these existing events with the `CloseWatcher`'s `close` event, a web developer could determine some information about the platform. (I.e., if they correlate with `keydown` events, the user is likely on desktop, or at least on a keyboard-attached mobile device.) This is similar to existing techniques which detect whether touch events or mouse events are fired, and user agents which want to emulate a different platform in order to mask the user's choice might want to apply similar mitigation techniques for close watchers as they do for other platform-revealing events.

## Stakeholder feedback

- W3C TAG: [w3ctag/design-review#594](https://github.com/w3ctag/design-reviews/issues/594)
- Browser engines:
  - Chromium: Positive; prototyped behind a flag
  - Gecko: [Positive](https://github.com/mozilla/standards-positions/issues/604)
  - WebKit: [No feedback so far](https://github.com/WebKit/standards-positions/issues/215)
- Web developers: [some positive signals](https://github.com/WICG/proposals/issues/18); see also [this comment](https://github.com/w3ctag/design-reviews/issues/594#issuecomment-890257686)

## Acknowledgments

This proposal is based on [an earlier analysis](https://github.com/slightlyoff/history_api/blob/67c29f3bd7710f2adff737bd665cd9338a68b25d/history_and_modals.md) by [@dvoytenko](https://github.com/dvoytenko).
