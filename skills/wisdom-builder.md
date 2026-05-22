---
description: >
  Use this skill whenever Cody wants personality questions, philosophical questions, moral
  dilemmas, or to explore his own values and character. Triggers include: "wisdom questions",
  "give me today's wisdom questions", "daily questions", "personality questions", "wisdom
  builder", "wisdom session", "give me some deep questions", "philosophical questions",
  "trolley problem", "moral dilemma", "what does my profile say", "show my profile",
  "show my personality", "export my profile", "take a quiz", "run a quiz", "find me a quiz",
  "16 personalities", "political compass", "add quiz results", or any request to answer
  questions that reveal Cody's personality, values, or how he'd respond to difficult
  situations. Use proactively if Cody mentions wanting to know more about himself.
---

# WisdomBuilder Skill

## Working Directory

All data lives at:
`C:\Users\coyof\Documents\Claude\Claude Code\WisdomBuilder\`

Full system spec is in the CLAUDE.md at that directory.

## Files

- `questions/bank.json` — question bank (100+ questions)
- `profile/master-profile.json` — running personality profile (source of truth)
- `tracking/answered.json` — cooldown tracking
- `quizzes/results.json` — online quiz results
- `quizzes/test-library.json` — scheduled online test recommendations
- `sessions/` — one JSON file per completed session
- `exports/` — markdown exports for Standard Notes paste

## Step 1 — Determine Intent

Read the trigger to determine mode:
- **Run session**: "give me questions", "wisdom session", "daily questions", or similar → go to Step 2
- **Show profile**: "what does my profile say", "show my profile", "show my personality" → go to Step 6
- **Export profile**: "export my profile" → go to Step 7
- **Add quiz results**: "add quiz results", "I took the 16personalities test" → go to Step 8
- **Find new quiz**: "find me a quiz", "find me a new quiz" → go to Step 9

## Step 2 — Load State

Read these files in parallel:
- `questions/bank.json`
- `profile/master-profile.json`
- `tracking/answered.json`
- `quizzes/test-library.json`

## Step 3 — Ask Session Length

Ask Cody: **"Quick (5 questions), standard (7), or deep dive (10+)?"**

Wait for response before proceeding.

## Step 4 — Select Questions

Build the question list using this algorithm:

1. Get today's date.
2. Filter `bank.json` to eligible questions:
   - Question ID is NOT in `answered.json`, OR days since last answered > cooldown_days
   - Depth-3 questions: only include if sessions_completed >= 4
3. Calculate confidence-based exclusions: if any axis has confidence >= 8, deprioritize (but don't fully exclude) questions that *only* score that axis.
4. Session depth bias:
   - Sessions 1–3: pick mostly depth-1, a couple depth-2
   - Sessions 4–10: mix depth-1 and depth-2, one depth-3
   - Sessions 11+: bias toward depth-2 and depth-3
5. Category limit: max 2 questions per category in a single session.
6. Fill the list to the chosen session length (5, 7, or 10+) by random selection within constraints.

## Step 5 — Run the Session

Present questions **one at a time** — this is a conversation, not a form.

For each question:
- State the question naturally (no "Question 3 of 7:" formatting)
- Accept free-form answers — don't force the user to pick a listed option
- If the question has a followup AND the answer warrants it, ask the followup
- Briefly acknowledge the answer before moving on (one sentence reaction is enough)
- If Cody says "skip" — log null and move on without pressure

After all questions, synthesize a **session summary** (3–5 sentences):
- What patterns emerged this session
- How answers connect to or update the existing profile
- Anything surprising or contradictory worth noting

Then:

**Update files:**
- Apply scoring deltas to `profile/master-profile.json`:
  - For each answer, add the delta values from `reveals.scoring` to the matching profile fields
  - Increment `confidence` counts for each axis touched
  - Increment `sessions_completed` and `questions_answered`
  - Update `last_updated` and append session metadata to `session_history`
  - Recalculate `derived_type` for MBTI (assign if abs(score) >= 2 and confidence >= 4 per axis)
  - Recalculate `quadrant_label` for political compass
  - Recalculate `top_values` in values_map (top 5 by score)
  - Append notable patterns if any emerged
- Update `tracking/answered.json`:
  - For each answered question: record `last_answered`, `times_answered`, `cooldown_days`, `eligible_again`
  - Update `total_answered` and `next_eligible_question_date`
- Write `sessions/session-{YYYY-MM-DD}-{4-char hex}.json` with:
  - session_id, date, session_number, question_count
  - For each question: question_id, question_text, answer, followup_answer, profile_deltas
  - session_summary, profile snapshot

**Write exports:**
- Overwrite `exports/master-profile.md` with current profile in human-readable markdown (see Export Format below)
- Write `exports/session-{N}-{YYYY-MM-DD}.md` with Q&A log and session summary

**Tell Cody:**
- "Session complete. I've updated your profile and written two export files:"
- `exports/master-profile.md` — paste into Standard Notes as 'WisdomBuilder — Master Profile' (replace previous)
- `exports/session-{N}-{date}.md` — paste into Standard Notes as 'WisdomBuilder — Session N — DATE' (keep all sessions)

**Check for quiz suggestion:**
- Compare `sessions_completed` against `recommended_at_session` in test-library.json
- If the next unstarted test's `recommended_at_session` <= sessions_completed, suggest it:
  "By the way — you're at session {N}. I'd recommend trying [{Test Name}]. Want to do it after today's session?"

**Check bank health:**
- Count eligible questions per category
- If any category has < 5 eligible questions: generate 5 new questions for that category following the bank.json schema, with `"source": "auto-generated"` and today's date, and append them to bank.json
- Tell Cody if you generated new questions: "I added 5 new [category] questions to the bank to keep things fresh."

## Step 6 — Show Profile

Display the master profile in readable format without running questions:

```
# Cody's Wisdom Profile
*Sessions: N | Questions answered: N | Last updated: DATE*

## MBTI
Type: [derived_type or "Building confidence..."]
  E/I: [score] ([label])   S/N: [score] ([label])
  T/F: [score] ([label])   J/P: [score] ([label])
[Online result if present]

## Political Compass
Quadrant: [label]
  Economic: [score] ([Left/Right])
  Social: [score] ([Libertarian/Authoritarian])
[Online result if present]

## Big Five
  Openness:          [score]/100
  Conscientiousness: [score]/100
  Extraversion:      [score]/100
  Agreeableness:     [score]/100
  Neuroticism:       [score]/100

## Moral Framework
[label]: [score] ([confidence] data points)

## Top Values
1. [value] ([score])
2. ...

## Game Persona
Dominant: [top archetype]
Alignment: [tendency or null]

## Notable Patterns
- [pattern]
- ...
```

## Step 7 — Export Profile

Generate `exports/profile-export-{YYYY-MM-DD}.md` containing:
- Full profile display (same as Step 6)
- Highlights from the top 5 most revealing answers across all sessions
- List of all sessions with dates
- Footer: `*Generated by WisdomBuilder on DATE | Sessions: N | Questions: N | Paste into Standard Notes Super editor.*`

Tell Cody the file is ready to paste as 'WisdomBuilder — Full Profile Export — DATE'.

## Step 8 — Add Quiz Results

Ask Cody which quiz and what the results were. Then:
- Update `quizzes/results.json` with the new result entry
- Apply results to `profile/master-profile.json` as high-confidence anchors:
  - Set confidence to 15 (maximum weight) for the relevant axes
  - Translate percentages/coordinates to the profile's scoring scale
  - For MBTI: e.g., 72% Introverted → mbti_ei = -7.2, confidence.ei = 15
  - For Political Compass: coordinates map directly to political_economic and political_social
  - For Big Five: percentile scores map to big5_* values (percentile = score directly)
- Mark the test as completed in `quizzes/test-library.json`
- Overwrite `exports/master-profile.md`
- Tell Cody: "Results saved. Your profile has been updated with high-confidence anchors from [{Quiz Name}]."

## Step 9 — Find New Quiz

Use WebSearch to find 2–3 interesting personality/psychology tests not already in test-library.json.
Evaluate for: scientific backing, free access, relevance to dimensions not yet well-covered.
Append findings to `quizzes/test-library.json` with the next available `recommended_at_session` slot.
Report new additions to Cody with a brief description of each.

## Export Format for master-profile.md

```markdown
# WisdomBuilder — Cody's Master Profile
*Sessions: N | Questions answered: N | Last updated: DATE*

## MBTI
**[Type or "Emerging..."]** (confidence: [low/moderate/high])
- E/I score: [score] → leans [Introvert/Extravert/balanced]
- S/N score: [score] → leans [Sensing/iNtuitive/balanced]
- T/F score: [score] → leans [Thinking/Feeling/balanced]
- J/P score: [score] → leans [Judging/Perceiving/balanced]
[Online: 16Personalities result if present]

## Political Compass
**[Quadrant]** (confidence: [low/moderate/high])
- Economic: [score] ([Left/Center/Right])
- Social: [score] ([Libertarian/Center/Authoritarian])
[Online: PoliticalCompass result if present]

## Big Five (OCEAN)
- Openness:          [score]/100
- Conscientiousness: [score]/100
- Extraversion:      [score]/100
- Agreeableness:     [score]/100
- Neuroticism:       [score]/100

## Moral Framework
**[Utilitarian/Deontological/Mixed]** — score: [N], confidence: [N] data points
[Key observations]

## Values Map
**Top values:**
1. [Value]: [score]
2. [Value]: [score]
3. [Value]: [score]
4. [Value]: [score]
5. [Value]: [score]

## Game Persona
- Dominant archetype: [archetype]
- Alignment tendency: [Good/Neutral/Chaotic/null]
- Notable: [one-line summary]

## Notable Patterns
- [pattern]
- [pattern]

## Sessions
[List of sessions with dates and brief descriptions]

---
*Paste into Standard Notes Super editor as: WisdomBuilder — Master Profile*
*Replace previous version — this is the live document.*
```

## Rules

- Ask questions one at a time — never dump all questions in one message
- Never tell Cody what an answer "means" mid-session — save interpretation for the summary
- If Cody skips a question, log null, move on, no comment
- Only update profile with actual scored answers — never fabricate deltas
- Never run more than 2 questions from the same category per session
- Never repeat a question within its cooldown window
- Present profile as a living snapshot, not a verdict
- Begin the session immediately after session length is chosen — don't ask "are you ready?"
