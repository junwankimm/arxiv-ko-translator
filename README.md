# arxiv-ko-translator

A Claude Code plugin that translates **arXiv papers into Korean (한국어)**.

Give it an arXiv ID, an `abs` URL, or a `pdf` URL. It fetches the LaTeX source,
translates the narrative text into Korean **without touching equations, commands, or
code**, compiles a Korean PDF with `kotex`, and then **deletes the bulky LaTeX source**.

The translation is done by **Claude itself** (via translation subagents) — no external
MT service — and the model is selectable per run.

## Features

- **Input flexibility** — arXiv ID (`2310.06825`), `arxiv.org/abs/...`, or `arxiv.org/pdf/...`.
- **Equation-safe** — math, `\label`/`\ref`/`\cite`, code blocks, and paths are copied verbatim.
- **Vocabulary knob (`LEVEL`)**:
  - `researcher` (default) — keep standard ML terms in English inline (attention, gradient,
    fine-tuning, baseline…); translate only the connective prose. Built for fast skimming.
  - `beginner` — translate terms into Korean with English in parentheses on first mention.
- **Selectable Claude model** — translation subagents inherit your session model by default;
  say "use opus" / "use sonnet" / "use haiku" to override.
- **Korean PDF** via XeLaTeX + `kotex` (Nanum / Apple fonts, or Docker fallback).
- **Auto-cleanup** — deletes the LaTeX source after a verified PDF build (`KEEP_SOURCE` to opt out).
- **Optional Korean technical report** — "also write a technical report".

## Install

### Option A — from a local clone (use it right now)
```
/plugin marketplace add /Users/junwankim/arxiv-ko-translator
/plugin install arxiv-ko-translator@arxiv-ko-translator
```

### Option B — from GitHub (after you publish)
```
/plugin marketplace add <your-github-username>/arxiv-ko-translator
/plugin install arxiv-ko-translator@arxiv-ko-translator
```

Then restart Claude Code (or run `/plugin` to confirm it loaded). The skill
`arxiv-paper-translator-ko` becomes available.

## Usage

Just ask, in English or Korean:

```
2310.06825 한국어로 번역해줘
translate https://arxiv.org/abs/2310.06825 to Korean
translate 2310.06825 for beginners, also write a technical report
translate https://arxiv.org/pdf/2310.06825 to Korean using opus, keep the tex source
```

Output lands in `arXiv_<ID>/`:
- `arXiv_<ID>/<ID>_ko.pdf` — the Korean PDF (always kept)
- `arXiv_<ID>/technical_report_ko.md` — if a report was requested
- LaTeX source — deleted by default (kept under `paper_ko/` only with "keep the tex source")

## Requirements

- **XeLaTeX with `kotex`** (`tlmgr install kotex cjk-ko`), **or Docker**
  (`ghcr.io/xu-cheng/texlive-debian`, which ships Nanum fonts).
- `wget` and `tar` for fetching the source.

## Publishing this plugin

This repo is **both** a Claude Code plugin and a single-plugin marketplace
(`.claude-plugin/plugin.json` + `.claude-plugin/marketplace.json`). To publish:

1. `git init && git add -A && git commit -m "arxiv-ko-translator"`
2. Create a public GitHub repo and push.
3. Others install with:
   `/plugin marketplace add <username>/arxiv-ko-translator` then
   `/plugin install arxiv-ko-translator@arxiv-ko-translator`.

Optionally list it on a community index (e.g. claudemarketplaces.com) by pointing it at
the GitHub repo. No npm publish is needed — Claude Code plugins are distributed via git
marketplaces, not npm.

## License

MIT
