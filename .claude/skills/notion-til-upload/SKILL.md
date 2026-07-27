---
name: notion-til-upload
description: Fetch today's (or a specified date's) TIL entry from the user's Notion "TIL" database and upload it to this repo as a markdown file, following the fixed folder/file/commit conventions below. Trigger this whenever the user says things like "TIL 업로드", "오늘 TIL 올려줘", "노션 TIL 커밋", "TIL 깃허브에 올려줘", or otherwise asks to sync/push/upload their daily Notion TIL to GitHub. This is a daily-habit automation for this specific repo and one specific Notion database — always use it end-to-end (fetch → convert → commit → push) rather than doing only part of the workflow, unless the user asks for just one step.
---

# Notion TIL → GitHub Upload

Automates the daily habit of taking a Notion TIL ("Today I Learned") entry and turning it into a committed, pushed markdown file in this repo. This skill exists because the user does this every day and the rules below (folder layout, callout formatting, commit message) were worked out together with them — follow them exactly rather than improvising a different structure, since consistency across ~4 months of daily entries is the whole point.

## Fixed facts about this setup

- **Notion data source**: `collection://3a5e7e8f-dd12-802e-8f87-000bd00b4c62` (the "TIL" data source). Each row/page is titled with its date, e.g. `2026-07-27`.
- **This repo**: tracks `origin/main` at `https://github.com/hm1n/CAREER-CAMP-TIL.git`. Run all git commands from the repo root (the directory containing this `.claude/` folder).
- **Camp start date**: 2026-07-22, which is **Week01, Day1**. The camp runs weekdays only (Mon–Fri); a new week folder begins every Monday, even if the previous week was a short/partial week (the first week is partial since it starts on a Wednesday).

## Step 1 — Fetch the Notion entry

1. Figure out the target date. Default to today's date (from the current session's date context). If the user names a different date, use that instead.
2. Use `notion-query-data-sources` (SQL mode) against the data source above to find the page whose title/date matches the target date, or use `notion-search`/`notion-fetch` directly if you already know the page URL from a prior conversation turn.
3. Fetch the full page with `notion-fetch`. If no page exists for that date yet, tell the user and stop — don't fabricate content.

## Step 2 — Convert to markdown

Build the file content from the page's `<content>` block:

1. **First line**: `# YYYY-MM-DD` (just the date, not the "✒️" emoji Notion adds to its own H2).
2. **Section headers** become `## 섹션명`, keeping the section's own name as written in Notion (typically 오늘 배운 내용, 더 알아보고 싶은 것, 나에게 적용한다면, 한 줄 회고 — but carry over whatever sections actually exist that day, e.g. an extra "오늘의 아이디어" section, rather than assuming a fixed list).
3. **Callout blocks → h4 with icon prefix.** Notion callouts (`<callout icon="...">text</callout>`) represent one bite-sized point. Convert each to a heading of the form:
   ```
   #### <icon> <callout text>
   ```
   followed by a blank line, then the paragraph(s) that follow the callout in Notion (up to the next callout or section header) as normal body text below it. If a callout has no following body text (common in the "나에게 적용한다면" section, where callouts are often just short idea titles with nothing after them), just emit the heading with no body paragraph — don't invent a body.
   - Known icon-to-section pairing so far: 📌 for 오늘 배운 내용, ❓ for 더 알아보고 싶은 것, 💡 for 나에게 적용한다면. But always use whatever icon is actually on the callout in Notion — don't hardcode the icon by section name, since that's just been the pattern, not a rule Notion enforces.
4. **한 줄 회고** is a plain blockquote (`> ...`), not a callout — leave it as `> text` under its `##` header. If it's empty (no reflection written yet for today), leave `>` with nothing after it rather than skipping the section.
5. Clean up Notion-specific markdown artifacts that shouldn't leak into the file:
   - Escaped characters like `\~` → `~`.
   - `<br>` inside blockquotes → a real line break (a new `> ` line).
   - `<mention-page url="...">` references → drop them or replace with plain text if the mentioned page's title is obvious from context; don't leave raw Notion mention tags in the output.
   - Notion's occasional double-bold artifacts around inline code (e.g. `**\`상황\`**** **`) → normalize to a single `**상황**` bold wrapper around the label.

## Step 3 — Place the file in the right week folder

Compute the week folder like this:
1. Find the Monday of the calendar week containing the **camp start date** (2026-07-22 → Monday 2026-07-20). Call this `week1Monday`.
2. Find the Monday of the calendar week containing the **target date**. Call this `targetMonday`.
3. `weekNumber = floor((targetMonday - week1Monday) / 7 days) + 1`. Zero-pad to two digits (`Week01`, `Week02`, ...).
4. The file goes at `WeekNN/YYYY-MM-DD.md` inside the repo root.

(Sanity check against known data: 2026-07-22/23/24 → Week01; 2026-07-27 → Week02, since 07-27 is the next Monday after the partial first week.)

If a file already exists at that path, **ask the user before overwriting it** — don't silently clobber a day's entry, since it may contain edits made after the last sync (like the 한 줄 회고 that gets filled in later in the day).

## Step 4 — Commit

The commit convention here is fixed and non-negotiable (the user set this deliberately so the repo history reads as one line per day):

- Stage **only** the one new/updated markdown file — never bundle multiple days or unrelated changes into one commit.
- Commit message is **exactly** `YYYY-MM-DD TIL 업로드` (the target date, a space, then the literal text "TIL 업로드"). No prefix, no scope, no extra description.
- If other unrelated files happen to be modified in the working tree (e.g. README changes from an earlier session), leave them uncommitted — they're out of scope for this skill. Only touch what this run produced.

```bash
git add "Week<NN>/<date>.md"
git commit -m "<date> TIL 업로드"
```

## Step 5 — Push

Push to `origin main`:

```bash
git push origin main
```

This repo and this exact push (today's single TIL commit to the user's own personal repo) is what the user built this skill to automate — proceed with the push as part of the normal flow without pausing to ask "should I push?" each time, the same way the rest of the pipeline doesn't pause after every step. Do still surface a short confirmation of what happened afterward (date, file path, commit hash, push result) so the user can see it landed correctly. If `git push` fails (e.g. remote has diverged, network issue, author identity not configured in a fresh environment), stop and report the actual error rather than retrying blindly or force-pushing.

## End-to-end summary

Given a date (default: today), this skill should go from "nothing" to "a correctly-formatted, correctly-placed, correctly-committed, and pushed markdown file" in one pass, and then report back concisely: which file was created, which week folder it landed in, the commit message used, and confirmation the push succeeded.
