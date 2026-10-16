# CLAUDE.md

Guidance for Claude Code (and humans) working in this repo.

## What this is

upppdf ("U triple P D F") is a Python CLI and library that removes password protection from PDF
files the user is authorised to access, writing an unlocked copy and leaving the original
untouched. It tries several backends in turn (a remembered password, a small known-password list,
the `pdfcrack` and `qpdf` command-line tools when present, then the PyMuPDF, pikepdf and PyPDF2
libraries), detects the encryption revision (R=1/2/3) up front to report whether a bypass is even
possible, verifies each result opens with real content, and caches any discovered password in a
local `password_memory.json`. Published to PyPI as `upppdf` and via a Homebrew tap. The project
documents that it cannot defeat strong encryption (R=3) and is intended for documents the user has
permission to unlock.

## Commands

```bash
# Set up a dev environment (either works)
./setup.sh                 # creates ./venv and installs requirements.txt (setup.bat on Windows)
python3 -m venv venv && source venv/bin/activate && pip install -r requirements.txt
pip install -e .           # editable install, exposes the `upppdf` command

# Run
upppdf                        # unlock every PDF in the current directory (default)
upppdf -f document.pdf        # unlock one file
upppdf -i ./in -o ./out       # custom input/output directories
upppdf --output-name out.pdf  # choose the output filename
upppdf --show-passwords       # print remembered passwords
upppdf --help

# Optional external tools (improve success rate; detected at runtime, not required)
brew install qpdf pdfcrack          # macOS
sudo apt-get install qpdf pdfcrack  # Debian/Ubuntu

# Test (unittest; pytest is NOT a dependency)
python -m unittest test_upppdf       # 3 tests; needs at least one PDF lib installed
```

Verified here: in a throwaway venv with `requirements.txt` installed, `python -m unittest
test_upppdf` passes all 3 tests on Python 3.13.

## Layout

```
upppdf/
  __init__.py          re-exports PDFUnlocker, PasswordMemory, main, __version__
  pdf_unlocker.py       all logic: PasswordMemory + PDFUnlocker (one unlock_* method per backend), main()
  version.py            __version__ / __version_info__ (currently 1.0.9)
test_upppdf.py          unittest suite (imports, PasswordMemory, PDFUnlocker construction)
requirements.txt        runtime deps: PyPDF2, pikepdf, pymupdf
pyproject.toml          primary packaging (setuptools backend, version 1.0.9)
setup.py                legacy packaging, kept in parallel (also version 1.0.9)
MANIFEST.in             sdist includes (README, LICENSE, requirements, QUICKSTART, *.py, the *.txt placeholders)
setup.sh / setup.bat    venv bootstrap for macOS-Linux / Windows
PDFs/, Unlocked_PDFs/   default-ish working folders; only their placeholder .txt files are tracked
homebrew-tap/           in-repo mirror of the Homebrew formula (Formula/upppdf.rb) + tap docs
HOMEBREW_SETUP.md, QUICKSTART.md   extra docs
.github/workflows/      test.yml, publish.yml, update-homebrew.yml
```

## Releasing

All release automation is tag-driven on `v*`.

- `.github/workflows/publish.yml` (tag `v*` or manual dispatch): `python -m build`, `twine check`,
  `twine upload` to PyPI (secret `PYPI_API_TOKEN`), then `gh release create` with the wheel and
  sdist (secret `PAT_TOKEN`).
- `.github/workflows/update-homebrew.yml` (on a published release, or manual with a version input):
  downloads the release source tarball, computes its SHA256, `sed`s the new `url` and `sha256` into
  `homebrew-tap/Formula/upppdf.rb`, and commits/pushes to `master` (secret `PAT_TOKEN`). So the
  in-repo formula is refreshed automatically after each release.
- `.github/workflows/test.yml` (push/PR to main or master): matrix Python 3.8/3.11/3.13; it does
  import checks, `pip install -e .`, and `upppdf --help`. It does **not** run `test_upppdf.py`.

Before tagging, bump the version in all four places that carry it (see gotchas). The Homebrew tap
itself lives in a separate repo (`abozaralizadeh/homebrew-brew`, installed via
`brew tap abozaralizadeh/brew`); `homebrew-tap/` here is a reference copy — HOMEBREW_SETUP.md
explains the mirroring.

## Conventions / gotchas

- **Version is duplicated in four places:** `pyproject.toml`, `setup.py`, `upppdf/version.py`, and
  the `url` tag + `sha256` in `homebrew-tap/Formula/upppdf.rb`. Keep the first three in sync by hand;
  the formula url/sha are updated by the release workflow. (The module docstring in
  `pdf_unlocker.py` says "v1.0.0" — cosmetic and stale; `version.py` is authoritative.)
- **Two packaging configs coexist** (`pyproject.toml` and `setup.py`). `python -m build` uses the
  pyproject/setuptools backend. `setup.py` also lists `py_modules=["pdf_unlocker"]`, which is stale
  (the module is inside the `upppdf/` package, not at the top level); the package is found via
  `find_packages()` / `[tool.setuptools.packages.find]`.
- **Default input/output directories are the current directory (`.`)**, per the argparse defaults in
  `main()` — not `PDFs/` and `Unlocked_PDFs/` as an older version of the README claimed. Unlocked
  files get an `unlocked_` prefix unless `--output-name` is given.
- **External tools are optional.** `qpdf` and `pdfcrack` are probed with `--version` at startup; if
  absent the tool silently falls back to the Python libraries. At least one of pikepdf / PyMuPDF /
  PyPDF2 must be importable or the module exits immediately with an install hint.
- **Password memory:** `password_memory.json` in the current directory, keyed by the MD5 of the PDF
  bytes. It is git-ignored (along with `*.pdf`), so don't commit it. `--show-passwords` prints it.
- `.gitignore` keeps `PDFs/*.txt` and `Unlocked_PDFs/*.txt` (the placeholders) but ignores every
  `*.pdf`, so sample PDFs never get committed.
- PyMuPDF is imported as `import fitz`; recent versions warn that `fitz` is deprecated in favour of
  `import pymupdf`. Harmless today, but worth migrating if touching the imports.
- Code uses print-based progress with emoji throughout; there is no logging framework.
