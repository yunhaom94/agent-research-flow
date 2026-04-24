---
name: parse-paper
description: >
  Use when the caller needs the full text of a single academic paper as a Markdown file — given
  a paper identifier (URL, DOI, arXiv ID, Semantic Scholar ID, title, or local PDF path) and a
  destination path, this skill locates the PDF, downloads it, and converts it to Markdown.
  Does NOT resolve bibliographic metadata and does NOT modify any literature index — callers
  handle those concerns themselves.
---

# Parse Paper

Locate a paper's PDF, download it, and convert it to Markdown at a specified output path. Three steps — no metadata extraction, no index updates.

## Inputs

The caller provides:
- **paper**: identifier for the paper — URL, DOI, arXiv ID, Semantic Scholar URL/ID, title, or a local PDF file path
- **output_md**: absolute path where the resulting `.md` file should be written
- **skill_dir**: path to this skill's directory (for referencing `scripts/download_pdf.py`)

The `S2_API_KEY` environment variable is available; include it as `-H "x-api-key: $S2_API_KEY"` on Semantic Scholar calls.

## Step 1: Find a PDF Source

Produce a list of candidate PDF URLs (or a local PDF path) to try. If the input is already a local PDF, skip to Step 3 using that path.

Try sources in this order based on the input type; collect every URL that looks viable so Step 2 has fallbacks:

- **Direct PDF URL** (ends in `.pdf` or Content-Type is `application/pdf`): use as-is.
- **arXiv ID or URL** (`2301.00001`, `arxiv.org/abs/...`): `https://arxiv.org/pdf/<arxiv_id>.pdf`.
- **DOI**:
  1. `curl -s -H "x-api-key: $S2_API_KEY" "https://api.semanticscholar.org/graph/v1/paper/DOI:<doi>?fields=openAccessPdf"` → `openAccessPdf.url`
  2. `curl -s "https://api.unpaywall.org/v2/<doi>?email=parse-paper@example.com"` → `best_oa_location.url_for_pdf`
- **Semantic Scholar URL/ID**: `curl -s -H "x-api-key: $S2_API_KEY" "https://api.semanticscholar.org/graph/v1/paper/<paper_id>?fields=openAccessPdf,externalIds"` → `openAccessPdf.url`, and if `externalIds.ArXiv` or `externalIds.DOI` are present, derive those candidates too.
- **Title only**: `curl -s -H "x-api-key: $S2_API_KEY" "https://api.semanticscholar.org/graph/v1/paper/search?query=<url_encoded_title>&limit=3&fields=title,authors,year,openAccessPdf,externalIds"`.
  - If the top hit's title matches the user's title closely (normalized case/whitespace), trust it.
  - If two or more hits have effectively the same title but are clearly different papers (different authors/year), **stop and ask the caller which one they mean** — do not guess.
  - On the chosen hit, take `openAccessPdf.url` if present, and also derive candidates from `externalIds.ArXiv` (→ `https://arxiv.org/pdf/<id>.pdf`) and `externalIds.DOI` (→ run the DOI path above). Use every candidate you can derive so Step 2 has fallbacks.
  - If no hit matches closely, use WebFetch/WebSearch to find an official PDF link (publisher page, author homepage, arXiv).

**Note on deriving candidates from `externalIds`:** any response that includes `externalIds` (Semantic Scholar ID path *or* title-search path) should expand into arXiv/DOI candidates the same way. Multiple PDF sources for the same paper is fine — just collect them all; Step 2 picks whichever works first.

**STOP condition:** If no source produces any candidate PDF URL — and the input isn't a local PDF — stop and report to the caller that the paper could not be found. Do not fabricate a URL. Do not proceed to Step 3 without a PDF.

## Step 2: Download the PDF

For each candidate URL from Step 1, attempt:

```bash
python3 "<skill_dir>/scripts/download_pdf.py" \
  --url "<candidate_url>" \
  --output "<project_root>/.tmp/<unique_name>.pdf"
```

`<project_root>` is the root of the project the caller is operating in (i.e. the user's current working project, not the skill directory). If the caller didn't supply one, use any scratch location such as `/tmp/parse-paper/<unique_name>.pdf`.

`<unique_name>` must be unique per invocation — use the arXiv ID, a sanitized DOI, or a short sanitized form of the title. Never hard-code `paper.pdf`, because parallel calls (e.g. from literature-search) will overwrite each other.

The script validates that the response is actually a PDF (checks `%PDF-` magic bytes) and retries transient network errors. A non-zero exit means that specific URL failed.

**If a candidate fails, try the next candidate.** If every candidate from Step 1 has been exhausted without a successful download, go back to Step 1 and look harder using the WebFetch and WebSearch tools — try a different query formulation, try an author-hosted copy, try a mirror. Only give up and report failure to the caller after those tools genuinely turn up nothing.

Create the temp directory (`mkdir -p`) before calling the script. Clean up the downloaded PDF in Step 3 after conversion.

## Step 3: Convert to Markdown

Invoke the `markitdown` skill (via the Skill tool) to convert the downloaded (or user-provided local) PDF to Markdown, writing the result to the `output_md` path the caller specified. Pass the input PDF path and the `output_md` destination as arguments — let the markitdown skill decide the concrete invocation (CLI vs. Python API) and handle availability checks.

After a successful conversion:
- Delete the downloaded PDF from the temp location. Leave user-provided local PDFs untouched.
- Verify the output `.md` file exists at `output_md` and is non-empty before returning.

## Output

Return a short status to the caller:

```
- **status**: ok | not_found | download_failed
- **output_md**: <path>   (only when status is ok)
- **source_url**: <url>   (the candidate that succeeded, or "local" for user-provided PDFs)
```

On `not_found` or `download_failed`, include a one-line reason so the caller knows which stage failed.

