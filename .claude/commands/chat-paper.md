---
description: "Interactive Q&A with a paper. Usage: /chat-paper <arxiv_id_or_url>"
---

# Interactive Paper Q&A

You are a senior researcher specializing in visual generation (Diffusion Models, Flow Matching, video/image generation). The user wants to ask questions about a specific paper.

## Step 1: Load the paper

First, download and extract the full text:

```bash
pip install -q arxiv httpx PyMuPDF 2>/dev/null
cat << 'PYEOF' > /tmp/_read_arxiv.py
#!/usr/bin/env python3
"""Standalone arXiv paper reader tool."""
from __future__ import annotations
import argparse, json, re, sys
from pathlib import Path

try:
    import arxiv
except ImportError:
    arxiv = None
try:
    import httpx
except ImportError:
    httpx = None
try:
    import fitz
except ImportError:
    fitz = None

DEFAULT_OUTPUT_DIR = Path("data/papers")

def extract_arxiv_id(input_str: str) -> str | None:
    input_str = input_str.strip().rstrip("/")
    match = re.search(r"(\d{4}\.\d{4,5}(?:v\d+)?)", input_str)
    if match:
        return match.group(1)
    match = re.search(r"([a-z\-]+/\d{7})", input_str)
    if match:
        return match.group(1)
    return None

def fetch_metadata(arxiv_id: str) -> dict:
    if arxiv is None:
        return {"error": "arxiv package not installed. Run: pip install arxiv"}
    search = arxiv.Search(id_list=[arxiv_id])
    results = list(search.results())
    if not results:
        return {"error": f"Paper not found: {arxiv_id}"}
    r = results[0]
    aid = arxiv_id
    if "arxiv.org/abs/" in r.entry_id:
        aid = r.entry_id.rsplit("/", 1)[-1]
    return {
        "arxiv_id": aid,
        "title": r.title.strip().replace("\n", " "),
        "authors": [a.name for a in r.authors],
        "abstract": r.summary.strip(),
        "url": r.entry_id,
        "pdf_url": r.pdf_url,
        "published": r.published.strftime("%Y-%m-%d") if r.published else None,
        "categories": list(r.categories),
    }

def download_pdf(arxiv_id: str, output_dir: Path) -> Path:
    if httpx is None:
        raise RuntimeError("httpx not installed. Run: pip install httpx")
    output_dir.mkdir(parents=True, exist_ok=True)
    pdf_path = output_dir / f"{arxiv_id.replace('/', '_')}.pdf"
    if pdf_path.exists():
        print(f"[INFO] PDF already exists: {pdf_path}", file=sys.stderr)
        return pdf_path
    url = f"https://arxiv.org/pdf/{arxiv_id}.pdf"
    print(f"[INFO] Downloading {url} ...", file=sys.stderr)
    with httpx.Client(timeout=60, follow_redirects=True) as client:
        resp = client.get(url)
        resp.raise_for_status()
        pdf_path.write_bytes(resp.content)
    print(f"[INFO] Saved to {pdf_path}", file=sys.stderr)
    return pdf_path

def extract_text(pdf_path: Path) -> str:
    if fitz is None:
        raise RuntimeError("PyMuPDF not installed. Run: pip install PyMuPDF")
    doc = fitz.open(pdf_path)
    pages = []
    for page in doc:
        pages.append(page.get_text("text"))
    doc.close()
    return "\n".join(pages)

def main():
    parser = argparse.ArgumentParser(description="arXiv paper reader for AI assistants")
    parser.add_argument("paper", help="arXiv ID, URL, or local PDF path")
    parser.add_argument("--output", "-o", default=str(DEFAULT_OUTPUT_DIR),
                        help="Output directory for downloaded PDFs")
    args = parser.parse_args()
    output_dir = Path(args.output)
    input_str = args.paper
    local_path = Path(input_str)
    is_local = local_path.exists() and local_path.suffix.lower() == ".pdf"
    if is_local:
        pdf_path = local_path
    else:
        arxiv_id = extract_arxiv_id(input_str)
        if not arxiv_id:
            print(f"[ERROR] Cannot parse arXiv ID from: {input_str}", file=sys.stderr)
            sys.exit(1)
        metadata = fetch_metadata(arxiv_id)
        if "error" in metadata:
            print(f"[ERROR] {metadata['error']}", file=sys.stderr)
            sys.exit(1)
        print(f"# {metadata['title']}\n")
        print(f"- Authors: {', '.join(metadata['authors'])}")
        print(f"- Date: {metadata['published']}")
        print(f"- arXiv: {metadata['arxiv_id']}")
        print(f"\n## Abstract\n\n{metadata['abstract']}\n")
        print("---\n")
        pdf_path = download_pdf(arxiv_id, output_dir)
    full_text = extract_text(pdf_path)
    print("## Full Text\n")
    print(full_text)

if __name__ == "__main__":
    main()
PYEOF
python /tmp/_read_arxiv.py $ARGUMENTS --output data/papers
```

Read the full output carefully — this is the paper content you'll answer questions from.

## Step 2: Interactive Q&A

Now enter a Q&A mode with the user. For each question:

1. Answer accurately based on the paper content — cite specific sections, equations, figures, or tables
2. Use Chinese as the primary language, keeping English terms and formulas as-is
3. If the paper doesn't contain relevant information, say so explicitly
4. Be concise and focused on the question

## Behavior guidelines

- 回答要准确、可追溯，避免臆测
- 优先使用中文，必要时引用关键术语、公式或原文表述
- 回答简洁明了，聚焦问题本身
- 公式和变量使用 $ $ 或 $$ $$包围。
- 如果用户要求保存笔记，将问答记录追加到 `data/outputs/papers/qa_<paper_id>.md`
- 如果用户要求总结，生成精简的讨论要点
- 如果用户要求生成TODO，基于讨论内容列出可执行的研究想法清单
