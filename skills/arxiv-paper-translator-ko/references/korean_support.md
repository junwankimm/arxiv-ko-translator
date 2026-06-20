# Korean Support Configuration (kotex + XeLaTeX)

Add this to the main `.tex` preamble (before `\begin{document}`). The `kotex` wrapper
auto-loads `xetexko` under XeLaTeX.

## 1. Load kotex
```latex
\usepackage{kotex}
```
If `\usepackage[T1]{fontenc}` is present, **remove it** — it conflicts with XeLaTeX/Unicode.

## 2. Hangul fonts

`kotex` exposes Hangul-specific font setters that don't disturb the Latin/math fonts:
```latex
\setmainhangulfont{<serif/myeongjo>}    % body text
\setsanshangulfont{<sans/gothic>}       % headings, \textsf
\setmonohangulfont{<mono>}              % \texttt, code
```

### Option A — Local compile (this macOS machine; fonts confirmed present)
```latex
\setmainhangulfont{AppleMyungjo}
\setsanshangulfont{Apple SD Gothic Neo}
\setmonohangulfont{D2Coding}
```
List installed Korean fonts to pick alternatives:
```bash
fc-list :lang=ko family | sort -u
```
Alternatives available here: `NanumSquare Neo`, `S-Core Dream`. Korean fonts have no true
italic — kotex maps emphasis sensibly; no extra config needed.

### Option B — Docker (`xu-cheng/texlive-debian`, Nanum preinstalled)
```latex
\setmainhangulfont{NanumMyeongjo}
\setsanshangulfont{NanumGothic}
\setmonohangulfont{NanumGothicCoding}
```
If you don't set fonts at all, `kotex` falls back to its bundled defaults (UnBatang/UnDotum),
which also compile — explicit fonts just look better.

## 3. Localize float / section labels
```latex
\renewcommand{\figurename}{그림}
\renewcommand{\tablename}{표}
\renewcommand{\abstractname}{초록}
\renewcommand{\refname}{참고문헌}
\renewcommand{\bibname}{참고문헌}
\renewcommand{\contentsname}{목차}
\renewcommand{\appendixname}{부록}
```

## 4. cleveref names (only if the paper uses `cleveref`)
```latex
\crefname{figure}{그림}{그림}
\Crefname{figure}{그림}{그림}
\crefname{table}{표}{표}
\Crefname{table}{표}{표}
\crefname{section}{절}{절}
\Crefname{section}{절}{절}
\crefname{equation}{식}{식}
\Crefname{equation}{식}{식}
\crefname{algorithm}{알고리즘}{알고리즘}
\Crefname{algorithm}{알고리즘}{알고리즘}
\crefname{appendix}{부록}{부록}
\Crefname{appendix}{부록}{부록}
```

## 5. Theorem-like environment names
Find `\newtheorem{...}{Name}` (preamble or `.sty`) and replace the display name:
```
Theorem → 정리        Proposition → 명제      Definition → 정의
Lemma → 보조정리       Corollary → 따름정리     Proof → 증명
Assumption → 가정      Remark → 비고           Example → 예시
```
```latex
\newtheorem{theorem}{정리}
\newtheorem{lemma}{보조정리}
```

## 6. Custom CJK/Hangul macros
If the source wraps Hangul/CJK in a custom command, redefine it to output its argument
directly so it doesn't fight `kotex`:
```latex
% before
\newcommand{\kr}[1]{\begin{CJK}...{#1}\end{CJK}}
% after
\newcommand{\kr}[1]{#1}
```

## 7. Page layout
```latex
\raggedbottom   % avoids vertical stretching on mixed Hangul/math pages
```

## 8. Macro adjacency reminder
Hangul is catcode-letter under kotex. A custom macro directly followed by Hangul must be
separated: `\ourmodel{}는`, not `\ourmodel는`. (See translation_guidelines.md.)
