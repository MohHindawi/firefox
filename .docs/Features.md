# Feature catalogue

How to use this file: fill in the **Decision** column with **Keep**, **Remove** (delete the code), or **Disable** (keep the code but switch it off with a build flag, pref or policy). **Suggested** is Claude's recommendation for a personal, privacy-respecting fork; ignore it freely.

**Difficulty** is how hard it is to delete cleanly:
- **Low:** self-contained, or a build flag already exists.
- **Med:** several callers or UI hooks need edits.
- **High:** used all over the tree, so disable it instead.

File counts include tests and were taken at the fork point.

Already gone upstream (nothing to do): Pocket on desktop, Shopping/Review Checker, Contile sponsored tiles.

---

## 1. Telemetry, data collection and Mozilla services

| Feature | Path(s) | What it does | Files | Difficulty | Existing switch | Suggested | Decision |
|---|---|---|---|---|---|---|---|
| Glean (FOG) | `toolkit/components/glean`, `third_party/rust/glean*`, `metrics.yaml` everywhere | Main telemetry SDK | ~470 + yamls | High (326 JS + 302 C++ callers) | Upload is compiled out unless `MOZ_TELEMETRY_REPORTING` (defaults to official builds only); policy `DisableTelemetry` | Disable | |
| Legacy Telemetry | `toolkit/components/telemetry` | Histograms, scalars, events, main/crash pings | 258 | Med-High | Same as Glean | Disable | |
| Pingsender / uninstall ping | `toolkit/components/telemetry/pingsender`, `pings/UninstallPing.sys.mjs` | Sends pings at shutdown and uninstall | ~10 | Low | Telemetry off | Remove | |
| Normandy | `toolkit/components/normandy` | Remote pref rollouts and surveys | 65 | Low-Med | `MOZ_NORMANDY` in `browser/moz.configure`; `app.normandy.enabled` | Remove | |
| Nimbus (experiments) | `toolkit/components/nimbus`, `browser/components/nimbus` | Experiments, rollouts, Firefox Labs | 96 | High (83 files read feature values) | `app.shield.optoutstudies.enabled`; policy `DisableFirefoxStudies` | Disable | |
| Crash reporter | `toolkit/crashreporter`, `toolkit/components/crashes`, `crashmonitor` | Breakpad minidumps, upload to Mozilla, about:crashes | ~1,100 | Low | `--disable-crashreporter`, `--with-crashreporter-url=` | Remove, or point at own server | |
| DAP | `toolkit/components/dap` | Privacy-preserving aggregated telemetry | 25 | Low-Med | Telemetry prefs | Remove | |
| gecko-trace | `toolkit/components/gecko-trace` | Tracing spans sent as Glean pings | 27 | Low-Med | none | Remove | |
| Captcha detection | `toolkit/components/captchadetection` | Telemetry-only CAPTCHA ping | 28 | Low | none | Remove | |
| Search/SERP telemetry | `browser/actors/SearchSERPTelemetry*`, parts of `browser/components/search` | Records search ad clicks, categorizes results pages | part of 224 | Med | `browser.search.serpEventTelemetryCategorization.enabled` | Remove | |
| Browser usage telemetry | `browser/modules/BrowserUsageTelemetry.sys.mjs`, `UIDensityTelemetry.sys.mjs`, `browser/components/StartupTelemetry.sys.mjs` | UI interaction metrics | ~4 | Med | Telemetry off | Disable | |
| Context ID | `browser/modules/ContextId.sys.mjs` | Stable ID for sponsored content and Suggest | 1 | Med | none | Remove with sponsored content | |
| Attribution / referrals | `browser/components/attribution`, `browser/components/referrals` | Install-source attribution, referral pings | 24 | Low | none | Remove | |
| Content relevancy | `toolkit/components/contentrelevancy` | Local interest classification for content and ads | 14 | Low | Nimbus/pref | Remove | |
| Remote Settings | `services/settings` | Remote data: add-on blocklist, CRLite cert revocation, search config, ML models | 191 | High | Policy `DisableRemoteSettingsAndAcceptSecurityConsequences` | Keep (security) | |
| Mozilla accounts (FxA) | `services/fxaccounts`, `browser/components/AccountsGlue.sys.mjs` | Login, devices, send-tab | 53+ | Med-High | `identity.fxaccounts.enabled`; policy `DisableFirefoxAccounts` | ? (needed for Sync) | |
| Sync | `services/sync`, `services/crypto`, `services/common`, `browser/components/syncedtabs` | Bookmark, history, password and tab sync via Mozilla servers | ~300 | Med | `MOZ_SERVICES_SYNC` in `browser/moz.configure` | ? | |
| Sponsored new tab content | `browser/extensions/newtab/lib/AdsFeed.sys.mjs`, `DiscoveryStreamFeed`, `NewTabAttribution*`, `browser/components/mozcachedohttp` | Sponsored tiles and stories (`ads.mozilla.org`) | part of 694 | Med | `browser.newtabpage.activity-stream.showSponsored*`; policy `FirefoxHome` | Remove | |
| Firefox Suggest / Merino | `browser/components/urlbar/private/*` | Sponsored and online address bar suggestions (AMP, Yelp, weather, ...) | ~140 | Med | `browser.urlbar.quicksuggest.enabled`; policy `FirefoxSuggest` | Remove | |
| Messaging / ASRouter | `browser/components/asrouter`, `toolkit/components/messaging-system`, `browser/components/messagepreview`, `browser/components/aboutwelcome` | Recommendations, infobars, spotlights, onboarding, product promos | ~495 | Med-High (onboarding uses it) | Policy `UserMessaging` | Disable, rewrite onboarding later | |
| Mozilla product promos | prefs `browser.contentblocking.report.vpn*` / `monitor*`, `browser.privatebrowsing.vpnpromourl`, `browser.preferences.moreFromMozilla` | VPN/Monitor/Relay ads | prefs | Low | prefs | Remove | |
| IP Protection (built-in VPN) | `browser/components/ipprotection`, `toolkit/components/ipprotection`, `browser/extensions/ipp-activator` | Mozilla VPN-backed proxy | 182 | Low-Med | `browser.ipProtection.enabled` (off by default) | Remove | |
| Default browser agent | `toolkit/mozapps/defaultagent` | Windows scheduled task that reports and nags about the default browser | 51 | Low | `--disable-default-browser-agent` | Remove | |
| App updater | `toolkit/mozapps/update`, `toolkit/components/maintenanceservice`, `bitsdownload` | Mozilla update service, MAR files, Windows maintenance service | ~460 | Low (flags) | `--disable-updater`, `--disable-maintenance-service`, `--disable-update-agent`, `--disable-bits-download` | ? (need our own update channel eventually) | |
| Background tasks | `toolkit/components/backgroundtasks`, `taskscheduler` | Headless tasks (update agent, pingsender, cleanup) | 93 | Med | `--disable-backgroundtasks` | Keep for now | |
| Captive portal detection | `toolkit/components/captivedetect` | Pings `detectportal.firefox.com` | 18 | Low | `network.captive-portal-service.enabled` | Keep, maybe own endpoint | |
| DoH rollout heuristics | `toolkit/components/doh` | Region-based automatic DNS-over-HTTPS | 29 | Low-Med | `doh-rollout.*`, `network.trr.mode` | ? | |
| Safe Browsing | `toolkit/components/url-classifier`, `reputationservice` | Google Safe Browsing, tracking lists, download reputation | 250 | High (tracking protection shares it) | `browser.safebrowsing.*` | Keep | |
| Report Broken Site | `browser/components/reportbrokensite`, `toolkit/components/reportbrokensite` | Sends site-compat reports to Mozilla | 40 | Low | Policy `DisableFeedbackCommands` | Remove | |
| Terms of Use modal | `asrouter/modules/PreonboardingSplash.sys.mjs` | Mozilla ToU acceptance | — | Low-Med | Policy `SkipTermsOfUse` | Remove | |

## 2. Core desktop UI (keep unless replacing)

| Feature | Path(s) | What it does | Files | Difficulty | Decision |
|---|---|---|---|---|---|
| Tabbed browser | `browser/components/tabbrowser` | Tabs, tab groups, smart grouping, previews | 380 | High | |
| Address bar | `browser/components/urlbar` | Awesomebar and its providers | 907 | High | |
| Search | `browser/components/search`, `toolkit/components/search` | Search engines and config | 435 | High | |
| Session restore | `browser/components/sessionstore`, `toolkit/components/sessionstore` | Restores windows and tabs | 420 | High | |
| Bookmarks and history | `browser/components/places`, `toolkit/components/places` | Places database and UI | 752 | High | |
| Preferences | `browser/components/preferences` | about:preferences | 431 | High | |
| Toolbar customization | `browser/components/customizableui` | Customize toolbars | 215 | High | |
| Downloads | `browser/components/downloads`, `toolkit/components/downloads` | Downloads panel and library | 109 | High | |
| Sidebar / vertical tabs | `browser/components/sidebar` | Sidebar revamp | 127 | Med | |
| Private browsing | `browser/components/privatebrowsing` | about:privatebrowsing | 81 | Med | |
| Protections dashboard / site info | `browser/components/protections`, `controlcenter` | about:protections, site identity panel | 39 | Med | |
| Containers | `browser/components/contextualidentity`, `originattributes` | Multi-account containers | 156 | Med | |
| Fingerprinting protection | `browser/components/resistfingerprinting`, `toolkit/components/resistfingerprinting` | RFP and FPP | 233 | High | |
| Permissions, prompts, WebRTC UI | `browser/components/permissions`, `prompts`, `webrtc` | Permission and media sharing UI | ~11 | High | |
| OS shell integration | `browser/components/shell` | Default browser, pinning, jump lists | 117 | Med | |
| Enterprise policies | `browser/components/enterprisepolicies`, `toolkit/components/enterprisepolicies` | Policy engine; handy for our own defaults | 220 | Keep | |
| Browser WebExtension APIs | `browser/components/extensions` | Tabs, windows, menus APIs, ... | 465 | High | |
| Themes | `browser/themes` (incl. `addons/` colorway themes) | Platform CSS and built-in themes | 554 | High (colorways Low) | |

## 3. Optional desktop features

| Feature | Path(s) | What it does | Files | Difficulty | Existing switch | Suggested | Decision |
|---|---|---|---|---|---|---|---|
| Smart Window (AI window) | `browser/components/aiwindow` | AI browsing window: chat, memories, agent; uses a remote Mozilla endpoint | 677 | Low-Med | `browser.smartwindow.enabled` (off) | ? | |
| AI chatbot sidebar / Link Preview / Page Assist | `browser/components/genai` | ChatGPT/Gemini/Claude etc. in the sidebar, AI link previews | 74 | Low-Med | `browser.ml.chat.enabled`, `browser.ml.linkPreview.enabled`; policy `GenerativeAI` | ? | |
| Mini window | `browser/components/miniwindow` | Small popup browser window | 41 | Low | — | ? | |
| Multiline editor | `browser/components/multilineeditor` | Rich editor used by the Smart Window bar | 16 | Low (with aiwindow) | — | Follows aiwindow | |
| Tab notes | `browser/components/tabnotes` | Notes attached to tabs | 25 | Low | `browser.tabs.notes.enabled` (off) | ? | |
| New tab page | `browser/extensions/newtab`, `browser/components/newtab`, `topsites` | Top sites, stories, weather, wallpapers, widgets | 840 | Med-High | Policies `FirefoxHome`, `NewTabPage` | Keep, strip sponsored parts | |
| Firefox View | `browser/components/firefoxview` | Recent, open and synced tabs hub | 82 | Med | `browser.firefox-view.*` | ? | |
| Screenshots | `browser/components/screenshots` | Built-in screenshot tool | 63 | Low | Policy `DisableFirefoxScreenshots` | Keep | |
| Translations | `browser/components/translations`, `toolkit/components/translations` | Local, on-device page translation | 355 | Low-Med | `browser.translations.enable` | Keep | |
| Picture-in-Picture | `toolkit/components/pictureinpicture`, `browser/extensions/pictureinpicture` | Pop-out video | 184 | Low-Med | `media.videocontrols.picture-in-picture.enabled` | Keep | |
| Password manager UI | `browser/components/aboutlogins`, `passwordmgr` | about:logins | 106 | Med | — | Keep | |
| Import from other browsers | `browser/components/migration` | Imports from Chrome, Edge, Safari | 150 | Low-Med | `MOZ_PROFILE_MIGRATOR`; policy `DisableProfileImport` | Keep | |
| Profile manager | `browser/components/profiles` | New profile switcher | 210 | Med | `browser.profiles.enabled` | ? | |
| Backup | `browser/components/backup` | Scheduled profile backup and restore | 162 | Low-Med | `browser.backup.enabled` | ? | |
| Content analysis (DLP) | `browser/components/contentanalysis`, `toolkit/components/contentanalysis` | Enterprise data-loss-prevention hook | 97 | Low-Med | Policy `ContentAnalysis` | Remove | |
| Taskbar Tabs | `browser/components/taskbartabs` | Pin websites as Windows apps | 42 | Low | — | ? | |
| Text recognition (OCR) | `browser/components/textrecognition` | Copy text from images | 11 | Low | — | Keep | |
| QR code / share | `browser/components/qrcode`, `sharing` | Page QR code, share dialog | 66 | Low | — | Keep | |
| Custom keyboard shortcuts | `browser/components/customkeys` | Rebind shortcuts | 16 | Low | — | Keep | |
| UITour | `browser/components/uitour` | Lets mozilla.org pages drive the UI | 40 | Low-Med | `browser.uitour.enabled` | Remove | |
| Remote-control indicator | `browser/components/remotecontrol` | Shows when WebDriver is controlling the browser | 9 | Low | Follows `--disable-webdriver` | Keep | |
| Page data | `browser/components/pagedata` | OpenGraph/Schema.org extraction | 31 | Low | — | ? | |
| about:unloads | `browser/components/tabunloader` | Tab unloading UI | 5 | Low | — | Keep | |
| Storybook | `browser/components/storybook` | UI component gallery (dev only) | 61 | Low | — | Keep (dev tool) | |
| Launch on login | `browser/components/defaultlaunchonlogin` | Start with Windows | 4 | Low | Policy `DisableLaunchOnLogin` | ? | |
| Touch Bar | `browser/components/touchbar` | macOS Touch Bar | 10 | Low | — | Remove (macOS) | |
| Protocol handlers | `browser/components/protocolhandler` | `registerProtocolHandler` | 11 | Med | — | Keep | |

## 4. Built-in system add-ons (`browser/extensions/`)

| Add-on | Files | What it does | Difficulty | Suggested | Decision |
|---|---|---|---|---|---|
| `newtab` | 694 | The new tab page (see above) | Med-High | Keep, strip sponsored | |
| `webcompat` | 645 | Per-site compat fixes and SmartBlock shims | Low (but sites break more without it) | Keep | |
| `formautofill` | 435 | Address and credit-card autofill | Med | Keep | |
| `pictureinpicture` | 51 | Per-site PiP fixes | Low | Keep | |
| `ipp-activator` | 31 | Activates IP Protection | Low | Remove with IP Protection | |
| `search-detection` | 13 | Detects add-ons hijacking search (telemetry) | Low | Remove | |

## 5. Shared toolkit features

| Feature | Path(s) | What it does | Files | Difficulty | Suggested | Decision |
|---|---|---|---|---|---|---|
| WebExtensions + Add-ons Manager | `toolkit/components/extensions`, `toolkit/mozapps/extensions` | Extension engine, AMO, blocklist | 1,648 | High | Keep | |
| Password manager | `toolkit/components/passwordmgr` | Logins, generation, Relay integration | 341 | High | Keep | |
| Form history / autofill | `satchel`, `formautofill`, `autocomplete` | Form history | 232 | High | Keep | |
| Anti-tracking | `antitracking`, `cleardata`, `content-classifier` | Enhanced Tracking Protection, cookie purging, bounce tracking | ~765 | High | Keep | |
| PDF viewer | `toolkit/components/pdfjs`, `aboutpdf` | PDF.js | 460 | Low | Keep | |
| Reader mode / Narrate | `reader`, `narrate` | Reader view, text-to-speech | 72 | Low-Med | Keep | |
| Local ML runtime | `toolkit/components/ml`, `aboutinference` | On-device ONNX/WASM inference; used by link preview, smart tab groups, Suggest ML, new tab | 208 | Med | ? | |
| Page extractor | `toolkit/components/pageextractor` | Page text extraction for AI features | 29 | Low | Follows AI choices | |
| Thumbnails | `toolkit/components/thumbnails` | Page thumbnails for new tab | 55 | Med | Keep | |
| Printing | `printing`, `printingui` | Print and print preview | 82 | High | Keep | |
| Notifications / push | `alerts`, `push`, `toolkit/mozapps/notificationserver`, `pushnotificationhelper` | Desktop notifications, web push | 58 | Med | Keep | |
| Diagnostic about: pages | `aboutconfig`, `aboutmemory`, `aboutprocesses`, `aboutthirdparty`, `aboutwebauthn`, `aboutwindowsmessages`, `aboutcheckerboard`, `certviewer` | about:config, about:memory, ... | ~190 | Low each | Keep | |
| app-services bridge (UniFFI) | `uniffi-bindgen-gecko-js`, `uniffi-js`, `third_party/application-services` | Rust components: logins, places, suggest, sync | 155 + 1,548 | High | Keep | |

## 6. Developer tools

| Feature | Path(s) | What it does | Files | Difficulty | Existing switch | Suggested | Decision |
|---|---|---|---|---|---|---|---|
| DevTools UI | `devtools/client` | Inspector, console, debugger, network, ... | 7,410 | Med | `MOZ_DEVTOOLS=server`; policy `DisableDeveloperTools` | Keep | |
| DevTools server | `devtools/server`, `devtools/shared` | Debugging actors; also used by Android remote debugging | 1,524 | High | always built | Keep | |
| DevTools docs | `devtools/docs` | Docs | 769 | Low | — | ? | |
| Remote Agent (WebDriver BiDi / Marionette) | `remote/` | Browser automation; the test harnesses rely on it | 1,118 | Med | `--disable-webdriver` | Keep (tests need it) | |
| geckodriver | `testing/geckodriver` | WebDriver proxy | 44 | Low | `--enable-geckodriver` | Keep | |

## 7. Android

### Apps
| App / module | Path | Files | What it is | Suggested | Decision |
|---|---|---|---|---|---|
| Fenix | `mobile/android/fenix` | 3,919 | Firefox for Android, the full app. Our base | Keep (rebrand) | |
| Focus | `mobile/android/focus-android` | 708 | Separate minimal privacy browser | Remove | |
| android-components | `mobile/android/android-components` | 9,568 | Library of browser building blocks used by Fenix | Keep | |
| GeckoView | `mobile/android/geckoview` | 621 | Engine as an Android library | Keep | |
| Sample apps | `geckoview_example`, `mobile/android/examples`, `android-components/samples` | ~430 | Dev demos | ? | |
| test_runner | `mobile/android/test_runner` | 27 | Runs GeckoView tests | Keep | |

### Third-party SDKs and Mozilla services in Fenix
These are wired in through `fenix/app/build.gradle` and `fenix/.../components/`.

| Dependency | What it does | Suggested | Decision |
|---|---|---|---|
| Adjust (`libs.adjust`) | Marketing install attribution | Remove | |
| Sentry (`lib-crash-sentry`) | Crash reporting to Sentry | Remove | |
| Google Play install referrer, ads ID, in-app review | Google Play integrations | Remove (keep review prompt?) | |
| Firebase push (`lib-push-firebase`) | Push messages (WebPush, Send Tab) | ? | |
| Glean (`service-glean`) | Telemetry | Disable/remove | |
| Nimbus (`service-nimbus`, `nimbus.fml.yaml`) | Experiments | Disable | |
| MARS ads (`service-mars`) | Sponsored content | Remove | |
| Pocket (`service-pocket`) | Home screen stories | Remove | |
| Google Lens (`components/lens`) | Image search | ? | |
| On-device and remote LLMs (`lib-llm-gemininano`, `lib-llm-mlpa`, `feature-summarize`) | Page summarization | ? | |

### Fenix feature areas (`fenix/app/src/main/java/org/mozilla/fenix/`)
settings, home, tabstray, browser, debugsettings, translations, downloads, library, onboarding, tabgroups, bookmarks, share, webcompat, search, termsofuse, ipprotection, addons, exceptions, collections, experiments, pdf, messaging, crashes, summarization, reviewprompt, microsurvey, wallpapers, privacyreport, pbmlock, listentopage, whatsnew.

Candidate cuts: termsofuse, ipprotection, experiments, messaging, microsurvey, reviewprompt, summarization. Decide each:

| Area | Decision |
|---|---|
| termsofuse | |
| ipprotection | |
| experiments | |
| messaging | |
| microsurvey | |
| reviewprompt | |
| summarization | |
| wallpapers | |
| collections | |

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
