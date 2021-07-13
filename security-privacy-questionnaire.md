# Close Watchers: Security and Privacy Questionnaire Answers

The following are the answers to the W3C TAG's [security and privacy self-review questionnaire](https://w3ctag.github.io/security-questionnaire/).

**What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?**

It exposes some information about user interaction, but that information was already available via other means (e.g. `keydown` listeners or the `window.history` API).

**Do features in your specification expose the minimum amount of information necessary to enable their intended uses?**

Yes.

**How do the features in your specification deal with personal information, personally-identifiable information (PII), or information derived from them?**

They do not consume such information.

**How do the features in your specification deal with sensitive information?**

They do not consume such information.

**Do the features in your specification introduce new state for an origin that persists across browsing sessions?**

No.

**Do the features in your specification expose information about the underlying platform to origins?**

To a small extent. In theory, by correlating signals from `CloseWatcher` with other signals (e.g. `keydown` events), one could try to determine what a platform's modal close signal is, and thus roughly into what "bucket" (desktop or not) the user falls. Such determination is fragile and coarse.

**Do features in this specification allow an origin access to sensors on a user’s device?**

No.

**What data do the features in this specification expose to an origin? Please also document what data is identical to data exposed by other features, in the same or different contexts.**

This question seems to be about cross-origin information exposure, which does not happen with this feature.

**Do features in this specification enable new script execution/loading mechanisms?**

No.

**Does this specification allow an origin access to other devices?**

No.

**Do features in this specification allow an origin some measure of control over a user agent’s native UI?**

Yes, in that the user agent may choose to expose its own UI (or platform UI, such as the Android back button) as a close signal. See the discussion about [abuse prevention](./README.md#abuse-analysis) in this regard.

**What temporary identifiers do the features in this specification create or expose to the web?**

None.

**How does this specification distinguish between behavior in first-party and third-party contexts?**

It does not.

**How do the features in this specification work in the context of a browser’s Private Browsing or Incognito mode?**

No differently.

**Does this specification have both "Security Considerations" and "Privacy Considerations" sections?**

Not yet.

**Do features in your specification enable downgrading default security characteristics?**

No.
