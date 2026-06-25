# Fork Notes

Forked from: https://github.com/safishamsi/graphify  
Fork remote: https://github.com/justinhchae/graphify.git  
Branch: v8-ps

---

## Usage (PS Fork)

### Environment variables

Create a `.env` file at the project root before running graphify:

```env
GRAPHIFY_OUT=graphify-out        # output directory; defaults to graphify-out if omitted
GEMINI_API_KEY=your-key-here     # required for semantic extraction
# GOOGLE_API_KEY=...             # alternative to GEMINI_API_KEY
```

`_load_dotenv()` runs at startup before the paths module loads, so `.env` values are available immediately at import time. `python-dotenv` is an optional dependency — if not installed, a built-in fallback parser reads the file directly (supports `KEY=value` and `KEY="value"`; no multiline values or shell substitution).

### ArcGIS Pro file types

`.pyt` (Python Toolbox) and `.bat` (batch launcher) files are treated as code and included in LLM semantic extraction. No configuration required — detection is automatic.

### Rebase workflow

```
git fetch upstream
git rebase upstream/main
```

Re-run `pip install -e .` and rebuild graphs after each rebase.

---

## Changelog

### 2026-06-25 — Esri PS extensions

- `graphify/__main__.py`: Restored `_load_dotenv()` call before `from graphify.paths import GRAPHIFY_OUT`. Loads `.env` from CWD at startup so `GRAPHIFY_OUT` and API keys are available at import time. Falls back to manual parse if `python-dotenv` is not installed.
- `graphify/detect.py`: Added `.pyt` to `CODE_EXTENSIONS`. ArcGIS Pro Python Toolbox files are treated as code (LLM semantic extraction).
- `graphify/detect.py`: Added `.bat` to `CODE_EXTENSIONS`. Windows workflow launcher scripts are treated as code (LLM semantic extraction).

### 2026-06-25 — Windows compatibility (v8-ps branch)

All fixes address pre-existing upstream failures on Windows; none alter POSIX behavior.

**Test infrastructure (skips)**
- `tests/test_detect.py`, `tests/test_extract.py`: 8 symlink tests skipped on Windows — `symlink_to()` requires elevated privileges (WinError 1314).
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
