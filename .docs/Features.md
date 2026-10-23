# Feature catalogue

How to use this file: every row has a **Decision** checklist, `[ ] Keep [ ] Remove [ ] Disable`. Mark one by changing its `[ ]` to `[x]`. Markdown has no real radio buttons inside tables, so this is plain text you edit in Zed.
- **Keep:** leave it as is.
- **Remove:** delete the code.
- **Disable:** keep the code but switch it off with a build flag, pref or policy.

**Suggested** is Claude's recommendation for a personal, privacy-respecting fork; ignore it freely. Rows that are already ticked record decisions you have made (see the telemetry decision below).

**What it does** is the one-line summary. **Details** explains how the feature works, then its impact on **privacy** and **performance** where that matters.

**Difficulty** is how hard it is to delete cleanly:
- **Low:** self-contained, or a build flag already exists.
- **Med:** several callers or UI hooks need edits.
- **High:** used all over the tree, so disable it instead.

File counts include tests and were taken at the fork point.

Already gone upstream (nothing to do): Pocket on desktop, Shopping/Review Checker, Contile sponsored tiles.

---

## Telemetry decision (2026-10-23): none

The browser ships **no telemetry**. Telemetry-only rows below are ticked accordingly.

### What the three telemetry pieces are
- **Glean** is Mozilla's current telemetry SDK. Firefox embeds it as "FOG" (Firefox on Glean); Fenix uses the Kotlin version.
  - Developers declare metrics in `metrics.yaml` files all over the tree: counters, timings, events, strings, booleans. Code then records into them.
  - Glean stores the values in a database in the profile (`datareporting/glean/`) and bundles them into **pings** on a schedule, e.g. `baseline` (on activity changes), `metrics` (daily), `events` and `usage-reporting`.
  - It uploads the pings to `incoming.telemetry.mozilla.org`, some through an OHTTP relay.
- **Legacy Telemetry** (`toolkit/components/telemetry`) is the older system that Glean is replacing. It still produces the `main` ping (per session and every 24 hours), plus `crash`, `event`, `new-profile`, `first-shutdown` and `update` pings, and it archives copies on disk (`datareporting/archived/`).
  - Its data types are histograms, scalars and events.
  - The `main` ping carries an **environment** block: OS and version, CPU, GPU, memory, screen, locale, installed add-ons, non-default prefs, profile age and default-browser status.
  - Many Glean metrics are mirrored into it (GIFFT).
- **Pingsender** is a tiny separate program (`pingsender.exe` on Windows) that Telemetry launches to upload pings without delaying Firefox's own shutdown. It sends the shutdown `main` ping and `first-shutdown` ping after Firefox has exited; a background-task variant does the same job.
  - The related **uninstall ping** (Windows) is a ready-made ping file Firefox keeps on disk so that the uninstaller can send it when you uninstall.

### Why they matter for privacy
- Every Glean ping carries a random per-profile **`client_id`**, and Legacy Telemetry has its own client ID (`datareporting/state.json`). Those IDs persist for the life of the profile, so every ping from one install can be linked into a long-term record.
- **Habits:** pings include usage measures such as active hours, number of pages loaded, searches per engine, tab and window counts, which features and buttons you use, and session lengths.
- **Device fingerprint:** the environment data (hardware, screen, add-on list, settings) is detailed enough to single out a device even without the ID.
- The receiving server always sees your IP address.

### Where things stand in our builds today
Upload is already off in builds made from this tree, but only by accident of configuration:
- Glean's uploader is compiled with `glean_disable_upload` unless `MOZILLA_OFFICIAL` is set. It then pretends each ping was sent and deletes it.
- Legacy Telemetry only sends when `MOZILLA_OFFICIAL`, `MOZ_TELEMETRY_REPORTING` and a non-debug build are all set (`Telemetry::IsOfficialTelemetry`).
- Fenix debug builds have `BuildConfig.TELEMETRY = false`. Release builds still have the "Send technical and interaction data" toggle, which defaults to on.

Data is still **recorded and written to the profile**. And `MOZ_TELEMETRY_REPORTING` defaults to on as soon as we set `MOZILLA_OFFICIAL`, which release-quality builds usually do.

### How we neutralise it
1. Make `glean_disable_upload` unconditional in `toolkit/library/rust/gkrust-features.mozbuild`. The Glean uploader can then never send, whatever the build type.
2. Remove the `MOZ_TELEMETRY_REPORTING` option from `toolkit/moz.configure` (or default it to off), so Legacy Telemetry's sender is never compiled in.
3. Default and lock these prefs to off:
   - `datareporting.healthreport.uploadEnabled`
   - `datareporting.usage.uploadEnabled`
   - `datareporting.policy.dataSubmissionEnabled`
   - `toolkit.telemetry.enabled`
   - `toolkit.telemetry.archive.enabled`

   With the first one off, Glean stops collecting and wipes what it stored.
4. Delete Pingsender, the uninstall ping, and the telemetry-only components (DAP, gecko-trace, captcha detection, SERP telemetry, usage telemetry, attribution, search-detection).
5. Remove the "Firefox Data Collection and Use" section from Settings.
6. Fenix: hard-wire Glean's upload off, remove the data-collection settings, and drop Adjust.

The ~600 files that call the Glean API stay, but the calls become no-ops. Ripping them all out is possible later; it is a large mechanical job with no extra privacy gain once nothing is recorded or sent.

---

## 1. Telemetry, data collection and Mozilla services

| Feature | Path(s) | What it does | Details | Files | Difficulty | Existing switch | Suggested | Decision |
|---|---|---|---|---|---|---|---|---|
| Glean (FOG) | `toolkit/components/glean`, `third_party/rust/glean*`, `metrics.yaml` everywhere | Main telemetry SDK | Records metrics declared in `metrics.yaml`, stores them in the profile and uploads them as pings to Mozilla (see the telemetry decision above). **Privacy:** high; a persistent per-profile `client_id` links usage habits and device details over time, and the server sees your IP. **Performance:** small; in-memory recording, periodic disk writes and ping assembly, a few ms at startup. | ~470 + yamls | High (326 JS + 302 C++ callers) | Upload is compiled out unless `MOZILLA_OFFICIAL`; policy `DisableTelemetry` | Disable (upload compiled out, collection locked off) | [ ] Keep [ ] Remove [x] Disable |
| Legacy Telemetry | `toolkit/components/telemetry` | Histograms, scalars, events, main/crash pings | The older telemetry system: the `main` ping with its detailed environment block (hardware, add-ons, settings), plus crash, event and new-profile pings, archived on disk. **Privacy:** high; separate client ID plus a fingerprint-grade environment block. **Performance:** small; histogram accumulation in hot paths, ping assembly every 24 hours and at shutdown, disk archiving. | 258 | Med-High | Sending needs `MOZILLA_OFFICIAL` + `MOZ_TELEMETRY_REPORTING`; prefs `toolkit.telemetry.*` | Disable (sender compiled out, recording and archive off) | [ ] Keep [ ] Remove [x] Disable |
| Pingsender / uninstall ping | `toolkit/components/telemetry/pingsender`, `pings/UninstallPing.sys.mjs` | Sends pings at shutdown and uninstall | A helper program that uploads telemetry pings after Firefox has closed, and a pre-written ping file the Windows uninstaller sends. Exists only to deliver telemetry. **Privacy:** sends data even after the browser has exited or been uninstalled. **Performance:** an extra process at shutdown. | ~10 | Low | Telemetry off | Remove | [ ] Keep [x] Remove [ ] Disable |
| Normandy | `toolkit/components/normandy` | Remote pref rollouts and surveys | Downloads "recipes" from Mozilla that change your prefs remotely, install study add-ons and show Heartbeat surveys; eligibility is evaluated locally against profile data. **Privacy:** lets Mozilla change browser behaviour without an update; enrolments are reported through telemetry. **Performance:** a periodic background check. | 65 | Low-Med | `MOZ_NORMANDY` in `browser/moz.configure`; `app.normandy.enabled` | Remove | [ ] Keep [ ] Remove [ ] Disable |
| Nimbus (experiments) | `toolkit/components/nimbus`, `browser/components/nimbus` | Experiments, rollouts, Firefox Labs | Downloads experiment definitions, decides locally (region, version, usage, profile age) whether to enrol you, and switches features and their settings remotely. Firefox Labs opt-in features also go through it. **Privacy:** remote control of features; targeting reads local usage data; enrolments reported via telemetry. **Performance:** minor; cached at startup. Code in 83 files reads its values, which fall back to defaults when it's off. | 96 | High (83 files read feature values) | `app.shield.optoutstudies.enabled`; policy `DisableFirefoxStudies` | Disable | [ ] Keep [ ] Remove [ ] Disable |
| Crash reporter | `toolkit/crashreporter`, `toolkit/components/crashes`, `crashmonitor` | Breakpad minidumps, upload to Mozilla, about:crashes | When a process crashes it writes a minidump (stack, registers, loaded modules, some memory) plus annotations, then offers to send it to `crash-stats.mozilla.org`; a crash ping also goes through telemetry. **Privacy:** minidumps can contain fragments of page content or URLs; sending needs consent by default. **Performance:** an exception handler in every process; negligible until a crash. Without it we get no crash diagnostics, unless we run our own server. | ~1,100 | Low | `--disable-crashreporter`, `--with-crashreporter-url=` | Remove, or point at own server | [ ] Keep [ ] Remove [ ] Disable |
| DAP | `toolkit/components/dap` | Privacy-preserving aggregated telemetry | Splits a measurement (e.g. ad conversions) into encrypted shares sent to two independent servers, so neither sees an individual value. Still telemetry. **Privacy:** better designed than normal pings, but the data still leaves and servers see your IP. **Performance:** negligible. | 25 | Low-Med | Telemetry prefs | Remove | [ ] Keep [x] Remove [ ] Disable |
| gecko-trace | `toolkit/components/gecko-trace` | Tracing spans sent as Glean pings | Records timing spans of internal operations (OpenTelemetry style) and submits them as Glean pings. **Privacy:** reveals what the browser was doing and when. **Performance:** tracing overhead when active. | 27 | Low-Med | none | Remove | [ ] Keep [x] Remove [ ] Disable |
| Captcha detection | `toolkit/components/captchadetection` | Telemetry-only CAPTCHA ping | Detects CAPTCHAs (reCAPTCHA, hCaptcha, Cloudflare Turnstile, ...) on the pages you visit and reports whether they appeared and were solved, to measure whether privacy protections cause more CAPTCHAs. **Privacy:** reports browsing-derived events. **Performance:** an actor runs on matching pages. | 28 | Low | none | Remove | [ ] Keep [x] Remove [ ] Disable |
| Search/SERP telemetry | `browser/actors/SearchSERPTelemetry*`, parts of `browser/components/search` | Records search ad clicks, categorizes results pages | On result pages of the big search engines, detects ads and whether you clicked them, and classifies the result domains into topics that are sent to Mozilla. Exists for search-revenue reporting. **Privacy:** high; reveals what kind of things you search for and your ad engagement. **Performance:** parses every results page. | part of 224 | Med | `browser.search.serpEventTelemetryCategorization.enabled` | Remove | [ ] Keep [x] Remove [ ] Disable |
| Browser usage telemetry | `browser/modules/BrowserUsageTelemetry.sys.mjs`, `UIDensityTelemetry.sys.mjs`, `browser/components/StartupTelemetry.sys.mjs` | UI interaction metrics | Counts tabs and windows, pages loaded, which toolbar buttons and menus you use, UI density, and startup settings. **Privacy:** usage habits. **Performance:** event listeners on UI. | ~4 | Med | Telemetry off | Remove | [ ] Keep [x] Remove [ ] Disable |
| Context ID | `browser/modules/ContextId.sys.mjs` | Stable ID for sponsored content and Suggest | A random ID stored in prefs and sent with sponsored tile, story and Suggest impressions and clicks; rotated periodically. **Privacy:** an identifier for ad interactions. | 1 | Med | none | Remove with sponsored content | [ ] Keep [ ] Remove [ ] Disable |
| Attribution / referrals | `browser/components/attribution`, `browser/components/referrals` | Install-source attribution, referral pings | Attribution reads campaign data stamped into the installer (which site and campaign led to the download) and reports it. Referrals generates a per-profile referral code for Mozilla's referral programme and sends referral pings. **Privacy:** marketing tracking. | 24 | Low | `browser.referrals.enabled` (off) | Remove | [ ] Keep [x] Remove [ ] Disable |
| Content relevancy | `toolkit/components/contentrelevancy` | Local interest classification for content and ads | Classifies your history into interest categories (via the app-services relevancy component) to personalise new tab content and ads. **Privacy:** builds an interest profile from your history, locally. **Performance:** periodic background pass over history. | 14 | Low | Nimbus/pref | Remove | [ ] Keep [ ] Remove [ ] Disable |
| Remote Settings | `services/settings` | Remote data: add-on blocklist, CRLite cert revocation, search config, ML models | Syncs signed data collections from `firefox.settings.services.mozilla.com`: the add-on blocklist, certificate revocation (CRLite) and intermediates, the search engine config, translation and ML model lists, password-manager site rules, and the data used by Normandy, Nimbus and messaging. **Privacy:** polls Mozilla roughly daily; requests carry version and channel but no user ID. **Performance:** small background sync. **Security:** cert revocation and malicious add-on blocking depend on it. | 191 | High | Policy `DisableRemoteSettingsAndAcceptSecurityConsequences` | Keep (security) | [ ] Keep [ ] Remove [ ] Disable |
| Mozilla accounts (FxA) | `services/fxaccounts`, `browser/components/AccountsGlue.sys.mjs` | Login, devices, send-tab | Signs you in to a Mozilla account; manages your device list and Send Tab; required by Sync, Relay and IP Protection. **Privacy:** ties your devices to an account, only when signed in. **Performance:** none when signed out. | 53+ | Med-High | `identity.fxaccounts.enabled`; policy `DisableFirefoxAccounts` | ? (needed for Sync) | [ ] Keep [ ] Remove [ ] Disable |
| Sync | `services/sync`, `services/crypto`, `services/common`, `browser/components/syncedtabs` | Bookmark, history, password and tab sync via Mozilla servers | Syncs bookmarks, history, passwords, open tabs, add-ons, settings, addresses and cards. **Privacy:** end-to-end encrypted with keys from your account password; Mozilla sees only metadata (timing, sizes, IP). The server can be self-hosted. **Performance:** periodic background syncs when signed in. | ~300 | Med | `MOZ_SERVICES_SYNC` in `browser/moz.configure` | ? | [ ] Keep [ ] Remove [ ] Disable |
| Sponsored new tab content | `browser/extensions/newtab/lib/AdsFeed.sys.mjs`, `DiscoveryStreamFeed`, `NewTabAttribution*`, `browser/components/mozcachedohttp` | Sponsored tiles and stories (`ads.mozilla.org`) | Fetches sponsored shortcut tiles and sponsored stories for your region through an OHTTP relay and reports impressions and clicks. **Privacy:** ad reporting with the Context ID. **Performance:** extra network fetches and images on new tab. | part of 694 | Med | `browser.newtabpage.activity-stream.showSponsored*`; policy `FirefoxHome` | Remove | [ ] Keep [ ] Remove [ ] Disable |
| Firefox Suggest / Merino | `browser/components/urlbar/private/*` | Sponsored and online address bar suggestions (AMP, Yelp, weather, ...) | Adds suggestions while you type: some come from a downloaded offline dataset; in the online mode your **keystrokes are sent to Mozilla's Merino server**. Includes sponsored results. **Privacy:** online mode shares what you type; sponsored clicks are reported. **Performance:** a local suggestion database of several MB. | ~140 | Med | `browser.urlbar.quicksuggest.enabled`; policy `FirefoxSuggest` | Remove | [ ] Keep [ ] Remove [ ] Disable |
| Messaging / ASRouter | `browser/components/asrouter`, `toolkit/components/messaging-system`, `browser/components/messagepreview`, `browser/components/aboutwelcome` | Recommendations, infobars, spotlights, onboarding, product promos | Shows Mozilla-authored messages (extension recommendations, infobars, full-screen spotlights, what's new, about:welcome onboarding) downloaded from Remote Settings and targeted with local profile data. **Privacy:** targeting reads usage data locally; impressions reported via telemetry. **Performance:** targeting evaluated at startup and on triggers. | ~495 | Med-High (onboarding uses it) | Policy `UserMessaging` | Disable, rewrite onboarding later | [ ] Keep [ ] Remove [ ] Disable |
| Mozilla product promos | prefs `browser.contentblocking.report.vpn*` / `monitor*`, `browser.privatebrowsing.vpnpromourl`, `browser.preferences.moreFromMozilla` | VPN/Monitor/Relay ads | Advertising for Mozilla's paid products inside the protections dashboard, private browsing page and settings. No direct privacy or performance effect. | prefs | Low | prefs | Remove | [ ] Keep [ ] Remove [ ] Disable |
| IP Protection (built-in VPN) | `browser/components/ipprotection`, `toolkit/components/ipprotection`, `browser/extensions/ipp-activator` | Mozilla VPN-backed proxy | Routes browser traffic through Mozilla's proxy service when you're signed in to a Mozilla account. **Privacy:** hides your IP from sites, but the proxy operator sees your connection metadata. **Performance:** added latency while active. Off by default. | 182 | Low-Med | `browser.ipProtection.enabled` (off by default) | Remove | [ ] Keep [ ] Remove [ ] Disable |
| Default browser agent | `toolkit/mozapps/defaultagent` | Windows scheduled task that reports and nags about the default browser | A Windows scheduled task that runs daily even when Firefox is closed, reports which browser is your default (and what it was before) through telemetry, and can show a notification asking you to switch back. **Privacy:** reports your browser choice outside browsing sessions. **Performance:** a daily background process. | 51 | Low | `--disable-default-browser-agent` | Remove | [ ] Keep [ ] Remove [ ] Disable |
| App updater | `toolkit/mozapps/update`, `toolkit/components/maintenanceservice`, `bitsdownload` | Mozilla update service, MAR files, Windows maintenance service | Asks `aus5.mozilla.org` for updates (sending version, build, OS, locale, channel), downloads MAR patches (via BITS on Windows) and applies them; the maintenance service is a SYSTEM service that installs updates without a UAC prompt. **Privacy:** update checks reveal version and OS, no user ID. **Performance:** background downloads. Points at Mozilla's servers, so useless for our builds until we run our own. | ~460 | Low (flags) | `--disable-updater`, `--disable-maintenance-service`, `--disable-update-agent`, `--disable-bits-download` | ? (need our own update channel eventually) | [ ] Keep [ ] Remove [ ] Disable |
| Background tasks | `toolkit/components/backgroundtasks`, `taskscheduler` | Headless tasks (update agent, pingsender, cleanup) | Runs Firefox headless in a temporary profile to do jobs while the browser is closed: background updates, ping sending, removing leftover directories. **Privacy:** depends on the task. **Performance:** spawns short-lived processes. | 93 | Med | `--disable-backgroundtasks` | Keep for now | [ ] Keep [ ] Remove [ ] Disable |
| Captive portal detection | `toolkit/components/captivedetect` | Pings `detectportal.firefox.com` | Fetches a known page over plain HTTP at startup and on network changes; if the response is altered, you're behind a login portal (hotel or airport Wi-Fi) and Firefox opens it. **Privacy:** Mozilla sees your IP and when you start the browser or change networks. **Performance:** one tiny request. | 18 | Low | `network.captive-portal-service.enabled` | Keep, maybe own endpoint | [ ] Keep [ ] Remove [ ] Disable |
| DoH rollout heuristics | `toolkit/components/doh` | Region-based automatic DNS-over-HTTPS | In some regions, turns on DNS-over-HTTPS automatically after checks for parental controls, enterprise DNS and the `use-application-dns.net` canary domain. Manual DoH in Settings is separate (`netwerk/`) and stays. **Privacy:** hides DNS from your ISP but hands it to the resolver (e.g. Cloudflare). **Performance:** DoH can be faster or slower than system DNS. | 29 | Low-Med | `doh-rollout.*`, `network.trr.mode` | ? | [ ] Keep [ ] Remove [ ] Disable |
| Safe Browsing | `toolkit/components/url-classifier`, `reputationservice` | Google Safe Browsing, tracking lists, download reputation | Downloads hashed prefixes of malware and phishing URLs from Google; when a URL matches a prefix it asks Google with the hash prefix only, never the full URL. Download reputation sends metadata of unknown Windows executables (hash, size, URL and referrers) to Google. The same engine applies the tracking-protection lists. **Privacy:** Google sees your IP and occasional hash prefixes; download checks send more (they can be turned off with `browser.safebrowsing.downloads.remote.enabled`). **Performance:** local lookups, negligible. | 250 | High (tracking protection shares it) | `browser.safebrowsing.*` | Keep (turn off remote download checks) | [ ] Keep [ ] Remove [ ] Disable |
| Report Broken Site | `browser/components/reportbrokensite`, `toolkit/components/reportbrokensite` | Sends site-compat reports to Mozilla | A form in the help menu that sends the URL, your description and technical details (settings, tracking protection state) to Mozilla. Only sends when you submit it; useless to us. | 40 | Low | Policy `DisableFeedbackCommands` | Remove | [ ] Keep [ ] Remove [ ] Disable |
| Terms of Use modal | `asrouter/modules/PreonboardingSplash.sys.mjs` | Mozilla ToU acceptance | A first-run dialog asking you to accept Mozilla's Terms of Use; not applicable to our browser. | — | Low-Med | Policy `SkipTermsOfUse` | Remove | [ ] Keep [ ] Remove [ ] Disable |

## 2. Core desktop UI (keep unless replacing)

| Feature | Path(s) | What it does | Details | Files | Difficulty | Decision |
|---|---|---|---|---|---|---|
| Tabbed browser | `browser/components/tabbrowser` | Tabs, tab groups, smart grouping, previews | The tab strip and everything around it. Smart tab grouping suggests group names with a local ML model. **Performance:** tab unloading under memory pressure lives here too. | 380 | High | [ ] Keep [ ] Remove [ ] Disable |
| Address bar | `browser/components/urlbar` | Awesomebar and its providers | Combines history, bookmarks, open tabs, search suggestions and Suggest into one dropdown. **Privacy:** search suggestions send your keystrokes to your search engine (`browser.search.suggest.enabled`). | 907 | High | [ ] Keep [ ] Remove [ ] Disable |
| Search | `browser/components/search`, `toolkit/components/search` | Search engines and config | Manages search engines, their order and URL parameters. The engine list comes from Remote Settings, and partner codes in search URLs earn Mozilla revenue. **Privacy:** queries go to the engine you choose. | 435 | High | [ ] Keep [ ] Remove [ ] Disable |
| Session restore | `browser/components/sessionstore`, `toolkit/components/sessionstore` | Restores windows and tabs | Saves open windows, tabs, history per tab, scroll positions and form data to `sessionstore.jsonlz4` about every 15 seconds. **Privacy:** local only. **Performance:** periodic disk writes (`browser.sessionstore.interval`). | 420 | High | [ ] Keep [ ] Remove [ ] Disable |
| Bookmarks and history | `browser/components/places`, `toolkit/components/places` | Places database and UI | The local `places.sqlite` database of history and bookmarks, frecency ranking and the Library window. **Privacy:** local only. **Performance:** idle-time database maintenance. | 752 | High | [ ] Keep [ ] Remove [ ] Disable |
| Preferences | `browser/components/preferences` | about:preferences | The Settings page. | 431 | High | [ ] Keep [ ] Remove [ ] Disable |
| Toolbar customization | `browser/components/customizableui` | Customize toolbars | Moving buttons between toolbars, menus and the overflow panel. | 215 | High | [ ] Keep [ ] Remove [ ] Disable |
| Downloads | `browser/components/downloads`, `toolkit/components/downloads` | Downloads panel and library | Download manager and history. Downloads are checked by Safe Browsing's download reputation. | 109 | High | [ ] Keep [ ] Remove [ ] Disable |
| Sidebar / vertical tabs | `browser/components/sidebar` | Sidebar revamp | The new sidebar with vertical tabs, history, bookmarks, synced tabs and the AI chatbot panel. | 127 | Med | [ ] Keep [ ] Remove [ ] Disable |
| Private browsing | `browser/components/privatebrowsing` | about:privatebrowsing | Private windows keep history, cookies and cache in memory and drop them on close. **Privacy:** positive. | 81 | Med | [ ] Keep [ ] Remove [ ] Disable |
| Protections dashboard / site info | `browser/components/protections`, `controlcenter` | about:protections, site identity panel | Shows blocked-tracker statistics (stored locally) and the per-site security and permission panel. Contains the Monitor and VPN promos. | 39 | Med | [ ] Keep [ ] Remove [ ] Disable |
| Containers | `browser/components/contextualidentity`, `originattributes` | Multi-account containers | Colour-coded tab containers with separate cookies and storage. **Privacy:** positive; isolates sites from each other. | 156 | Med | [ ] Keep [ ] Remove [ ] Disable |
| Fingerprinting protection | `browser/components/resistfingerprinting`, `toolkit/components/resistfingerprinting` | RFP and FPP | Reduces what sites can learn about the device (canvas, fonts, screen, timers). FPP is on in strict mode and private windows; RFP is the stricter Tor-style mode. **Privacy:** positive. **Performance:** small; some site breakage under RFP. | 233 | High | [ ] Keep [ ] Remove [ ] Disable |
| Permissions, prompts, WebRTC UI | `browser/components/permissions`, `prompts`, `webrtc` | Permission and media sharing UI | Permission prompts (camera, microphone, location, notifications) and the sharing indicators. | ~11 | High | [ ] Keep [ ] Remove [ ] Disable |
| OS shell integration | `browser/components/shell` | Default browser, pinning, jump lists | Setting the default browser, taskbar pinning, Windows jump lists, desktop background. | 117 | Med | [ ] Keep [ ] Remove [ ] Disable |
| Enterprise policies | `browser/components/enterprisepolicies`, `toolkit/components/enterprisepolicies` | Policy engine; handy for our own defaults | Reads `policies.json` (or the Windows registry) and locks settings. Useful for shipping our own defaults. | 220 | Keep | [ ] Keep [ ] Remove [ ] Disable |
| Browser WebExtension APIs | `browser/components/extensions` | Tabs, windows, menus APIs, ... | The browser-specific extension APIs (tabs, windows, menus, sidebar, ...). Extensions need them. | 465 | High | [ ] Keep [ ] Remove [ ] Disable |
| Themes | `browser/themes` (incl. `addons/` colorway themes) | Platform CSS and built-in themes | The browser's CSS per platform, plus the built-in light, dark and alpenglow themes and the colorway themes. | 554 | High (colorways Low) | [ ] Keep [ ] Remove [ ] Disable |

## 3. Optional desktop features

| Feature | Path(s) | What it does | Details | Files | Difficulty | Existing switch | Suggested | Decision |
|---|---|---|---|---|---|---|---|---|
| Smart Window (AI window) | `browser/components/aiwindow` | AI browsing window: chat, memories, agent; uses a remote Mozilla endpoint | A separate window type built around an AI assistant: chat about pages, "memories" it learns about you, and an agent that acts in tabs. **Privacy:** high; your prompts and page content go to Mozilla's remote model endpoint, and memories are a profile of you. **Performance:** on demand only. | 677 | Low-Med | `browser.smartwindow.enabled` (off) | ? | [ ] Keep [ ] Remove [ ] Disable |
| AI chatbot sidebar / Link Preview / Page Assist | `browser/components/genai` | ChatGPT/Gemini/Claude etc. in the sidebar, AI link previews | The chatbot sidebar loads the provider's website (ChatGPT, Claude, Gemini, Copilot, Le Chat); context-menu actions paste the selection or page into it. Link Preview summarises a link with a local model. **Privacy:** the chatbot is a third-party site and sees what you send; Link Preview stays on device. **Performance:** Link Preview runs a local model (CPU, RAM, a model download). | 74 | Low-Med | `browser.ml.chat.enabled`, `browser.ml.linkPreview.enabled`; policy `GenerativeAI` | ? | [ ] Keep [ ] Remove [ ] Disable |
| Mini window | `browser/components/miniwindow` | Small popup browser window | An always-on-top small window with a simplified toolbar: either a whole tab moved into it, or a cropped section of a page that stays interactive. | 41 | Low | — | ? | [ ] Keep [ ] Remove [ ] Disable |
| Multiline editor | `browser/components/multilineeditor` | Rich editor used by the Smart Window bar | Only used by the Smart Window. | 16 | Low (with aiwindow) | — | Follows aiwindow | [ ] Keep [ ] Remove [ ] Disable |
| Tab notes | `browser/components/tabnotes` | Notes attached to tabs | Lets you attach short text notes to tabs; stored locally. | 25 | Low | `browser.tabs.notes.enabled` (off) | ? | [ ] Keep [ ] Remove [ ] Disable |
| New tab page | `browser/extensions/newtab`, `browser/components/newtab`, `topsites` | Top sites, stories, weather, wallpapers, widgets | Shortcuts from your history, recommended stories, weather, wallpapers and widgets. **Privacy:** stories, weather and wallpapers are fetched from Mozilla (by region); impressions go to telemetry. **Performance:** a heavy page that is preloaded in the background (`browser.newtab.preload`). | 840 | Med-High | Policies `FirefoxHome`, `NewTabPage` | Keep, strip sponsored parts | [ ] Keep [ ] Remove [ ] Disable |
| Firefox View | `browser/components/firefoxview` | Recent, open and synced tabs hub | A page listing recently closed, open and synced tabs and history. Local only. | 82 | Med | `browser.firefox-view.*` | ? | [ ] Keep [ ] Remove [ ] Disable |
| Screenshots | `browser/components/screenshots` | Built-in screenshot tool | Captures a region, visible area or full page to the clipboard or a file. Local only; the old cloud upload is gone. | 63 | Low | Policy `DisableFirefoxScreenshots` | Keep | [ ] Keep [ ] Remove [ ] Disable |
| Translations | `browser/components/translations`, `toolkit/components/translations` | Local, on-device page translation | Translates pages fully on your machine with the Bergamot engine (WASM); each language model is downloaded once from Remote Settings. **Privacy:** page text never leaves the device. **Performance:** CPU-heavy while translating; models take tens of MB each. | 355 | Low-Med | `browser.translations.enable` | Keep | [ ] Keep [ ] Remove [ ] Disable |
| Picture-in-Picture | `toolkit/components/pictureinpicture`, `browser/extensions/pictureinpicture` | Pop-out video | Plays a video in a floating always-on-top window. Local. | 184 | Low-Med | `media.videocontrols.picture-in-picture.enabled` | Keep | [ ] Keep [ ] Remove [ ] Disable |
| Password manager UI | `browser/components/aboutlogins`, `passwordmgr` | about:logins | Lets you view, edit, import and export saved logins. Breach alerts compare against a downloaded list of breached sites, locally. | 106 | Med | — | Keep | [ ] Keep [ ] Remove [ ] Disable |
| Import from other browsers | `browser/components/migration` | Imports from Chrome, Edge, Safari | Reads another browser's bookmarks, history, passwords and cards from its profile on disk. Local only. | 150 | Low-Med | `MOZ_PROFILE_MIGRATOR`; policy `DisableProfileImport` | Keep | [ ] Keep [ ] Remove [ ] Disable |
| Profile manager | `browser/components/profiles` | New profile switcher | A switcher for several profiles (work, personal), each with its own data. Local. | 210 | Med | `browser.profiles.enabled` | ? | [ ] Keep [ ] Remove [ ] Disable |
| Backup | `browser/components/backup` | Scheduled profile backup and restore | Periodically archives your profile to a folder you choose; passwords, cookies and cards are only included when you set a backup password (encryption). **Privacy:** backup files hold your data wherever you store them. **Performance:** periodic compression work in idle time. | 162 | Low-Med | `browser.backup.enabled` | ? | [ ] Keep [ ] Remove [ ] Disable |
| Content analysis (DLP) | `browser/components/contentanalysis`, `toolkit/components/contentanalysis` | Enterprise data-loss-prevention hook | When a company configures it by policy, sends uploads, pastes and prints to a local DLP agent for approval. Inactive otherwise. | 97 | Low-Med | Policy `ContentAnalysis` | Remove | [ ] Keep [ ] Remove [ ] Disable |
| Taskbar Tabs | `browser/components/taskbartabs` | Pin websites as Windows apps | Opens a site in its own window with its own taskbar icon, like a web app. Windows only. | 42 | Low | — | ? | [ ] Keep [ ] Remove [ ] Disable |
| Text recognition (OCR) | `browser/components/textrecognition` | Copy text from images | Context-menu "Copy text from image". Only has a macOS backend (`widget/cocoa/TextRecognition.mm`); on Windows and Linux it is unavailable, so for us it is dead code. | 11 | Low | — | Remove (no backend on our platforms) | [ ] Keep [ ] Remove [ ] Disable |
| QR code / share | `browser/components/qrcode`, `sharing` | Page QR code, share dialog | Shows a QR code for the current page and the OS share dialog. Local. | 66 | Low | — | Keep | [ ] Keep [ ] Remove [ ] Disable |
| Custom keyboard shortcuts | `browser/components/customkeys` | Rebind shortcuts | Lets you change built-in keyboard shortcuts. | 16 | Low | — | Keep | [ ] Keep [ ] Remove [ ] Disable |
| UITour | `browser/components/uitour` | Lets mozilla.org pages drive the UI | A JS API that allowlisted Mozilla sites (www.mozilla.org, support.mozilla.org) use to highlight buttons, open menus and read some browser state. **Privacy:** remote web pages can query browser state. | 40 | Low-Med | `browser.uitour.enabled` | Remove | [ ] Keep [ ] Remove [ ] Disable |
| Remote-control indicator | `browser/components/remotecontrol` | Shows when WebDriver is controlling the browser | The striped address bar and robot icon shown during automation. | 9 | Low | Follows `--disable-webdriver` | Keep | [ ] Keep [ ] Remove [ ] Disable |
| Page data | `browser/components/pagedata` | OpenGraph/Schema.org extraction | Extracts page title, description and image from page metadata for richer cards in other features. Local. | 31 | Low | — | ? | [ ] Keep [ ] Remove [ ] Disable |
| about:unloads | `browser/components/tabunloader` | Tab unloading UI | Lists tabs by memory use and lets you unload them. **Performance:** positive. | 5 | Low | — | Keep | [ ] Keep [ ] Remove [ ] Disable |
| Storybook | `browser/components/storybook` | UI component gallery (dev only) | A dev tool for viewing UI components; not shipped. | 61 | Low | — | Keep (dev tool) | [ ] Keep [ ] Remove [ ] Disable |
| Launch on login | `browser/components/defaultlaunchonlogin` | Start with Windows | Registers the browser to start when you log in to Windows. **Performance:** slower login if on. | 4 | Low | Policy `DisableLaunchOnLogin` | ? | [ ] Keep [ ] Remove [ ] Disable |
| Touch Bar | `browser/components/touchbar` | macOS Touch Bar | macOS only; goes with platform removal. | 10 | Low | — | Remove (macOS) | [ ] Keep [x] Remove [ ] Disable |
| Protocol handlers | `browser/components/protocolhandler` | `registerProtocolHandler` | Lets web apps handle links like `mailto:` (e.g. webmail). | 11 | Med | — | Keep | [ ] Keep [ ] Remove [ ] Disable |

## 4. Built-in system add-ons (`browser/extensions/`)

| Add-on | Files | What it does | Details | Difficulty | Suggested | Decision |
|---|---|---|---|---|---|---|
| `newtab` | 694 | The new tab page (see above) | See "New tab page" in section 3. | Med-High | Keep, strip sponsored | [ ] Keep [ ] Remove [ ] Disable |
| `webcompat` | 645 | Per-site compat fixes and SmartBlock shims | Site-specific fixes (user-agent overrides, CSS and JS patches) for sites that break in Firefox, plus SmartBlock: stand-ins for blocked tracking scripts so pages keep working. **Privacy:** positive. **Performance:** small scripts only on listed sites. | Low (but sites break more without it) | Keep | [ ] Keep [ ] Remove [ ] Disable |
| `formautofill` | 435 | Address and credit-card autofill | Detects address and payment forms and fills them from local storage; cards are encrypted with an OS key. **Performance:** form heuristics run on page load. | Med | Keep | [ ] Keep [ ] Remove [ ] Disable |
| `pictureinpicture` | 51 | Per-site PiP fixes | Site wrappers so PiP controls and captions work on YouTube, Netflix and others. | Low | Keep | [ ] Keep [ ] Remove [ ] Disable |
| `ipp-activator` | 31 | Activates IP Protection | Only used by IP Protection. | Low | Remove with IP Protection | [ ] Keep [ ] Remove [ ] Disable |
| `search-detection` | 13 | Detects add-ons hijacking search (telemetry) | Watches for extensions that change or redirect your searches and reports it through telemetry; it doesn't block anything. | Low | Remove | [ ] Keep [x] Remove [ ] Disable |

## 5. Shared toolkit features

| Feature | Path(s) | What it does | Details | Files | Difficulty | Suggested | Decision |
|---|---|---|---|---|---|---|---|
| WebExtensions + Add-ons Manager | `toolkit/components/extensions`, `toolkit/mozapps/extensions` | Extension engine, AMO, blocklist | Runs extensions and manages installing and updating them. **Privacy:** daily update checks to addons.mozilla.org list your installed add-on IDs; the about:addons recommendations fetch from AMO. **Performance:** depends on the extensions you install. | 1,648 | High | Keep | [ ] Keep [ ] Remove [ ] Disable |
| Password manager | `toolkit/components/passwordmgr` | Logins, generation, Relay integration | Saves and fills logins (`logins.json`, encrypted with `key4.db` and an optional primary password), generates passwords and offers Firefox Relay email masks (needs a Mozilla account). | 341 | High | Keep | [ ] Keep [ ] Remove [ ] Disable |
| Form history / autofill | `satchel`, `formautofill`, `autocomplete` | Form history | Remembers what you typed in form fields (`formhistory.sqlite`) and suggests it again. **Privacy:** local, but stores everything you type in forms (`browser.formfill.enable`). | 232 | High | Keep | [ ] Keep [ ] Remove [ ] Disable |
| Anti-tracking | `antitracking`, `cleardata`, `content-classifier` | Enhanced Tracking Protection, cookie purging, bounce tracking | Blocks known trackers, isolates cookies per site (Total Cookie Protection), purges bounce-tracker cookies, and provides "clear data". **Privacy:** core protection. **Performance:** positive; blocked trackers never load. | ~765 | High | Keep | [ ] Keep [ ] Remove [ ] Disable |
| PDF viewer | `toolkit/components/pdfjs`, `aboutpdf` | PDF.js | Built-in PDF viewer and form filler written in JS. Local. | 460 | Low | Keep | [ ] Keep [ ] Remove [ ] Disable |
| Reader mode / Narrate | `reader`, `narrate` | Reader view, text-to-speech | Strips clutter from articles; Narrate reads them aloud with the OS speech engine. Local. | 72 | Low-Med | Keep | [ ] Keep [ ] Remove [ ] Disable |
| Local ML runtime | `toolkit/components/ml`, `aboutinference` | On-device ONNX/WASM inference; used by link preview, smart tab groups, Suggest ML, new tab | Runs AI models on your machine. Models are downloaded from Mozilla's model hub on first use. **Privacy:** inference is local; downloads reveal which feature you used. **Performance:** heavy CPU, RAM or GPU while running; models take hundreds of MB on disk. | 208 | Med | ? | [ ] Keep [ ] Remove [ ] Disable |
| Page extractor | `toolkit/components/pageextractor` | Page text extraction for AI features | Pulls readable text out of pages for the AI features. | 29 | Low | Follows AI choices | [ ] Keep [ ] Remove [ ] Disable |
| Thumbnails | `toolkit/components/thumbnails` | Page thumbnails for new tab | Captures page previews for new tab shortcuts; may load a site in a hidden background browser to capture it. **Performance:** background page loads. | 55 | Med | Keep | [ ] Keep [ ] Remove [ ] Disable |
| Printing | `printing`, `printingui` | Print and print preview | Print dialog, preview and save as PDF. | 82 | High | Keep | [ ] Keep [ ] Remove [ ] Disable |
| Notifications / push | `alerts`, `push`, `toolkit/mozapps/notificationserver`, `pushnotificationhelper` | Desktop notifications, web push | Shows site notifications through the OS, and receives Web Push through Mozilla's push server (payloads are end-to-end encrypted). **Privacy:** the push server knows a random device ID and which subscriptions it has. **Performance:** a persistent connection to the push server. | 58 | Med | Keep | [ ] Keep [ ] Remove [ ] Disable |
| Diagnostic about: pages | `aboutconfig`, `aboutmemory`, `aboutprocesses`, `aboutthirdparty`, `aboutwebauthn`, `aboutwindowsmessages`, `aboutcheckerboard`, `certviewer` | about:config, about:memory, ... | Diagnostic and power-user pages. Local, no cost unless opened. | ~190 | Low each | Keep | [ ] Keep [ ] Remove [ ] Disable |
| app-services bridge (UniFFI) | `uniffi-bindgen-gecko-js`, `uniffi-js`, `third_party/application-services` | Rust components: logins, places, suggest, sync | Lets JS call the shared Rust components that Mozilla also uses on Android: logins, Suggest, Sync, relevancy, Remote Settings client. | 155 + 1,548 | High | Keep | [ ] Keep [ ] Remove [ ] Disable |

## 6. Developer tools

| Feature | Path(s) | What it does | Details | Files | Difficulty | Existing switch | Suggested | Decision |
|---|---|---|---|---|---|---|---|---|
| DevTools UI | `devtools/client` | Inspector, console, debugger, network, ... | The developer tools panels. **Performance:** nothing loads until you open them. | 7,410 | Med | `MOZ_DEVTOOLS=server`; policy `DisableDeveloperTools` | Keep | [ ] Keep [ ] Remove [ ] Disable |
| DevTools server | `devtools/server`, `devtools/shared` | Debugging actors; also used by Android remote debugging | The backend the panels talk to; also lets desktop debug the Android app over USB. | 1,524 | High | always built | Keep | [ ] Keep [ ] Remove [ ] Disable |
| DevTools docs | `devtools/docs` | Docs | Contributor docs; not shipped. | 769 | Low | — | ? | [ ] Keep [ ] Remove [ ] Disable |
| Remote Agent (WebDriver BiDi / Marionette) | `remote/` | Browser automation; the test harnesses rely on it | Remote-control protocols for automation (Selenium, Puppeteer, Playwright) and our test harnesses. Only listens when started with `--marionette` or `--remote-debugging-port`. **Security:** exposes full control when enabled. | 1,118 | Med | `--disable-webdriver` | Keep (tests need it) | [ ] Keep [ ] Remove [ ] Disable |
| geckodriver | `testing/geckodriver` | WebDriver proxy | A separate program that translates W3C WebDriver to Marionette for Selenium. | 44 | Low | `--enable-geckodriver` | Keep | [ ] Keep [ ] Remove [ ] Disable |

## 7. Android

### Apps
| App / module | Path | Files | What it is | Details | Suggested | Decision |
|---|---|---|---|---|---|---|
| Fenix | `mobile/android/fenix` | 3,919 | Firefox for Android, the full app. Our base | The complete Android browser: UI, settings, home screen and all integrations. Built from android-components on top of GeckoView. | Keep (rebrand) | [ ] Keep [ ] Remove [ ] Disable |
| Focus | `mobile/android/focus-android` | 708 | Separate minimal privacy browser | A second app (Firefox Focus/Klar) that erases everything on close. We only ship one app. | Remove | [ ] Keep [x] Remove [ ] Disable |
| android-components | `mobile/android/android-components` | 9,568 | Library of browser building blocks used by Fenix | Reusable Kotlin modules (toolbar, tabs, downloads, prompts, add-ons, ...) that Fenix assembles into the app. | Keep | [ ] Keep [ ] Remove [ ] Disable |
| GeckoView | `mobile/android/geckoview` | 621 | Engine as an Android library | Wraps the Gecko engine (the same one as desktop) as an Android library. | Keep | [ ] Keep [ ] Remove [ ] Disable |
| Sample apps | `geckoview_example`, `mobile/android/examples`, `android-components/samples` | ~430 | Dev demos | Small demo apps for testing components in isolation. Not shipped. | ? | [ ] Keep [ ] Remove [ ] Disable |
| test_runner | `mobile/android/test_runner` | 27 | Runs GeckoView tests | The app that hosts GeckoView's automated tests. | Keep | [ ] Keep [ ] Remove [ ] Disable |

### Third-party SDKs and Mozilla services in Fenix
These are wired in through `fenix/app/build.gradle` and `fenix/.../components/`.

| Dependency | What it does | Details | Suggested | Decision |
|---|---|---|---|---|
| Adjust (`libs.adjust`) | Marketing install attribution | A third-party marketing SDK that reports to Adjust's servers which ad campaign led to the install, with device information. **Privacy:** third-party tracking. | Remove | [ ] Keep [x] Remove [ ] Disable |
| Sentry (`lib-crash-sentry`) | Crash reporting to Sentry | Sends crash and exception reports (stack traces, device model, Android version) to Mozilla's Sentry instance; release builds only. **Privacy:** crash context leaves the device. | Remove | [ ] Keep [ ] Remove [ ] Disable |
| Google Play install referrer, ads ID, in-app review | Google Play integrations | Install referrer tells the app which Play Store link it came from; the advertising ID is a cross-app device identifier for ad tracking; in-app review shows the Play rating dialog. **Privacy:** the advertising ID and referrer are tracking data. | Remove (keep review prompt?) | [ ] Keep [ ] Remove [ ] Disable |
| Firebase push (`lib-push-firebase`) | Push messages (WebPush, Send Tab) | Receives web push and Send Tab while the app is in the background through Google's Firebase Cloud Messaging. Needs Google Play Services. **Privacy:** Google sees push tokens and delivery timing; payloads stay encrypted. Without it, web push only works while the app is open. | ? | [ ] Keep [ ] Remove [ ] Disable |
| Glean (`service-glean`) | Telemetry | Same as Glean on desktop. Debug builds already don't upload (`BuildConfig.TELEMETRY = false`); release builds have the "Send technical and interaction data" toggle, which defaults to on. | Disable (hard-wired off) | [ ] Keep [ ] Remove [x] Disable |
| Nimbus (`service-nimbus`, `nimbus.fml.yaml`) | Experiments | Same as Nimbus on desktop; most Fenix features are configured through its manifest, so it stays as a local config source. | Disable | [ ] Keep [ ] Remove [ ] Disable |
| MARS ads (`service-mars`) | Sponsored content | Fetches sponsored shortcuts and stories for the home screen and reports impressions and clicks. **Privacy:** ad reporting. | Remove | [ ] Keep [ ] Remove [ ] Disable |
| Pocket (`service-pocket`) | Home screen stories | Fetches recommended stories for the home screen. **Privacy:** requests reveal your region and locale. | Remove | [ ] Keep [ ] Remove [ ] Disable |
| Google Lens (`components/lens`) | Image search | Hands an image to Google Lens for visual search. **Privacy:** the image goes to Google when used. | ? | [ ] Keep [ ] Remove [ ] Disable |
| On-device and remote LLMs (`lib-llm-gemininano`, `lib-llm-mlpa`, `feature-summarize`) | Page summarization | Summarises pages with Gemini Nano on supported phones (on device, via Android AICore) or with Mozilla's remote model proxy (MLPA). **Privacy:** the remote path sends page content to Mozilla. | ? | [ ] Keep [ ] Remove [ ] Disable |

### Fenix feature areas (`fenix/app/src/main/java/org/mozilla/fenix/`)
settings, home, tabstray, browser, debugsettings, translations, downloads, library, onboarding, tabgroups, bookmarks, share, webcompat, search, termsofuse, ipprotection, addons, exceptions, collections, experiments, pdf, messaging, crashes, summarization, reviewprompt, microsurvey, wallpapers, privacyreport, pbmlock, listentopage, whatsnew.

Candidate cuts: termsofuse, ipprotection, experiments, messaging, microsurvey, reviewprompt, summarization. Decide each:

| Area | What it does | Decision |
|---|---|---|
| termsofuse | Prompt to accept Mozilla's Terms of Use | [ ] Keep [ ] Remove [ ] Disable |
| ipprotection | UI for the Mozilla VPN-backed proxy | [ ] Keep [ ] Remove [ ] Disable |
| experiments | Nimbus enrolment and the Studies settings screen | [ ] Keep [ ] Remove [ ] Disable |
| messaging | Mozilla promo and nudge messages (e.g. set as default browser), driven by Nimbus | [ ] Keep [ ] Remove [ ] Disable |
| microsurvey | Short in-app surveys whose answers are sent through Glean | [ ] Keep [ ] Remove [ ] Disable |
| reviewprompt | Asks you to rate the app on the Play Store | [ ] Keep [ ] Remove [ ] Disable |
| summarization | AI page summaries (see On-device and remote LLMs above) | [ ] Keep [ ] Remove [ ] Disable |
| wallpapers | Home screen backgrounds downloaded from Mozilla's CDN | [ ] Keep [ ] Remove [ ] Disable |
| collections | Saved groups of tabs on the home screen; local | [ ] Keep [ ] Remove [ ] Disable |

### android-components modules of note
- **service-\*** (Mozilla backends): `nimbus`, `pocket`, `firefox-accounts`, `firefox-relay`, `mars` (ads), `merino-weather`, `merino-manifest`, `sync-*`, `glean`, `location`.
- **feature-\***: `search`, `prompts`, `downloads`, `addons`, `pwa`, `awesomebar`, `autofill`, `media`, `readerview`, `sitepermissions`, `customtabs`, `tabs`, `contextmenu`, `qr`, `findinpage`, `app-links`, `listentopage`, `fxsuggest`, `summarize`, `webnotifications`, `privatemode`, `ipprotection`, `protection-dashboard`, `importer`, `password-importer`, ...
- **browser-engine-system** (163 files): a WebView-based engine alternative, not used by Fenix. Can be removed.

---

## 8. Switches available today (no code changes)

### Configure flags (mozconfig `ac_add_options`)
| Flag | Removes |
|---|---|
| `--disable-crashreporter` | Crash reporter |
| `--disable-updater` | Updater (also maintenance service and update agent) |
| `--disable-maintenance-service`, `--disable-update-agent`, `--disable-bits-download` | Windows update pieces |
| `--disable-default-browser-agent` | Default browser agent |
| `--disable-notification-server`, `--disable-push-notification-helper` | Windows notification and push helpers |
| `--disable-backgroundtasks` | Background tasks (needed by the update and default-browser agents) |
| `--disable-webdriver` | Remote Agent (breaks most test harnesses) |
| `--disable-parental-controls` | OS parental controls |
| `--disable-eme` | DRM (Widevine); breaks Netflix and similar |
| `--disable-webrtc`, `--disable-webspeech`, `--disable-accessibility` | Core platform features (not recommended) |

In `browser/moz.configure`, the `imply_option` lines for `MOZ_NORMANDY`, `MOZ_SERVICES_HEALTHREPORT`, `MOZ_SERVICES_SYNC` and `MOZ_PROFILE_MIGRATOR` can be deleted. `MOZ_DEVTOOLS` can be set to `"server"`.

### Enterprise policies (`distribution/policies.json`)
`DisableTelemetry`, `DisableFirefoxStudies`, `DisableFirefoxAccounts`, `DisableAppUpdate`, `DisableDefaultBrowserAgent`, `DisableFeedbackCommands`, `DisableSystemAddonUpdate`, `DisableFirefoxScreenshots`, `DisableProfileImport`, `FirefoxHome`, `FirefoxSuggest`, `UserMessaging`, `GenerativeAI`, `AIControls`, `TranslateEnabled`, `SkipTermsOfUse`, `NewTabPage`, `SearchEngines`, `Preferences`, `ExtensionSettings`. The full list is in `browser/components/enterprisepolicies/Policies.sys.mjs`.
