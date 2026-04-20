# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working in this repository.

## Project Overview

A personal study-notes repository. The workflow is: **discuss a topic with Claude, then save the important takeaways as a markdown file** here. Topics are general (git, linux, networking, tooling, etc.) — not scoped to any single language or domain.

## Structure

- Organize notes by topic, one folder per topic (e.g. `git/`, `linux/`, `networking/`).
- One markdown file per focused subtopic. Prefer several small files over one sprawling file.
- Use kebab-case filenames (e.g. `rebase-vs-merge.md`).
- A topic folder may have an `index.md` linking to its notes once it grows.

## Workflow

1. User opens a discussion on a topic.
2. Claude explains, answers questions, explores edge cases.
3. When the user says "save this" (or equivalent), Claude writes a markdown file capturing the **conclusions and non-obvious details** from the conversation — not a transcript.

## Writing Guidelines

### Think before writing
- State assumptions explicitly. If uncertain, ask rather than guess.
- If multiple interpretations exist, present them — don't pick silently.
- If something is unclear, stop and name what's confusing.

### Content
- Capture the **reasoning and motivation** behind decisions and the **non-obvious** details, not textbook definitions easily found elsewhere.
- Include concrete examples, commands, and gotchas the user hit during discussion.
- Link related notes with relative markdown links.
- Keep a short summary/TL;DR at the top of each note.

### Style
- Markdown only. Use headings, fenced code blocks with language tags, and tables where they help.
- Use mermaid fenced blocks for diagrams/charts (flowcharts, sequence, state, etc.) when a visual clarifies the idea.
- Be concise. A study note is for the user's future self — not a tutorial for strangers.
- No filler ("In this document, we will discuss..."). Start with the substance.
- Match existing note style once a pattern is established.

### Surgical edits
- When updating an existing note, touch only what changed. Don't reformat or rewrite adjacent sections.
- If a note grows unwieldy, suggest splitting it — don't silently restructure.

## Conventions

- Prefer editing an existing note over creating a new one when the topic overlaps.
- Do not create a note unless the user explicitly asks to save. Discussion alone does not imply a save.
- Do not delete or rewrite prior notes unless asked — they are a record of what the user learned at that point in time.
