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
| `APPENDIX` | `exclude` \| `include` | `exclude` | "include the appendix", "부록도 번역" |
| `TABLES` | `keep` \| `translate` | `keep` | "translate table contents too" |
| `LAYOUT_QA` | `on` \| `off` | `on` | "skip layout check", "don't do layout QA" |
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
- **APPENDIX** `exclude` (default) skips translating everything from the appendix /
  supplementary boundary onward (and the bibliography) — those are kept **verbatim in
  English** so the document still compiles and refs resolve, but no translation tokens are
  spent on them. See Step 2.3 for boundary detection.
- **TABLES** `keep` (default) leaves the contents of `tabular`/`array` cells in English and
  translates only the table `\caption`. This protects column widths/alignment from Hangul
  reflow. `translate` opts into translating cell text.
- **LAYOUT_QA** `on` (default) renders the compiled PDF and visually checks for
  layout damage caused by EN→KO reflow (see Step 5.5).
- Announce the resolved parameters once at the start, e.g.:
  `> Translating 2310.06825 — level=researcher, model=inherit(session), appendix=excluded, tables=keep, layout-QA=on, delete source after compile.`

## Workflow Overview
1. **Resolve ID & Download** — Parse the arXiv ID from any input form; fetch LaTeX source
2. **Translate** — Translate English prose to Korean per the level knob, preserving LaTeX
3. **REVIEW PHASE** — **MUST COMPLETE** before compiling
4. **Korean Support** — Add `kotex`, set Hangul fonts, localize float labels
5. **Compile** — Generate the Korean PDF with XeLaTeX
6. **Report (optional)** — Korean technical summary
7. **Cleanup** — Move the PDF out, then **delete the LaTeX source** (unless `KEEP_SOURCE`)

## Prerequisites — trust the one-time sentinel (don't re-probe every run)

Dependency installation is handled **once** by the `arxiv-ko-setup` skill, which writes a
sentinel on success. This skill should **not** re-run a full dependency probe on every
translation — just check the sentinel:

```bash
SENTINEL="${XDG_CACHE_HOME:-$HOME/.cache}/arxiv-ko-translator/deps-ok"
if [ -f "$SENTINEL" ]; then echo "deps OK (verified $(cat "$SENTINEL"))"; else echo "NO-SENTINEL"; fi
```

- **Sentinel present** → assume the toolchain is ready and proceed straight to Step 1. (If a
  compile later fails for a missing-package reason, only *then* fall back to a check.)
- **Sentinel absent** (first-ever run on this machine) → do **not** silently install. Tell the
  user once and hand off:
  "First run — I need XeLaTeX + kotex + a Korean font. Run the **`arxiv-ko-setup`** skill once
  (installs without Docker), then I'll continue." Offer to invoke `arxiv-ko-setup` now.

This keeps every normal run check-free; setup is the single place that verifies/installs.

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

### 2.3 Determine the translation scope (main file + appendix boundary)

```bash
# Main file (contains \documentclass)
find paper_ko/ -name "*.tex" -exec grep -l '\\documentclass' {} \; | head -1
```

**Appendix / supplementary boundary** (when `APPENDIX=exclude`, default). Papers vary, so
detect the boundary using the strongest signal present, in this priority order:
1. The `\appendix` macro — everything **after** it is appendix. Most reliable.
2. `\section{Appendix}` / `\section*{Appendix}` / `\section*{Supplementary...}` /
   `\appendices` (IEEE) — boundary at that line.
3. The bibliography — `\bibliography{...}`, `\begin{thebibliography}`, or `\printbibliography`.
   Everything from References onward is not narrative to translate anyway.
4. Separate appendix **files** included via `\input{}`/`\include{}` whose names match
   `appendix*`, `supp*`, `supplement*`, `appendices*` — exclude those files wholesale.

```bash
# Locate the boundary marker in the main file:
grep -nE '\\appendix\b|\\appendices\b|\\section\*?\{(Appendix|Appendices|Supplement)' paper_ko/<main>.tex
grep -nE '\\(bibliography|printbibliography)\b|\\begin\{thebibliography\}' paper_ko/<main>.tex
```

- If a boundary is found, **translate only the content before it**; leave everything from the
  boundary onward (appendix + bibliography) **unchanged/verbatim**. The document still
  compiles and all `\ref`/`\cite`/`\appendix` references still resolve.
- If you cannot confidently locate a boundary, **state that and translate the whole body**
  (safer than silently dropping main-text sections). Mention it in the start-of-run summary.
- `APPENDIX=include` → translate the appendix too (still never the bibliography entries).

**Scope summary to print:** e.g. `Body: sections 1–6 (intro.tex … conclusion.tex). Appendix A–C + references: kept in English (excluded).`

### 2.4 Dispatch Translation Tasks — parallel, edits inside subagents

**Always do the edits inside translation subagents, not in the main thread.** This both
(a) parallelizes and (b) keeps the per-file Edit diffs out of your main view — the
orchestrator only prints short progress lines (see "Progress vs. diffs" below).

**Ordering & parallelism:**
1. **Main file first, sequentially** — it seeds the shared terminology table. (If the main
   file is mostly `\input{}`s with little prose, this is quick.)
2. **All remaining in-scope section files → dispatch in parallel** in a *single message*
   with multiple Agent calls (not one-at-a-time). This is the main speed lever.
3. **Single-file paper** (all body text in one big `.tex`, 3+ sections): optionally split to
   parallelize — see 2.5.

**Each translation Task:**
- Agent tool, `subagent_type: general-purpose`
- **`model`**: if the user set `MODEL`, pass it (`opus`/`sonnet`/`haiku`); otherwise omit to inherit the session model.
- Input: one in-scope `.tex` file (or chunk) path under `paper_ko/`
- Action: Read → translate → write back (Edit), all within the subagent
- Must follow [references/translation_prompt.md](references/translation_prompt.md) and use the gathered context (title, abstract, structure, terminology, **resolved `LEVEL`**, **`TABLES` policy**, and which trailing portion is the excluded appendix).

**Progress vs. diffs:** report progress as one line per file/chunk — e.g.
`✅ Methods (3/7) · ⏳ Experiments …`. Do **not** paste translated `.tex` content or diffs
into the conversation. (Diffs produced *inside* a subagent are not streamed to the main
view, so routing all edits through subagents is what suppresses them. The orchestrator's
own messages stay at the progress-line level.)

### 2.5 (Optional) Split a single large file to parallelize

Only when the body is one big `.tex` with several top-level sections and translation feels slow:

1. Keep the preamble (everything up to and including `\begin{document}`, plus title/abstract)
   as `chunk_00`; translate it first (terminology seed).
2. Split the body at top-level `\section{` boundaries into `chunk_01.tex`, `chunk_02.tex`, …
   (stop at the appendix boundary from 2.3 — excluded part stays in a final untranslated chunk).
3. Translate `chunk_01..N` **in parallel** (one subagent each), then concatenate the chunks
   back in original order into the file and **diff against the original line count / structure**
   to confirm nothing was dropped or reordered.
4. Risk to watch: terminology drift across chunks (mitigated by the shared terminology table)
   and split/reassembly errors. If reassembly doesn't verify cleanly, fall back to translating
   the file as a single Task. Don't split files under ~3 sections — overhead isn't worth it.

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

Follow [references/korean_support.md](references/korean_support.md) to configure `kotex`, Hangul fonts, and localized labels.

**Pick Hangul fonts by runtime detection — do NOT hardcode** (a hardcoded family may be
absent on another machine and break the build). First see what's actually installed:
```bash
fc-list :lang=ko family | sort -u
```
Then choose the first present family from each priority list (these are fontconfig-visible
across platforms / macOS built-ins):
- main (serif/myeongjo): `NanumMyeongjo` → `UnBatang` → `AppleMyungjo` → `Apple SD Gothic Neo`
- sans (gothic): `NanumGothic` → `NanumBarunGothic` → `Apple SD Gothic Neo` → `AppleGothic`
- mono (optional): `D2Coding` → `NanumGothicCoding` → *(skip if none — fall back to the sans)*

```latex
\usepackage{kotex}
\setmainhangulfont{<first available main>}
\setsanshangulfont{<first available sans>}
% set mono only if a Korean mono font exists; otherwise omit this line
```
**If none of the named families are present, omit the `\set*hangulfont` lines entirely** —
`\usepackage{kotex}` alone renders Hangul with its built-in default font, so the build still
succeeds. (Verified: kotex-only compiles Korean on XeLaTeX.) Never set a font you didn't
confirm with `fc-list`.

**Preserve the original arXiv/ML-paper look (font fidelity):** set **only the Hangul**
fonts above. **Do NOT** set `\setmainfont`/`\setsansfont` — leaving them unset keeps the
paper's original Latin/math fonts (NeurIPS/ICML Times, Computer Modern, etc.), so English
text and equations render exactly as in the original. Keep the paper's own `\documentclass`
and Latin font packages (`times`, `newtxtext`, …); only **remove `\usepackage[T1]{fontenc}`**
(incompatible with XeLaTeX). Pick a Hangul font that matches the body: a **Myeongjo** (serif,
e.g. `AppleMyungjo` / Docker `NanumMyeongjo`) for the usual serif ML body; a Gothic only if
the paper's body is sans. See [references/korean_support.md](references/korean_support.md) § Font fidelity.
**Float labels & section titles are LEVEL-dependent:**
- `LEVEL=researcher` (default): **keep them English.** Do *not* localize — leave
  `Figure`/`Table`/`Algorithm`/`References`/`Abstract` and all `\section{...}` titles in
  English so "Table 3"/"Fig. 2" cross-references read as in the original. (Section titles
  are simply left untranslated during Step 2.)
- `LEVEL=beginner`: localize the labels (and section titles are translated in Step 2):
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

Also scan the compile log for layout warnings:
```bash
grep -nE 'Overfull \\hbox|Overfull \\vbox|Float too large|LaTeX Warning: .*float' paper_ko/<main>.log | head
```

## Step 5.5: Layout QA (when `LAYOUT_QA=on`, default)

EN→KO reflow changes line breaks and paragraph lengths, which can push floats (figures/
tables) to odd spots, overlap text, or squeeze `minipage`/`wrapfigure`/inline layouts. Do a
**visual** pass — log warnings alone don't catch misplacement.

1. Render the PDF pages to images (use the first available tool):
   ```bash
   cd arXiv_${ARXIV_ID}/paper_ko
   pdftoppm -png -r 110 <main>.pdf qa_page        # poppler  → qa_page-01.png, ...
   # fallbacks: pdftocairo -png -r 110 <main>.pdf qa_page   |   sips -s format png <main>.pdf --out qa_page.png   |   magick -density 110 <main>.pdf qa_page.png
   ```
2. **Read** the `qa_page-*.png` images (Read tool reads images) — or Read the PDF directly
   via the Read tool's `pages` param — and look for: figures/tables overlapping text,
   captions split from floats, content past margins, a float dumped pages away from its
   reference, broken `minipage`/side-by-side figures.
3. Apply **targeted** LaTeX fixes and recompile, e.g.:
   - Float drifting: tighten placement `[t]`/`[h]`, add `\FloatBarrier` (needs `placeins`)
     before the next section, or `\clearpage`.
   - Overlap/overfull: shrink with `\resizebox{\linewidth}{!}{...}`, reduce `\includegraphics`
     `width`, or relax `minipage` widths (`0.48\linewidth` → `0.46\linewidth`).
   - Caption split: wrap the float so caption and graphic stay together.
4. Re-render and re-check the affected pages. Iterate a few times.

**Honest limitation:** this is best-effort QA, not a guarantee. Common float drift and
overfull boxes are fixable this way; pathological hand-tuned layouts (tightly packed
multi-`minipage` grids, absolute positioning) may still need manual touch-up. Report any
pages you couldn't fully resolve rather than claiming perfect layout.

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
