# Roadmap

## Phase 0: Knowledge base
- [x] Create `.docs/` (README, TechStack, Features, Platforms, RepoOverhead, Roadmap)

## Phase 1: Repo overhead
- [x] Remove editor, Mercurial, Mozilla-process and CI leftovers (see RepoOverhead.md)
- [x] Remove the linters that only checked those files
- [x] Rewrite the root README
- [ ] Verify with `./mach help`, `./mach lint --list`, `./mach python-test tools/lint/test` (blocked on the build environment below)

## Phase 0.5: Build environment (blocks Phase 2)
- [ ] Install MozillaBuild and Visual Studio 2022 Build Tools on the Windows dev machine
- [ ] `./mach bootstrap`, then a baseline `./mach build` and `./mach run`; record the mozconfig in TechStack.md
- [ ] Linux build environment (WSL2 or native) for GTK and Android builds
- [ ] Android mozconfig and a `./gradlew :fenix:app:assembleDebug` baseline

## Phase 2: Remove unsupported platforms (see Platforms.md)
- [ ] Configure: triplets, toolkits, defines, macOS SDK and host logic, exotic CPUs and JIT backends
- [ ] moz.build branches
- [ ] Delete platform-only directories and files
- [ ] Shared mac IDL/IPDL interfaces
- [ ] Packaging, themes, mozconfigs, taskcluster mac/iOS kinds (leave toolchain scripts untouched)
- [ ] Remove `mobile/android/focus-android` and its gradle and taskcluster references
- [ ] Incremental `#ifdef XP_MACOSX` / `__APPLE__` / JS `macosx` cleanup

## Phase 3: Features
- [ ] Fill in the Decision column in Features.md
- [ ] Disable via configure flags, prefs and policies
- [ ] Delete removed features with their registrations and tests
- [ ] Android: strip Adjust, Sentry, Firebase, Play, MARS, Pocket as decided

## Later
- [ ] Rebranding: name, icons, `browser/branding/`, `mobile/android/branding/`, app IDs
- [ ] Our own update channel and crash reporting endpoint (or none)
- [ ] CI on GitHub Actions
