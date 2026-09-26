---
name: update-skill-from-chat
description:Update a local skill's SKILL.md from changes implemented or decided 
in the current agent chat. Use when the user asks to sync, update, revise, or 
fold chat changes into a skill, or names this skill / update-skill-from-chat.
disable-model-invocation: true

---

# Update Skill From Chat

Fold durable learnings from this conversation into the target local skill so a future agent following that skill alone reproduces the improved behavior.

## 1. Resolve the target skill

1. If the user names a skill (path, folder name, or `name:`), use that.
2. Otherwise infer from chat: which skill was read/followed, whose domain the work matched, or which `SKILL.md` is under discussion.
3. If still ambiguous among multiple candidates, ask once which skill to update. Do not guess.

## 2. Read before editing

Open the target `SKILL.md` (and any linked reference files if relevant). Do not edit from memory alone.

## 3. Extract durable learnings (this chat only)

**Capture** new rules, corrected defaults, workflows, constraints, examples, and gotchas that were **implemented or firmly decided** in this conversation.

**Ignore** one-off task details, temporary debugging, unrelated code changes, and speculative ideas that were not adopted.

## 4. Edit the skill in place

- Update instructions so following the skill alone would reproduce the improved behavior.
- Match the existing skill's style (concise; no fluff).
- Prefer surgical edits over full rewrites.
- Keep `name` / `description` accurate; refresh description trigger terms if scope changed.
- Do not invent rules that were not established in the chat.
- Do not auto-copy to `~/.cursor/skills/` or `~/.claude/skills/` (repo is source of truth).
- Do not commit unless the user asks.

## 5. Report

- Which skill file was updated.
- Brief bullets of what changed and why (tied to chat decisions).
