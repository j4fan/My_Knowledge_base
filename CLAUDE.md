# Wiki Operations Manual

This file governs how to operate the LLM wiki at `Q:\src\wiki`.
**Read this file at the start of every wiki session before taking any action.**

---

## Layout

```
Q:\src\wiki\
├── CLAUDE.md          ← this file; read first every session
├── raw\               ← immutable user-supplied sources; NEVER edit
└── wiki\
    ├── index.md       ← master catalog; one row per wiki page
    ├── log.md         ← append-only event log
    ├── overview.md    ← high-level orientation page
    ├── sources\       ← one .md per raw source
    ├── entities\      ← named-thing pages (people, systems, products, orgs)
    └── concepts\      ← idea/pattern/technique pages
```

**Rules:**
- `raw\` is read-only. Never create, edit, or delete files there.
- All wiki output goes under `wiki\`.
- Use plain Markdown only. No Obsidian wikilinks. No YAML frontmatter except the four-field block defined below.
- All links use standard relative paths: `[text](../concepts/foo.md)`.

---

## Page Types

| Type | Location | Purpose |
|------|----------|---------|
| `source-summary` | `wiki/sources/<slug>.md` | Summarises one raw source document |
| `entity` | `wiki/entities/<slug>.md` | Named people, systems, products, orgs |
| `concept` | `wiki/concepts/<slug>.md` | Ideas, patterns, techniques, terms |
| `overview` | `wiki/overview.md` | Orientation: what this knowledge base covers |
| `index` | `wiki/index.md` | Catalog of every wiki page |
| `log` | `wiki/log.md` | Append-only event log |

**Slug format:** lowercase, hyphens only, no special characters. Example: `vector-search.md`, `john-smith.md`.

---

## Frontmatter

Every wiki page (all types except `index.md` and `log.md`) begins with this block:

```markdown
---
title: Human-readable page title
type: source-summary | entity | concept | overview
date: YYYY-MM-DD
sources: N
---
```

- `date`: date this page was created or last substantially updated.
- `sources`: number of raw source documents this page draws from. Set to `0` for derived pages.
- Do not add any other frontmatter fields.

---

## Link Conventions

- All links are relative to the current file's location.
- From `wiki/sources/foo.md` to `wiki/concepts/bar.md`: `[bar](../concepts/bar.md)`
- From `wiki/concepts/foo.md` to `wiki/entities/bar.md`: `[bar](../entities/bar.md)`
- From `wiki/overview.md` to `wiki/sources/foo.md`: `[foo](sources/foo.md)`
- Index and log use the same relative scheme from `wiki/`.
- Never use absolute paths. Never use `[[wikilink]]` syntax.

---

## Ingest Workflow

Run when the user says a new source is ready, or explicitly asks to ingest a file.

**Step 1 — Identify the source.**
List files in `raw\`. Find any that have no corresponding page in `wiki/sources/`. The slug for `raw/my-document.pdf` is `my-document`.

**Step 2 — Read and summarise.**
Read the raw source. Write `wiki/sources/<slug>.md` with:
- Frontmatter (type: source-summary, sources: 1)
- `## Summary` — 3–5 sentence abstract
- `## Key Points` — bulleted list of the most important facts, claims, or data
- `## Entities Mentioned` — list of named things with one-line descriptions
- `## Concepts Mentioned` — list of ideas/terms with one-line descriptions
- `## Cross-Links` — links to existing entity or concept pages this source informs

**Step 3 — Update or create entity pages.**
For each entity in "Entities Mentioned" that either has no page or gains new information:
- Create `wiki/entities/<slug>.md` (type: entity) or open the existing one.
- Add or update a `## Sources` section listing source pages that mention this entity.
- Add or update factual sections as needed. Do not delete existing content; extend it.

**Step 4 — Update or create concept pages.**
Same process for concepts. Each `wiki/concepts/<slug>.md` should include:
- `## Definition` — one-paragraph explanation
- `## Context` — where/how this concept appears in the knowledge base
- `## Related Concepts` — links to related concept pages
- `## Sources` — links to source pages that discuss this concept

**Step 5 — Update index.**
Add one row to the catalog table in `wiki/index.md` for each new page created. Do not duplicate existing rows; update the `Date` cell if a page was substantially changed.

**Step 6 — Update overview.**
If the new source introduces a major topic not yet covered in `wiki/overview.md`, extend the overview. Do not rewrite existing sections.

**Step 7 — Append to log.**
Prepend one log entry to `wiki/log.md` (format below).

---

## Query Workflow

Run when the user asks a question about knowledge base contents.

**Step 1 — Search the index.**
Read `wiki/index.md` to identify candidate pages by title and type.

**Step 2 — Read relevant pages.**
Read the identified pages. Follow cross-links to related pages if needed.

**Step 3 — Answer.**
Answer directly, citing pages with relative links: "See [vector-search](wiki/concepts/vector-search.md)."

**Step 4 — Optionally file the result.**
If the answer is substantive and reusable, ask the user if they want it saved as a wiki page. If yes, create the page, update the index, and prepend a log entry (type: query).

Do not save query results without user confirmation.

---

## Lint Workflow

Run when the user asks for a health check. Report violations; do not auto-fix without user confirmation.

1. **Orphan pages** — any page in `wiki/sources/`, `wiki/entities/`, or `wiki/concepts/` not listed in `wiki/index.md`.
2. **Missing frontmatter** — any wiki page (except `index.md` and `log.md`) missing the four required frontmatter fields.
3. **Broken links** — any relative link whose target file does not exist.
4. **Un-ingested sources** — any file in `raw\` with no corresponding page in `wiki/sources/`.
5. **Stale overview** — `wiki/overview.md` last updated more than 30 days before the most recent log entry.
6. **Empty pages** — pages with frontmatter but no body content beyond section headers.

Report as a checklist, naming the file and problem. Offer to fix individually.

---

## Log Entry Format

Entries are **prepended** (newest first) under the `## Entries` heading in `wiki/log.md`.

**Ingest:**
```markdown
## [YYYY-MM-DD] ingest | <source-slug>

- Source: [<slug>](sources/<slug>.md)
- Pages created: <comma-separated list of new page links>
- Pages updated: <comma-separated list of updated page links>
- Notes: <one sentence or "none">
```

**Query:**
```markdown
## [YYYY-MM-DD] query | <topic>

- Question: <verbatim user question, truncated to 100 chars>
- Pages read: <comma-separated list>
- Result filed: [yes: <link>] | no
```

**Lint:**
```markdown
## [YYYY-MM-DD] lint | health-check

- Violations found: <N>
- Summary: <one sentence>
```

Rules:
- Never delete or edit existing log entries.
- Always prepend; do not append to the bottom.
- Use `##` (h2) so each entry is individually linkable.

---

## Index Row Format

`wiki/index.md` contains one catalog table with columns: `Title | Type | Path | Date | Sources`

```markdown
| [Vector Search](concepts/vector-search.md) | concept | concepts/vector-search.md | 2026-04-14 | 3 |
```

Rules:
- `Title` cell contains the linked page title.
- `Path` is relative to `wiki/` (no leading `wiki/`).
- Sort rows: sources first, then entities, then concepts, then overview.
- Do not include `index.md` or `log.md` as rows.
- When a page is updated, change the `Date` cell and increment `Sources` if new sources were added.
