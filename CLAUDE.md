# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Project Is

WisdomBuilder is a personal personality profiling system for CoYoFroYo (Cody). Sessions are run conversationally — Claude asks questions, Cody answers, and the profile accumulates over time. There is no build system, no tests, and no server. Everything is JSON data files and Markdown exports.

The global skill that drives sessions lives at `C:\Users\coyof\.claude\skills\wisdom-builder.md`. Read that file before running any session — it contains the full session flow, scoring logic, and export instructions.

## Data Files

| File | Purpose |
|---|---|
| `questions/bank.json` | 100+ questions with full scoring metadata. Schema: id, category, depth (1–3), format, options, followup, reveals (dimension/axis/scoring), cooldown_days. |
| `profile/master-profile.json` | Source of truth for Cody's personality profile. All cumulative scores live here. |
| `tracking/answered.json` | Cooldown tracker — which questions were answered and when, with `eligible_again` dates. |
| `sessions/` | One JSON file per completed session. Never modify retroactively. |
| `exports/` | Markdown files for manual Standard Notes paste. `master-profile.md` is overwritten each session; session files are permanent. |
| `quizzes/results.json` | Online quiz results captured after Cody takes external tests. |
| `quizzes/test-library.json` | Scheduled online test recommendations (10 tests across sessions 3–20). |

## Scoring Scales

| Dimension | Scale | Direction |
|---|---|---|
| `mbti_ei`, `mbti_sn`, `mbti_tf`, `mbti_jp` | -10 to +10 | negative = I/N/F/P, positive = E/S/T/J |
| `political_economic` | -10 to +10 | negative = Left, positive = Right |
| `political_social` | -10 to +10 | negative = Libertarian, positive = Authoritarian |
| `big5_*` | 0–100 | initialized at 50; each question shifts ±2–5 |
| `moral_framework` | -10 to +10 | negative = Deontological, positive = Utilitarian |
| `values_*` clusters | 0–100 | cumulative, initialized at 0 |

**Derived MBTI type:** assign when `abs(score) >= 2` AND `confidence >= 4` on that axis.
**Online quiz results** are high-confidence anchors — set confidence to 15 and score proportionally.

## Session Rules

- Ask session length at the start: **Quick (5)**, **Standard (7)**, or **Deep Dive (10+)**
- Max 2 questions per category per session
- Never repeat a question within its `cooldown_days` window (default 60, classics 90)
- Session depth bias: sessions 1–3 → depth-1; sessions 4–10 → mix; sessions 11+ → depth-2/3
- Suppress depth-3 questions until `sessions_completed >= 4`
- Present questions one at a time conversationally — never as a numbered list

## Modifying the Question Bank

Every entry in `bank.json` must have all required fields: `id`, `text`, `category`, `subcategory`, `depth`, `format`, `options` (null for open_ended), `followup` (null if none), `reveals` (with `dimension`, `axis`, `scoring`, `notes`), `tags`, `cooldown_days`, `source`.

Auto-expansion: if any category drops below 5 eligible (non-cooldown) questions after a session, generate 5 new questions for that category and append them with `"source": "auto-generated"`.

## Standard Notes Export Conventions

| File | Standard Notes title |
|---|---|
| `exports/master-profile.md` | `WisdomBuilder — Master Profile` (replace each session) |
| `exports/session-N-YYYY-MM-DD.md` | `WisdomBuilder — Session N — YYYY-MM-DD` (keep all) |
| `exports/profile-export-YYYY-MM-DD.md` | `WisdomBuilder — Full Profile Export — DATE` (on demand) |

## Obsidian Vault (Primary Export Target)

After every session, write files **directly to the Obsidian vault** in addition to the `exports/` folder. The vault is at:

```
G:\My Drive\My Life Vault\My Life Vault\20 Areas\Personal Development\Wisdom Building\
```

**Every file written to Obsidian must include this YAML frontmatter:**

```yaml
---
category: areas
tags: ["status/active", "topic/personal-development", "topic/wisdom"]
created: YYYY-MM-DD   ← date the session/profile was created
modified: YYYY-MM-DD  ← today's date
---
```

**Files to write after each session:**

| Obsidian file | Action | Content |
|---|---|---|
| `WisdomBuilder — Cody's Master Profile.md` | Overwrite | Full master profile (same as `exports/master-profile.md`, with frontmatter, without the Standard Notes footer line) |
| `WisdomBuilder — Session N — YYYY-MM-DD.md` | Create new | Full session Q&A log (same as `exports/session-N-YYYY-MM-DD.md`, with frontmatter, without the Standard Notes footer line) |

**Do not** include the `*Paste into Standard Notes as: ...*` footer lines in Obsidian files.
**Do not** push the Obsidian vault path to GitHub — it is a local personal vault.
