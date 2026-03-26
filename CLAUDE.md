# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A Chinese-language technical notes site built with MkDocs Material. Content covers Linux system programming, distributed systems, and CS fundamentals. Articles are deep-dive technical documents with ASCII diagrams, code examples, tables, and admonitions.

## Commands

```bash
pip install -r requirements.txt   # Install dependencies (just mkdocs-material)
mkdocs serve                      # Local dev server at http://localhost:8000
mkdocs build                      # Build static site to site/
```

Pushing to `main` triggers GitHub Actions to auto-deploy to GitHub Pages via `mkdocs gh-deploy --force`.

## Writing Articles

- All content is in `docs/` as Markdown files, organized by topic subdirectory (`linux/`, `fundamentals/`)
- After creating a new article, update **both** `mkdocs.yml` (add to `nav:`) and `docs/index.md` (add a link with short description to the appropriate category)
- Articles are written in Chinese (Simplified)
- Available markdown extensions: admonitions (`!!! note/warning/tip`), code highlighting with line numbers, tabbed content, collapsible details, superfences, tables, attribute lists
- Article style: deep technical detail with ASCII diagrams, code examples in C/C++/Python/Java, comparison tables, and practical application sections
- Typical structure: concept introduction → internal mechanism → implementation details → practical applications
- After finishing an article, directly `git commit` and `git push` — GitHub Actions will auto-deploy
