# Repo overhead

Non-feature parts of the repository: Mozilla process, CI, other editors and version control systems.

## Removed (Phase 1)
| Path | Why it went |
|---|---|
| `.vscode/` | VSCode config; we use Zed |
| `.codex/` | Codex config; we use Claude |
| `.idea/` | Android Studio ktfmt plugin config; Gradle configures ktfmt itself |
| `.github/` | Mozilla's `pr-handler.yml` auto-closes and locks every PR from outside Mozilla's GitHub team, including ours |
| `.hgignore`, `.hgtags`, `.hg-annotate-ignore-revs` | Mercurial only. `.hgignore` went together with the `ignorefile` linter, which compared it to `.gitignore` |
| `.agents/` | Byte-identical copy of `.claude/skills`. Went together with the `agent-skills-sync` linter |
| `review-context.toml` | Mozilla AI-review config. Went together with its linter |
| `mots.yaml` | Mozilla module owners. `mach file-info` and the docs build handle it being missing |
| `.lando.ini` | Mozilla's landing service (`mach try` only) |
| `.trackerignore`, `.watchmanconfig` | Empty file / watchman only |

Follow-up edits made with the removal:
- Root `moz.build`: dropped the `Files()` metadata blocks for the deleted files, and repointed `SPHINX_TREES["ai-agent-tools/skills"]` to `.claude/skills`.
- Linters: `tools/lint/test/python.toml`, `tools/lint/test-manifest-toml.yml` and the `pyproject.toml` ruff excludes. Removed the `ignorefile-test`, `agent-skills-sync` and `review-context-toml` jobs from `taskcluster/kinds/source-test/mozlint.yml`.
- Docs: the ignorefile linter page and the review-context page.
- `.claude/skills` script paths that pointed into `.agents/skills`.
- `.gitignore`: the obsolete `lint-ignore-next-line` markers.

## Kept on purpose
| Path | Why |
|---|---|
| `taskcluster/` | `./mach bootstrap` generates toolchain task definitions from it and downloads Mozilla's prebuilt toolchains by digest. Changing toolchain definitions or scripts breaks bootstrap (see TechStack.md) |
| `.cron.yml`, `.taskcluster.yml` | Read by taskgraph code; tiny, kept to avoid risk |
| `.arcconfig` | `python/mozbuild/mozbuild/vendor/rewrite_mozbuild.py` uses it to find the repo root for `mach vendor` |
| `.ycm_extra_conf.py` | `build/moz.build` installs it into the objdir |
| `.lldbinit` | Loads the Gecko lldb helpers (`python/lldbutils`); useful with lldb on Linux and Windows, not only macOS. `build/moz.build` installs a loader for it into the objdir |
| `.zed/`, `.claude/`, `.mcp.json` | Our tools |
| `.git-blame-ignore-revs`, `.mailmap`, lint/format configs | Used by git, `mach lint` and `mach format` |
| `testing/mozbase`, `testing/mozharness`, `testing/web-platform/tests/tools` | mach's Python environment imports from them |
| `testing/talos`, `third_party/webkit/PerformanceTests` (Speedometer, webaudio) | Used by PGO builds |
| `other-licenses/7zstub`, `other-licenses/nsis` | The Windows installer |

## Tests
About 75% of the 485k files are tests (web-platform-tests ~198k, test262 ~58k, jit-test ~9.6k, layout reftests ~19k). **Decision: keep them**, except tests for removed platforms and features.

Two gotchas when removing tests:
- `moz.build` test manifests are read on every configure, even with `--disable-tests`. A deleted test directory still listed in a `*_MANIFESTS` variable breaks the build.
- Every `mach_commands.py` is registered in `python/mach/mach/command_util.py`. Deleting one without removing its entry breaks `mach help`.

## Git history
The clone has full history: about 1M commits and a 5.22 GiB pack. Deleting files adds commits; it does not shrink `.git`. Options, all deferred:
- Re-clone with `--filter=blob:none` (partial clone) to save local disk.
- Start a fresh-history repository from a cleaned tree. This is the only way to get a small repo, and it loses upstream blame and history.
