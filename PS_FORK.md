# Fork Notes

See [PS_SETUP.md](PS_SETUP.md) for installation, environment setup, and per-project graph configuration.

Forked from: https://github.com/safishamsi/graphify  
PS fork: https://github.com/EsriPS/graphify  
Origin (maintainer): https://github.com/justinhchae/graphify.git  
Branch: v8-ps

---

## Changelog

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

### `_load_dotenv()`

`_load_dotenv()` runs at startup before the paths module loads, so `.env` values are available immediately at import time. `python-dotenv` is an optional dependency — if not installed, a built-in fallback parser reads the file directly (supports `KEY=value` and `KEY="value"`; no multiline values or shell substitution).

### ArcGIS Pro file types

`.pyt` (Python Toolbox) and `.bat` (batch launcher) files are treated as code and included in LLM semantic extraction. No configuration required — detection is automatic.

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
