---
name: arxiv-ko-setup
description: Check for and install the system dependencies the Korean arXiv translator needs — XeLaTeX, kotex/xetexko, latexmk, and Korean fonts — without Docker. Use when the translator reports a missing dependency, or when the user asks to "set up", "install dependencies", "doctor", or "check requirements" for arxiv-ko-translator.
license: MIT
---

# arxiv-ko-translator — Dependency Setup (no Docker)

Detects and installs the toolchain the translator needs to compile Korean PDFs:
**XeLaTeX**, **kotex/xetexko**, **latexmk**, and a **Korean font**. Docker-free.

> **sudo note:** TeX package installs need `sudo`, and the assistant cannot type your
> password. Run any `sudo ...` line yourself by typing it in the prompt with a leading `!`
> (e.g. `! sudo tlmgr install collection-langkorean`) — that runs interactively so you can
> enter your password, and the output comes back into the conversation.

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
- **All of xelatex/latexmk/kotex/xetexko present and fonts > 0** → already set up. Stop and
  confirm; optionally run the Step 3 smoke test.
- Otherwise go to Step 2 for the user's platform.

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

If Homebrew is missing: install it from https://brew.sh, or download the MacTeX `.pkg`
directly from https://tug.org/mactex/ and run the installer.

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
xelatex -interaction=nonstopmode -halt-on-error ko_test.tex >/dev/null 2>&1 \
  && echo "✅ Korean compile OK ($(pwd)/ko_test.pdf)" \
  || echo "❌ compile failed — re-check kotex/fonts"
```
A `✅` here means the translator is ready to run.

## Notes
- No Docker required by this skill (the main translator only falls back to Docker if you have
  no local TeX at all).
- A plugin cannot auto-install on `/plugin install`; this skill is the explicit, consented
  setup step — run it once per machine.
