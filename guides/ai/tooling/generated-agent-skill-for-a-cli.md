---
type: guide
title: Generating an agent skill from the CLI itself
description: Let a CLI generate and install its own agent skill from its command tree, with a version check and a drift test.
tags:
- cli
- ai-agents
- skills
- claude-code
- copilot
- codex
- cursor
status: draft
resource:
created: 2026-10-25T19:11:45Z
updated: 2026-10-25T19:15:12Z
generated:
  by: claude/sonnet-5.5
  at: 2026-10-25T19:15:12Z
verified: []
stale_after: 2027-10-25T19:11:45Z
sources:
- id: 2026-10-25-agent-first-cli-concept
  resource: private:/sources/ai/tooling/2026-10-25-agent-first-cli-concept.md
relations: []
superseded_by:
---

# Generating an agent skill from the CLI itself

A hand-written skill file next to a CLI drifts: a new option is added, the skill still describes the old one. Instead
the CLI **generates its own skill** from its live command tree, so the installed skill always describes exactly the
installed version. Use this for any CLI that AI agents should operate (see the
[concept](../../../concepts/ai/tooling/agent-first-cli-for-web-apps.md)).

## Prerequisites

- A CLI whose commands are a tree you can walk at run time (System.CommandLine, Cobra, clap, ...).
- A JSON output per command, ideally with source-generated type metadata (no reflection when you compile to Native AOT).
- A place to put one list of exit codes.

## Steps

### 1. Design the three skill commands

```text
tool skill install [--target claude|agents|copilot|cursor] [--dir <skills-folder>]
tool skill show                      # prints the manifest as JSON (for agents and tests)
tool skill status [--target ...]     # current | outdated | missing | unreadable
```

`install` writes the skill; `show` prints what would be written; `status` compares the installed version with the
binary. All three support `--json` like every other command.

### 2. Generate the files from the live program

Walk the command tree and write two files per target:

- `SKILL.md`: front matter (`name`, `description`), then the **rules for agents**, how to sign in, how to read and (if
  applicable) change data, every command with its options, the JSON shape of each result, and the exit code table.
- a manifest (`<tool>-skill.json`) with a schema version, the tool version and the same command list as data, so `status`
  and tests do not have to parse markdown.

Keep two kinds of content separate:

| Content | Source |
|---|---|
| command names, arguments, options, descriptions, exit codes, version | generated from the command tree and constants |
| JSON result shapes | generated from the JSON type metadata of the output records; for documents built at run time (keys follow the chosen columns), a one-sentence note per command |
| the rules for agents (use `--json`, check the exit code first, never put secrets on the command line, never touch production, run `--dry-run` before a write) | hand-written text in the generator |

Do not reflect over model types if you publish with Native AOT; use the source-generated JSON metadata or hand-maintained
notes.

### 3. Install into every agent's skills folder

An agent reads skills from a folder in the user's home directory. Support the common ones and let the user limit or
override them:

| Target | Folder |
|---|---|
| `claude` | `~/.claude/skills/<tool>` |
| `agents` | `~/.agents/skills/<tool>` |
| `copilot` | `~/.copilot/skills/<tool>` |
| `cursor` | `~/.cursor/skills/<tool>` |

`--target` takes one or more names, `--dir` writes to another skills folder (the two are mutually exclusive). Do not touch
files whose content is unchanged, and report `installed`, `updated` or `unchanged` per target.

### 4. Run the install automatically

The installer and the package manager package should run `tool skill install` for the user (post-install hook,
`chocolateyinstall.ps1`, an installer custom action). A portable zip has no hook: document "run `tool skill install`
once". Re-run it after every update; `skill status` tells whether that is needed (`outdated`).

### 5. Add a drift test

The point of generation is that nothing is forgotten, so make forgetting fail:

- a registry maps every runnable command to its result type (or `null` plus a note);
- a test walks the real command tree and asserts that every command is in the registry and in the manifest, that every
  command without a fixed JSON type has a note, and that the version in the manifest equals the binary's version;
- a new exit code must be added to the exit code table or the test fails.

A new command then cannot ship without a skill entry.

## Verify

```bash
tool skill show --json | jq '.commands | length'      # every command is listed
tool skill install --dir /tmp/skills-test --json      # installed
tool skill status --dir /tmp/skills-test --json       # current
```

Change the tool's version (or add an option) and run `status` again: it must say `outdated` until you `install`.

## Troubleshooting

- **The agent does not pick the skill up.** Check the folder for that agent, the front matter (`name` and `description`
  are what the agent uses to decide when to load it) and restart the agent session.
- **`status` says `unreadable`.** The manifest is missing or from an incompatible schema version; run `install`.
- **The skill and the repository guidance disagree.** The generated skill is for *users of the CLI*. Rules for
  *contributors* (how to add a command, the parity rule) belong in the repository's agent guidance and repo skills, not in
  the generated one.

## Related

- [Agent-first CLI for a web app](../../../concepts/ai/tooling/agent-first-cli-for-web-apps.md)
- [Building an OpenClaw agent skill](openclaw-skill-development.md)
- [Packaging a .NET Native AOT CLI with Velopack and Chocolatey](../../development/ci/native-aot-cli-velopack-chocolatey.md)

## Sources

- [Original notes (private)](../../../../../sources/ai/tooling/2026-10-25-agent-first-cli-concept.md)
