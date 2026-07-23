# Capacitor iOS: `CapacitorCookies` pauses `<video>` playback

Minimal reproduction showing that, with the `CapacitorCookies` plugin enabled on iOS,
**every `document.cookie` read or write pauses any playing `<video>` (or `<audio>`) element**.

## Why it happens

When `CapacitorCookies` is enabled, Capacitor's injected bridge replaces the
`document.cookie` accessor. On iOS the replacement uses `window.prompt()` as a
synchronous JS→native channel
(`node_modules/@capacitor/ios/Capacitor/Capacitor/assets/native-bridge.js`):

```js
Object.defineProperty(document, 'cookie', {
  get: function () {
    if (platform === 'ios') {
      // Use prompt to synchronously get cookies.
      const payload = {
        type: 'CapacitorCookies.get',
      };
      const res = prompt(JSON.stringify(payload));
      return res;
    }
    ...
```

The native side (`WebViewDelegationHandler.swift`, the `WKUIDelegate`
`runJavaScriptTextInputPanelWithPrompt` handler) recognises the
`CapacitorCookies.*` payloads and answers them immediately without presenting
any UI.

However, WebKit treats `window.prompt()` as a modal JavaScript dialog: while the
dialog is pending, the page's active DOM objects — including playing media
elements — are suspended, and playing media is left **paused** afterwards. That
happens in the web process before the embedder's `WKUIDelegate` is consulted, so
answering the prompt natively (without showing a dialog) does not avoid it. You
can see the same base behaviour in plain Safari: play a video, run `prompt('x')`
in the Web Inspector console, and the video pauses.

The result: any code that touches `document.cookie` — analytics snippets, auth
SDKs, ad/consent libraries, your own code — silently pauses video playback in a
Capacitor iOS app with `CapacitorCookies` enabled. Setting a cookie goes through
the same `prompt()` channel (`type: 'CapacitorCookies.set'`) and has the same
effect. The bridge also issues one `prompt()` call for
`CapacitorCookies.isEnabled` on every page load on iOS, even when the plugin is
disabled.

## Project layout

- `capacitor.config.json` — enables the plugin:
  ```json
  "plugins": { "CapacitorCookies": { "enabled": true } }
  ```
- `www/index.html` — the whole test app: a looping muted video, a live
  play/pause status readout, an event log, and buttons that trigger each
  variant of the bridge call.
- `ios/` — not committed; it is 100% stock output of `npx cap add ios`
  (no manual changes), so it is generated as part of the setup steps below.

## Running the repro

Requires macOS with Xcode and CocoaPods.

```bash
npm install
npx cap add ios
npx cap open ios
```

Then build & run on a simulator or device (reproduces on both).

## Steps

1. Tap **“Start test”**. The video starts playing; after 3 seconds the app reads
   `document.cookie` once.
2. Watch the status bar / event log.

**Expected:** reading `document.cookie` has no effect on playback.

**Actual:** the moment `document.cookie` is read, the video fires a `pause`
event and stops. The log shows the cookie read completing in a few ms —
no dialog is ever visible — yet playback is paused.

The other buttons isolate each variant:

- **Read document.cookie** — one getter call → video pauses.
- **Set document.cookie** — setter also goes through `prompt()` → video pauses.
- **Raw prompt() bridge call** — calls
  `prompt(JSON.stringify({type: 'CapacitorCookies.get'}))` directly, exactly
  what the patched getter does internally → video pauses. This isolates the
  root cause independent of the cookie patch.
- **Toggle 2s cookie polling** — simulates a library that polls cookies: the
  video can never play for more than 2 seconds (restart it with the native
  controls and watch it get paused again).

## Control experiment

Set `"enabled": false` for `CapacitorCookies` in `capacitor.config.json` and run
`npx cap sync ios` again. The page header shows the `document.cookie` patch as
inactive, and the read/set buttons no longer pause the video — while the
**Raw prompt() bridge call** button still does, confirming the `prompt()`-based
bridge (not cookie access itself) is what pauses playback.

## Versions

- `@capacitor/core` / `@capacitor/cli` / `@capacitor/ios`: 7.6.8 (pinned
  exactly in `package.json`)
- Video: any playing HTML5 media element; this repro streams a public sample
  MP4 (Big Buck Bunny). To run fully offline, drop a small MP4 at
  `www/assets/sample.mp4` — the page prefers it automatically (re-run
  `npx cap sync ios` after adding it).
