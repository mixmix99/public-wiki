---
type: guide
title: 'Zed: region folding with custom-code-folding'
description: 'Set up the custom-code-folding Zed extension for #region/#endregion folding, including the fix needed for C#.'
tags:
- zed
- editor
- folding
- csharp
- extensions
status: draft
resource: https://github.com/ali-ramadhan/zed-custom-code-folding
created: 2026-09-27T17:47:57Z
updated: 2026-09-27T19:04:10Z
generated:
  by: hermes/gpt-6-sol
  at: 2026-09-27T19:04:10Z
verified: []
stale_after: 2027-09-27T17:47:57Z
sources:
- id: 2026-09-27-zed-custom-code-folding
  resource: private:/sources/development/editors/2026-09-27-zed-custom-code-folding.md
- id: zed-custom-code-folding-repo
  resource: https://github.com/ali-ramadhan/zed-custom-code-folding
relations: []
superseded_by:
---

# Zed: region folding with custom-code-folding

Zed has no built-in support for `#region`/`#endregion`-style folding (unlike VS Code). The
[zed-custom-code-folding](https://github.com/ali-ramadhan/zed-custom-code-folding) extension adds
it via an LSP server that implements `textDocument/foldingRange`, covering 40+ languages including
C#. Use this guide to install it and, if you write C#, fix its region syntax so it actually matches.

## Prerequisites

- Zed editor.

## Steps

### 1. Install the extension

Open the command palette (`Ctrl+Shift+P`) → `zed: extensions` → search "Custom Code Folding" →
Install. No build step is needed; it's published on the Zed extension marketplace.

### 2. Enable LSP folding ranges

The extension only works once LSP folding ranges are enabled. This replaces Zed's built-in
indent/tree-sitter folding, but structural folds (functions, classes) are preserved because Zed
merges ranges from all active language servers. Add to `settings.json`:

```json
{
  "document_folding_ranges": "on"
}
```

At this point the extension's two default patterns work: `region` (requires a comment prefix —
`//`, `/*`, or `#` — before `#region`/`#endregion`) and `plus-minus` (`# +++ Section Name` … `# ---`).

### 3. Add a custom pattern for C#

C#'s native region syntax is a bare preprocessor directive — `#region Name` / `#endregion` with
**no** comment prefix. The default `region` pattern requires a prefix, so it does **not** match C#
out of the box, even though `CSharp` is in the extension's supported language list. Add a custom
pattern to `settings.json`:

```json
{
  "document_folding_ranges": "on",
  "lsp": {
    "custom-code-folding": {
      "initialization_options": {
        "include_defaults": true,
        "patterns": [
          {
            "name": "csharp-region",
            "start": "^\\s*#region\\b\\s*(?P<label>.*?)\\s*$",
            "end": "^\\s*#endregion"
          }
        ]
      }
    }
  }
}
```

The `(?P<label>...)` capture group is shown as the label when a region is folded.

## Verify

Open a C# file with `#region`/`#endregion` blocks and confirm they now fold. Zed's default folding
shortcuts (Windows):

| Shortcut | Action |
|---|---|
| `Ctrl+K Ctrl+0` | `editor::FoldAll` — folds everything foldable |
| `Ctrl+K Ctrl+J` | `editor::UnfoldAll` |
| `Ctrl+K Ctrl+L` | `editor::ToggleFold` |
| `Ctrl+K Ctrl+1`–`9` | `editor::FoldAtLevel_N` — fold by indentation depth |

## Troubleshooting

- **Regions still don't fold after enabling `document_folding_ranges`:** confirm the language
  server actually attached to the file (it must be a language the extension supports) and that the
  custom pattern's `start`/`end` regexes match your exact comment style.
- **Want to fold only regions, not every foldable block:** not directly possible. The extension
  only supplies fold *ranges* to Zed via LSP — it cannot add new editor actions or filter `FoldAll`
  by kind, since extensions can't register custom editor commands. `FoldAtLevel_N` can approximate
  "regions only" if your regions consistently sit at one indentation depth, but it will also fold
  any other block at that depth. Auto-folding regions on file open is not supported either — no
  such setting exists in Zed.

## Related

- [zed-custom-code-folding on GitHub](https://github.com/ali-ramadhan/zed-custom-code-folding)

## Source captures (private)

- [Original Zed custom-folding notes (private)](../../../../../sources/development/editors/2026-09-27-zed-custom-code-folding.md)
