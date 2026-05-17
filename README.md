# skills

Personal skill collection for [pi](https://github.com/earendil-works/pi-coding-agent) and other coding agents that use SKILL.md files.

Each skill lives in its own directory with a `SKILL.md` file that the agent reads and follows when invoked.

## Skills

| Skill | Description |
|-------|-------------|
| **ruthless-review** | Dual-hatted Tech Lead + CEO codebase audit. Finds what's actually wrong with your architecture — no sugarcoating, no style nitpicks, no generic praise. Delivers a structured report with executive summary, architectural breakdown with file citations, and a prioritized backlog of fixes with impact/complexity scoring. |
| **skill-creator** | Create, edit, and evaluate coding agent skills. Draft skills, run them against test prompts, review results with quantitative evals, and iterate. Also optimizes skill descriptions for better triggering accuracy. |
| **text-to-md** | Convert plain text or subtitle files (`.vtt`, `.srt`, `.txt`) into clean Markdown. Strips timestamps, detects headings and lists, cleans formatting, fixes common OCR errors. |

## Workspaces

| Directory | Purpose |
|-----------|---------|
| `ruthless-review-workspace/` | Evaluation data and iteration artifacts for the ruthless-review skill |

## Installation

Clone into your agent's skills directory. For pi:

```bash
git clone https://github.com/blaiseliu/skills.git ~/.pi/agent/skills/blaise-skills
```

## License

Skills are provided as-is. The skill-creator skill is adapted from [Anthropic's skill-creator](https://github.com/anthropics/skills) under its original license (see `skill-creator/LICENSE.txt`).
