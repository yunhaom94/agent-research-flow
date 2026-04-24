---
name: literature-search
description: Use when the user asks to find related work, search literature, survey prior art, or check whether someone has already done X for a research topic or problem statement.
---

# Literature Search

Find papers relevant to a research topic and save the selected ones under `{project_root}/Notes/Literatures/`.

**Inputs from the caller:**
- `topic_summary` — description of the research area
- `problem_statement` — the specific research question

**Project layout:** The plugin's `SessionStart` hook injects the project structure in a `<project-structure>` block; treat it as authoritative.

## Workflow

1. **Search** — one `search_papers.py` call with 2-4 queries.
2. **Filter in-context** — keep the relevant ones by title + abstract.
3. **Parse PDFs** — dispatch `parse-paper` subagents in parallel.
4. **Deep relevance check** — dispatch reader subagents in parallel.
5. **Confirm with user** — summarize and ask which to save.
6. **Store** — move selected markdowns to `Notes/Literatures/` and append entries to `Literatures.md`.

## Temp files

- All intermediate files (PDFs, parsed markdown) go in `{project_root}/.tmp/`. Create it at start.
- Search output stays in context — do not save stdout to disk, do not shell out with `jq` or `python -c` to parse it.
- Delete `.tmp/` at the end. If deletion is blocked, tell the user to clean it up.

## 1. Search

Extract 3-5 technical keywords from the topic and problem, form 2-4 queries that an academic search engine would understand, then run:

```bash
python "<skill-dir>/scripts/search_papers.py" \
  --queries "query one" "query two" "query three" \
  --max-results 50
```

The script searches Semantic Scholar + arXiv, deduplicates, and prints a **compact digest** to stdout (one block per paper: index, year, source, PDF availability, arXiv/DOI, title, venue, full abstract). Read it directly. Progress goes to stderr. `S2_API_KEY` is already in the environment.

Pass `--format json` only when a later step needs full records. Do not redirect to a file.

An optional `scripts/search_scholar.py` supplements Google Scholar via the `scholarly` package — use it only if results are thin.

**Query example:** Topic "efficient vector similarity search" + problem "approximate nearest neighbor for high-dimensional data" → `"approximate nearest neighbor" high-dimensional`, `vector similarity search efficient`, `ANN index high-dimensional`.

## 2. Title + Abstract Filtering

Classify each paper in-context:
- **highly relevant** — directly addresses the problem or a core aspect of the topic
- **somewhat relevant** — related methodology, adjacent problem, or useful background
- **not relevant** — different domain or tangential

Keep the first two classes. Target 10-20 papers. If more, tighten criteria; if fewer than 5, broaden queries and re-run.

If the digest is too long to keep in context, re-run with tighter queries rather than caching output.

## 3. Parse PDFs

Dispatch one `parse-paper` subagent per kept paper, in parallel (single message, multiple tool calls). For each, pass the paper's title, DOI, arXiv ID, and/or `pdf_url`, plus an output path under `{project_root}/.tmp/<unique>.md`.

Skip any paper where `parse-paper` returns `not_found` or `download_failed`, and note it in the Phase 5 summary.

## 4. Full-Text Relevance Check

For each parsed markdown, dispatch a reader subagent in parallel. Each returns a classification (highly / somewhat / not relevant) plus a 1-2 sentence relevance note.

## 5. Confirm with User

Summarize the classified results and ask the user which papers to save.

## 6. Store

**6a. Ensure `Literatures.md` exists.** Create `{project_root}/Notes/Literatures/` if missing. If `Literatures.md` does not exist, copy it from `<plugin_root>/references/Literatures.md` (plugin root = two levels above this SKILL.md). The template defines the per-entry format.

**6b. Deduplicate.** Read the existing `Literatures.md` and skip any paper whose title already appears.

**6c. Move and append.** For each selected paper, move its markdown from `.tmp/` to `Notes/Literatures/<sanitized_title>.md`, then append a summary entry to `Literatures.md` using the template format.

**6d. Cleanup.** Delete `{project_root}/.tmp/`.

## Output

Return a short success message with the number of papers saved.

## Notes

- **Filename sanitization:** Replace `:` `/` `\` `?` `*` `"` `<` `>` `|` with underscores. Spaces are allowed — be consistent across a run. Truncate to ~100 chars.
- **Manual dedup:** The script dedups by DOI + title similarity. If you spot an arXiv preprint and a published version of the same work, keep the published one.
- **Partial results are fine:** If one API source fails, continue with the others.
- **Direct Semantic Scholar calls:** If you curl `api.semanticscholar.org`, include `-H "x-api-key: $S2_API_KEY"` for higher rate limits.
