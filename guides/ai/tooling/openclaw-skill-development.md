---
type: guide
title: Building an OpenClaw agent skill
description: How to structure a skill folder, SKILL.md frontmatter, and a Python CLI pattern for OpenClaw agent skills.
tags:
- openclaw
- agent
- skills
- python
status: draft
resource:
created: 2026-09-27T17:15:28Z
updated: 2026-09-27T17:15:28Z
generated:
  by: claude/sonnet-5
  at: 2026-09-27T17:15:28Z
verified: []
stale_after: 2027-09-27T17:15:28Z
sources: []
relations: []
superseded_by:
---

# Building an OpenClaw agent skill

OpenClaw is an agent framework whose capabilities are extended through **skills**: folders under
`~/.openclaw/workspace/skills/<name>/` containing a `SKILL.md` the agent reads at runtime, plus
optional executable code. This guide covers the recommended layout, frontmatter schema, and a
Python CLI pattern for implementing one.

## Prerequisites

- `uv` installed (used to run the skill's Python CLI without manual venv management).
- Python `>=3.12` available to `uv`.

## Steps

### 1. Create the skill folder layout

```
skills/myskill/
├── SKILL.md
├── pyproject.toml
├── myskill_api.py      # optional library (pure stdlib = no deps)
└── scripts/
    ├── __init__.py
    └── cli.py          # CLI entry point
```

### 2. Write `SKILL.md` frontmatter

```yaml
---
name: myskill
description: "One-liner shown in the skills list."
version: "1.0.0"
pythonVersion: ">=3.12"
metadata:
  openclaw:
    emoji: "🔧"
    primaryEnv: MY_API_KEY        # credential highlighted in web UI
    requires:
      anyBins: [uv]               # runtime binaries required
    install:
      - id: uv-install
        kind: shell
        command: "curl -LsSf https://astral.sh/uv/install.sh | sh"
        bins: [uv]
        label: "Install uv"
env:                              # shown as settings in the web UI
  - name: MY_API_KEY
    required: true
    description: "API key for the service."
  - name: MY_BASE_URL
    required: false
    description: "Base URL (optional, has a built-in default)."
---
```

Use the **top-level `env:` array** for declaring settings, not `metadata.openclaw.requires.env` —
only the top-level form renders with descriptions and required/optional labels in the web UI.

### 3. Set up `pyproject.toml`

```toml
[project]
name = "myskill"
version = "1.0.0"
requires-python = ">=3.12"
dependencies = []                    # add e.g. "httpx>=0.27" if needed

[project.scripts]
myskill = "scripts.cli:main"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["scripts"]
```

### 4. Write the CLI entry point

A proper CLI invoked via `uv run` avoids quoting/escaping issues when the agent passes complex
content (Markdown, special characters, multiline strings) inline to the exec tool:

```python
import argparse, json, os, sys

# Make the skill root importable (e.g. to import myskill_api)
sys.path.insert(0, os.path.dirname(os.path.dirname(os.path.abspath(__file__))))
import myskill_api

def cmd_list(args):
    items = myskill_api.list_items()
    if args.json:
        print(json.dumps(items, indent=2))
        return
    for item in items:
        print(f"[{item['id']}] {item['name']}")

def main():
    parser = argparse.ArgumentParser(prog="myskill")
    sub = parser.add_subparsers(dest="command", required=True)

    p = sub.add_parser("list")
    p.add_argument("--json", action="store_true")
    p.set_defaults(func=cmd_list)

    args = parser.parse_args()
    try:
        args.func(args)
    except Exception as e:
        print(f"Error: {e}", file=sys.stderr)
        sys.exit(1)

if __name__ == "__main__":
    main()
```

Invoke it from `SKILL.md`'s documented commands as:

```bash
SKILL=~/.openclaw/workspace/skills/myskill
uv run --project $SKILL myskill list
uv run --project $SKILL myskill search "term"
```

The first run creates `.venv` and installs the package (~1s); subsequent runs are fast.

### 5. Use the file-based content pattern for large inputs

For commands that accept large or complex content (Markdown pages, configs), always add a
`--content-file` flag rather than relying only on inline `--content` — this eliminates shell
quoting failures entirely:

```python
p.add_argument("--content-file", metavar="FILE")
p.add_argument("--content")          # fallback for short strings

# In the handler:
if args.content_file:
    with open(args.content_file) as f:
        content = f.read()
elif args.content:
    content = args.content
```

Expected agent workflow: write content to a temp file via heredoc, call the CLI with
`--content-file <path>`, then clean up the temp file.

### 6. Resolve secrets from multiple sources

Skills should work regardless of which user Python runs as:

```python
def _get_secret(env_var: str, filename: str) -> str:
    if os.environ.get(env_var):
        return os.environ[env_var].strip()
    candidates = [
        os.path.join(os.environ.get("HOME", ""), ".openclaw", "secrets", filename),
        os.path.expanduser(f"~/.openclaw/secrets/{filename}"),
        f"/home/openclaw/.openclaw/secrets/{filename}",  # hard fallback
    ]
    for path in candidates:
        if os.path.exists(path):
            with open(path) as f:
                return f.read().strip()
    raise Exception(f"{env_var} not set and {filename} not found.")
```

`os.path.expanduser("~")` returns `/root` when running as root, so always include the
`/home/openclaw` hard fallback.

### 7. Document the skill in `SKILL.md`

```markdown
## Trigger Phrases
- "Do X with [thing]"     ← natural language variations the agent matches on
- "Search for [term]"

## Commands
### `list` — short description
\`\`\`bash
uv run --project $SKILL myskill list
\`\`\`

## Setup
| Setting | Source |
|---------|--------|
| API Key | `~/.openclaw/secrets/my-key.txt` or `$MY_API_KEY` |
| Base URL | `~/.openclaw/secrets/my-url.txt` or `$MY_BASE_URL` |

## Notes
- Known workarounds and gotchas
```

Write trigger phrases that cover how a user would naturally phrase requests — include both
imperative ("Add a page") and interrogative ("What pages are in the wiki?") forms.

## Verify

- `uv run --project $SKILL myskill list` runs without error and the `.venv` is created on first
  invocation.
- The skill appears in OpenClaw's web UI with the settings declared under `env:` shown with their
  descriptions and required/optional labels.
- A `--content-file`-based command round-trips a multiline Markdown file without quoting errors.

## Troubleshooting

- **Env var settings don't show descriptions in the web UI:** the `env:` array must be top-level
  in `SKILL.md` frontmatter, not nested under `metadata.openclaw.requires.env`.
- **Secret lookup fails when the skill runs as root:** `os.path.expanduser("~")` resolves to
  `/root` for the root user — add the hard-coded `/home/openclaw/...` fallback path.
- **Quoting/escaping errors when passing Markdown or code inline:** switch the affected command to
  the `--content-file` pattern instead of inline `--content`.

## Related

- Applies to any OpenClaw skill implementation; the same Python CLI + `uv run` pattern generalizes
  to other agent-skill frameworks that shell out to a script.
