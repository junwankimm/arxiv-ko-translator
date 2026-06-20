# LaTeX → Korean Translation Guidelines

Guidelines for translating arXiv papers from English to Korean while keeping the
LaTeX source compilable and the equations untouched.

## Core Principles

1. **Preserve LaTeX structure** — every command, environment, and macro stays valid.
2. **Never touch math** — equations, inline math, and code are copied verbatim.
3. **Keep it compilable** — the result must build with XeLaTeX + `kotex`.
4. **Selective translation** — translate only prose meant for human readers.
5. **Respect the LEVEL knob** — vocabulary handling depends on `researcher` vs `beginner`.

---

## The LEVEL Knob (vocabulary policy)

Korean ML writing naturally code-switches: researchers read English terms inline far
faster than their Korean calques. The knob picks how aggressively to localize vocabulary.
**Sentence structure, particles (조사), and verbs are always Korean** in both modes — only
*term* handling differs.

### `LEVEL=researcher` (DEFAULT)

Goal: a Korean ML researcher skims the paper fast in their native sentence flow, without
re-decoding unfamiliar Korean calques of standard terms.

- **Keep common ML / technical terms in English**, inline, with no Korean gloss. Examples
  that stay English: `attention`, `transformer`, `embedding`, `gradient`, `fine-tuning`,
  `pre-training`, `loss`, `logit`, `baseline`, `benchmark`, `token`, `prompt`, `inference`,
  `latency`, `throughput`, `checkpoint`, `optimizer`, `learning rate`, `batch`, `epoch`,
  `regularization`, `ablation`, `representation`, `latent`, `decoder`, `encoder`,
  `self-supervised`, `zero-shot`, `in-context`, `state-of-the-art (SOTA)`, etc.
- Attach Korean particles directly to the English term: `attention을`, `gradient의`,
  `transformer는`, `baseline보다`. (kotex handles English+조사 spacing.)
- Translate only the **connective prose, verbs, and logical glue** into Korean.
- Do **not** invent Korean for a term just to avoid English. If in doubt, keep English.
- Proper nouns / model names always stay English (BERT, ResNet, LLaMA, Adam).
- **Keep the structural scaffolding in English** (researcher mode):
  - **Section/subsection titles** stay English: `\section{Introduction}` stays
    `\section{Introduction}` (not `서론`), `\subsection{Related Work}` unchanged, etc.
    Researchers navigate faster by the familiar English headings.
  - **Float name labels** stay English: keep `Figure`, `Table`, `Algorithm`, `Section`,
    `Equation` — do **not** localize to 그림/표/알고리즘. (i.e. skip the `\figurename`/
    `\tablename` renames and the Korean `cleveref` names — see korean_support.md.)
  - **Cross-references stay as the paper wrote them**: `Table 3`, `Tab. 3`, `Fig. 2`,
    `Section 4`, `Eq. (5)` — translate the surrounding sentence but leave the reference
    phrasing and `\cref`/`\ref` output untouched.
  - The paper's `\title{...}` also stays English in researcher mode.
  - Still translated in researcher mode: body prose, caption *text* (the descriptive part,
    not the auto "Figure N:" prefix), footnotes.

**Example (researcher):**
```latex
% Before
We train the encoder with a contrastive loss to improve representation alignment.
For each mini-batch we sample positive and negative pairs and minimize InfoNCE.

% After
우리는 contrastive loss로 encoder를 학습시켜 representation의 alignment를 개선한다.
각 mini-batch에서 positive/negative pair를 sampling하여 InfoNCE를 최소화한다.
```

### `LEVEL=beginner`

Goal: approachable for students / non-specialists.

- **Translate terms into Korean**, with the **English in parentheses on first mention**,
  then use the Korean form afterward.
- Acronyms: first mention `한국어 (Full Name, ABBR)`, then `ABBR`.
- Still keep universally-untranslated names (model names, proper nouns) in English.
- **Localize the structural scaffolding**: translate section/subsection titles to Korean and
  localize float labels (그림/표/알고리즘) and `cleveref` names. (This is the opposite of
  researcher mode.) **Exception:** the paper `\title{...}` stays English even here.

**Example (beginner):**
```latex
% After
우리는 대조 손실(contrastive loss)로 인코더(encoder)를 학습시켜 표현(representation)의
정렬(alignment)을 개선한다. 각 미니배치(mini-batch)에서 양성/음성 쌍(positive/negative pair)을
샘플링하여 InfoNCE를 최소화한다.
```

### Quick decision table

| Content | researcher | beginner |
|---------|-----------|----------|
| Standard ML term (attention, loss, gradient...) | keep English | Korean + (English) on 1st mention |
| Everyday word (method, result, however, we show) | Korean | Korean |
| Acronym (RAG, MoE, SOTA) | keep acronym, optionally expand once | 한국어 (Full Name, ABBR), then ABBR |
| Model / proper noun (BERT, LLaMA) | English | English |
| Math / code / `\label` / `\cite` | unchanged | unchanged |

### Domain generality (not just ML)

The knob itself is **field-agnostic** — it is a policy about *whether established
domain jargon stays in its original English form*, which holds for any discipline where
Korean researchers routinely read English terms inline (physics, statistics, math,
systems, bioinformatics, economics, etc.). Only the *example term list above is ML-flavored*.

When you gather context in Step 2.2, **infer the paper's field and build the "keep in
English" set from that field's conventions**, not from a fixed ML list. Examples of what
`researcher` mode keeps in English by field:
- **Statistics / ML theory**: estimator, prior, posterior, likelihood, regret, covariate, p-value
- **Systems / DB**: throughput, latency, cache, replication, consistency, sharding, commit
- **Physics**: Hamiltonian, Lagrangian, eigenstate, renormalization, lattice
- **Bio / chem**: assay, in vitro, motif, expression, residue, knockout
- **Math**: manifold, homomorphism, compact, almost surely, supremum

Caveat: in a field with strong, settled Korean terminology (e.g. some pure-math or
clinical terms have universal Korean equivalents), `researcher` mode may keep *fewer*
things in English than in ML. Use judgment: keep English when the Korean calque would
*slow a domain expert down*; otherwise use the standard Korean term. If a paper spans
multiple fields, prefer the dominant field's conventions and stay consistent.

---

## What to Translate

### ✅ Always translate (both modes)
1. **Body prose** — paragraphs, sentences (Korean sentence structure + particles)
2. **Caption text** — the descriptive prose in `\caption{...}` (not the auto "Figure N:" prefix)
3. **Reader notes** — `\footnote{...}`, `\thanks{...}`, margin notes

### ✋ Never translate the paper title (both modes)
- **`\title{...}` / `\icmltitle{...}`** stays in the original English in *all* modes. The
  title identifies the paper; keep it verbatim. (Optionally add a Korean gloss as a
  `%` comment, never in the rendered title.)

### ↕ Translate in beginner mode only (kept English in researcher mode)
- **Section titles** — `\section{...}`, `\subsection{...}`, `\subsubsection{...}`
- **Float name labels & cross-ref words** — Figure/Table/Algorithm/Section/Equation
- **Hard-coded template labels** in `.sty`/`.cls` — `Abstract`, `Equal contribution`,
  `Correspondence to:`, `Preprint`, `Under review`. (In researcher mode keep them English;
  `Equal contribution`-type author notes may still be localized if you wish — minor.)

```latex
% \section{Introduction}
%   researcher →  \section{Introduction}     (unchanged)
%   beginner   →  \section{서론}
```

### ❌ Never translate
1. **Math** — `\begin{equation}...\end{equation}`, `$...$`, `\[ ... \]` — verbatim.
2. **LaTeX commands/environments** — `\includegraphics`, `\label`, `\begin{figure}`.
3. **Code blocks** — `lstlisting`, `minted`, `verbatim` contents (translate only `caption`).
4. **Raw data in tables** — AI dialogue, traceback, user-input examples, code cells.
   (Rule of thumb: "evidence/data" → keep; "narrative" → translate.)
5. **Algorithm pseudocode** — keep `\STATE`/`\FOR` and code; translate only the `\caption`.
6. **Person names & proper nouns** — author names, model names (ResNet, BERT) stay English.
   Institutions: well-known Korean institutions may use the official Korean name
   (e.g. Seoul National University → 서울대학교); otherwise keep English.
7. **Paths & refs** — `\input{...}`, `\includegraphics{...}`, `\cite{...}`, `\ref{...}`,
   `\label{...}` — unchanged.
8. **URLs** — `\url{...}`, `\href{...}{...}` target — unchanged (link text may be translated).
9. **Inline code/math** — `$E=mc^2$`, `\texttt{...}`, `\verb|...|` — unchanged.

---

## Special Cases

### Inline math in a sentence
Keep the math, translate around it. Add a particle after the math:
```latex
% Before:  The loss function $\mathcal{L}$ measures the error.
% After:   loss function $\mathcal{L}$은 오차를 측정한다.        (researcher)
% After:   손실 함수(loss function) $\mathcal{L}$은 오차를 측정한다.  (beginner)
```

### Tables (default `TABLES=keep`: do NOT translate cell contents)
Hangul is wider than Latin and reflows differently, so translating cells routinely breaks
column widths and alignment. **By default, leave everything inside `tabular`/`array`/`tabularx`
untouched — including header cells — and translate only the table `\caption`.**
```latex
% default (TABLES=keep): cells stay English, caption translated
\begin{tabular}{lcc}
Method & Accuracy & Latency \\   % ← unchanged
Ours   & 95.2\%   & 10ms    \\
\end{tabular}
\caption{ImageNet에서의 성능 비교}   % ← translated
```
Only when the user sets `TABLES=translate` do you translate descriptive headers/narrative
cells (keep numbers/units), and then re-check the layout in the Layout-QA step.

### Layout-affecting content (figures, minipages, floats)
EN→KO reflow can move floats and squeeze `minipage`/`wrapfigure`/inline layouts. **Do not
change float placement specifiers, `\includegraphics` sizes, or `minipage` widths during
translation** — translate only the human-readable text (captions, surrounding prose). Layout
repair is a separate, deliberate step (SKILL Step 5.5 Layout QA), done by viewing the
rendered PDF — not something to guess at while translating.

### Appendix / supplementary (default excluded)
When `APPENDIX=exclude` (default), do not translate content from the appendix boundary
onward (`\appendix`, `\section*{Appendix}`, `\appendices`, or the bibliography) — leave it
**verbatim in English**. It still compiles and `\ref`/`\cite`/`\appendix` references still
resolve; this just saves translation tokens. Translate the appendix only if `APPENDIX=include`.
Never translate bibliography entries themselves.

### Custom macros + Hangul (CRITICAL for compilation)
A custom macro immediately followed by a Hangul syllable must be separated with `{}`,
because `kotex` makes Hangul catcode-letter, so `\ourmodel는` parses as one undefined
command `\ourmodel는`.
```latex
% wrong:   본 논문은 \ourmodel는 제안한다
% right:   본 논문은 \ourmodel{}을 제안한다
```
Same for English-term macros: `\bert{}는`.

### Acronyms
- researcher: keep the acronym; you may expand once: `state-of-the-art (SOTA)`, then `SOTA`.
- beginner: `검색 증강 생성(Retrieval-Augmented Generation, RAG)`, then `RAG`.

### LaTeX comments (`%`-prefixed)
Leave as-is; do not translate (saves tokens).

### Quotation marks
Convert English `"..."` to LaTeX `` ``...'' `` or Korean `“...”`.

---

## Korean Academic Writing Style

Aim for natural written Korean, not word-for-word translation. Both LEVEL modes follow these.

| Goal | Rule | Example |
|------|------|---------|
| Written register | Use the plain declarative `~한다/~이다`, consistently | "보여준다", "제안한다" |
| Right verb | innovation = "제안하다"; overview = "소개하다/설명하다" | "본 논문은 X를 **제안한다**" |
| Drop redundant subject | Trim repeated "우리는"; use "본 논문" or subjectless | "본 논문은 X를 제안한다" |
| No filler | Cut empty intensifiers; cite numbers | "약 100배 빠르다" not "매우 효율적이다" |
| Concise particles | Avoid stacking "~을 통하여 ~하는 것을" | prefer "~로 ~한다" |
| Consistent term | Same concept → same word (mode-appropriate) throughout |  |

**Verb cues:**
- "This paper proposes X" → "본 논문은 X를 제안한다"
- "Section 2 introduces the background" → "2절에서는 배경을 설명한다"
- "achieves 95% accuracy" → "95%의 정확도를 달성한다"
- "We show that" → "본 논문은 ~임을 보인다"

**Citations / units** stay as-is: `(Smith et al., 2020)`, `ms`, `GB`, `Hz`.

---

## File Organization

```
paper_source/      # original (deleted after compile)
paper_ko/          # translated, compiled here, then deleted (unless KEEP_SOURCE)
├── main.tex       # translated
├── sections/*.tex # translated independently; \input{} unchanged
├── figures/*      # copied as-is
├── *.sty / *.cls  # copied; localize hard-coded labels
└── references.bib # copied (titles optionally left English)
```

For detailed automated review steps, see [review_checklist.md](review_checklist.md).
