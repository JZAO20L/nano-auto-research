---
name: auto-research-s1-github-search
description: Search GitHub for open-source baseline repositories using the GitHub API. Evaluates repo quality and produces entries for baselines.md.
metadata:
  version: "0.1"
---

# S1 GitHub Search

Find and evaluate open-source repositories for baseline methods.

## 1. Search via GitHub API

```bash
curl -s "https://api.github.com/search/repositories?q=KEYWORD+language:python&sort=stars&order=desc&per_page=10"
```

Example:
```bash
curl -s "https://api.github.com/search/repositories?q=LLM+jailbreak+language:python&sort=stars&order=desc&per_page=10"
```

For multi-word queries, use `+` between terms. Add qualifiers:
- `language:python` — filter by language
- `topic:llm` — filter by topic tag
- `in:readme` — search in README content

## 2. Response Fields to Check

From each result item, extract:
- `full_name` — repo identifier (e.g., "patrickrchao/JailbreakingLLMs")
- `stargazers_count` — popularity signal
- `updated_at` — last activity date
- `description` — one-line summary
- `topics` — tag list
- `html_url` — repo URL
- `license.spdx_id` — license type

## 3. Evaluation Criteria

A repo is **usable** if it meets ALL:
- Stars > 50 (relax to > 10 for niche research areas where few implementations exist)
- Updated within the last 1 year
- Has a README with usage instructions
- Contains training/evaluation scripts (not just a notebook demo)
- License permits research use (MIT, Apache-2.0, BSD, or explicit research license)

Additional check:
- README links to a paper → verify it matches the baseline paper we're looking for
- If no paper link, check if the repo name/description clearly corresponds

## 4. Reproduction Difficulty Classification

| Level | Meaning |
|-------|---------|
| `ready` | Has runnable scripts, requirements.txt, clear instructions |
| `needs-adaptation` | Code exists but needs modification for our setup |
| `code-only` | Code present but no instructions, unclear entry point |
| `paper-only` | No usable code found; must implement from paper |

## 5. Output Format

Append to `docs/baselines.md`:

```markdown
| Method | Paper | Repo | Stars | Last Update | License | Reproduction | Status |
|--------|-------|------|-------|-------------|---------|--------------|--------|
| GCG | arxiv:2307.15043 | [llm-attacks/llm-attacks](https://github.com/llm-attacks/llm-attacks) | 812 | 2025-03 | MIT | ready | pending |
```

## 6. Rate Limits

- **Unauthenticated**: 10 search requests/minute, 60 API calls/hour
- **With token** (`-H "Authorization: Bearer {TOKEN}"`): 30 search requests/minute, 5000 calls/hour

Space requests with `sleep 6` between searches if unauthenticated.

## 7. Fallback

If the API returns 403 (rate limited) or fails:

1. Wait 60 seconds and retry (rate limit resets per-minute for search).
2. Try the GitHub API search endpoint directly:
```bash
curl -s --max-time 20 "https://api.github.com/search/repositories?q=KEYWORD&sort=stars&per_page=10"
```

**Note**: `web_fetch` is unreliable in this environment (safety classifier blocks, proxy timeouts). Always prefer the GitHub API via `curl`.

## 8. Batch Search Strategy

For a baseline method, try multiple keyword variants:
1. Method acronym (e.g., "GCG attack")
2. First author + topic (e.g., "Zou adversarial suffix LLM")
3. Paper title keywords (e.g., "universal transferable adversarial attacks")

Stop after finding one qualifying repo per method.
