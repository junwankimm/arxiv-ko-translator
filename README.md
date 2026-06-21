# arxiv-ko-translator

A Claude Code plugin that translates **arXiv papers into Korean (한국어)**. Give it an arXiv
ID or URL — it fetches the LaTeX source, translates the prose **without touching equations,
commands, or code**, compiles a Korean PDF with `kotex`, and deletes the bulky source.
Translation is done by **Claude itself** (no external MT service); the model is selectable per run.

## Features

- **Any input** — arXiv ID (`2602.05951`), `arxiv.org/abs/...`, or `arxiv.org/pdf/...`.
- **Equation-safe** — math, `\label`/`\ref`/`\cite`, code blocks, and paths are copied verbatim.
- **Vocabulary knob** — `researcher` (default): keep standard ML/technical terms in English for
  fast skimming, plus section titles, the paper title, and `Figure`/`Table` labels stay English;
  `beginner`: translate terms to Korean with English glosses.
- **Appendix excluded by default** — detects the appendix/bibliography boundary and skips it to save tokens.
- **Table-layout safe** — table cells are left in English by default so Hangul reflow can't break columns.
- **Layout QA** — renders the compiled PDF and fixes figures/tables that drift or overlap after reflow.
- **Font fidelity** — only Hangul fonts are added; the paper's original Latin/math fonts are preserved.
- **Parallel translation** across sections, with selectable Claude model and an optional Korean technical report.

## Installation

> **macOS only.**

```
/plugin marketplace add junwankimm/arxiv-ko-translator
/plugin install arxiv-ko-translator@arxiv-ko-translator
```
Restart Claude Code. Two skills load: `arxiv-paper-translator-ko` (the translator) and
`arxiv-ko-setup` (dependency installer).

## Requirements

macOS with **XeLaTeX + `kotex` + a Korean font**. If they're missing, the translator installs
them **automatically on first run — no admin password needed** (a user-local TeX Live in
`~/texlive`; takes a few minutes the first time). You can also trigger setup anytime:

```
set up arxiv-ko-translator
```

## Usage

Ask in English or Korean:
```
translate https://arxiv.org/abs/2602.05951 to Korean
https://arxiv.org/abs/2602.05951 한국어로 번역해줘
translate 2602.05951 to Korean
```

Control behavior by adding any of these phrases (combine freely):

| Want | Say | Default |
|------|-----|---------|
| Beginner-friendly vocab | `for beginners` | researcher |
| Pick the model | `use opus` / `use sonnet` / `use haiku` | session model |
| Translate the appendix too | `include the appendix` | excluded |
| Translate table cells | `translate table contents` | cells kept English |
| Skip the layout check | `skip layout check` | on |
| Keep the LaTeX source | `keep the tex source` | deleted after PDF |
| Add a technical report | `also write a technical report` | off |

Example combining several:
```
translate https://arxiv.org/abs/2602.05951 for beginners, use opus, include the appendix, keep the tex source
```

Output lands in `arXiv_<ID>/`:
- `arXiv_<ID>/<ID>_ko.pdf` — the Korean PDF
- `arXiv_<ID>/technical_report_ko.md` — if a report was requested

## License

MIT
