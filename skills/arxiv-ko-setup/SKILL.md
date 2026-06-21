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

## No sudo by default

The **default** install (Option A below) is a **user-tree TeX Live** in `~/texlive` — owned by
you, so `tlmgr` and the assistant need **no admin password**. This makes first-run setup fully
hands-off. Only the optional faster routes (BasicTeX / MacTeX, which install into the
root-owned `/usr/local/texlive/…`) need `sudo`; for those, the user runs the `sudo` line via
the `!` prefix (e.g. `! sudo tlmgr install collection-langkorean`).

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

## Step 2: Install (macOS)

### A. Default — no-sudo, user-tree TeX Live (fully automatic, no password)

Installs a user-owned TeX Live into `~/texlive`, so `tlmgr` and the assistant need **no sudo**.
The assistant can run all of this itself:
```bash
cd "$(mktemp -d)"
curl -fsSL https://mirror.ctan.org/systems/texlive/tlnet/install-tl-unx.tar.gz | tar xz
TEXDIR="$HOME/texlive/2026"
./install-tl-*/install-tl --no-interaction --scheme=scheme-small --texdir="$TEXDIR"
export PATH="$TEXDIR/bin/universal-darwin:$PATH"
tlmgr install collection-langkorean latexmk xetex          # no sudo (user tree)
# persist PATH for future shells:
grep -q "$TEXDIR/bin" "$HOME/.zshrc" 2>/dev/null || \
  echo "export PATH=\"$TEXDIR/bin/universal-darwin:\$PATH\"" >> "$HOME/.zshrc"
```
Downloads a few hundred MB and takes a few minutes the first time. `collection-langkorean`
includes `kotex`, `cjk-ko`, `xetexko`, and Nanum fonts.

### B. Faster but needs your admin password — BasicTeX (~100 MB)
Note: the `--cask basictex` install itself triggers a macOS **admin password prompt** (the
`.pkg` installer needs it), and the `tlmgr` step needs `sudo` too — so this route is *not*
password-free (unlike Option A). Both must be run by the user:
```bash
! brew install --cask basictex          # prompts for your macOS password
eval "$(/usr/libexec/path_helper)"
! sudo tlmgr update --self
! sudo tlmgr install collection-langkorean latexmk xetex
```

### C. Full MacTeX, no GUI (~5 GB, needs sudo) — only if the user wants the whole distribution
```bash
brew install --cask mactex-no-gui
```
(Homebrew, needed only for B/C, is at https://brew.sh.)

After installing, re-run **Step 1** to confirm ✅, then **Step 3**. (For Option A you may need a
new shell or the `export PATH` line above for `tlmgr`/`xelatex` to be visible.)

### About fonts
This skill does **not** ship a font. The translator detects an installed Korean font at
runtime (`fc-list :lang=ko`):
- macOS already has `AppleMyungjo` / `Apple SD Gothic Neo` / `AppleGothic` built in — nothing to install.
- Worst case, `\usepackage{kotex}` renders Hangul with its built-in default, so the build never
  hard-fails for fonts alone.
- (TeX's `collection-langkorean` Nanum is **Type1** — not XeLaTeX-name-callable — which is why
  we rely on system fonts / the kotex default for XeLaTeX.)

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
