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
  In researcher mode the **structural scaffolding stays English** too — section titles,
  the paper title, and float labels/cross-refs ("Table 3", "Fig. 2") — so it reads like the
  original with Korean prose woven in. Beginner mode localizes those.
- **Selectable Claude model** — translation subagents inherit your session model by default;
  say "use opus" / "use sonnet" / "use haiku" to override.
- **Appendix excluded by default** — detects the appendix/supplementary/bibliography boundary
  and leaves it untranslated to save tokens ("include the appendix" to override).
- **Table-layout safe** — by default table cells are left in English (only captions
  translated) so Hangul reflow doesn't break column widths ("translate table contents" to override).
- **Layout QA pass** — renders the compiled PDF and visually checks for figures/tables that
  drifted or overlap text after EN→KO reflow, then applies targeted LaTeX fixes (best-effort).
- **Font fidelity** — only Hangul fonts are added; the paper's original Latin/math fonts and
  document class are preserved, so it keeps the arXiv/ML-paper look.
- **Parallel translation** — section files (or split chunks of a single big file) are
  translated concurrently; per-file diffs run inside subagents, so the chat shows progress, not walls of diff.
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
translate 2310.06825 to Korean, include the appendix, translate table contents
translate 2310.06825 to Korean, skip layout check
```

Defaults: researcher level · inherit session model · appendix excluded · table cells kept in
English · layout-QA on · source deleted after a verified PDF build.

Output lands in `arXiv_<ID>/`:
- `arXiv_<ID>/<ID>_ko.pdf` — the Korean PDF (always kept)
- `arXiv_<ID>/technical_report_ko.md` — if a report was requested
- LaTeX source — deleted by default (kept under `paper_ko/` only with "keep the tex source")

## Requirements

- **XeLaTeX with `kotex`** + a Korean font. Don't have it? Run the bundled **`arxiv-ko-setup`**
  skill — it installs everything **without Docker** (BasicTeX + `collection-langkorean`) and
  walks you through the one `sudo` step. Docker (`ghcr.io/xu-cheng/texlive-debian`) is only a
  fallback for users who'd rather not install TeX locally.
- `wget` and `tar` for fetching the source.

### One-time setup (no Docker)
A plugin can't auto-install system packages on `/plugin install`, so dependency setup is an
explicit, consented step. Either run the `arxiv-ko-setup` skill ("set up arxiv-ko-translator"),
or do it by hand on macOS:
```bash
brew install --cask basictex
eval "$(/usr/libexec/path_helper)"
sudo tlmgr update --self
sudo tlmgr install collection-langkorean latexmk xetex
```
(Linux: `sudo apt-get install texlive-xetex texlive-lang-korean texlive-latex-extra latexmk fonts-nanum`.)

### Optional (for the Layout-QA pass)
None are mandatory — the skill uses whichever it finds, and can fall back to reading the PDF
directly. Any **one** of: `pdftoppm`/`pdftocairo` (poppler), `sips` (macOS built-in),
`magick` (ImageMagick), or `gs` (Ghostscript). Because nothing here is a hard dependency,
**publishing is unaffected** — Claude Code plugins can't install system packages anyway, so
the skill is written to degrade gracefully and the optional tools are just documented here.

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
