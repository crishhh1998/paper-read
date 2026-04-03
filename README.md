# Paper Read

A set of [Claude Code](https://claude.ai/code) slash commands for reading, chatting with, and searching arXiv papers.

## Skills

| Command | Description |
|---------|-------------|
| `/read-paper <arxiv_id_or_url>` | Download an arXiv paper and generate a structured analysis |
| `/chat-paper <arxiv_id_or_url>` | Interactive multi-turn Q&A with a paper |
| `/search-paper <query>` | Search arXiv papers by keyword |

## Installation

### Option 1: Copy into your project

Copy the `.claude/commands/` directory into your project:

```bash
# In your project root
mkdir -p .claude/commands
curl -sL https://raw.githubusercontent.com/crishhh1998/paper-read/main/.claude/commands/read-paper.md -o .claude/commands/read-paper.md
curl -sL https://raw.githubusercontent.com/crishhh1998/paper-read/main/.claude/commands/chat-paper.md -o .claude/commands/chat-paper.md
curl -sL https://raw.githubusercontent.com/crishhh1998/paper-read/main/.claude/commands/search-paper.md -o .claude/commands/search-paper.md
```

### Option 2: Clone the repo

```bash
git clone https://github.com/crishhh1998/paper-read.git
cd paper-read
```

Then open Claude Code in this directory.

## Usage

Once installed, open Claude Code in the project directory and type:

```
/read-paper 2312.00752
/read-paper https://arxiv.org/abs/2312.00752
/chat-paper 2312.00752
/search-paper diffusion video generation --category cs.CV --limit 5
```

Python dependencies (`arxiv`, `httpx`, `PyMuPDF`) are installed automatically on first run.

## Requirements

- [Claude Code](https://claude.ai/code)
- Python 3.10+
