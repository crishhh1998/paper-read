---
description: "Download and analyze an arXiv paper. Usage: /read-paper <arxiv_id_or_url>"
---

# Read & Analyze arXiv Paper

You are a senior researcher specializing in visual generation (Diffusion Models, Flow Matching, video/image generation). The user wants you to read and analyze an arXiv paper.

## Step 1: Download and extract paper text

Run the following to download the PDF and extract text. The argument `$ARGUMENTS` contains the arXiv ID or URL.

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

def extract_sections(text: str) -> dict[str, str]:
    lower = text.lower()
    section_defs = [
        ("abstract", ["abstract"]),
        ("introduction", ["introduction", "1 introduction", "1. introduction"]),
        ("related_work", ["related work", "2 related", "2. related"]),
        ("method", ["method", "approach", "model", "architecture", "3 method", "3. method"]),
        ("experiments", ["experiment", "evaluation", "results", "4 experiment", "4. experiment"]),
        ("conclusion", ["conclusion", "5 conclusion", "6 conclusion"]),
        ("limitations", ["limitation", "future work"]),
    ]
    sections = {}
    for name, keywords in section_defs:
        indices = [lower.find(kw) for kw in keywords]
        indices = [i for i in indices if i >= 0]
        if not indices:
            continue
        start = min(indices)
        sections[name] = text[start : start + 5000].strip()
    return sections

def main():
    parser = argparse.ArgumentParser(description="arXiv paper reader for AI assistants")
    parser.add_argument("paper", help="arXiv ID, URL, or local PDF path")
    parser.add_argument("--output", "-o", default=str(DEFAULT_OUTPUT_DIR),
                        help="Output directory for downloaded PDFs")
    parser.add_argument("--meta-only", action="store_true",
                        help="Only output metadata (no full text)")
    parser.add_argument("--sections", action="store_true",
                        help="Extract and output key sections instead of full text")
    parser.add_argument("--json", action="store_true",
                        help="Output as JSON")
    args = parser.parse_args()
    output_dir = Path(args.output)
    input_str = args.paper
    local_path = Path(input_str)
    is_local = local_path.exists() and local_path.suffix.lower() == ".pdf"
    if is_local:
        metadata = {
            "title": local_path.stem.replace("_", " ").replace("-", " "),
            "source": "local",
            "path": str(local_path.resolve()),
        }
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
        if args.meta_only:
            if args.json:
                print(json.dumps(metadata, ensure_ascii=False, indent=2))
            else:
                print(f"Title: {metadata['title']}")
                print(f"Authors: {', '.join(metadata['authors'])}")
                print(f"Date: {metadata['published']}")
                print(f"arXiv: {metadata['arxiv_id']}")
                print(f"Categories: {', '.join(metadata['categories'])}")
                print(f"\nAbstract:\n{metadata['abstract']}")
            return
        pdf_path = download_pdf(arxiv_id, output_dir)
    full_text = extract_text(pdf_path)
    if args.json:
        result = {**metadata}
        if args.sections:
            result["sections"] = extract_sections(full_text)
        else:
            result["full_text"] = full_text
        print(json.dumps(result, ensure_ascii=False, indent=2))
    else:
        if not is_local:
            print(f"# {metadata['title']}\n")
            print(f"- Authors: {', '.join(metadata['authors'])}")
            print(f"- Date: {metadata['published']}")
            print(f"- arXiv: {metadata['arxiv_id']}")
            print(f"- Categories: {', '.join(metadata['categories'])}")
            print(f"\n## Abstract\n\n{metadata['abstract']}\n")
            print("---\n")
        if args.sections:
            sections = extract_sections(full_text)
            for name, content in sections.items():
                print(f"## {name.replace('_', ' ').title()}\n")
                print(content[:3000])
                print()
        else:
            print("## Full Text\n")
            print(full_text)

if __name__ == "__main__":
    main()
PYEOF
python /tmp/_read_arxiv.py $ARGUMENTS --output data/papers --sections
```

## Step 2: Read the full extracted text

If the `--sections` output is too brief or you need more context, re-run without `--sections` and read the full text:
```bash
python /tmp/_read_arxiv.py $ARGUMENTS --output data/papers
```

Or directly read the downloaded PDF file from `data/papers/`.

## Step 3: Analyze the paper

Based on the extracted paper content, provide a structured Chinese analysis covering these 6 parts and 20 dimensions:

### 第一部分：基本信息概览
1. **文献元数据**：作者名单、发表年份、论文全名及发表刊物/会议
2. **核心问题**：本研究解决的具体痛点、技术瓶颈或理论缺口
3. **核心假设**：作者提出的核心科学假设或理论构想

### 第二部分：方法论与研究设计
4. **整体思路**：研究的宏观框架与逻辑链条
5. **核心算法与理论**：数学推导、模型架构创新、目标函数改进
6. **实验设置细节**：硬件环境 / 超参数 / 评估基准与数据集
7. **数据分析方法**：统计手段、性能评价指标

### 第三部分：实验结果与发现
8. **核心发现**：最关键的科学发现或观测现象
9. **定量结果**：关键性能指标、SOTA 对比
10. **定性结论**：视觉效果、生成质量等非数值化结论
11. **辅助结果**：消融实验及其他次要结果

### 第四部分：可视化证据索引
12. **图表导读**：按顺序列出所有 Figure 和 Table — [编号] - [标题] - [核心展示内容]

### 第五部分：学术贡献与评价
13. **领域贡献**：理论创新、技术突破、方法改进、应用拓展
14. **批判性分析**：开创性 / 局限性 / 争议性
15. **个人评价**：方法严谨性、逻辑完备性、结论可靠性评分与点评

### 第六部分：思考与延伸
16. **潜在疑问**：逻辑断点、未详尽解释之处
17. **启发与未来方向**：新思路与延伸方向
18. **关键引文**：1-2 篇最具代表性的参考文献

## Output requirements

- 使用中文输出，必要时保留英文术语和公式
- 内容准确、完整、可读性强，避免臆测
- 信息不足时明确说明
- 每个维度使用完整段落，方法/实验/结论给出关键细节
- 将分析结果保存到 `data/outputs/papers/abstract_<paper_id>.md`
