---
name: text-to-md
description: |
  Convert plain text or subtitle files (.txt, .vtt, .srt) into clean Markdown.
  Strips timestamps from subtitles, detects headings and lists, cleans formatting,
  fixes common OCR errors, and outputs a .md file. Use when asked to "convert this
  to markdown", "turn subtitles into notes", "clean up this transcript", "format
  this text file as markdown", or when working with .vtt/.srt files that need to
  become readable documents.
---

# Text to Markdown Converter

Convert plain text or subtitle files into clean, well-structured Markdown documents.

## Overview

This skill handles two categories of input:
1. **Subtitle files** (.vtt, .srt) — strip timestamps and cue markers, leaving clean text
2. **Plain text files** (.txt) or raw text — clean up and apply Markdown structure

## Workflow

### Step 1: Get the input

Determine what the user is providing:

- If a **file path** is given, read it. Verify it ends in `.txt`, `.vtt`, or `.srt`. If the file doesn't exist, tell the user.
- If **raw text** is pasted or provided inline, treat it as plain text input.
- If the user doesn't specify an input, ask them.

### Step 2: Parse the content

**For .vtt / .srt subtitle files:**

Strip out all timing and metadata, keeping only the spoken text:
- Remove WEBVTT headers and metadata blocks
- Remove timestamp lines (e.g., `00:01:23.456 --> 00:01:25.789`)
- Remove cue numbers (e.g., `1`, `42`)
- Keep only the text lines
- If a cue has multiple text lines, join them with a space
- Deduplicate repeated consecutive lines (common in subtitles)

Example transformation:
```
WEBVTT

1
00:00:01.000 --> 00:00:04.500
Welcome to this presentation on

2
00:00:04.500 --> 00:00:08.200
the history of ancient Rome.
```
becomes:
```
Welcome to this presentation on the history of ancient Rome.
```

**For plain text (.txt) or raw text:**
- Preserve the text as-is for the cleaning step

### Step 3: Clean the text

Apply these cleanups in order:

1. **Normalize line endings** — convert `\r\n` to `\n`
2. **Collapse blank lines** — multiple consecutive blank lines → single blank line
3. **Trim whitespace** — remove trailing spaces from each line, remove leading/trailing whitespace from the whole document
4. **Fix common OCR errors** — apply these replacements:
   - `"1"` at the start of a line followed by a space/period → `"I"` (when it looks like the pronoun)
   - Isolated `"0"` where context suggests `"O"` → `"O"`
   - `"|"` used as `"I"` or `"l"` → restore based on context
   - Broken ligatures and common artifact patterns
   - Don't over-correct — when in doubt, leave the original

5. **Normalize smart quotes** — if text has mixed straight/curly quotes, normalize to curly quotes for readability

### Step 4: Apply Markdown structure

Convert the cleaned text to Markdown by detecting structure:

**Headings — detect these patterns and convert to `#` headers:**
- Lines that are short, standalone, and in ALL CAPS or Title Case, especially if followed by a blank line
- Lines matching patterns like `Chapter N`, `Part N`, `Section N`, `Introduction`, `Conclusion`, `Summary`
- Lines that are clearly section titles based on content and position
- For VTT/SRT output: use the first meaningful line as an `# H1` title
- Don't force headings — err on the side of leaving text as paragraphs rather than creating false headings

**Paragraphs:**
- Groups of text separated by blank lines become separate paragraphs
- Single line breaks within a paragraph are joined (Markdown doesn't need explicit joining — just remove the single `\n`)

**Lists — detect these patterns and convert to Markdown lists:**
- Lines starting with `-`, `*`, `•`, `·`, `–`, `—` followed by a space → unordered list (`- `)
- Lines starting with a number followed by `.` or `)` and a space (e.g., `1.`, `1)`) → ordered list
- Consecutive list items are grouped into a single list block
- If detection is ambiguous, prefer paragraphs over lists

**Bold/Italic:**
- Don't add bold/italic unless the source clearly marks it (e.g., `*word*` or `_word_` in the original)
- If the source uses ALL CAPS for emphasis, convert those words to `**bold**` only when clearly intentional (short runs, not full paragraphs)

**Blockquotes:**
- If text contains citations or clearly quoted material, wrap in `> `
- Only use blockquotes when the quoted nature is unambiguous

**Escape special characters:**
- Escape bare `*`, `_`, `#`, `[`, `]`, `<`, `>`, `` ` ``, `|` that appear in the original text and aren't part of intentional Markdown
- Don't double-escape already-escaped characters

### Step 5: Write the output

- **Default filename**: Same as the input file but with `.md` extension (e.g., `lecture.vtt` → `lecture.md`). For raw text input, use a descriptive name based on the content or let the user specify.
- **User-specified path**: If the user provides an output path, use it.
- Write the Markdown content to the output file.
- Report back: the file path, line count, and a brief summary of what was done.

## Example

**Input** (`notes.srt`):
```
1
00:00:01.000 --> 00:00:03.500
Chapter 1: The Beginning

2
00:00:03.500 --> 00:00:07.200
In the beginning, there was nothing.

3
00:00:07.200 --> 00:00:11.800
And then there was something.
```

**Output** (`notes.md`):
```markdown
# Chapter 1: The Beginning

In the beginning, there was nothing.

And then there was something.
```

## Guardrails

- Never modify the original input file — always write a new `.md` file
- When uncertain about structure (is this a heading? is this a list?), default to plain paragraphs — it's better to under-structure than to create false structure
- Preserve the original meaning — don't paraphrase or summarize
- If a file is very large (>10,000 lines), process it in chunks and combine the results
- If the user provides a file in an unsupported format, explain the supported formats (.txt, .vtt, .srt) and ask what they'd like to do
