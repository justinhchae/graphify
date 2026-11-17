# Fork Notes

See [PS_SETUP.md](PS_SETUP.md) for installation, environment setup, and per-project graph configuration.

Forked from: https://github.com/safishamsi/graphify  
PS fork: https://github.com/EsriPS/graphify  
Origin (maintainer): https://github.com/justinhchae/graphify.git  
Branch: v8-ps

---

## Changelog

### 2026-11-16 — Upstream sync (upstream/v8 → 480b61ed, v0.9.81–v0.9.111)

**901 upstream commits, 31 versions.** Reset-and-reapply again (backup branch
`v8-ps-backup-20261116`). Every re-applied patch's diff size matched its documented
size exactly (`__main__.py` 33, `extract.py` 10, `analyze.py`/`detect.py`/`watch.py` 2,
`gen.py` 1), so nothing was reformatted.

**Patch set changes**
- **Dropped patch 7** (`paths.py` read-only guard). Upstream fixed the same bug in
  `_atomic_replace`, with a deliberately narrower scope: it refuses only on the generic
  write path, so install/cache callers of `os_replace_with_fallback` can still replace
  their own read-only copies. Our patch blocked all eight callers, which contradicted
  that intent.
- **Patch 3 adapted.** Upstream now routes suffix checks through `effective_suffix`
  (`rcfile.py`), so the `py_paths`/`py_results` filter became
  `effective_suffix(p) in (".py", ".pyt")`. Two further Python-*source* gates upstream
  added after v0.9.80 also include `.pyt`: `_is_python_file` (member-call resolver) and
  the builtin-name guard. `.pyt` now appears at six sites in `extract.py`.
- **`pyyaml` is now a declared core dependency** upstream; the "install it separately"
  step is gone.
- **Considered and rejected:** upstream's `.graphifyrc` language overrides (#2961)
  would let a per-project `language.pyt=python` replace patches 2 and 3. Rejected so
  `.pyt` works in every repo with no per-repo file. It was never tested end to end, and
  it would not have covered patch 4 either (`analyze._cross_language` reads the raw
  suffix).

**Old test patches: none carried.** None of the v0.9.80 skips or fixes (see the
historical sections below) were re-applied; the full run showed none of those
failures, so upstream absorbed them or its tests now skip natively.

**Test suite.** The first full run on the patched tree had 19 failures; the last
full run before the final test edits had 8 failed / 10990 passed / 468 skipped.
Disposition of the 19:

| Count | Tests | Disposition |
|---|---|---|
| 5 | `test_hooks.py` (checkout no-op, worktree guard ×2, toplevel anchor, baked viz limit, cdpath) | skipped on Windows: hook scripts need a POSIX `sh` |
| 1 | `test_language_override_lifecycle[src]` | test fixed: expected value now `.as_posix()` (`source_file` is POSIX-canonical, #2627) |
| 7 | `test_ext_tagged_id_clash.py` | env: `tree-sitter-sql` was not installed, so `.sql` produced no nodes |
| 2 | `test_multilang.py` (T-SQL bracketed procedure, escaped double quote) | env: `tree-sitter-sql` 0.3.13 parses differently from upstream's locked 0.3.11; pinned to 0.3.11 |
| 2 | `test_code_only_update_all_hosts.py` (devin, aider) | skipped on Windows: bash runbook cannot run a backslash interpreter path |
| 2 | `test_resolve_memoization.py`, `test_source_key_memoization.py` | tests fixed: warm the absolute key first (see below) |
| 2 | `test_llm_backends.py` (deepseek base_url) | **open**: see Known Upstream Windows Failures |

**Upstream behavior worth knowing (new this sync)**
- Windows `Path.resolve()` calls `os.getcwd()` on a memo *miss* (measured: 2 calls for
  one nonexistent absolute path). The two memoization tests asserted no `getcwd` call
  on the first absolute lookup, which only holds on POSIX. Warming the absolute key
  first makes them assert the real contract: a memo *hit* never calls `getcwd`.
- The aider and devin skills run `bash -c "$(cat graphify-out/.graphify_python) -c ..."`.
  `.graphify_python` holds a Windows interpreter path with backslashes, which unquoted
  bash strips to `C:Users...` (`command not found`). A Windows user running those two
  skills from a bash shell hits the same failure. Upstream's runbook; not patched here.
- `tree-sitter-sql` is not pinned in `pyproject.toml` but `uv.lock` pins 0.3.11, and
  two tests depend on ERROR-recovery details that changed in 0.3.13.

### 2026-10-08 — Upstream sync (upstream/v8 → 6478eb7, v0.9.43–v0.9.80)

**750 upstream commits, 38 versions.** Too large to rebase: `extract.py` alone grew
+3038 lines across 112 commits, so replaying the PS commit would have meant resolving
conflicts against substantially restructured code. Used **reset-and-reapply** instead
(see PS_SETUP.md). Zero conflicts; every patch re-applied by hand against the new code.

**Patch set changes**
- **Dropped** the `hooks.py` `sys.platform == "win32"` guard in `_reject_windows_path`.
  Upstream's `os.name == "nt"` is functionally identical on Windows; the fork was
  carrying a pure style preference that cost a conflict every sync.
- **Dropped** all ruff autoformat changes. A formatter had been run over the fork at
  some point, accounting for ~64 of the 65 changed lines in `hooks.py` and most of the
  ~720 changed test lines. It created permanent conflict surface for zero behavior
  change. Ruff is now disabled for this workspace to stop it recurring.
- **Documented** two patches that had been carried silently since before 2026-07:
  `watch.py` `_queue_pending` and `tools/skillgen/gen.py` `_git_show` (see Patch
  Inventory below). Both verified still needed against v0.9.80.
- **Added** patch 7 (`paths.py`): the first PS patch that fixes an upstream *bug*
  rather than adding PS behavior. Worth reporting upstream so it can be dropped.

**Test suite: 26 failures → 0**

Every one of the 26 was reproduced on clean upstream before being touched, so none
were caused by PS patches.

- 13 skipped as genuinely inapplicable (POSIX primitives absent, symlink privilege,
  CWD-delete, `sh`, POSIX-only path assertions, `MAX_PATH`) — see Windows Test Skips.
- 10 fixed as real test defects (see Windows Test Fixes).
- 2 fixed by installing missing dev-env packages (`setuptools>=83.0.0`, `pyyaml`),
  not by code changes.
- 1 fixed by patching an upstream product bug (patch 7, below).

**Upstream behavior worth knowing (cost real time to rediscover)**
- `gemini` user-scope resolves to `~/.agents/` on **Windows only**, which collides with
  the `agents` platform. `_refresh_stale_skills` therefore skips gemini on Windows by
  design (it refuses to refresh any directory written by two installers), and the
  stale-skill warning names `--platform agents`, not gemini.
- `claude` and `windows` share `.claude/skills/graphify/`. On Windows the refresh
  deliberately picks the `windows` variant; they differ only in host shell (bash vs
  PowerShell bodies).
- `tests/conftest.py` has an autouse `_sandbox_home` fixture patching `HOME`,
  `USERPROFILE`, `LOCALAPPDATA`, and `Path.home`. Tests never touch the real home
  directory — do not assume otherwise when triaging install/uninstall failures.
- The dev environment needs `setuptools>=83.0.0` or
  `test_built_wheel_ships_the_full_skill_payload` fails on the wheel build. Runtime
  envs do not need it.

---

## Patch Inventory

The complete set of PS modifications as of **2026-11-16 (v0.9.111)**. Verify all of
these after any upstream sync. There are **six** (patch 7 was dropped).

| # | File | Change | Why |
|---|---|---|---|
| 1 | `graphify/__main__.py` | `_load_dotenv()` defined and called before the `graphify.paths` import | `.env` must populate `GRAPHIFY_OUT` and API keys at import time |
| 2 | `graphify/detect.py` | `.pyt`, `.bat` in `CODE_EXTENSIONS` | ArcGIS Pro toolboxes and batch launchers are source |
| 3 | `graphify/extract.py` | `.pyt` in `_LANG_FAMILY_BY_EXT`, `_DISPATCH`, the `python_member_calls` frozenset, the `py_paths`/`py_results` cross-file import filter (`effective_suffix(p) in (".py", ".pyt")`), `_is_python_file`, and the builtin-name guard (`effective_suffix(...) in (".py", ".pyt")`) | full Python AST coverage for `.pyt`; six sites |
| 4 | `graphify/analyze.py` | `.pyt` in the `_LANG_FAMILY` python set | language-family classification |
| 5 | `graphify/watch.py` | `_queue_pending` writes `p.as_posix()` rather than `os.fspath(p)` | source identities are normalised to POSIX elsewhere (`watch.py` 541/586/1788); backslash entries in the pending file would not match |
| 6 | `tools/skillgen/gen.py` | `errors="replace"` on the `_git_show` subprocess | cp1252 is the Windows default and crashes on UTF-8 skill content |

Patch 7 (`paths.py` read-only guard) was **dropped on 2026-11-16**; see below.

New files (no upstream counterpart, so they never conflict): `PS_FORK.md`,
`PS_SETUP.md`, `.env_example`.

### Where `.pyt` deliberately does NOT go

Upstream has many `.py`-gated sites in `extract.py` (`_python_absolute_import_alias_files`,
`_repoint_python_package_imports`, `_repoint_python_sibling_imports`, the
`resolution_context_nodes` loop, and others). They build maps of **importable module
names**, so they stay as upstream wrote them. A `.pyt` file cannot be imported as a
module — ArcGIS loads it specially — so adding it there would fabricate import targets
nothing can reference. The split is:

- `.pyt` as import/call **source** → include (patch 3: the import filter, `_is_python_file`,
  the builtin-name guard)
- `.pyt` as import **target** → exclude (leave upstream alone)

When upstream adds a new `.py`-gated site, decide which side it is on before touching it.

### Patch 7 — DROPPED 2026-11-16 (absorbed upstream; kept for the diagnosis)

Upstream v0.9.111 fixes this in `_atomic_replace` (`paths.py`): on Windows it raises
`PermissionError` before calling `os_replace_with_fallback` when the destination is
read-only. Its scope is intentionally narrower than ours was — install/cache callers
must still be able to replace their own read-only copies. Do not re-apply the old
patch; it would block those callers. Confirm with
`tests/test_atomic_writes.py` after each sync.

The original finding, for the record: `os_replace_with_fallback` treated **any**
`PermissionError` as the transient lock #3508 targets and fell through to a
copy-based swap that clobbered a read-only destination. It was the only PS patch that
fixed an upstream *defect*.
`test_atomic_writes.py::test_write_text_atomic_refuses_a_readonly_destination_without_leaking_a_temp`
is `skipif os.name != "nt"`, so upstream's POSIX CI has never executed it.

`os_replace_with_fallback` treated **any** `PermissionError` from `os.replace` as
the transient lock #3508 targets and fell through to a copy-based swap. Against a
destination carrying the Windows read-only attribute:

1. `os.replace` raises `PermissionError` — correct
2. the fallback runs; `os.rename(dst, backup)` **succeeds**, because the read-only
   attribute blocks delete and overwrite but not rename
3. `os.rename(tmp_copy, dst)` succeeds → the read-only file is overwritten
4. `os.unlink(backup)` fails, error swallowed → backup temp leaks
5. `os.unlink(src)` fails → `PermissionError` propagates, *after* the damage

So `pytest.raises(PermissionError)` passed for the wrong reason while the file was
already gone. Affects all eight call sites (`cache.py` ×2, `export.py`,
`install.py` ×3, `watch.py` ×2, `_atomic_replace`).

Implementation notes for anyone re-applying it:
- The guard must sit **after** the existing `src == dst` early return, or replacing
  a read-only path with itself stops being a no-op and breaks
  `test_os_replace_with_fallback_is_a_noop_when_src_equals_dst`.
- The original exception is bound to `replace_error` inside the `except` block,
  because Python unbinds `exc` once the block exits.
- `_is_readonly` uses `os.lstat`, not `os.stat`: `os.replace` swaps a symlink
  itself rather than following it (#3286), so the link's own attribute is what
  matters.
- Gated on `os.name == "nt"` so POSIX behavior is provably unchanged (there,
  `os.replace` succeeds against a read-only file anyway).

### Windows MAX_PATH limit on the cache (upstream, unfixed)

Not patched — recorded so it is not rediagnosed. `cache.py::save_cached` writes
`<cache_root>/graphify-out/cache/ast/<version>/<64-char-sha256>.<rand>.tmp`, which
adds roughly **110 characters** on top of the project path. Windows rejects paths
over 260 chars with `FileNotFoundError` (`ERROR_PATH_NOT_FOUND`) unless they carry
the `\\?\` extended-length prefix, and `save_cached` does not apply it.

So **a project nested deeper than ~150 characters cannot write its AST cache on
Windows** — plausible for OneDrive-synced or deeply nested corporate trees.

`detect.py::_os_path()` already exists to add the prefix, and `cache.py` line ~465
strips it for key normalization, so the codebase understands the problem; the write
path just never applies it. A real fix needs the prefix at every cache I/O site
(`mkstemp`, the replace target, the unlink cleanup, `cache_dir()`'s `mkdir`, the
`load_cached` read path, and the cache-tree globs), which carries meaningful
regression risk in Windows path handling.

Left unfixed deliberately: PS repo paths are ~44 characters, leaving ~215 of
headroom. `test_c_include_out_of_root_target_id_is_deterministic_across_checkout_paths`
is skipped on Windows because its deliberately long checkout name trips this; the
determinism behavior it covers is unaffected.

### PyYAML is an undeclared dependency (upstream) — FIXED upstream by v0.9.111

`pyproject.toml` now declares `pyyaml>=6.0` in core `dependencies`, so a normal
install pulls it in. Historical note: through v0.9.80 it appeared nowhere, and
`extractors/markdown.py` silently fell back to `_parse_frontmatter_fallback`, a flat
`key: value` parser that **dropped nested blocks** with no warning.

### Install via extras, not bare package names

PS_SETUP.md previously instructed `pip install openai watchdog mcp`, which
bypasses the version caps upstream declares — notably `mcp>=1,<3`, capped because
3.x is outside the tested range. Use `pip install -e ".[openai,mcp,watch]"` so the
caps are honoured and the set stays correct as upstream edits its extras.

### tree-sitter-sql must be installed AND pinned (dev env)

`tree-sitter-sql` lives in the optional `sql` extra. Without it a `.sql` file silently
yields **no nodes** (seven `test_ext_tagged_id_clash` tests failed with
`expected one node 'x.sql'...; got []`), and tests that `importorskip` it skip
quietly, so a missing grammar hides real failures.

It must also match upstream's lock. `pyproject.toml` does not pin it, so `pip` takes
the latest (0.3.13 on 2026-11-16), but `uv.lock` pins **0.3.11**, and
`test_multilang.py::test_sql_tsql_bracketed_procedure_is_recovered` and
`...::test_sql_escaped_double_quote_in_routine_name_is_consumed` assert ERROR-recovery
behaviour that changed in 0.3.13. Both pass on 0.3.11. Install
`pip install tree-sitter-sql==0.3.11` in the dev env.

---

## Known Upstream Windows Failures

**One open item as of 2026-11-16:** `test_llm_backends.py::test_base_url_env_overrides[deepseek-...]`
and `test_base_url_defaults_without_env[deepseek-...]` fail with
`PermissionError: [WinError 5] Access is denied` while launching a subprocess. The
identical kimi and gemini variants pass, and a plain `pytest ... -k "deepseek and
base_url"` command line was itself refused with `Access is denied.`. A `git commit -m`
whose message contained the word was refused the same way. That points to an
endpoint-security policy matching the string `deepseek` in a command line. **This is
an inference, not verified.** If `python -c "print('deepseek')"` is also refused, it
is a machine policy: document it and leave the tests alone. Do not try to evade the
control.

Everything else from the 2026-11-16 run is resolved (see the changelog table). All 19
failures found on the first full run were reproduced or explained before being touched;
none were caused by PS patches.

Keep this section as the landing place for the next sync's triage. The standing
unfixed limitation is the `MAX_PATH` cache issue documented above — real, but not
reachable from PS-length paths.

### Known flaky test (load-sensitive, not a regression)

`test_incremental_mtime_collision.py::test_same_size_rewrite_in_one_tick_is_requeued`
fails intermittently under a loaded full-suite run and passes in isolation in under
a second. Seen once in four full runs on 2026-10-08. **Re-run it alone before
investigating** — do not treat it as a real failure.

Cause: the assertion depends on `detect._mtime_may_hide_a_rewrite`, which fires only
when `seen - current_mtime` is below `_MTIME_SUBSECOND_S` (**50 ms**). `seen` is
stamped by `save_manifest`, so the fixture's `detect()` + `save_manifest()` must
complete within 50 ms of the corpus being written. Under load that window is missed,
the file reads as unchanged, nothing is queued, and the assertion fails.

The test's docstring claims the mtime pinning makes this deterministic. It does not:
it pins the file's mtime to the pre-write `stat_before.st_mtime_ns`, while the
comparison is against the wall-clock `seen` recorded in the manifest row, so the
timing dependency the docstring says was removed is still present. A deterministic
version would pin the file's mtime to the row's `seen` value instead. Left unpatched
— it is upstream's test, and the flake is rare.

---

## Windows Test Patches (current, v0.9.111)

The test changes this fork carries today:

| File | Change | Reason |
|---|---|---|
| `test_hooks.py` | `skipif win32` on 5 tests: `test_checkout_hook_skips_same_head_noop_at_runtime`, `test_worktree_guard_runs_on_primary_skips_linked`, `test_toplevel_anchor_runs_from_the_top_when_git_starts_the_hook_in_a_subdir`, `test_baked_viz_limit_yields_to_an_explicit_per_run_override`, `test_worktree_guard_survives_exported_cdpath` | hook scripts need a POSIX `sh` (the cdpath one is probable, not confirmed) |
| `test_code_only_update_all_hosts.py` | `skipif win32` on the parametrized test (devin, aider) | bash runbook vs a backslash interpreter path |
| `test_language_override_lifecycle.py` | expected `source_file` built with `.as_posix()` | `source_file` is POSIX-canonical (#2627) |
| `test_resolve_memoization.py`, `test_source_key_memoization.py` | warm the absolute key before patching `getcwd` | Windows `resolve()` calls `getcwd` on a miss |

## Windows Test Skips — HISTORICAL (v0.9.80, not carried at v0.9.111)

Thirteen tests that cannot execute on Windows. Each asserts behavior requiring a
platform primitive Windows lacks — these are inapplicable, not failures. **None were
re-applied on 2026-11-16 and none of them failed**, so treat this table as a record of
what to look for if they come back.

| File | Tests | Blocker |
|---|---|---|
| `test_non_regular_files.py` | `test_fifo_is_rejected`, `test_symlink_pointing_at_a_fifo_is_rejected` | `os.mkfifo` does not exist |
| `test_non_regular_files.py` | `test_unix_socket_is_rejected` | `socket.AF_UNIX` does not exist |
| `test_non_regular_files.py` | `test_symlink_to_a_regular_file_is_accepted`, `test_broken_symlink_is_rejected_without_raising` | symlink creation needs elevation (WinError 1314) |
| `test_hooks.py` | `test_checkout_hook_skips_same_head_noop_at_runtime`, `test_worktree_guard_runs_on_primary_skips_linked`, `test_baked_viz_limit_yields_to_an_explicit_per_run_override` | hook scripts need a POSIX shell (WinError 2) |
| `test_watch.py` | `test_rebuild_code_deleted_cwd_without_repo_root_returns_false`, `test_rebuild_code_deleted_cwd_uses_graphify_repo_root` | Windows cannot delete the CWD (WinError 32) |
| `test_install.py` | `test_hermes_skill_destination_posix_uses_home` | asserts the POSIX `~/.hermes` destination |
| `test_install_roundtrip.py` | `test_skill_roundtrip_at_real_destination[user-hermes]` | hermes user-scope resolves under `LOCALAPPDATA` on Windows |
| `test_extract.py` | `test_c_include_out_of_root_target_id_is_deterministic_across_checkout_paths` | exceeds `MAX_PATH` — see the cache limitation above |

## Windows Test Fixes — HISTORICAL (v0.9.80, not carried at v0.9.111)

Ten tests repaired rather than skipped on v0.9.80 — each was a genuine defect in the
test, not a product bug. Not re-applied on 2026-11-16; none failed.

**Missing UTF-8 encoding** (the file is UTF-8; `read_text`/`write_text` default to
cp1252 on Windows and mangle or reject non-ASCII):
- `test_install.py::test_codex_skill_uses_graphify_with_existing_graph` — em/en dashes
- `test_languages.py::test_markdown_wikilink_fallback_unicode_normalization` — Korean text
- `test_merge_chunks_validation.py::test_merge_chunks_accepts_unicode_id` — CJK node id

**Hardcoded POSIX skill destinations** (gemini user-scope is `~/.agents` on Windows):
- `test_uninstall_scope.py` — `PLATFORMS` split into separate user and project dot-dirs;
  fixes `test_bare_call_still_removes_global[gemini]` and
  `test_remove_user_skill_opt_in_with_project_dir[gemini]`
- `test_install_references.py::test_gemini_install_references_all_resolve`

**POSIX-only refresh assumptions**:
- `test_skill_auto_refresh.py::test_every_stale_platform_is_refreshed_not_only_the_detected_one`
  — uses `windows` instead of `claude` and drops gemini on Windows
- `test_skill_auto_refresh.py::test_a_stale_gemini_skill_gets_the_warning_too` — expects
  `--platform agents` on Windows

**Windows path separators**:
- `test_terraform_modules.py::test_same_named_directories_and_cross_file_references_stay_separate`
  — built its expected `source_file` with `str(Path(...))`, producing `dev\app\use.tf`
  while upstream canonicalises `source_file` to POSIX (#2627); now uses `.as_posix()`

**POSIX-only subprocess environment**:
- `test_extract.py::test_python_external_calls_survive_real_incremental_context` — its
  minimal env allowlist passed `HOME`/`TMPDIR`, but Windows resolves `Path.home()`
  from `USERPROFILE` (or `HOMEDRIVE`+`HOMEPATH`) and temp from `TEMP`/`TMP`. The CLI
  died at startup with "Could not determine home directory" before extraction began.
  Allowlist now includes the Windows equivalents.

### 2026-08-13 — Upstream sync (upstream/v8 → 7fe58b0, v0.9.29–v0.9.42)

Routine rebase onto `upstream/v8`. No new PS-specific functionality. 14 versions (v0.9.29–v0.9.42), ~100 upstream commits. All 6 PS patches survived; `extract.py`, `analyze.py`, `hooks.py`, `__main__.py` auto-merged cleanly — only `detect.py` required manual `.pyt`/`.bat` re-add (upstream reformatted `CODE_EXTENSIONS` to a single-line set).

Key upstream changes relevant to PS:
- `fix(hooks)`: upstream replaced `DETACHED_PROCESS` with `CREATE_NO_WINDOW` on Windows (complements our `_reject_windows_path` win32 guard; no conflict).
- `fix(paths)`: upstream added cross-platform absolute path detection and read-only bit clearing before atomic-write temp unlink on Windows.
- `test`: upstream added probe-and-skip for symlink creation unavailability — overlaps with our `@_win_no_symlink` markers; upstream version is broader and more portable.
- `fix(extract)`: upstream canonicalized `source_file` to POSIX separators — benefits Windows runs.
- `fix(serve)`: dual-compat with MCP SDK 1.x and 2.x.

Sync process change: the fork had accumulated 13 PS commits since the last sync. Introduced squash-first workflow — squash all PS commits to 1 before rebasing (`git rebase -i <fork-point>`) so conflict resolution is a single pass. See `PS_SETUP.md` for updated procedure.

### 2026-07-27 — Upstream sync (upstream/v8 → 1644230, v0.9.17–v0.9.28)

Routine rebase onto `upstream/v8`. No new PS-specific functionality; changes below are conflict resolutions that preserve existing PS patches while integrating upstream additions (21 commits, v0.9.17 through v0.9.28).

- `graphify/detect.py`: took upstream; re-applied `.pyt` and `.bat` in `CODE_EXTENSIONS`. Upstream added `_CREDENTIAL_STORE_DIRS`/`_AMBIGUOUS_SENSITIVE_DIRS` split, improved `_SENSITIVE_PATTERNS`, NFC normalization for manifest keys, and `gitignore: bool = True` parameter to `detect()`.
- `graphify/build.py`: took upstream. Upstream added `_load_existing_graph()` helper, grouped extraction-error breakdown, and `_prune_match()` for form-insensitive prune matching.
- `graphify/export.py`: took upstream. Upstream added `MALFORMED_GRAPH` sentinel and `existing_graph_node_count()`.
- `graphify/analyze.py`: took upstream; re-applied `.pyt` in `_LANG_FAMILY`. Upstream added Swift framework symbols to `_BUILTIN_NOISE_LABELS` and updated `_is_builtin_god_node` to use `_is_file_node_label()`.
- `graphify/extract.py`: took upstream; re-applied `.pyt` in 4 locations (`_LANG_FAMILY_BY_EXT`, `_DISPATCH`, `python_member_calls` frozenset, cross-file import filter). Upstream refactored into per-language extractor split, added C# namespace-aware resolution, Swift computed/observed properties, and canonicalized import target IDs.
- `tests/test_detect.py`: merged. Upstream added `test_detect_surfaces_unreadable_dir_instead_of_silent_skip` — preserved our Windows skip guard (`sys.platform == "win32"`) and the `os.geteuid()` root check.
- `tests/test_install.py`: took upstream. Upstream added `test_codex_hook_command_is_a_real_cli_subcommand` (#2165).

### 2026-07-15 — `.pyt` AST extraction support

`.pyt` (ArcGIS Python Toolbox) files are valid Python. Added `.pyt` as a Python alias in 5 locations so they receive full AST extraction, symbol resolution, language-family classification, and cross-file import edges — identical coverage to `.py`.

- `graphify/extract.py` (`_LANG_FAMILY_BY_EXT`): added `".pyt": "python"`
- `graphify/extract.py` (`_DISPATCH`): added `".pyt": extract_python`
- `graphify/extract.py` (`LanguageResolver` python_member_calls): added `".pyt"` to frozenset
- `graphify/extract.py` (cross-file import filter): extended `.py` suffix checks to `{".py", ".pyt"}`
- `graphify/analyze.py` (`_LANG_FAMILY`): added `".pyt"` to python set

`.bat` intentionally excluded — DOS CMD syntax is incompatible with the tree-sitter bash grammar; LLM-only extraction is correct for `.bat`.

---

## PS Patch Behavior

See **Patch Inventory** above for the authoritative list of modifications.

### `_load_dotenv()`

`_load_dotenv()` runs at startup before the paths module loads, so `.env` values are available immediately at import time. `python-dotenv` is an optional dependency — if not installed, a built-in fallback parser reads the file directly (supports `KEY=value` and `KEY="value"`; no multiline values or shell substitution).

### ArcGIS Pro file types

`.pyt` (Python Toolbox) and `.bat` (batch launcher) files are treated as code and included in LLM semantic extraction. No configuration required — detection is automatic. `.pyt` additionally receives full Python AST extraction; `.bat` does not (DOS CMD syntax has no compatible tree-sitter grammar, so `.bat` is LLM-only).

---

## Changelog

### 2026-07-08 — Upstream sync (upstream/v8 → 20bfdf6)

Routine rebase onto `upstream/v8`. No new PS-specific functionality; changes below are conflict resolutions that preserve existing PS patches while integrating upstream additions.

- `graphify/detect.py` (`CODE_EXTENSIONS`): kept upstream additions `.mts` and `.cts`; PS additions `.pyt` and `.bat` remain.
- `graphify/detect.py` (`_SKIP_DIRS`): kept upstream's context-aware `__snapshots__` handling and new `_JS_SNAPSHOT_TEST_ROOTS` constant.
- `graphify/detect.py`: kept upstream's new `_resolves_under_root()` security helper (prevents symlink traversal outside scan root).
- `graphify/export.py`: kept upstream's full learning overlay node logic.
- `graphify/hooks.py` (`_reject_windows_path`): retained PS `sys.platform == "win32"` guard (upstream used `os.name == "nt"`; both equivalent, PS form kept for consistency).
- `tests/test_detect.py`, `tests/test_extract.py`: applied `@_win_no_symlink` to 4 new upstream-added out-of-root symlink tests (`test_detect_skips_out_of_root_symlinked_directory_even_when_following`, `test_detect_skips_out_of_root_symlinked_file_by_default`, `test_collect_files_skips_out_of_root_symlinked_directory`, `test_collect_files_skips_out_of_root_symlinked_file_by_default`).

### 2026-07-08 — Post-sync Windows fixes

All fixes address new upstream test failures on Windows; none alter POSIX behavior.

**Source fix**
- `graphify/build.py` (`_semantic_id_remap`): added `or sf_norm.startswith("/")` to the absolute-path guard. `Path("/abs/...").is_absolute()` returns `False` on Windows (no drive letter), so POSIX-style absolute `source_file` paths were incorrectly remapped instead of being left untouched.

**Test infrastructure (skips)**
- `tests/test_image_vision.py`: `test_read_files_skips_out_of_root_symlink` and `test_build_image_refs_skips_out_of_root_symlink` skipped on Windows — `symlink_to()` requires elevated privileges (WinError 1314).
- `tests/test_watch.py`: `test_rebuild_code_deleted_cwd_without_repo_root_returns_false` and `test_rebuild_code_deleted_cwd_uses_graphify_repo_root` skipped on Windows — Windows cannot remove a directory that is the current working directory (WinError 32).
- `tests/test_ollama_retry_cap.py`: all tests skipped when `openai` package is not installed — `pytest.importorskip("openai")` added at module level.

**Test assertion fix**
- `tests/test_cache.py` (`test_semantic_prune_removes_orphan_entries`): clears `_stat_index` between the two `write_text` calls. Both test strings are the same byte length (16 bytes); on Windows `st_mtime_ns` may not update between rapid sequential same-size writes, causing the stat-fastpath in `file_hash` to return the stale hash for content B (`h_b == h_a`), which makes `prune_semantic_cache` find nothing to prune.

### 2026-06-25 — Esri PS extensions

- `graphify/__main__.py`: Restored `_load_dotenv()` call before `from graphify.paths import GRAPHIFY_OUT`. Loads `.env` from CWD at startup so `GRAPHIFY_OUT` and API keys are available at import time. Falls back to manual parse if `python-dotenv` is not installed.
- `graphify/detect.py`: Added `.pyt` to `CODE_EXTENSIONS`. ArcGIS Pro Python Toolbox files are treated as code (LLM semantic extraction).
- `graphify/detect.py`: Added `.bat` to `CODE_EXTENSIONS`. Windows workflow launcher scripts are treated as code (LLM semantic extraction).

### 2026-06-25 — Windows compatibility (v8-ps branch)

All fixes address pre-existing upstream failures on Windows; none alter POSIX behavior.

**Test infrastructure (skips)**
- `tests/test_detect.py`, `tests/test_extract.py`: 14 symlink tests skipped on Windows — `symlink_to()` requires elevated privileges (WinError 1314).
- `tests/test_read_hook.py`: all 8 tests skipped on Windows — hook command uses `sh -c` which is not available natively.
- `tests/test_hooks.py`: `test_windows_hookspath_rejected_no_junk_dir` skipped on Windows — tests POSIX-specific junk-dir guard; Windows paths are valid on native Windows.
- `tests/test_install_roundtrip.py`: `test_skill_roundtrip_at_real_destination[user-hermes]` skipped on Windows — Hermes user-scope uses `LOCALAPPDATA` on Windows, covered by a dedicated test.

**Test assertion fixes**
- `tests/test_cpp_preprocess.py`: `startswith("/")` changed to `os.path.isabs()` so the absolute-path guard passes on Windows drive paths.
- `tests/test_install.py`: `str(dst).endswith(...)` changed to `dst.as_posix().endswith(...)` for the hermes POSIX destination test.
- `tests/test_install.py`: `read_text()` changed to `read_text(encoding="utf-8")` in `test_codex_skill_uses_graphify_with_existing_graph` — default cp1252 on Windows mangles the UTF-8 em dash in the skill file.
- `tests/test_extract.py`: `_legacy_collect_files` oracle deduplicates with `sorted(set(...))` — case-insensitive rglob on Windows returns duplicate paths.
- `tests/test_install_references.py`: `test_gemini_install_references_all_resolve` uses `mainmod._platform_skill_destination("gemini")` instead of hardcoding `.gemini` (Windows installs to `.agents`).

**Source fixes**
- `graphify/hooks.py` (`_reject_windows_path`): added `if sys.platform == "win32": return` — Windows paths are valid on native Windows; the junk-dir guard only applies on POSIX/WSL.
- `graphify/llm.py` (`_build_image_refs`): `str(p.relative_to(root))` changed to `p.relative_to(root).as_posix()` — prevents backslash image ref paths on Windows.
- `graphify/watch.py` (`_queue_pending`): `os.fspath(p)` changed to `p.as_posix()` — pending-changes file uses forward-slash paths consistently.
- `graphify/manifest_ingest.py` (`_parse_apm_fallback`): extracts `version:` field when PyYAML is not installed, fixing `KeyError: 'version'` in `test_apm_parses_name_and_deps`.
- `tools/skillgen/gen.py` (`_git_show`): added `encoding="utf-8"` and `errors="replace"` to `subprocess.run` — prevents `UnicodeDecodeError` on Windows cp1252 when reading git blobs.
- `graphify/export.py` (`to_obsidian`): compute `_fname_limit` dynamically from `out.resolve()` on Windows so the total path stays under MAX_PATH (260). Short output paths keep the existing 200-byte cap unchanged; deep paths shrink the cap proportionally.
