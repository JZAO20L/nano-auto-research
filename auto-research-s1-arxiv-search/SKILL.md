---
name: arxiv-search
description: Search and retrieve paper metadata from arxiv via curl API access
source: auto-skill
extracted_at: '2026-07-21'
---

## When to use

When needing to search academic papers on arxiv. `web_fetch` is unreliable in this environment (safety classifier blocks, proxy timeouts, internal model errors), use `curl` to access the arxiv Atom API directly.

## API Endpoint

```
https://export.arxiv.org/api/query
```

## Query Parameters

| Parameter | Description | Example |
|---|---|---|
| `search_query` | Search expression with field prefixes and boolean operators | `all:reinforcement+AND+all:jailbreak` |
| `id_list` | Comma-separated arxiv IDs for exact lookup | `2606.09701,2605.19485` |
| `start` | Result offset for pagination (0-indexed) | `0` |
| `max_results` | Number of results per page (max 100) | `10` |
| `sortBy` | Sort field: `submittedDate`, `lastUpdatedDate`, `relevance` | `submittedDate` |
| `sortOrder` | `ascending` or `descending` | `descending` |

### Field Prefixes for search_query

- `all:` — search all fields
- `ti:` — title
- `au:` — author
- `abs:` — abstract
- `cat:` — category (e.g., `cat:cs.CL`)

### Boolean Operators

- `AND`, `OR`, `ANDNOT`
- Parentheses for grouping: `(all:RL+OR+all:reinforcement)+AND+all:jailbreak`
- URL-encode spaces as `+`, parentheses as `%28` `%29`

## Usage Examples

### Keyword search (sorted by date)

```bash
curl -s --max-time 20 \
  "https://export.arxiv.org/api/query?search_query=all:reinforcement+AND+all:red-teaming+AND+all:language+model&start=0&max_results=10&sortBy=submittedDate&sortOrder=descending"
```

### Exact paper lookup by ID

```bash
curl -s --max-time 20 \
  "https://export.arxiv.org/api/query?id_list=2606.09701,2605.19485"
```

### Title-specific search

```bash
curl -s --max-time 20 \
  "https://export.arxiv.org/api/query?search_query=ti:GRPO+AND+cat:cs.CL&start=0&max_results=5&sortBy=submittedDate&sortOrder=descending"
```

### Filter output for quick scanning

```bash
curl -s --max-time 20 \
  "https://export.arxiv.org/api/query?search_query=all:KEYWORDS&start=0&max_results=10&sortBy=submittedDate&sortOrder=descending" \
  | grep -E "<title>|<id>http|<published>|<summary>"
```

## Response Fields (per entry)

| Field | Content |
|---|---|
| `<id>` | arxiv URL (e.g., `http://arxiv.org/abs/2606.09701v1`) |
| `<title>` | Paper title |
| `<summary>` | **Full abstract** |
| `<published>` | First submission date |
| `<updated>` | Latest revision date |
| `<author><name>` | All authors |
| `<category term>` | Subject categories (cs.CL, cs.AI, cs.LG, etc.) |
| `<arxiv:comment>` | Page/figure count, venue acceptance (e.g., "Accepted to ICML 2026") |
| `<arxiv:primary_category>` | Primary subject area |
| `<link href>` | Abstract page URL + PDF URL |

## Full Text Access

arxiv provides HTML versions of most papers (2023+). Use this instead of PDF since no PDF parsing library is installed.

### Download and read full text

```bash
# Download HTML version
curl -s --max-time 30 -o /tmp/paper.html "https://arxiv.org/html/2606.09701v1"

# Extract text with Python
python3 -c "
from html.parser import HTMLParser
import re

class TextExtractor(HTMLParser):
    def __init__(self):
        super().__init__()
        self.text = []
        self.skip = False
    def handle_starttag(self, tag, attrs):
        if tag in ('script', 'style', 'nav', 'header', 'footer'):
            self.skip = True
    def handle_endtag(self, tag):
        if tag in ('script', 'style', 'nav', 'header', 'footer'):
            self.skip = False
    def handle_data(self, data):
        if not self.skip:
            self.text.append(data)

with open('/tmp/paper.html', 'r') as f:
    html = f.read()

parser = TextExtractor()
parser.feed(html)
text = ' '.join(parser.text)
text = re.sub(r'\s+', ' ', text)
print(text[:5000])  # adjust slice as needed
"
```

### URL patterns

| Content | URL |
|---|---|
| Abstract page | `https://arxiv.org/abs/{id}` |
| PDF | `https://arxiv.org/pdf/{id}` |
| **HTML full text** | `https://arxiv.org/html/{id}` |

Note: `{id}` includes version, e.g., `2606.09701v1`. HTML may not exist for older papers — fall back to abstract-only in that case.

## Limitations

- **No PDF parsing**: poppler-utils / pymupdf / pdfminer not installed. Use HTML version instead.
- **Rate limiting**: arxiv API may return 429 if called too frequently. Add `sleep 3` between consecutive queries.
- **No citation graph**: Cannot get references/citations. Use Semantic Scholar API for that (if accessible).
- **Query parsing**: arxiv treats `language model` as `language OR model` unless quoted. Use `%22language+model%22` for exact phrase.
- **HTML availability**: Not all papers have HTML versions (mostly 2023+). Check HTTP status code before parsing.

## Workflow

1. **Search**: curl API with keywords → get titles + abstracts
2. **Filter**: Identify relevant papers from abstracts
3. **Detail**: Query by `id_list` for specific papers if needed
4. **Full text**: `curl -o /tmp/paper.html "https://arxiv.org/html/{id}"` → Python extract text → read/analyze
5. **Batch**: For multiple searches, chain with `sleep 3` between calls to avoid 429
