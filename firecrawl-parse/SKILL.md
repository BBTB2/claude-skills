---
name: firecrawl-parse
description: "Use this skill whenever the user is working with the Firecrawl API /parse endpoint. Triggers include: uploading local files (PDF, DOCX, XLSX, HTML) to Firecrawl, parsing documents with Firecrawl, extracting markdown or structured JSON from files using Firecrawl, choosing between /parse and /scrape, configuring PDF parsing modes (fast/auto/ocr), setting output formats, enabling zero data retention, or any question about Firecrawl document parsing. Also triggers when the user says 'parse this file with Firecrawl', 'use Firecrawl to extract', or 'Firecrawl /parse'."
---

# Firecrawl /parse Endpoint Skill

This skill governs how to assist the user with Firecrawl's `/parse` endpoint for uploading and converting local or non-public files into LLM-ready data.

Official docs: https://docs.firecrawl.dev/api-reference/endpoint/parse.md

---

## Core Facts

- **Endpoint:** `POST https://api.firecrawl.dev/v2/parse`
- **Auth:** Bearer token in `Authorization` header
- **Content-Type:** `multipart/form-data`
- **Max file size:** 50 MB per request
- **Engine:** Rust-based (up to 5× faster than alternatives)
- **Zero Data Retention:** Supported — requires opt-in via `zeroDataRetention: true` (contact help@firecrawl.dev to enable)

---

## Supported File Types

`.pdf`, `.docx`, `.doc`, `.odt`, `.rtf`, `.xlsx`, `.xls`, `.html`, `.htm`

---

## When to Use /parse vs /scrape

| Situation | Endpoint |
|---|---|
| Local file or non-public bytes | `POST /parse` |
| Public URL pointing to a document | `POST /scrape` |

Always clarify this with the user first — if they have a public URL, redirect them to `/scrape`.

---

## Request Structure

Two `multipart/form-data` fields:
1. `file` — the binary file bytes
2. `options` — JSON string with ParseOptions (optional)

### Python example:
```python
import httpx

with open("document.pdf", "rb") as f:
    response = httpx.post(
        "https://api.firecrawl.dev/v2/parse",
        headers={"Authorization": "Bearer YOUR_API_KEY"},
        files={"file": ("document.pdf", f, "application/pdf")},
        data={"options": '{"formats": [{"type": "markdown"}]}'}
    )
print(response.json())
```

### Node.js / fetch example:
```javascript
const form = new FormData();
form.append("file", fileBlob, "document.pdf");
form.append("options", JSON.stringify({
  formats: [{ type: "markdown" }],
  parsers: [{ type: "pdf", mode: "auto" }]
}));

const res = await fetch("https://api.firecrawl.dev/v2/parse", {
  method: "POST",
  headers: { "Authorization": "Bearer YOUR_API_KEY" },
  body: form
});
```

---

## Output Formats

Specify one or more in the `formats` array:

| Format | Description |
|---|---|
| `markdown` | Clean markdown (default) |
| `html` | Cleaned HTML (removes scripts, styles, nav) |
| `rawHtml` | Exact unmodified HTML |
| `links` | All links extracted from the document |
| `images` | Images extracted |
| `summary` | AI-generated summary |
| `json` | Structured JSON — requires a `schema` (JSON Schema) and optional `prompt` |

Default if omitted: `[{ "type": "markdown" }]`

### JSON extraction example:
```json
{
  "formats": [
    {
      "type": "json",
      "schema": {
        "type": "object",
        "properties": {
          "title": { "type": "string" },
          "date": { "type": "string" },
          "findings": { "type": "array", "items": { "type": "string" } }
        }
      },
      "prompt": "Extract the report title, date, and key findings."
    }
  ]
}
```

---

## PDF Parsing Modes

Set via `parsers` option:

| Mode | Behaviour |
|---|---|
| `fast` | Text-only extraction — fastest, no OCR |
| `auto` | Text-first, falls back to OCR if needed — **default** |
| `ocr` | OCR on every page — best for scanned documents |

```json
{
  "parsers": [{ "type": "pdf", "mode": "ocr", "maxPages": 50 }]
}
```

`maxPages`: integer, 1–10000. Limits pages parsed per request.

---

## Key ParseOptions Reference

| Option | Type | Default | Notes |
|---|---|---|---|
| `formats` | array | `[markdown]` | Output formats |
| `parsers` | array | `[pdf auto]` | Parser config per file type |
| `onlyMainContent` | boolean | `true` | Strips headers/footers/nav |
| `includeTags` | string[] | — | HTML tags to include |
| `excludeTags` | string[] | — | HTML tags to exclude |
| `removeBase64Images` | boolean | `true` | Replaces base64 images with alt text |
| `blockAds` | boolean | `true` | Blocks ads and cookie popups |
| `timeout` | integer | 30000 | Max ms (up to 300000) |
| `zeroDataRetention` | boolean | `false` | No data stored after response |
| `proxy` | string | — | `basic` or `auto` |

---

## Response Structure

```json
{
  "success": true,
  "data": {
    "markdown": "# Document Title\n...",
    "html": null,
    "rawHtml": null,
    "links": [],
    "images": [],
    "summary": null,
    "metadata": {
      "title": "...",
      "sourceURL": "...",
      "statusCode": 200,
      "contentType": "application/pdf"
    }
  }
}
```

---

## Error Codes

| Code | Meaning |
|---|---|
| 400 | Bad request — invalid multipart form |
| 402 | Payment required |
| 429 | Rate limit exceeded |
| 500 | Server error |

---

## Behaviour Guidelines

- Always confirm whether the user has a **local file** or **public URL** before suggesting `/parse` vs `/scrape`.
- When helping with PDF extraction, ask if the document is **text-based or scanned** to recommend the right mode (`auto` vs `ocr`).
- For structured data extraction, help the user draft a **JSON Schema** that matches what they want to pull out.
- When zero data retention matters (e.g. medical records, confidential reports), proactively remind the user to enable `zeroDataRetention: true`.
- Always show the **complete working code snippet** in the user's preferred language.
