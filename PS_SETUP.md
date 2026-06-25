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
pip install -e .
pip install openai watchdog mcp
```

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

```
git fetch upstream
git rebase upstream/v8
git push --force-with-lease
```

If conflicts occur during rebase, resolve each one then:

```
git rebase --continue
```

To abort and return to pre-rebase state:

```
git rebase --abort
```

After a rebase, verify the PS patches are intact:
- `_load_dotenv()` call in `graphify/__main__.py`
- `.pyt` and `.bat` in `graphify/detect.py`
- `.pyt` AST support in `graphify/extract.py`: `_LANG_FAMILY_BY_EXT`, `_DISPATCH`, `LanguageResolver` frozenset, cross-file import suffix filter
- `.pyt` in `graphify/analyze.py`: `_LANG_FAMILY`
- Windows compatibility fixes (see [PS_FORK.md](PS_FORK.md))

Then re-run the test suite to confirm nothing broke.

---

### When a new upstream version branch ships (e.g. v9)

```
git fetch upstream
git rebase upstream/v9
git push --force-with-lease
```

Same as a routine sync — just target the new branch. Expect more conflicts if it is a major version.

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
