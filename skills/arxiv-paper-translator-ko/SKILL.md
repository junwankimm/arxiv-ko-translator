---
name: arxiv-paper-translator-ko
description: Translate academic papers from arXiv into Korean (한국어). Use when the user gives an arXiv paper ID (e.g. "2310.06825"), an abs URL (arxiv.org/abs/...), or a pdf URL (arxiv.org/pdf/...) and wants a Korean translation or a Korean technical report. Downloads the LaTeX source, translates narrative text while preserving equations/commands, compiles a Korean PDF, and deletes the bulky source afterward.
license: MIT
---

# arXiv Paper Translator (Korean / 한국어)

Translate arXiv papers into Korean by downloading the LaTeX source, translating
content while preserving structure, compiling a Korean PDF with `kotex`, and then
**deleting the LaTeX source to save disk space**.

## Run Parameters (detect from the user's request; otherwise use defaults)

| Parameter | Values | Default | How the user sets it |
|-----------|--------|---------|----------------------|
| `LEVEL` | `researcher` \| `beginner` | `researcher` | "for beginners", "초보자용", "researcher level", "skim mode" |
| `MODEL` | a Claude model id, or *inherit* | *inherit session* | "use opus", "translate with sonnet", "use haiku" |
| `KEEP_SOURCE` | `true` \| `false` | `false` | "keep the tex source", "don't delete source" |
| `REPORT` | `true` \| `false` | `false` | "also write a technical report", "기술 리포트도" |

- **LEVEL** controls vocabulary. `researcher` (default): keep common ML/technical
  terms in English inline, translate only the connective prose — for fast skimming
  by ML researchers. `beginner`: translate terms into Korean with the English in
  parentheses on first mention. See [references/translation_guidelines.md](references/translation_guidelines.md).
- **MODEL** is the model used for the **translation subagents** (Step 2). By default
  they inherit the current session's model. If the user names a model, pass it as the
  `model` argument when dispatching each translation Task (Agent tool `model` field:
  `opus`, `sonnet`, or `haiku`).
- Announce the resolved parameters once at the start, e.g.:
  `> Translating 2310.06825 — level=researcher, model=inherit(session), delete source after compile.`

## Workflow Overview
1. **Resolve ID & Download** — Parse the arXiv ID from any input form; fetch LaTeX source
2. **Translate** — Translate English prose to Korean per the level knob, preserving LaTeX
3. **REVIEW PHASE** — **MUST COMPLETE** before compiling
4. **Korean Support** — Add `kotex`, set Hangul fonts, localize float labels
5. **Compile** — Generate the Korean PDF with XeLaTeX
6. **Report (optional)** — Korean technical summary
7. **Cleanup** — Move the PDF out, then **delete the LaTeX source** (unless `KEEP_SOURCE`)

## Prerequisites

Check for a local XeLaTeX install (required for Korean via `kotex`):
```bash
xelatex --version && kpsewhich kotex.sty
```
If `kotex.sty` is missing, install it: `tlmgr install kotex cjk-ko` (TeX Live), or use Docker.

If XeLaTeX is not installed locally, check Docker:
```bash
docker --version
```
Recommended image (Nanum Korean fonts preinstalled): `ghcr.io/xu-cheng/texlive-debian` (mirror: `ghcr.1ms.run/xu-cheng/texlive-debian`).

**If neither local XeLaTeX nor Docker is available, STOP** and ask:
"XeLaTeX (with kotex) or Docker is required to compile the Korean PDF. Which do you want to use? I'll help set it up."

## Step 1: Resolve arXiv ID and Download Source

Accept **any** of these input forms and extract `ARXIV_ID`:
- Bare ID: `2310.06825` or `2310.06825v2`
- abs URL: `https://arxiv.org/abs/2310.06825`
- pdf URL: `https://arxiv.org/pdf/2310.06825` or `.../pdf/2310.06825v2.pdf`
- Old-style ID: `cs/0309040`

```bash
# Extract the ID from any of the forms above (strip URL, .pdf, abs/pdf path).
RAW="<user input>"
ARXIV_ID=$(printf '%s' "$RAW" | sed -E 's#https?://arxiv\.org/(abs|pdf)/##; s#\.pdf$##' | tr -d ' ')
echo "ARXIV_ID=${ARXIV_ID}"

mkdir -p arXiv_${ARXIV_ID}
# arXiv e-print endpoint returns the LaTeX source tarball regardless of abs/pdf input.
wget -q "https://arxiv.org/e-print/${ARXIV_ID}" -O arXiv_${ARXIV_ID}/paper_source.tar.gz
mkdir -p arXiv_${ARXIV_ID}/paper_source
tar -xzf arXiv_${ARXIV_ID}/paper_source.tar.gz -C arXiv_${ARXIV_ID}/paper_source
```

**Verify extraction:**
```bash
ls -R arXiv_${ARXIV_ID}/paper_source
```
If the download is a single `.tex` file (not a tarball), see Common Issues below.

## Step 2: Translate LaTeX Files

**IMPORTANT**: Before translating, read [references/translation_guidelines.md](references/translation_guidelines.md) for the detailed rules and the `LEVEL` knob behavior.

### 2.1 Copy source → `paper_ko/`
```bash
cd arXiv_${ARXIV_ID}
mkdir -p paper_ko
rsync -a paper_source/ paper_ko/   # or: cp -r paper_source/* paper_ko/
```
All `.tex` files in `paper_ko/` are translated in-place.

### 2.2 Gather Context (MANDATORY)

Before ANY translation, extract:
1. **Paper Title** — from `\title{...}` (or `\icmltitle{...}`) in the main file
2. **Abstract** — from `\begin{abstract}...\end{abstract}` or `\abstract{...}`
3. **Paper Structure** — list all sections and which `.tex` file holds each
4. **Field** — infer the paper's discipline (ML, stats, systems, physics, bio, math, …).
   The vocabulary knob adapts to it; don't assume ML.
5. **Key Terminology** — build a terminology table for *this field*. Under `LEVEL=researcher`,
   established domain jargon is marked **(keep)** = stay in English; under `LEVEL=beginner`,
   give a Korean gloss for each. The "keep in English" set comes from the field's
   conventions, not a fixed ML list (see [references/translation_guidelines.md](references/translation_guidelines.md)
   § Domain generality). If a term is genuinely ambiguous, ASK the user.

Read [references/translation_prompt.md](references/translation_prompt.md) for the prompt template.

### 2.3 Identify files to translate
```bash
# Main file (contains \documentclass)
find paper_ko/ -name "*.tex" -exec grep -l '\\documentclass' {} \; | head -1
```
- Translate the **main file first** (sequential) to establish shared terminology.
- Then the section files: **3+ files → dispatch in parallel**; 1–2 → sequential.

### 2.4 Dispatch Translation Tasks

**Each translation Task:**
- Agent tool, `subagent_type: general-purpose`
- **`model`**: if the user set `MODEL`, pass it (`opus`/`sonnet`/`haiku`); otherwise omit to inherit the session model.
- Input: one `.tex` file path under `paper_ko/`
- Action: Read file → translate → write back (Edit)
- Must follow [references/translation_prompt.md](references/translation_prompt.md) and use the gathered context (title, abstract, structure, terminology, **and the resolved `LEVEL`**).

## Step 3: Review Translation

After all Tasks complete, review per [references/review_checklist.md](references/review_checklist.md):
1. File completeness
2. LaTeX command spelling (diff command sets)
3. Hangul catcode / macro-adjacency issues
4. Terminology consistency (respecting the `LEVEL` knob)
5. Translation quality spot-check
6. Equations / `\label` / `\ref` / `\cite` untouched

**CRITICAL** — before Step 4 confirm: all checks done, issues fixed, equations intact.

## Step 4: Add Korean Support

Follow [references/korean_support.md](references/korean_support.md) to configure `kotex`, Hangul fonts, and localized labels. Core preamble (XeLaTeX):
```latex
\usepackage{kotex}
% Local macOS fonts (this machine has these). For Docker, prefer Nanum (see korean_support.md).
\setmainhangulfont{AppleMyungjo}
\setsanshangulfont{Apple SD Gothic Neo}
\setmonohangulfont{D2Coding}
```
And localize float labels:
```latex
\renewcommand{\figurename}{그림}
\renewcommand{\tablename}{표}
\renewcommand{\abstractname}{초록}
\renewcommand{\refname}{참고문헌}
\renewcommand{\contentsname}{목차}
```

## Step 5: Compile the Korean PDF

### Option 1 — Local XeLaTeX (preferred; this machine supports it)
```bash
cd arXiv_${ARXIV_ID}/paper_ko
latexmk -xelatex -interaction=nonstopmode main.tex   # main = actual main filename
# or manually: xelatex → bibtex → xelatex → xelatex
```

### Option 2 — Docker (Nanum fonts preinstalled)
```bash
cd arXiv_${ARXIV_ID}
docker run --rm -v "$(pwd)/paper_ko":/workspace -w /workspace \
  ghcr.1ms.run/xu-cheng/texlive-debian:20260101 \
  latexmk -xelatex -interaction=nonstopmode main.tex
```
Fix compile errors and recompile until a PDF is produced. See Common Issues.

## Step 6: Generate Technical Report (only if `REPORT=true`)

Spawn a subagent following [references/summary_prompt.md](references/summary_prompt.md), writing into [assets/report_template.md](assets/report_template.md). Save to `arXiv_${ARXIV_ID}/technical_report_ko.md`. The report is written in Korean, honoring the same `LEVEL` knob.

## Step 7: Cleanup — Delete the LaTeX Source

This is a headline feature: the LaTeX source is large and disposable once the PDF exists.

```bash
cd arXiv_${ARXIV_ID}
MAIN=main   # actual main filename (without .tex)
# 1) Preserve the compiled PDF at the top level.
cp "paper_ko/${MAIN}.pdf" "./${ARXIV_ID}_ko.pdf"
# 2) Verify the PDF exists and is non-empty BEFORE deleting anything.
test -s "./${ARXIV_ID}_ko.pdf" || { echo "PDF missing/empty — NOT deleting source"; exit 1; }
# 3) Delete the bulky source.
rm -rf paper_source paper_source.tar.gz paper_ko
```

- **Default is to delete** (`KEEP_SOURCE=false`). Skip Step 7's deletion if the user
  asked to keep the source — in that case leave `paper_ko/` in place.
- **Never delete unless the PDF was verified** (the `test -s` guard above). If compilation
  failed, keep the source and report the failure.
- Report exactly what was removed and what remains.

## Final Deliverables

- **Korean PDF**: `arXiv_${ARXIV_ID}/${ARXIV_ID}_ko.pdf`
- **Technical report** (if requested): `arXiv_${ARXIV_ID}/technical_report_ko.md`
- LaTeX source: deleted by default (kept under `paper_ko/` only if `KEEP_SOURCE=true`)

## Common Issues & Solutions

| Issue | Solution |
|-------|----------|
| Download is a single `.tex`, not `.tar.gz` | `mv paper_source.tar.gz paper_source.tex`, move it into `paper_source/` dir |
| Main file not `main.tex` | `find . -name "*.tex" -exec grep -l '\\documentclass' {} \;` |
| `kotex.sty` not found | `tlmgr install kotex cjk-ko`, or compile via Docker |
| Hangul renders as blank boxes | Hangul font not set/found — set `\setmainhangulfont{...}` to an installed font (`fc-list :lang=ko family`) |
| Encoding error on compile | `file *.tex`; convert with `iconv -f ISO-8859-1 -t UTF-8` |
| Command misspelled during translation (e.g. `\footnotext`) | Review checklist step 2 — diff command sets |
| Custom macro glued to Hangul → "Undefined control sequence" | Insert `{}`: `\ourmodel{}는` not `\ourmodel는` (Hangul is catcode-letter under kotex) |
| `\usepackage[T1]{fontenc}` present | Remove it — conflicts with XeLaTeX/Unicode |
| Undefined `\ref`/`\cite` | Ensure every referenced file is copied into `paper_ko/`, even untranslated ones |
| Mixed Hangul/Latin spacing looks off | `\usepackage{kotex}` handles most; add `\xetexkostartpunct`/disable only if needed |

## References

- **Translation rules + LEVEL knob**: [references/translation_guidelines.md](references/translation_guidelines.md)
- **Translation prompt**: [references/translation_prompt.md](references/translation_prompt.md)
- **Review checklist**: [references/review_checklist.md](references/review_checklist.md)
- **Korean support**: [references/korean_support.md](references/korean_support.md)
- **Report template**: [assets/report_template.md](assets/report_template.md)
- **Makefile (optional)**: [assets/Makefile.template](assets/Makefile.template)
