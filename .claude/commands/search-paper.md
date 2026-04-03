---
description: "Search arXiv papers by keyword. Usage: /search-paper <query> [--category cs.CV] [--limit 10]"
---

# Search arXiv Papers

Search for recent papers on arXiv related to the user's query.

## Step 1: Search

Run a Python one-liner to search arXiv. Parse `$ARGUMENTS` to extract the query and optional flags.

Default category: `cs.CV`, default limit: `10`.

```bash
python -c "
import arxiv, sys
query = '''$ARGUMENTS'''.strip()
# Parse flags
category = 'cs.CV'
limit = 10
parts = query.split()
clean_parts = []
i = 0
while i < len(parts):
    if parts[i] == '--category' and i+1 < len(parts):
        category = parts[i+1]; i += 2
    elif parts[i] == '--limit' and i+1 < len(parts):
        limit = int(parts[i+1]); i += 2
    else:
        clean_parts.append(parts[i]); i += 1
q = ' '.join(clean_parts)
search_query = f'cat:{category} AND all:{q}'
search = arxiv.Search(query=search_query, max_results=limit, sort_by=arxiv.SortCriterion.SubmittedDate)
for r in search.results():
    aid = r.entry_id.rsplit('/', 1)[-1] if 'arxiv.org/abs/' in r.entry_id else r.entry_id
    authors = ', '.join(a.name for a in r.authors[:3])
    if len(r.authors) > 3: authors += ' et al.'
    date = r.published.strftime('%Y-%m-%d') if r.published else '?'
    print(f'{aid} | {date} | {authors} | {r.title.strip()[:80]}')
"
```

## Step 2: Present results

Format the search results as a clean table for the user. Include arXiv ID, date, authors, and title.

If the user wants to read a specific paper, suggest using `/read-paper <arxiv_id>`.
