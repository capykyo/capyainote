# AINotes — GitHub Repo Reader

## Project Purpose
A personal knowledge base for reading and annotating GitHub repositories.
Each repo gets its own folder. All notes are in Markdown format.

## Structure
```
ainote/
├── CLAUDE.md           # This file — project instructions for Claude
├── README.md           # Reading list (index of all tracked repos)
└── <owner>-<repo>/     # One folder per GitHub repo
    ├── overview.md     # Summary, purpose, key concepts
    ├── architecture.md # Code structure, modules, design patterns
    └── notes.md        # Personal reading notes, questions, highlights
```

## Conventions
- Folder naming: `<github-owner>-<repo-name>` (lowercase, hyphens)
- `README.md` at root is the reading list — links to each repo folder
- All content in `.md` files only
- Each repo folder should have at least `overview.md`

## Workflow
1. Add a new repo: create `<owner>-<repo>/overview.md`, add entry to `README.md`
2. Take notes in `notes.md` inside the repo folder
3. Document architecture findings in `architecture.md`

## Claude Guidance
- When adding a new repo, always update `README.md` with a link
- Keep notes factual and tied to the actual source code
- Use headings, code blocks, and links to specific files/lines when possible
- Do not create files outside this structure without asking
