# Seminar LaTeX materials — AI & AI Agents (18 topics)

Each topic has TWO files:
- `abstract_NN_key.tex`  — one-page seminar abstract (article class)
- `slides_NN_key.tex`    — Beamer presentation (navy CET theme, TikZ diagrams)

## Before you compile
1. Open **`seminar_details.tex`** and fill in your name, roll number, class roll, and guide. Every file reads from it.
2. (Optional) drop a `cet_logo.png` in this folder to show the college crest on every title slide.

## How to compile
**Overleaf (easiest):** upload this whole folder as a project, open any `.tex`, and click Recompile.
**Local (needs a LaTeX install):** run `pdflatex` twice on a file (twice so the table of contents / slide numbers settle), e.g.:
```
pdflatex slides_01_deeprag.tex
pdflatex slides_01_deeprag.tex
```

## Notes
- Decks use the Beamer *Madrid* theme recolored navy, Palatino (`mathpazo`) font, and the 15-section structure from your sample.
- Diagrams are original TikZ (no copyrighted figures).
- `lint_report.txt` is an automated brace/environment smoke check — review any WARN entries.

## Images
The slide decks now embed diagrams from the `images/` folder via \includegraphics. Keep the `images/` folder next to the `.tex` files (Overleaf: upload the whole folder). Each diagram is a PNG named `<topic>_<n>.png`.
