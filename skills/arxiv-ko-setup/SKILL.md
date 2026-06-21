---
name: arxiv-ko-setup
description: One-time setup for the Korean arXiv translator — checks for and installs XeLaTeX, kotex/xetexko, latexmk, and a Korean font without Docker, then writes a sentinel so the translator never re-probes. Use on first run, when the translator reports a missing dependency, or when the user asks to "set up", "install dependencies", "doctor", or "check requirements" for arxiv-ko-translator.
license: MIT
---

# arxiv-ko-translator — Dependency Setup (no Docker)

Detects and installs the toolchain the translator needs to compile Korean PDFs:
**XeLaTeX**, **kotex/xetexko**, **latexmk**, and a **Korean font**. Docker-free. Run this
**once per machine** — on success it writes a sentinel that the translator trusts, so normal
translations never re-probe dependencies.

## Why sudo (and how to avoid it)

A system-mode TeX (MacTeX/BasicTeX installs into `/usr/local/texlive/…`, owned by root), so
`tlmgr install` and the `.pkg` installer need admin rights. The assistant cannot type your
password, so **you** run the `sudo` lines via the `!` prefix (e.g.
`! sudo tlmgr install collection-langkorean`) — it runs interactively and the output returns
to the conversation.

**No-sudo alternative (user-tree TeX Live):** install TeX Live into your home directory; then
`tlmgr` writes there without root. See "Option C" below. Trade-off: a longer first install.

## Step 1: Diagnose

Run the doctor check first and report a table of ✅/❌:
```bash
echo "OS       : $(uname -s)"
echo "xelatex  : $(command -v xelatex || echo MISSING)"
echo "latexmk  : $(command -v latexmk || echo MISSING)"
echo "kotex    : $(kpsewhich kotex.sty 2>/dev/null || echo MISSING)"
echo "xetexko  : $(kpsewhich xetexko.sty 2>/dev/null || echo MISSING)"
echo "ko fonts : $(fc-list :lang=ko family 2>/dev/null | wc -l | tr -d ' ') families"
echo "tlmgr    : $(command -v tlmgr || echo MISSING)"
echo "brew     : $(command -v brew || echo MISSING)"
echo "apt-get  : $(command -v apt-get || echo MISSING)"
echo "dnf      : $(command -v dnf || echo MISSING)"
```
- **All of xelatex/latexmk/kotex/xetexko present and fonts > 0** → already set up. **Skip
  Step 2** and go straight to **Step 3** — the smoke test writes the sentinel so the
  translator stops re-checking. (This is the path for a machine that already has TeX.)
- Otherwise go to Step 2 for the user's platform, then Step 3.

## Step 2: Install (pick the platform)

### macOS

Two TeX options — recommend **BasicTeX** (small) unless the user wants everything.

**A. BasicTeX (~100 MB, recommended):**
```bash
# 1) assistant can run this (Homebrew, no sudo):
brew install --cask basictex
# 2) make tlmgr visible in the current shell:
eval "$(/usr/libexec/path_helper)"
```
Then the user runs the sudo steps (via `!`):
```bash
! sudo tlmgr update --self
! sudo tlmgr install collection-langkorean latexmk xetex
```
`collection-langkorean` includes `kotex`, `cjk-ko`, `xetexko`, and Nanum fonts.

**B. Full MacTeX, no GUI (~5 GB):** `brew install --cask mactex-no-gui` — already bundles
everything above; no extra `tlmgr install` needed.

**C. No-sudo, user-tree TeX Live (any platform, no admin rights):**
```bash
# Installs into ~/texlive (user-owned) — tlmgr then needs no sudo.
cd "$(mktemp -d)" && curl -fsSL https://mirror.ctan.org/systems/texlive/tlnet/install-tl-unx.tar.gz | tar xz
TEXDIR="$HOME/texlive/2026"
./install-tl-*/install-tl --no-interaction --scheme=scheme-small --texdir="$TEXDIR"
export PATH="$TEXDIR/bin/universal-darwin:$TEXDIR/bin/$(uname -m)-linux:$PATH"   # add the line that exists
tlmgr install collection-langkorean latexmk xetex   # no sudo needed (user tree)
```
Add the correct `bin/...` dir to your shell profile so it persists. Fonts: on Linux without
system Korean fonts, also `tlmgr install` won't give XeLaTeX a name-callable OTF — install a
system font (`fonts-nanum`) or rely on the kotex default.

If Homebrew is missing: install it from https://brew.sh, or download the MacTeX `.pkg`
directly from https://tug.org/mactex/ and run the installer.

### About fonts (what gets installed)
This skill does **not** ship a font. The translator uses fonts **already on the system**,
detected at runtime:
- **macOS** already has `AppleMyungjo` / `Apple SD Gothic Neo` / `AppleGothic` built in — no
  font install needed.
- **Linux**: `fonts-nanum` (installed above) provides `NanumMyeongjo`/`NanumGothic`.
- **Worst case**: `\usepackage{kotex}` renders Hangul with its built-in default font even if
  no system Korean font is set. So the build never hard-fails for fonts alone.
- Note: TeX's `collection-langkorean` Nanum is **Type1** (not XeLaTeX-name-callable), which is
  why we rely on system fonts / the kotex default for XeLaTeX, not on that package's fonts.

### Linux (Debian/Ubuntu)
```bash
! sudo apt-get update
! sudo apt-get install -y texlive-xetex texlive-lang-korean texlive-latex-extra latexmk fonts-nanum
```
### Linux (Fedora/RHEL)
```bash
! sudo dnf install -y texlive-xetex texlive-collection-langkorean latexmk
```

After install, re-run **Step 1** to confirm everything flipped to ✅. (You may need to open a
new shell, or re-run `eval "$(/usr/libexec/path_helper)"` on macOS, for PATH changes.)

## Step 3: Smoke test

Verify a Korean doc actually compiles (catches font/encoding issues that mere presence won't):
```bash
cd "$(mktemp -d)" && cat > ko_test.tex <<'EOF'
\documentclass{article}
\usepackage{amsmath}
\usepackage{kotex}
\begin{document}
\section{테스트}
한글과 English term과 수식 $E=mc^2$ 이 함께 렌더링됩니다.
\end{document}
EOF
if xelatex -interaction=nonstopmode -halt-on-error ko_test.tex >/dev/null 2>&1; then
  echo "✅ Korean compile OK ($(pwd)/ko_test.pdf)"
  # Write the sentinel so the translator skips per-run dependency checks.
  SENTINEL="${XDG_CACHE_HOME:-$HOME/.cache}/arxiv-ko-translator/deps-ok"
  mkdir -p "$(dirname "$SENTINEL")"
  printf '%s | %s\n' "$(date -u +%Y-%m-%dT%H:%MZ)" "$(xelatex --version | head -1)" > "$SENTINEL"
  echo "🟢 sentinel written: $SENTINEL  → translations will no longer re-check deps"
else
  echo "❌ compile failed — re-check kotex/fonts (no sentinel written)"
fi
```
A `✅` + sentinel means the translator is ready and will **not** re-probe dependencies on
future runs. (Re-run this skill anytime to re-verify; delete the sentinel to force a recheck.)

## Notes
- No Docker required by this skill (the main translator only falls back to Docker if you have
  no local TeX at all).
- A plugin cannot auto-install on `/plugin install`; this skill is the explicit, consented
  setup step — run it once per machine. The sentinel makes it a one-time action.
