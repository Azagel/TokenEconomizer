# TokenEconomizer on Windows

This fork makes [microsoft/TokenEconomizer](https://github.com/microsoft/TokenEconomizer) run in
Claude Code on Windows. Upstream lists Windows as unsupported (`docs/PLUGIN.md`). Everything
else is the upstream project, by its authors (Microsoft Research, UC Berkeley, Microsoft
Security), under the Apache License 2.0. Windows port by [Azagel](https://github.com/Azagel).

## Install (Claude Code, Windows)

You need Python 3.10+ (from python.org, or the `py` launcher) and Git for Windows, which Claude
Code on Windows already uses to run its shell and hooks.

```sh
git clone https://github.com/Azagel/TokenEconomizer.git
claude plugin marketplace add "C:/path/to/TokenEconomizer"
claude plugin install tokeneconomizer@tokeneconomizer
```

Keep the marketplace path absolute and leave the clone in place: on Windows the installed copy
finds the package in this clone (see fix 3). Then follow the upstream
[Quick start](README.md#quick-start): `/tokeneconomizer:status`, and once per machine
`/tokeneconomizer:toolprune refresh-catalog`.

To try it without installing, for one session: `claude --plugin-dir "C:/path/to/TokenEconomizer/plugin"`.

## What broke on Windows, and the fix

1. **`python3` is the Microsoft Store alias.** On a stock Windows, `python3` resolves to
   `WindowsApps\python3.exe`, which prints an install prompt and runs nothing, so every hook
   silently answered `{}`. The launchers (`plugin/scripts/te`, `te-hook`) now try an alias under
   `WindowsApps` before using it, and fall back to `python`, then `py -3`.
2. **Backslashes in `$0`.** Claude Code passes the hook path with backslashes, so
   `${0%/*}` found no directory. The launchers turn them into forward slashes first.
3. **The `lib` symlink is a text file.** Git for Windows checks symlinks out as plain files
   holding the target, and Claude Code's install copy carries that file, not the package. The
   bootstrap (`te_bootstrap.py`) now reads such a stub, and for an install copy under
   `~/.claude/plugins/cache/` finds the package in the clone recorded in
   `~/.claude/plugins/known_marketplaces.json`.
4. **Background work was killed.** Claude Code runs each hook in a Windows job object that kills
   its children when the hook returns, so the fold (what learns from a finished session) and the
   loop guard's judge never ran. `tokeneconomizer/plugin/winspawn.py` starts them with
   `CREATE_BREAKAWAY_FROM_JOB`, or through WMI `Win32_Process.Create` when the job forbids
   breakaway.
5. **Session logs read in the ANSI code page.** Python on Windows opens text as cp1252, and the
   fold crashed on the first non-ASCII byte of a transcript. The launchers set `PYTHONUTF8=1`.
6. **CRLF launchers.** With `core.autocrlf=true`, `sh` fails on the launchers' `\r`.
   `.gitattributes` keeps them LF.
7. **No file locking.** The state store's lock was a no-op without `fcntl`; it now uses
   `msvcrt.locking`.
8. **Paths in shell commands.** Commands are Git Bash text, but `cd` targets were resolved with
   Windows path rules, so learned command forms came out as `cd ..\elsewhere`. They are now
   compared in one POSIX spelling (`C:\r`, `C:/r` and `/c/r` are the same directory).

## Verified

On Windows 11, Python 3.12, Git for Windows, Claude Code 2.1.312 (2026-10-28):

- `python -m pytest tests`: all pass. Skipped on Windows: tests built on POSIX permission bits,
  real symlinks, `sh` stand-in executables, the `ps`-based Copilot `-p` detection, and the Pi
  extension. A new test covers fix 3.
- A real `claude -p` session with the plugin loaded (`--plugin-dir`), then with
  `claude plugin install --scope local`: hooks run, the session is folded in the background,
  and `te status` reports its tokens.

## Not covered

- Codex, Copilot CLI and Pi on Windows: not tested.
- Copilot's `-p` (headless) is not told apart on Windows (no `ps`).
- Claude Code 2.1.312 is newer than the range upstream tested (2.1.300 to 2.1.310).
- No claim of savings: the cost results in the README are the upstream paper's, measured on
  Linux.

## Files changed from upstream

`.gitattributes`, `WINDOWS.md` (new), `plugin/scripts/te`, `plugin/scripts/te-hook`,
`plugin/scripts/te_bootstrap.py`, `tokeneconomizer/plugin/winspawn.py` (new),
`tokeneconomizer/plugin/context.py`, `tokeneconomizer/plugin/store.py`,
`tokeneconomizer/plugin/hosts.py`, `tokeneconomizer/microskills/observe.py`, and under `tests/`:
`test_live_core.py`, `test_live_hooks.py`, `test_live_repomap.py`, `test_live_repos.py`,
`test_live_safety.py`, `test_plugin_package.py`.
