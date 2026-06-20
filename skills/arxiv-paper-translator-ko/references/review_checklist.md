# Translation Review Checklist (Korean)

Run after all translation Tasks complete, before compiling.

## 1. File completeness
```bash
# All .tex present in both trees?
diff <(cd paper_source && find . -name "*.tex" -type f | sort) \
     <(cd paper_ko && find . -name "*.tex" -type f | sort)
# All non-.tex assets copied?
diff <(cd paper_source && find . -type f -not -name "*.tex" | sort) \
     <(cd paper_ko && find . -type f -not -name "*.tex" | sort)
```

## 2. LaTeX command spelling (catch typos introduced in translation)
```bash
diff <(cd paper_source && grep -ohrIE '\\[a-zA-Z]+' | sort -u) \
     <(cd paper_ko && grep -ohrIE '\\[a-zA-Z]+' | sort -u) | grep '^>'
```
Right-side-only commands are suspect. Verify each — if not a deliberate addition
(`\figurename`, `\setmainhangulfont`, `\crefname`...), it is likely a typo. Fix it.

## 3. Hangul catcode / macro adjacency
Find custom macros glued to Hangul without `{}`:
```bash
grep -rnE '\\[a-zA-Z]+[가-힣]' paper_ko/ --include='*.tex'
```
Each hit needs `{}` between macro and Hangul: `\xmax{}는`. Background: kotex makes Hangul
catcode-letter, so `\xmax는` parses as one undefined command.

## 4. Equations & references untouched (most important)
Spot-check that math and refs are byte-identical to the source:
```bash
# Inline/display math and key commands should match between trees.
for f in $(cd paper_ko && find . -name '*.tex'); do
  diff <(grep -oE '\\(label|ref|cite|eqref|includegraphics|input)\{[^}]*\}' "paper_source/$f" | sort) \
       <(grep -oE '\\(label|ref|cite|eqref|includegraphics|input)\{[^}]*\}' "paper_ko/$f" | sort) \
    && echo "OK refs: $f" || echo "CHECK refs: $f"
done
```
- [ ] `\begin{equation}`/`$...$` contents unchanged
- [ ] `\label`/`\ref`/`\cite`/`\eqref` keys unchanged
- [ ] `\includegraphics`/`\input` paths unchanged

## 5. LEVEL consistency
- [ ] **researcher**: standard ML terms kept in English throughout (not sporadically Koreanized)
- [ ] **beginner**: each term Korean + English on first mention, Korean thereafter
- [ ] Same concept uses the same wording everywhere
- [ ] Acronyms handled per mode (researcher: keep; beginner: expand once)
- [ ] Model names / proper nouns kept English in both modes

## 6. Coverage spot-check
- [ ] `\title{...}` / `\icmltitle{...}` translated
- [ ] All section/subsection titles translated
- [ ] Figure/table captions translated
- [ ] `\footnote{...}` / `\thanks{...}` translated
- [ ] Code blocks (`lstlisting`/`minted`/`verbatim`) left in English (only captions translated)

## 7. Template hard-coded labels
```bash
grep -rnE '(Equal contribution|Correspondence to|Under review|Preprint|Proceedings of|Abstract)' \
  paper_ko/ --include='*.sty' --include='*.cls' --include='*.tex'
```
- [ ] Conference/journal labels localized or overridden (e.g. `Equal contribution` → `동등 기여`)
- [ ] Affiliations handled (official Korean name for well-known KR institutions, else English)

## 8. Quality
- [ ] Natural written Korean (`~한다/~이다`), no machine-translation artifacts
- [ ] No empty intensifiers; numbers used instead
- [ ] No colloquialisms
