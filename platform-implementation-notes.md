# Close watchers on today's platforms

## Close requests

Although the specification is specifically designed to be future-extensible to new platforms, or to adapt if a platform changes how close signals are communicated, here is how we currently anticipate close requests being implemented.

### On platforms with keyboards

Pressing down on the <kbd>Esc</kbd> key is a close request.

Notes:

* This includes not just desktop platforms, but also mobile or tablet platforms with keyboards attached. In some cases these keyboards don't have an <kbd>Esc</kbd> key explicitly, but do have an equivalent [key combination](https://osxdaily.com/2019/04/12/how-type-escape-key-ipad/).
* This includes virtual keyboards with <kbd>Esc</kbd> keys or key combinations available, e.g. on Windows 11.
* From what we've seen, most operating systems specifically treat pressing down, and not releasing, as the close request.

### When assistive technology is being used

The current plan of the [AOM group](https://github.com/WICG/aom) is that, when an assistive technology's dismiss or escape gesture is used, this will [synthesize an <kbd>Esc</kbd> keypress](https://github.com/WICG/aom/blob/gh-pages/explainer.md#user-action-events-from-assistive-technology). Then, implementations can treat the pressing-down part of this synthesized keypress as a close request, giving [indistinguishable behavior](https://w3ctag.github.io/design-principles/#do-not-expose-use-of-assistive-tech) between assistive technology users and keyboard users.

### On Android

If in the 2-button navigation or 3-button navigation mode is being used for [system navigation](https://support.google.com/android/answer/9079644?hl=en), then the software back button at the bottom of the screen is a close request.

If gesture navigation mode is being used, then swiping from the left or right of the screen is a close request.

Notes:

* In the Chromium implementation, we found that we automatically got support for all three modes of system navigation from the same code path. Hooray!
* Android devices can also be have keyboards attached, or be used with assistive technology; the categories are not mutually exclusive.
* Because the back button is also used, in web apps, for navigating through the session history, there are a number of complications. Anti-back-trapping measures are discussed [in the README](./README.md#abuse-analysis), and below we have an appendix on [interaction with fragment navigation](#appendix-android-and-fragment-navigation).

### On iOS

As of now, we expect that iOS would only trigger close requests if assistive technology is being used, or if a keyboard is attached to the device.

Additionally, we note that the Apple Human Interface Guidelines [indicate](https://developer.apple.com/design/human-interface-guidelines/ios/app-architecture/modality/) that a swipe-down gesture can be used to close certain types of modals. However, we suspect this would not be suitable for integration with `CloseWatcher`, since there's no guarantee that a `CloseWatcher` is being used for a sheet-type modal (as opposed to a fullscreen-type modal, or a non-modal control like a sidebar or picker). We'd welcome more thoughts from iOS experts.

### On game consoles

We are not very confident in the desired user experience for console web browsers, or other apps implemented on consoles using web technology. But in our brief experience, most consoles have a button that is usually used as a "cancel" or "back" button. We expect that this being a close request might work well.

## Appendix: Android and fragment navigation

Consider a popup which contains a lot of text, such as a terms of service with a table of contents, or a preview overlay that shows an interactive version of this very document. Within such a popup, it'd be possible to perform fragment navigation. Although these situations are hopefully rare in practice, we need to figure out how they impact the proposed API.

We suggest that for Android, the back button still acts as a close request whenever any `CloseWatcher` is active. That is, it should only do fragment navigation back through the history list when there are no active close watchers, even if fragment navigation has occurred since the popup was shown and the `CloseWatcher` was constructed.

This does mean that the popup could be closed, while the page's current fragment and its history list still reflect fragment navigations that were performed inside the popup. The application would need to clean up such history entries to give a good user experience. We believe this is best done through a history-specific API, e.g. the navigation API, since this sort of batching and reversion of history entries is a generic problem and not popup-specific.

The alternative is for implementations to treat back button presses as fragment navigations, only treating them as close requests once the user has returned to the same history entry as the popup was opened in. But discussion with framework authors indicated this would be harder to deal with and reason about. In particular, this would introduce significant divergences between platforms: on Android, closing a popup would not require any history-stack-cleaning, whereas on other platforms it would. So, in accordance with our [goal](./README.md#goals) of discouraging platform-specific code, we encourage implementations to go with the model suggested above, which enables web developers to use the same cleanup code without platform-specific branches. (Or they can avoid the problem altogether, by not providing opportunities for fragment navigation while popups are opened.)
