# Close watcher platform-specific implementation notes

## Android: interaction with fragment navigation

Consider a popup which contains a lot of text, such as a terms of service with a table of contents, or a preview overlay that shows an interactive version of this very document. Within such a popup, it'd be possible to perform fragment navigation. Although these situations are hopefully rare in practice, we need to figure out how they impact the proposed API.

In particular, the problem case is again Android, where the back button serves as both a potential close signal and a way of navigating the history list. We suggest that for Android, the back button still acts as a close signal whenever any `CloseWatcher` is active. That is, it should only do fragment navigation back through the history list when there are no active close watchers, even if fragment navigation has occurred since the popup was shown and the `CloseWatcher` was constructed.

This does mean that the popup could be closed, while the page's current fragment and its history list still reflect fragment navigations that were performed inside the popup. The application would need to clean up such history entries to give a good user experience. We believe this is best done through a history-specific API, e.g. the existing `window.history` API or the proposed [app history API](https://github.com/WICG/app-history), since this sort of batching and reversion of history entries is a generic problem and not popup-specific.

The alternative is for implementations to treat back button presses as fragment navigations, only treating them as close signals once the user has returned to the same history entry as the popup was opened in. But discussion with framework authors indicated this would be harder to deal with and reason about. In particular, this would introduce significant divergences between platforms: on Android, closing a popup would not require any history-stack-cleaning, whereas on other platforms it would. So, in accordance with our [goal](#goals) of discouraging platform-specific code, we encourage implementations to go with the model suggested above, which enables web developers to use the same cleanup code without platform-specific branches. (Or they can avoid the problem altogether, by not providing opportunities for fragment navigation while popups are opened.)

## iOS: VoiceOver and keyboard integration

iOS does not have a system-provided back button like Android does. But it still provides some close signals. In particular, [VoiceOver users have a specific gesture they can use](https://support.apple.com/guide/iphone/learn-voiceover-gestures-iph3e2e2281/ios#:~:text=Dismiss%20an%20alert%20or%20return%20to%20the%20previous%20screen), and iOS users who have connected a keyboard to their mobile device can use the <kbd>Esc</kbd> key (or [key combination](https://osxdaily.com/2019/04/12/how-type-escape-key-ipad/)).

Additionally, we note that the Apple Human Interface Guidelines [indicate](https://developer.apple.com/design/human-interface-guidelines/ios/app-architecture/modality/) that a swipe-down gesture can be used to close certain types of modals. However, we suspect this would not be suitable for integration with `CloseWatcher`, since there's no guarantee that a `CloseWatcher` is being used for a sheet-type modal (as opposed to a fullscreen-type modal, or a non-modal control like a sidebar or picker). We'd welcome more thoughts from iOS experts.