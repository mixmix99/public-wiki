---
type: guide
title: Packaging a .NET Native AOT CLI with Velopack and Chocolatey
description: Publish a single-exe .NET CLI with Native AOT, fail on trim warnings, and ship it as installer, portable zip and Chocolatey package from a CI release workflow.
tags:
- dotnet
- native-aot
- velopack
- chocolatey
- release
- ci
- windows
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

# Packaging a .NET Native AOT CLI with Velopack and Chocolatey

Turn a .NET 10 console application into one native `tool.exe` (Native AOT, about 12 MB, no runtime needed), keep the
trimmer honest, and ship the same binary as a Setup installer, a portable zip and a Chocolatey package from one script
that local runs and CI share. The CLI is described in the [concept](../../../concepts/ai/tooling/agent-first-cli-for-web-apps.md).

> [!note] Status
> The publish and pack stages were run locally. The CI release workflow described in step 6 was written but had not
> run yet when this was captured.

## Prerequisites

- .NET 10 SDK.
- For Native AOT on Windows: Visual Studio **C++ build tools** (component `Microsoft.VisualStudio.Component.VC.Tools.x86.x64`)
  and a Windows SDK.
- Velopack CLI `vpk` (a local dotnet tool in the repository's tool manifest, so CI and developers use the same version).
- Chocolatey (`choco`) for packing; a package feed that accepts NuGet pushes (a Gitea package registry works).
- A Windows build machine or runner (the exe is `win-x64`).

## Steps

### 1. Make the project Native AOT friendly

In the console project: `PublishAot` for Release, a fixed runtime identifier (`win-x64`), and the version in one shared
props file imported by every project. A Debug build can stay a normal framework-dependent build.

Rules for the code (the trimmer cannot see through reflection):

- JSON through a **source-generated** `JsonSerializerContext`; use `JsonNode` for documents built at run time.
- No reflection-based serialization, no dynamic assembly loading, no libraries that need them (check each new
  package).
- Do not call the generic `JsonArray.Add<T>`; cast to `JsonNode?` to pick the non-generic overload.

### 2. One script for publish and pack

Write `pack-cli.ps1` with stages so CI cannot drift from local runs:

| Stage | Does |
|---|---|
| `Publish` | `dotnet publish -r win-x64 -c Release` into `artifacts/bm`; **fails when the output contains any `IL2xxx` or `IL3xxx` warning** |
| `Pack` | `vpk pack` of the publish folder (Setup, portable, feed files), then `choco pack` |
| `All` | both, in that order (default for local runs) |

The warning check is what keeps AOT safe over time: a new package that is not trim-compatible breaks the build instead of
the user. The script reads the version from the published exe (`FileVersion`) and gives Velopack and Chocolatey the
version **without leading zeros** (`26.08.28.0` becomes `26.8.28`), because NuGet versions do not accept them.

Version scheme used here: `yy.MM.dd` for a release (from the date), `99.99.99.99` for Debug builds so a local build is
never mistaken for a release.

### 3. Velopack: installer, portable zip, and the PATH problem

`vpk pack --packId <Id> --packVersion <v> --packDir artifacts/bm --mainExe tool.exe` produces `...-Setup.exe`, a
portable zip, and the update feed files. Two facts that are easy to miss:

- **Velopack does not put the app on `PATH`.** A CLI needs that. Handle Velopack's install hooks in the app itself: when
  the first argument starts with `--veloapp-` (install, updated, uninstall), run the hook code that adds or removes the
  install folder (`%LocalAppData%\<Id>\current`) in the **user** `PATH`, and exit. Normal calls never load Velopack, so
  the 30 ms start-up is untouched. Re-check start-up time and the AOT warnings whenever the Velopack version changes.
- Run the agent-skill installer in the same hook (see
  [Generating an agent skill from the CLI itself](../../ai/tooling/generated-agent-skill-for-a-cli.md)).

### 4. Chocolatey package

A `.nuspec` and a `tools/chocolateyinstall.ps1`. Chocolatey shims `tool.exe` onto the `PATH` by itself, so the install
script only has to run `tool skill install`. The package contains the same exe that Velopack ships.

```powershell
choco pack <nuspec> --outputdirectory artifacts/choco
choco push artifacts/choco/<id>.<version>.nupkg --source <feed-url> --api-key <key from the secret store>
```

Install:

```powershell
choco install <id> --source <feed-url>
```

Push to your own feed, not the community feed. The API key is a CI secret; never commit it.

### 5. Local check before every release

```powershell
dotnet test <solution>                       # all tests, in-process
pwsh scripts/pack-cli.ps1 -Stage Publish     # 0 trim/AOT warnings, note the size
pwsh scripts/pack-cli.ps1 -Stage Pack        # Setup, portable zip, nupkg
```

Do not run the Release exe for "just a quick check" if its default host is production; point a debug build at a local
backend instead.

### 6. Release workflow

A workflow on the Windows runner, triggered by a release tag or manually: restore tools, `pack-cli.ps1 -Stage All`
with `BUILD_VERSION` from the date, upload the Setup/zip as release assets, `choco push` to the feed. Secrets needed:
the feed URL (variable), the feed API key, and a token to create the release. Keep all logic in the script and only
call it from the workflow.

## Verify

- `tool --version` or the status page shows the version you built.
- A fresh terminal finds `tool` on the `PATH` after Setup or `choco install`; `tool skill status` says `current`.
- Uninstall removes the folder and the `PATH` entry.

## Troubleshooting

- **`IL2xxx`/`IL3xxx` warning fails the publish.** A package or your own code uses reflection. Replace it with a
  source-generated alternative or drop the package.
- **Native code generation fails.** The C++ build tools or Windows SDK are missing on the machine.
- **`choco pack` rejects the version.** Leading zeros; strip them.
- **SmartScreen warns on the installer.** There is no code-signing certificate; sign the binaries if that matters to your
  users.
- **The tool is not found after Setup.** Open a new terminal (the user `PATH` changed); check the install hooks ran.

## Related

- [Agent-first CLI for a web app](../../../concepts/ai/tooling/agent-first-cli-for-web-apps.md)
- [Generating an agent skill from the CLI itself](../../ai/tooling/generated-agent-skill-for-a-cli.md)

## Sources

- [Original notes (private)](../../../../../sources/ai/tooling/2026-10-25-agent-first-cli-concept.md)
