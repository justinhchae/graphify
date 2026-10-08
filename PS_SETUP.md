# graphify — Esri PS Maintenance Guide

This is a primer for Esri PS usage. For full documentation, see [README.md](README.md).

See [PS_FORK.md](PS_FORK.md) for PS-specific patches, Windows compatibility fixes, and fork changelog.

---

## First-time setup

**1. Clone the repo:**

```
git clone https://github.com/EsriPS/graphify.git
cd graphify
```

**2. Create the conda env and install:**

```
conda create -n graphify-ps python=3.12
conda activate graphify-ps
pip install -e ".[openai,mcp,watch]"
pip install pyyaml
```

This is the **runtime** env — it backs the CLI and the MCP server. Keep test-only
packages out of it.

Install via the **declared extras**, not bare package names. The extras carry
version caps that a bare `pip install mcp` ignores — `mcp` is declared
`mcp>=1,<3` because 3.x is outside the tested range — and they stay correct as
upstream changes them. Quote the argument: PowerShell treats bare `[...]` as
glob syntax.

| Extra | Pulls | Needed for |
|---|---|---|
| `openai` | `openai`, `tiktoken` | the OpenAI/Azure LLM backend; `tiktoken` gives exact token counts instead of estimates |
| `mcp` | `mcp>=1,<3`, `starlette>=1.3.1,<2` | the MCP server (`starlette` only for the HTTP transport, not stdio) |
| `watch` | `watchdog` | `graphify watch` |

Other extras worth considering by corpus: `office` (.docx/.xlsx), `pdf`,
`leiden` (better community detection), `terraform`, `sql`.

`pyyaml` is installed separately because **upstream does not declare it** — it
appears in no extra, yet `extractors/markdown.py` imports it. Without it markdown
frontmatter silently falls back to a flat `key: value` parser that drops nested
blocks.

**2b. (Maintainers) Create the dev env for running tests:**

```
conda create -n graphify-ps-dev python=3.12
conda activate graphify-ps-dev
pip install -e ".[openai,mcp,watch]"
pip install pyyaml
pip install pytest "setuptools>=83.0.0" build
```

`setuptools>=83.0.0` and `build` are required by
`test_built_wheel_ships_the_full_skill_payload`, which builds a real wheel and
fails (rather than skips) when the build backend is missing. `pyyaml` is required
by `test_markdown_nested_frontmatter_survives`.

**Note:** `pip install -e .` fails with `WinError 32` if the MCP server is running
— it holds a lock on `graphify-mcp.exe`. Stop the MCP server in VS Code first, or
`taskkill /F /IM graphify-mcp.exe`.

**3. Set up `.env`:**

Copy `.env_example` to `.env` at the project root and fill in your Azure key and `GRAPHIFY_OUT`:

```
# Windows
copy .env_example .env

# Mac/Linux
cp .env_example .env
```

---

## LLM backend

Add the following to your `.env`:

```
GRAPHIFY_OPENAI_MODEL=gpt-5.0-mini
```

Run extraction with:

```
graphify extract . --backend openai
```

`gpt-5.0-mini` is the recommended default. Extraction is a structured task (symbol and relationship identification), not reasoning-heavy — mini is sufficient and costs roughly 1/3 the output price of the full model. Step up to `gpt-5.0` only if extraction quality looks degraded on complex files (e.g. `.pyt` toolboxes with deeply nested class hierarchies).

---

## Fork maintenance (maintainer only)

### Remotes

| Remote | URL | Purpose |
|---|---|---|
| `upstream` | https://github.com/safishamsi/graphify | Source of truth (open source) |
| `esrips` | https://github.com/EsriPS/graphify | Esri PS working target (private) |

Working branch: `v8-ps`

---

### Routine work (PS-specific changes)

```
# make changes, commit as normal
git add -A
git commit -m "your message"
git push
```

Pushes to `esrips/v8-ps` (the tracking remote).

---

### Check Remotes

```
git remote -v
```

The following should apear:

```
esrips  https://github.com/EsriPS/graphify.git (fetch)
esrips  https://github.com/EsriPS/graphify.git (push)
origin  https://github.com/justinhchae/graphify.git (fetch)
origin  https://github.com/justinhchae/graphify.git (push)
upstream        https://github.com/safishamsi/graphify.git (fetch)
upstream        https://github.com/safishamsi/graphify.git (push)
```

### Upstream sync (when safishamsi/graphify v8 gets new commits)

**Step 1 — Fetch and size the gap**
```
git status --short                      # must be clean before anything else
git fetch upstream --tags
git log --oneline HEAD..upstream/v8 | find /c /v ""
git diff --stat HEAD upstream/v8 -- graphify/__main__.py graphify/detect.py graphify/extract.py graphify/analyze.py graphify/watch.py tools/skillgen/gen.py
```

**Step 2 — Pick the strategy from the gap size**

| Gap | Strategy |
|---|---|
| Under ~150 commits, patched files largely unchanged | Rebase (Step 3a) |
| Larger, or `extract.py` churned by thousands of lines | **Reset-and-reapply (Step 3b)** |

Rebasing replays the PS commit against upstream's code. Once upstream has
restructured the files the patches live in, `--ours` plus manual re-application
is guesswork, and it has to be repeated for every conflicting commit. The PS
patch set is only ~45 lines across 6 files, so re-applying it by hand against
known-good upstream is both faster and safer. The 2026-10-08 sync (750 commits,
`extract.py` +3038 lines) used reset and hit zero conflicts.

**Step 3a — Rebase (small gaps)**
```
git rebase -i <fork-point>    # squash all PS commits to 1: first `pick`, rest `f`
git rebase upstream/v8
```
On conflict: `git checkout --ours -- <file>`, `git add <file>`, re-apply the
missing patches, `git rebase --continue`. `git rebase --abort` to back out.

**Step 3b — Reset and re-apply (large gaps)**
```
git branch v8-ps-backup-<yyyymmdd>
git reset --hard upstream/v8
git checkout v8-ps-backup-<yyyymmdd> -- PS_FORK.md PS_SETUP.md .env_example
```
Then re-apply the six code patches from the **Patch Inventory** table in
[PS_FORK.md](PS_FORK.md). The backup branch keeps the old history reachable;
`git reset --hard v8-ps-backup-<yyyymmdd>` undoes everything.

**Step 4 — Verify every patch landed**

Work the Patch Inventory table in [PS_FORK.md](PS_FORK.md) top to bottom. Grep
for each one; do not assume a patch survived because a neighbouring one did.
Note the inventory also records where `.pyt` deliberately does **not** go —
upstream keeps adding `.py`-gated sites that must be left alone.

**Step 5 — Test, and triage against a clean baseline**
```
conda activate graphify-ps-dev
pip install -e . --no-deps
pytest tests/ -q
```

Before changing any failing test, prove it is not ours:
```
git stash
pytest <the failing tests> -q
git stash pop        # run as its own command and CONFIRM it succeeded
```
If it fails on clean upstream it is an upstream issue; if it passes, a PS patch
broke it — fix the patch, not the test. [PS_FORK.md](PS_FORK.md) lists the
currently-known upstream Windows failures and the standing Windows skips.

**Step 6 — Update the documentation in the same commit**

Add a changelog entry to [PS_FORK.md](PS_FORK.md), and amend the Patch Inventory
if the patch set changed. Undocumented patches are expensive: the 2026-10-08
sync lost time reverse-engineering two patches nobody had written down.

**Step 7 — Push**
```
git push --force-with-lease origin v8-ps
git push --force-with-lease esrips v8-ps
```

---

### When a new upstream version branch ships (e.g. v9)

Same procedure, targeting `upstream/v9`. A major-version jump is exactly the case
where reset-and-reapply (Step 3b) beats rebasing.

---

## Project setup — per-project MCP (VS Code)

Each project gets its own workspace-scoped MCP server. Do not add project graphs to the user-level `mcp.json`.

**Convention:**

| Item | Location |
|---|---|
| Graph output | `<project-root>/graphify-out/` |
| MCP config | `<project-root>/.vscode/mcp.json` |
| Exclusions | `<project-root>/.graphifyignore` |

**`.vscode/mcp.json` template:**

```json
{
  "servers": {
    "graphify": {
      "type": "stdio",
      "command": "<conda-envs-root>\\graphify-ps\\Scripts\\graphify-mcp.exe",
      "args": ["<absolute-path-to-project>\\graphify-out\\graph.json"],
      "env": {
        "GRAPHIFY_OUT": "<absolute-path-to-project>\\graphify-out"
      }
    }
  }
}
```

The MCP server is named `graphify` in every project — VS Code loads only the workspace-scoped config, so there is no collision.

**Initial build:**

```
cd <project-root>
graphify .
graphify cluster-only . --backend azure
```

If AST extraction completed but semantic extraction failed (e.g. missing package), fix the issue and retry without re-doing AST:

```
graphify . --update
```

graphify respects `.gitignore` automatically. Add a `.graphifyignore` for anything git-tracked that should be excluded (same syntax as `.gitignore`).

Typical `.graphifyignore` patterns:

```
# Graphify graph (self)
graphify-out/

# Docs-only directories (markdown, no code)
docs/

# Other extraneous directories
figures/
data/           # only if not already in .gitignore

# VS Code and GitHub Copilot configuration — not project source
.github/
.vscode/
.docs/

# Environment secrets
.env
```

**Watch mode (active development):**

Run from the project root in a persistent terminal:

```
graphify watch .
```

**Post-commit auto-rebuild (recommended):**

```
graphify hook install
```

This embeds the current interpreter path into the hook, so it fires correctly outside terminal sessions (GUI git clients, etc.). Re-run after upgrading graphify.

**Install the graphify skill for your AI assistant:**

From the project root, using the agent-specific install command:

```
graphify <agent> install
# examples: graphify claude install, graphify vscode install
```

This installs to `.<agent>/skills/graphify/SKILL.md` (e.g., `.claude/skills/graphify/SKILL.md`). Move it to the Esri PS skills directory so Copilot picks it up:

```
move .<agent>\skills\graphify\SKILL.md .github\skills\graphify\SKILL.md
```

The target path is `.github/skills/graphify/SKILL.md` relative to the project root.
