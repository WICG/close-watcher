# Close Watcher Extension: Confirmation

This document contains an extension to [the main proposal](./README.md) which allows asking for confirmation as part of the close-signal-initiated closing flow. The idea would be to add a new event, `beforeclose`, which can be canceled to trigger a browser-mediated "Are you sure?" UI.

## The API

The `beforeclose` event would be used like the following:

```js
watcher.onbeforeclose = e => {
  if (hasUnsavedData) {
    e.preventDefault();
  }
};
```

If the event is canceled via `preventDefault()`, then the browser will show some UI saying something like "Close modal? Changes you made may not be saved." For [abuse prevention](#abuse-analysis), this text is _not_ configurable.

Furthermore, unlike `Window`'s `beforeunload` event, this browser UI does _not_ block the event loop. Instead, it simply has the effect of delaying the `close` event until the user says "Yes".

If the user says "No" (i.e., keep the component open), then the `CloseWatcher` remains active, and can receive future `beforeclose` and `close` events. But once the user says "Yes" (i.e., confirms closing the component), the `CloseWatcher` receives the final `close` event, and becomes inactive.

The exact UI for this confirmation is up to the user agent. It could be a `beforeunload`-style modal dialog, but the resulting "modal on top of a modal" might be ugly. _TODO maybe we can get some mock-ups of better ideas._

Note that the `beforeclose` event is not fired when the user navigates away from the page: i.e., it has no overlap with `beforeunload`. `beforeunload` remains the best way to confirm a page unload, with `beforeclose` only used for confirming a close signal.

If called from within [transient user activation](https://html.spec.whatwg.org/multipage/interaction.html#transient-activation-gated-api), `watcher.signalClosed()` also invokes `beforeclose` event handlers, and will potentially cause a confirmation UI to come up which could delay the `close` event. If called without user activation, then it skips straight to the `close` event.

## Abuse analysis

### Non-configurable strings

We saw much abuse of `Window`'s `beforeunload` event, during the era where it allowed user-provided strings (via `event.returnValue` or the event handler return value). The issue there is putting web-developer-supplied strings into trusted browser UI. Thus, we have designed this API to avoid this; the only action the web developer can take is the binary signal of either canceling the event, or letting it proceed.

We recognize this gives less flexibility than what people are doing today, which is part of why we're unsure about this extension and so are not promoting it to the main proposal yet.

### User in control

Because the user makes the ultimate decision, they are always able to destroy the close watcher with at most one additional click (on the "Yes" button). That is, there's no way to trap the user by creating a `CloseWatcher` which always cancels the `beforeclose` event: doing so will only prevent `CloseWatcher` destruction if the user says "No".

This is especially important on Android, where the back button is used to try to "escape" a site. See the related discussion [in the main README](./README.md#abuse-analysis).

## Realistic example: a custom dialog

For a dialog, which might contain unsaved data, code like the following would work:

```js
class MyCustomDialog extends HTMLElement {
  #watcher;
  #hasUnsavedData;

  get hasUnsavedData() {
    return this.#hasUnsavedData;
  }
  set hasUnsavedData(v) {
    this.#hasUnsavedData = Boolean(v);
  }

  showModal() {
    // Normal dialog implementation left to the reader.

    this.#setupWatcherIfPossible();
    this.addEventListener('close', () => this.#watcher.?destroy());
  }

  close() {
    // Normal dialog implementation left to the reader.

    this.dispatchEvent(new Event('close'));
  }

  #setupWatcherIfPossible() {
    try {
      #watcher = new CloseWatcher();
    } catch {
      // The web developer using the component called showModal() outside of
      // transient user activation, while another watcher was already active.
      return;
    }

    #watcher.onbeforeclose = e => {
      if (this.#hasUnsavedData) {
        e.preventDefault();
      }
    };

    #watcher.onclose = () => {
      this.close();
    };
  }
}
```

## Alternatives considered

### Loosening the abuse protection

Similar to the ["Not gating on transient user activation"](./README.md#not-gating-on-transient-user-activation) section in the main README, if we relax our protections against abuse this whole problem space becomes simpler.

In particular, if we hold ourselves only to the same standards as today's `history.pushState()` anti-abuse measures, we could:

* Allow the original `close` event (described in the main README) to be canceled, as long as such calls to `event.preventDefault()` are similarly throttled.

* Make canceling the `close` event not show any browser UI, but instead just prevent `CloseWatcher` destruction, allowing further interception of close signals.

Note that we would need to keep the user activation requirement for close watcher creation; this would be analogous to the browser intervention where the back button ignores history entries created without user activation.

The end result is that if the user mashes the Android back button hard enough to overcome the throttling, they will destroy the close watcher, and so their next press will navigate back through the history list. Whereas if they calmly tap the back button every second or so, the page would be able to continue calling `event.preventDefault()` on the `close` event, and thus preventing the back button press from being interpreted as a history navigation.

It's unclear whether this kind of relaxation is OK. Essentially, it comes down to a question as to whether the current `history.pushState()` anti-abuse measures are sufficient to protect against back button hijacking, or whether browsers want to maintain the freedom to improve those over time. If the latter is the case, then building `CloseWatcher` on those shifting foundations could be problematic.
