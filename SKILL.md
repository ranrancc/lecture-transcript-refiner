---
name: dictation-corrector 讲座稿处理助手
description: "转写整理器。处理讲座/访谈/录音稿：先听写校对，再按需要书面整理成 Obsidian 笔记。默认作为“理”入口下的转写执行器：输入像转写稿、讲座稿、录音转文字、字幕纠错时调用。NOT for：普通文章润色、一般写作重写、inbox 分流。"
---

# Dictation Correction Skill

This skill is the transcript-processing executor for lecture, interview, recording, and subtitle text. It first corrects ASR/dictation errors with `prompt.md`; when the user asks for a written lecture note, it then rewrites the corrected transcript with `prompt_rewrite.md` and publishes the result into the lecture-recording Obsidian area.

## 默认主路 / 不该抢的活

**默认主路**：讲座稿、访谈稿、录音转文字、字幕错字修复、书面整理。

**不要抢这些活**：
- 一般文章润色 → `humanizer-zh` / 写作链路
- inbox 分流 / 入库 → `inbox-publisher`
- 议题研究 / 内容扩写 → 其他研究或写作 skill

## 触发条件

Use this skill when the user provides or points to transcript-like material and asks to correct, clean, or convert it into lecture notes.

**Strong triggers**:
- "听写校正" (Dictation Correction)
- "校正录音稿" (Correct Transcript)
- "整理语音转文字" (Clean up voice-to-text)
- "修复字幕错别字" (Fix subtitle typos)
- "书面整理" (Rewrite into written notes)
- "整理成文稿" (Rewrite into written transcript/notes)
- "讲座稿处理" / "访谈稿整理" / "字幕整理"

**Weak triggers that still apply when the input is clearly a transcript**:
- "帮我整理这个录音转文字"
- "把这份转写稿理一下"
- "转成 Obsidian 讲座笔记"

**Do NOT use this skill for**:
- General article polishing (润色文章)
- Grammar checking of written essays (一般文章语法检查)
- General rewriting that is not based on transcript/recording text
- Inbox routing or publishing only; use `inbox-publisher`
- Web capture or media download; use the capture/archive skill first

## Input Contract

Accept these input forms:
1. A local text/markdown/subtitle file path.
2. Pasted transcript text.
3. A folder containing transcript-like files, when the user clearly asks for batch handling.

If no processable text is available, ask for the file path or pasted transcript. If the input is a media URL or audio/video file with no transcript, route to the appropriate transcription/archive skill first instead of improvising.

## Workflow

### 1. Read prompts and source text

- Read `${SKILL_DIR}/prompt.md` for `MODE=proofread` and `MODE=strict`.
- Read `${SKILL_DIR}/prompt_rewrite.md` only when `MODE=rewrite` is needed.
- Preserve the original source file. Never overwrite the raw transcript.

### 2. Decide mode

Use the safest mode that satisfies the user request:

- `MODE=proofread` by default: correct ASR errors, punctuation, paragraphing, and obvious oral noise while preserving the speaker's wording, sequence, examples, and information density. Do not summarize or compress.
- `MODE=strict` only when the user says "保留口语", "尽量原话", "不要书面化", "strict", or equivalent.
- `MODE=rewrite` only when the user asks for "书面整理", "整理成文稿", "讲座回顾", "Obsidian 笔记", or equivalent.

For rewrite requests, always run a 2-pass workflow:
1. Pass A: run `MODE=proofread` on the original text and save a `（已校正）` file beside the source.
2. Pass B: run `MODE=rewrite` on the corrected text using `prompt_rewrite.md`, then save the Obsidian note.

### 3. Process text with chunking

Chunk only when needed:

- `MODE=proofread` or `MODE=strict`: chunk when source text is longer than 25,000 characters.
- `MODE=rewrite`: chunk when corrected text is longer than 180,000 characters.

Chunking rules:

- Use 20,000-25,000 characters per chunk for proofread/strict.
- Prefer paragraph breaks, speaker turns, or sentence-ending punctuation.
- Keep 200-400 characters of overlap between adjacent chunks.
- Process chunks sequentially and keep the original order.
- After joining chunks, remove duplicated overlap text without deleting unique sentences.
- If a chunk cannot be fully processed in the current context, stop and report that continuation/chunked processing is required. Do not submit a partial "summary" as if complete.

### 4. Save outputs

For `MODE=proofread` or `MODE=strict`:

- Save beside the source file.
- Append `（已校正）` before the extension.
- If the target file already exists, append `-2`, `-3`, etc. rather than overwriting.
- Example: `/path/to/讲座（已校正）.txt`

For `MODE=rewrite`:

- Destination root: `/Users/zhangyiran/Library/Mobile Documents/iCloud~md~obsidian/Documents/ZYR/01-摘录库/讲座录制/`
- Folder name: prefer `YYYYMMDD 讲者或主题` extracted from the source filename or nearby folder. If no date exists, use `undated 讲者或主题`.
- File name: prefer `序号 讲者 讲座标题.md`. If speaker or title is uncertain, use a conservative descriptive placeholder and mention the uncertainty to the user.
- Add YAML front matter:

```yaml
---
title: "1 苏涛 AI时代的大学课堂"
type: "讲座回顾"
status: "待整理"
tags: ["讲座录制", "讲座回顾", "自动生成"]
source_date: "2026-03-04"
speaker: "苏涛"
speaker_title: "西安电子科技大学本科生院院长、教授"
generated_at: "2026-03-06"
---

**原始文件**: /path/to/讲座（已校正）.txt

# 讲座内容
```

When metadata cannot be confidently inferred, keep the field with an empty string or conservative value rather than inventing facts.

### 5. Verify before reporting

Before notifying the user, check:

- The raw source remains unchanged.
- The proofread output exists and is non-empty.
- For rewrite requests, the Obsidian note exists, contains YAML front matter, includes the original corrected-file link, and has substantive lecture content.
- Proofread/strict output is not accidentally summarized: examples, speaker turns, and main argument chains are still present.

### 6. Notify

Report the completed mode and output path(s). If metadata was uncertain or any fallback was used, state that briefly.

## Output Naming

- Proofread output: append `（已校正）` before extension (save in source directory).
- Written rewrite output (only for "书面整理"): save to Obsidian directory with YAML front matter.

## Test Prompts

Use these prompts when evaluating this skill:

1. `请把 /path/to/20260304 苏涛 AI时代的大学课堂.txt 做听写校正。`
   - Expected: runs `MODE=proofread`, saves a non-overwriting `（已校正）` file beside the source, preserves information density.
2. `这份讲座转写稿已经校过一遍了，请保留口语风格，尽量原话，只修错别字和标点。`
   - Expected: runs `MODE=strict`, avoids rewriting, summarizing, or adding headings.
3. `把这个录音转文字书面整理成 Obsidian 讲座回顾笔记。`
   - Expected: runs proofread first, then rewrite, saves corrected transcript plus an Obsidian note with YAML front matter and original-file link.
