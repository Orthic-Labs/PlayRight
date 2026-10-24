# PlayRight architecture

**Revision:** 0.1, 24 October 2026. **Status:** design, pre-implementation. No player code, service login, DRM session or measurement has been produced for this document. Targets are goals to measure against.

## 1. What it is

A desktop media hub for Windows and macOS that:

- opens each subscription video service (Apple TV, Prime Video, JioHotstar, Netflix, YouTube) in its own isolated, persistent sign-in window using the operating system's web engine, one service at a time, torn down when playback stops;
- plays everything else natively: Stremio addon sources, debrid links, torrents, local files;
- stays small when idle because nothing is loaded until you choose something.

It is not a browser, not a downloader, and not a service that hosts or relays anything. Each user signs in to the services they already pay for. What a person streams through the open sources is their responsibility under their jurisdiction's law; the architecture is the same for public-domain films and Linux ISOs as for anything else.

## 2. Principles

1. One active playback session at a time; no hidden tabs, no preloading.
2. A service's web runtime exists only while that service is playing. Stop destroys it; it is never merely suspended.
3. Sign-in profiles survive restarts and updates. Service-imposed expiry is shown, not hidden.
4. Provider pages get no native bridge, no filesystem, no shell, no vault.
5. Measure the whole process tree at equal playback quality against a clean browser window before claiming anything about memory.
6. No central server, telemetry, account or subscription of ours.
7. Keep the platform sandbox, transport validation and the provider's own content-protection flow. No user-agent impersonation, no protection bypass.

## 3. Two kinds of source, two kinds of player

| Source | Player | Why |
|---|---|---|
| Subscription services with DRM (Apple TV, Prime Video, JioHotstar, Netflix) | The OS web engine (WebView2 on Windows, WKWebView on macOS) hosting the provider's official web player in an isolated window | Only the system engines carry the licensed content-decryption modules; no native player can lawfully play these |
| YouTube | The official IFrame Player API or youtube.com signed out, in an isolated window; the signed-in experience is handed to the default browser | Google disables account sign-in inside embedded web views, WebView2 included |
| Stremio addons, debrid links, torrents, local files | A native player (an mpv sidecar process for full format coverage; the web engine's `<video>` with on-the-fly remux as the lighter fallback) | No DRM, so no web engine is needed at all |
| Music (later) | librespot for Spotify; local files through the native player | A native Spotify client already exists in the open; no browser needed |

## 4. Process model

```text
   Hub window (tiles, library, settings)         on demand; closed or hidden while playing
            |
   Engine (Rust): providers, profiles, library, one-playback lease, Stremio addon client, torrent engine
            |
   +--------+-----------------------------+------------------------------+
   |                                      |                              |
 Service window                     Native player                  External handoff
 (OS web engine, own profile,       (mpv sidecar or <video>)       (default browser / provider app;
  created on play, destroyed                                        reported as not app-owned)
  on stop)
```

**Lease.** Starting a source allocates a new generation, stops and destroys the previous player, waits for a bounded shutdown receipt, then starts the next one. Stale events from an old generation are ignored. Login, purchase and rental actions are never retried automatically.

**Profiles.** One persistent store per service and account slot: a separate data directory for WebView2, a separate website data store identifier for WKWebView (macOS 14 and later). Clearing the app's caches never touches a sign-in profile. Sign out, clear cache and delete slot are three different actions.

**Idle.** No service window and no torrent engine exist until the user picks something. The hub itself is small by construction: tiles, a library list and settings.

## 5. Subscription services

Per service: canonical home and content URLs, an allowlist of navigation origins (provider, its CDNs, its authentication origins), an allowlist of popup origins for sign-in, visible-error classification, capability observation (what the page reports about playback, recorded as evidence with its origin), and stop semantics. No shared DOM-selector bundle that breaks all providers at once.

What the design does not promise, and why:

- **Quality.** Browsers receive software-level DRM, so services cap browser playback (commonly 720p or 1080p; Prime documents HD and stereo on computers; JioHotstar names Safari for 1080p). Netflix's higher tiers depend on a hardware DRM path the embedded engine may not expose. The app reports the quality the page reports, or "not reported".
- **Sign-in with passkeys.** WKWebView presents passkeys only for an app's own associated domains or for entitled browsers; WebView2's WebAuthn support in an embedded host is unverified. Service sign-in is therefore password or one-time code. Accounts that are passkey-only are routed to a documented external flow.
- **YouTube signed in.** Not available in-app (section 3). In-app YouTube is signed out: ads, no history, no Premium.
- **Terms.** Services list supported browsers. An embedded system engine is a browser to them; the realistic outcome of an unsupported one is a degraded or blocked page, not an account action. The P0 spike records which it is per service.

A provider page's message such as "signed in" or "playing 4K" is an observation, never an authorisation and never proof of quality.

## 6. Open sources: Stremio, debrid, torrents, local files

- **Stremio addons.** The addon protocol is plain HTTP (manifest, catalog, meta, stream, subtitles). The app embeds `stremio-core` (MIT, Rust), the same engine behind Stremio's own apps, for addon transport, catalogs, metadata, library and player state. Addons such as Cinemeta, OpenSubtitles and torrent indexers are added by URL, including any debrid configuration they carry.
- **Debrid.** A debrid account turns a magnet into a direct HTTPS file link. It is the lowest-resource and simplest path: no peers, no seeding, a plain stream for the native player.
- **Torrents.** `librqbit` (Rust) streams a torrent over a local HTTP endpoint with range requests, prioritises the pieces being played, and idles in tens of megabytes. It runs only while a torrent plays, with a seeding policy the user sets.
- **Local files.** Opened directly by the native player; the library indexes folders the user adds.
- **Native player.** An mpv sidecar process rendering into an app-owned window: MKV, HEVC, AV1, multi-audio, ASS subtitles, HDR passthrough, hardware decode. The lighter alternative for simple files is the web engine's `<video>` fed by an on-the-fly ffmpeg remux to fragmented MP4 with subtitles converted to WebVTT. Neither path loads user scripts or remote resolvers; sources are validated before fetch and private-network fetches require an explicit grant.

## 7. Media keys, keep-awake, power

Both system web engines route hardware media keys to the page's own media session; when a service plays, the page owns the keys and the app registers nothing. For the native player the app publishes now-playing state and receives media keys itself (one owner at a time, never both). A keep-awake assertion is held only while something plays and released on pause, stop, error and quit. Sleep and resume pause and re-acquire the session rather than spinning.

## 8. Storage and privacy

Logical layout: `library.db` (bookmarks, recents, capability receipts; encrypted when private history is on), `settings.json` (non-secret preferences and secret references only), `profiles/<service>/<slot>/` (engine-managed sign-in stores, owner-only permissions), `cache/artwork/` (size-bounded, disposable), redacted logs. Debrid and addon tokens live in the OS credential store. No cookie, token, signed URL, licence payload or page body ever reaches logs or an agent. No telemetry.

## 9. Agents

A local agent (or a script) can list services, request playback of a validated reference, pause, stop, read safe status and ask the user to complete a sign-in, through a typed local interface. Login, account switching, purchases, profile deletion and external launches require the user. There is no `evaluate_javascript`, no cookie export, no stream-key tool.

## 10. Measurement

Record, per OS, the whole app-owned process tree (private working set and commit on Windows, footprint on macOS) in four states: idle hub, service playing, native player playing a torrent or debrid stream, and after fifty start-stop cycles. Pair each service measurement with a clean Edge or Safari window playing the same title at the same quality. Targets to measure against, not promises: idle under 150 MiB on Windows and under 80 MiB on macOS with nothing playing; no retained growth over fifty cycles; UI actions under 100 ms while background work stalls.

## 11. Milestones

| Milestone | Content | Exit |
|---|---|---|
| P0 | A minimal isolated window per service: protected episode and live stream on JioHotstar, Prime and Apple TV; Netflix; sign-in persistence across restart; media keys; whole-tree memory against a clean browser | A receipt or an explicit unsupported result per service and OS |
| P1 | Hub, profiles, lease, close-to-tray, keep-awake, deep links, settings | No service window at launch; teardown returns to baseline |
| P2 | Stremio addons, debrid, `librqbit`, native player, subtitles, local library | A film plays from an addon and from a local file with subtitles and track selection |
| P3 | Per-service policies, YouTube signed-out mode with external handoff, capability display | Real-account tests pass per advertised route |
| P4 | Measurement receipts, hub hidden while playing, service-switch teardown proof | Published per-OS footprint and capability table |
| P5 | Music via librespot; agent interface; polish; signed release | Installed artefact receipts |

## 12. Open questions

Hub toolkit (a thin web-engine window on the shared desktop shell, or a native immediate-mode UI; the player and tiles need no rich text, so both are viable); mpv sidecar versus `<video>` plus remux as the default native player; minimum Windows and macOS versions after P0; licence.

## 13. References

- WebView2 feature differences from Microsoft Edge, including Google Authentication: https://learn.microsoft.com/en-us/microsoft-edge/webview2/concepts/browser-features
- WebView2 user data folders: https://learn.microsoft.com/en-us/microsoft-edge/webview2/concepts/user-data-folder
- WKWebView website data stores and encrypted media (WWDC22 session 10049): https://developer.apple.com/videos/play/wwdc2022/10049/
- YouTube IFrame Player API and required minimum functionality: https://developers.google.com/youtube/iframe_api_reference
- Prime Video computer requirements: https://www.primevideo.com/help?nodeId=GUX9FYHU5D8LC9EJ
- JioHotstar supported devices and quality: https://help.hotstar.com/
- stremio-core (MIT): https://github.com/Stremio/stremio-core
- rqbit / librqbit: https://github.com/ikatson/rqbit
- mpv: https://mpv.io/manual/stable/
