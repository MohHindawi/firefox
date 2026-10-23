# Tech stack

## Languages
| Language | Where | Used for |
|---|---|---|
| C++ | `dom/`, `layout/`, `netwerk/`, `gfx/`, `widget/`, `xpcom/`, `js/src/`, ... | Most of the engine (Gecko) |
| Rust | `servo/`, `gfx/wr/`, `toolkit/library/rust/`, `third_party/rust/` (636 vendored crates) | Style engine, WebRender, many components |
| JavaScript (ES modules, `.sys.mjs`) | `browser/`, `toolkit/`, `devtools/` | The whole desktop UI and most browser features |
| HTML / CSS / XUL / Fluent (`.ftl`) | `browser/`, `toolkit/` | UI markup, styling, localization |
| Kotlin / Java | `mobile/android/` | Android: GeckoView, android-components, Fenix |
| Python | `python/`, `build/`, `testing/`, `tools/` | `mach` (the build and dev CLI), configure, test harnesses, linters |

## Engine components
| Component | Path | Role |
|---|---|---|
| Gecko DOM | `dom/` | Web APIs (WebIDL bindings, workers, media, WebGPU, ...) |
| Layout | `layout/` | Box layout, frames, painting display lists |
| Stylo | `servo/` | Parallel CSS engine (Rust, shared with Servo) |
| WebRender | `gfx/wr/` | GPU renderer (Rust) |
| Graphics | `gfx/` | 2D (Moz2D/Skia/Cairo), layers/compositor, fonts (thebes), ANGLE, WebGL |
| SpiderMonkey | `js/` | JavaScript and WebAssembly engine (interpreter, Baseline, Ion/Warp JITs) |
| Necko | `netwerk/` | Networking: HTTP/1-3, cache, DNS/DoH, cookies, WebSockets |
| IPC / Fission | `ipc/`, `dom/ipc/` | Multi-process (site isolation), IPDL protocols |
| XPCOM | `xpcom/` | Component model, threads, strings, IO |
| Security | `security/` | NSS (TLS/crypto), PSM certificate handling, process sandboxing |
| NSPR | `nsprpub/` | Portable runtime used by NSS |
| Widget | `widget/` | OS windowing/input: `windows/`, `gtk/` (Linux), `android/` |
| Accessibility | `accessible/` | Platform a11y bridges (MSAA/UIA, ATK, Android) |
| Media | `dom/media/`, `media/` | Playback, WebRTC (`third_party/libwebrtc`), codecs (dav1d, aom, vpx, ffmpeg) |
| Toolkit | `toolkit/` | App-independent features: extensions, places, telemetry, updater, crash reporter, ... |

## Applications
| App | Path | Notes |
|---|---|---|
| Firefox desktop | `browser/` | Front-end (JS/HTML/CSS) on top of `toolkit/` and Gecko |
| GeckoView | `mobile/android/geckoview` | Gecko packaged as an Android library (AAR) |
| android-components | `mobile/android/android-components` | Reusable browser building blocks (`feature-*`, `service-*`, `browser-*`, `lib-*`, `ui-*`) |
| Fenix | `mobile/android/fenix` | Firefox for Android: the full app, built from GeckoView + android-components. Our Android base |
| Focus | `mobile/android/focus-android` | Separate minimal privacy browser. To be removed |

## Build system
- **`./mach`** is the single entry point (`mach build`, `mach run`, `mach test`, `mach lint`, `mach format`, ...). On Windows, mach runs inside the MozillaBuild shell.
- **Configure:** `moz.configure`, `build/moz.configure/*.configure`, `toolkit/moz.configure`, `browser/moz.configure`, `js/moz.configure`. These are written in a Python sandbox DSL (`option`, `depends`, `imply_option`, `set_config`, `set_define`). Options come from a `mozconfig` file.
- **Build description:** a `moz.build` file in each directory (`DIRS`, `SOURCES`, `UNIFIED_SOURCES`, `EXPORTS`, `XPIDL_SOURCES`, test manifests). Conditions use `CONFIG["OS_TARGET"]`, `CONFIG["MOZ_WIDGET_TOOLKIT"]`, etc.
- **Rust:** a Cargo workspace (`Cargo.toml`) with everything vendored into `third_party/rust`. `mach vendor rust` refreshes it, and Cargo needs every crate in `Cargo.lock`, even those used only on other OSes.
- **Front-end packaging:** `jar.mn` manifests and `browser/installer/package-manifest.in`.
- **Android:** Gradle (`build.gradle`, `settings.gradle`, `gradlew`) drives the Java/Kotlin parts, while mach builds the native GeckoView libraries.
- **Toolchains:** `./mach bootstrap` and `--enable-bootstrap` download prebuilt clang, rust, nasm, node, sccache, etc. from **Mozilla's public Taskcluster index** into `~/.mozbuild`. The index key for each toolchain is a digest of its definition in `taskcluster/kinds/toolchain` and `taskcluster/kinds/fetch` and of its build scripts (`taskcluster/scripts/misc/*`, `build/build-clang/*`). **Changing those files breaks bootstrap**, so `taskcluster/` stays even though we don't use Mozilla CI.

## Developer setup (Windows)
Not set up yet on this machine. Standard requirements:
1. MozillaBuild (provides the shell, Python and tools mach expects).
2. Visual Studio 2022 Build Tools (C++ workload and Windows SDK). Mozilla builds with clang-cl, which bootstrap downloads.
3. `./mach bootstrap` (choose "Firefox for Desktop"), then `./mach build` and `./mach run`.

Baseline mozconfig: to be recorded here once the first build succeeds.
