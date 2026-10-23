# Project docs

Working notes for this fork of Mozilla Firefox (forked from mozilla-central at `e33ac8a9b64e`, 2026-10-22).

| File | Contents |
|---|---|
| [TechStack.md](TechStack.md) | Languages, engine components, build system, how toolchains are obtained |
| [Features.md](Features.md) | Catalogue of every feature, with privacy and performance notes and a Decision checklist (Keep / Remove / Disable) |
| [Platforms.md](Platforms.md) | Supported targets and how to strip the unsupported ones |
| [RepoOverhead.md](RepoOverhead.md) | Non-feature repo overhead: what was removed, what must stay and why |
| [Roadmap.md](Roadmap.md) | Phases and status |

## Goal
Turn Firefox into our own browser for **Windows, Linux and Android**, developed with Zed and Claude.

## Decision log
| Date | Decision |
|---|---|
| 2026-10-22 | **Hard fork.** We do not plan to merge mozilla-central again, so deleting code and `#ifdef` branches outright is acceptable. Security fixes must be backported by hand if wanted. |
| 2026-10-22 | **Targets:** Windows (x86, x64, arm64), Linux (x64, arm64), Android (arm, arm64, x86_64). |
| 2026-10-22 | **No macOS at all**, neither as a target nor as a build host. Also drop iOS, the BSDs, Solaris and exotic CPUs. |
| 2026-10-22 | **Tests are kept**, except tests that belong to removed platforms or features. |
| 2026-10-22 | **Android:** base our app on Fenix (Firefox for Android). Remove Focus. |
| 2026-10-22 | Editor is Zed, assistant is Claude: VSCode, Codex and IntelliJ config removed. |
| 2026-10-23 | **No telemetry.** Nothing is collected or sent: Glean and Legacy Telemetry are hard-disabled (upload compiled out, collection prefs locked off), and telemetry-only components are removed. See the telemetry section in Features.md. |
