# Platforms

## Supported
| OS | CPUs | Widget toolkit |
|---|---|---|
| Windows | x86, x86_64, aarch64 | `cairo-windows` (`widget/windows`) |
| Linux | x86_64, aarch64 | `cairo-gtk3*` (`widget/gtk`, Wayland and X11) |
| Android | arm (armv7), aarch64, x86_64 | `cairo-android` (`widget/android`) |

Build hosts: Windows and Linux only. **No macOS, as target or host.**

## Removed or to be removed
macOS, iOS, FreeBSD, OpenBSD, NetBSD, DragonFly, Solaris/illumos, kFreeBSD, Hurd. CPUs: ia64, s390/s390x, ppc/ppc64, alpha, hppa, sparc/sparc64, m68k, mips32/mips64, sh4, riscv64, loongarch64.

## How platform selection works
1. **Triplet to OS:** `split_triplet()` in `build/moz.configure/init.configure` maps a target triplet to an `OS`, `Kernel` and `CPU`. `target_variables` derives `OS_TARGET` and `OS_ARCH`. Further down, the same file sets the defines `XP_WIN`, `XP_UNIX`, `XP_LINUX`, `ANDROID`, `XP_DARWIN`, `XP_MACOSX`, `XP_IOS`, `XP_FREEBSD`, `XP_OPENBSD`, `XP_NETBSD` and `XP_SOLARIS`. The enums live in `python/mozbuild/mozbuild/configure/constants.py`.
2. **Toolkit:** `toolkit_choices()` in `toolkit/moz.configure` maps WINNT to `cairo-windows`, OSX to `cairo-cocoa`, iOS to `cairo-uikit`, Android to `cairo-android`, and **everything else (including the BSDs and Solaris) to GTK**. This sets `MOZ_WIDGET_TOOLKIT` and `MOZ_WIDGET_<NAME>`.
3. **moz.build** files branch on `CONFIG["MOZ_WIDGET_TOOLKIT"]`, `CONFIG["OS_ARCH"]` (`"Darwin"`, `"FreeBSD"`, ...), `CONFIG["OS_TARGET"]` and `CONFIG["TARGET_KERNEL"]`.
4. **Sources** use `#ifdef XP_MACOSX` and similar in C++, `cfg(target_os = "macos")` in Rust, and `AppConstants.platform == "macosx"` in JS. Test manifests use `os == "mac"`.
5. **JIT backends:** `js/moz.configure` picks one of `js/src/jit/{x86,x64,arm,arm64,mips64,loong64,riscv64,wasm32,none}`, plus optional simulators.

## Removal order
Configure first, so removed code is never compiled. Then build files, then delete files.
1. Configure:
   - In `split_triplet()`, drop the darwin, ios, BSD, solaris, kfreebsd and hurd branches and the exotic CPUs.
   - Drop the related `set_define`s.
   - Drop `cairo-cocoa` and `cairo-uikit` from `toolkit_choices()`.
   - Drop the `target_is_osx`, `target_is_ios` and `host_is_osx` blocks in `toolchain.configure`, `toolkit/moz.configure`, `update-programs.configure` and the other configure files.
   - Drop the mips64, loong64, riscv64 and wasm32 JIT and simulator options.
2. moz.build: remove the cocoa, uikit, Darwin and BSD branches (`widget/moz.build`, `toolkit/library/moz.build`, `toolkit/xre/moz.build`, `ipc/*/moz.build`, `xpcom/*/moz.build`, `gfx/*/moz.build`, `browser/app/moz.build`, ...).
3. Delete platform-only directories (about 800 first-party files): `widget/cocoa`, `widget/uikit`, `accessible/mac`, `accessible/ios`, `hal/cocoa`, `security/sandbox/mac`, `security/mac`, `dom/media/platforms/apple`, `browser/themes/osx`, `toolkit/themes/osx`, `browser/app/macbuild`, `mobile/ios`, `build/macosx`, `toolkit/mozapps/macos-frameworks`, `toolkit/mozapps/dmgInstallHelper`, the mac files in `toolkit/xre`, `xpcom`, `ipc`, `gfx` and `toolkit/crashreporter`, plus the BSD and Solaris files.
4. Shared interfaces: `nsIMac*.idl`, `nsILocalFileMac.idl`, `nsIMacShellService.idl`, the `MacIOSurface`/`IOSurfacePort` members of the IPDL `SurfaceDescriptor`, and mach-port IPC.
5. Packaging and CI:
   - The osx entries in `jar.mn`, the `#ifdef XP_MACOSX` blocks in `package-manifest.in`, `browser/config/mozconfigs/macosx*`, and the DMG/pkg Python.
   - In `taskcluster/`, the mac and iOS kinds and the mac toolchain/fetch entries. **Never edit a script used by a Windows, Linux or Android toolchain** (see TechStack.md).
6. Gradually: dead `#ifdef XP_MACOSX` / `__APPLE__` / `XP_DARWIN` paths and JS `platform == "macosx"` checks.

Note: the `toolchain-parity` lint (`taskcluster/kinds/source-test/mozlint.yml`) checks that local toolchains cover every tier-1 *host* platform, macOS included. Update it when the macOS host toolchains are removed.

## Must not be cut
- `XP_UNIX` covers Linux and Android. `XP_DARWIN || XP_LINUX` conditions need care.
- `MOZ_WIDGET_GTK` code isn't BSD-specific; it's our Linux UI.
- The `wasm32-wasi` triplet (`allow_wasi=True`) is used for RLBox/wasm2c library sandboxing. Keep `wasm32` in `split_triplet()` and `CPU_bitness` even after removing `js/src/jit/wasm32`.
- `js/src/jit/none` is needed for `--disable-jit` and the portable baseline interpreter.
- The arm and arm64 JIT simulators are useful for testing ARM code generation on x64.
- `other-licenses/bsdiff` is the updater's diff tool, not BSD-specific. `other-licenses/7zstub` and `other-licenses/nsis` are the Windows installer.
- Apple-only vendored Rust crates (`objc2*`, `core-foundation*`, `mach2`, `cubeb-coreaudio`, ...) stay, because Cargo vendors dependencies for all targets.
- Vendored upstream libraries (breakpad, libwebrtc, skia, NSS, cairo, libvpx/libaom configs): trim their moz.build inclusion and leave their files alone.

## Size of the job
| Item | Count |
|---|---|
| `XP_MACOSX` | 315 files / 1,083 lines |
| `XP_DARWIN` | 128 files |
| `__APPLE__` | 81 files |
| `MOZ_WIDGET_COCOA` | 40 files |
| `XP_IOS` | 36 files |
| BSD/Solaris macros | ~150 files (many vendored) |
| moz.build files with `"cocoa"` / `"Darwin"` | 77 / 70 |
| Rust files with `target_os = "macos"/"ios"` | 53 |
| JS files with `platform == "macosx"` | 374 |
| Test manifests with `os == "mac"` | 2,172 |
