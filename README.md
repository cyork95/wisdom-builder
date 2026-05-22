# WisdomBuilder

A conversational personality profiling system for [Claude Code](https://claude.ai/code). Ask yourself hard questions every day. Build a living archive of who you actually are.

---

## What it does

WisdomBuilder runs question sessions through Claude Code. Each session delivers a mix of philosophical dilemmas, MBTI-style questions, political compass questions, video game persona questions, moral thought experiments, and weird hypotheticals. Your answers accumulate into a master profile that tracks:

- **MBTI** — derived from answers, then anchored by 16Personalities
- **Big Five (OCEAN)** — openness, conscientiousness, extraversion, agreeableness, neuroticism
- **Political Compass** — economic and social axes
- **Moral framework** — utilitarian vs deontological
- **Values map** — security, authenticity, honesty, loyalty, meaning, and more
- **Game persona** — how you play reveals how you think

After every session, Markdown exports are written for manual paste into Standard Notes (or anywhere else).

---

## How to use it

### 1. Clone this repo

```bash
git clone https://github.com/YOUR_USERNAME/wisdombuilder.git
cd wisdombuilder
```

### 2. Initialize your data files

Copy the template files to start your own profile:

```bash
cp profile/master-profile.template.json profile/master-profile.json
cp tracking/answered.template.json tracking/answered.json
cp quizzes/results.template.json quizzes/results.json
```

Or create them fresh — schemas are documented below.

### 3. Install the skill file

Copy the skill trigger to your Claude Code global skills folder:

**Windows:**
```
C:\Users\YOUR_USERNAME\.claude\skills\wisdom-builder.md
```

**Mac/Linux:**
```
~/.claude/skills/wisdom-builder.md
```

The skill file is at `skills/wisdom-builder.md` in this repo.

### 4. Start a session

Open Claude Code in the WisdomBuilder directory and say any of:

> "give me today's wisdom questions"  
> "let's do a deep dive"  
> "wisdom builder"  
> "show my profile"

Claude will ask how many questions you want — **Quick (5)**, **Standard (7)**, or **Deep Dive (10+)** — then run the session.

---

## File structure

```
WisdomBuilder/
├── CLAUDE.md                          ← Instructions for Claude Code
├── README.md                          ← This file
├── questions/
│   └── bank.json                      ← 100+ questions with full scoring metadata
├── profile/
│   └── master-profile.json            ← Your running personality profile (gitignored)
├── sessions/
│   └── session-YYYY-MM-DD-{hex}.json  ← One file per completed session (gitignored)
├── tracking/
│   └── answered.json                  ← Cooldown tracker (gitignored)
├── exports/
│   └── master-profile.md              ← Markdown export for Standard Notes (gitignored)
│   └── session-N-YYYY-MM-DD.md        ← Session logs (gitignored)
└── quizzes/
    ├── test-library.json              ← Scheduled online test recommendations
    └── results.json                   ← Your quiz results (gitignored)
```

Personal data files are gitignored by default. The question bank and system files are what's shared.

---

## Question bank schema

Every question in `questions/bank.json` follows this schema:

```json
{
  "id": "Q001",
  "text": "The question text.",
  "category": "moral_dilemma",
  "subcategory": "trolley_problem",
  "depth": 1,
  "format": "binary_with_followup",
  "options": ["Option A", "Option B"],
  "followup": "A followup question, or null.",
  "reveals": {
    "dimension": "moral_framework",
    "axis": "utilitarian_vs_deontological",
    "scoring": {
      "Option A": { "utilitarian_vs_deontological": 2 },
      "Option B": { "utilitarian_vs_deontological": -2 }
    },
    "notes": "What this question is actually probing."
  },
  "tags": ["classic", "trolley"],
  "cooldown_days": 90,
  "source": "curated"
}
```

**Categories:** `moral_dilemma`, `mbti_style`, `political_compass`, `video_game_persona`, `values_priorities`, `philosophical`, `aesthetic`, `hypothetical`, `historical`, `quick_binary`

**Depth:** 1 = surface/fun, 2 = moderate, 3 = deep/confronting

**Formats:** `binary`, `binary_with_followup`, `scale_1_5`, `ranked_list`, `open_ended`

---

## Scoring reference

| Dimension | Scale | Direction |
|---|---|---|
| `mbti_ei` | -10 to +10 | negative = Introvert, positive = Extravert |
| `mbti_sn` | -10 to +10 | negative = iNtuitive, positive = Sensing |
| `mbti_tf` | -10 to +10 | negative = Feeling, positive = Thinking |
| `mbti_jp` | -10 to +10 | negative = Perceiving, positive = Judging |
| `political_economic` | -10 to +10 | negative = Left, positive = Right |
| `political_social` | -10 to +10 | negative = Libertarian, positive = Authoritarian |
| `big5_*` | 0–100 | initialized at 50, each question shifts ±1–5 |
| `utilitarian_vs_deontological` | -10 to +10 | negative = Deontological, positive = Utilitarian |
| values clusters | 0–100 | cumulative signal strength, initialized at 0 |

**Derived MBTI type:** assigned when `abs(score) >= 2` and `confidence >= 4` on that axis.  
**Online quiz results** act as high-confidence anchors — confidence set to 15, score set proportionally.

---

## Scheduled online tests

WisdomBuilder recommends external tests at milestone sessions to anchor the profile with validated external data:

| Session | Test | URL |
|---|---|---|
| 3 | 16Personalities (MBTI) | 16personalities.com/free-personality-test |
| 5 | Political Compass | politicalcompass.org/test |
| 7 | Big Five IPIP-NEO | openpsychometrics.org/tests/IPIP-NEO |
| 9 | Truity Enneagram | truity.com/test/enneagram-personality-test |
| 10 | VIA Character Strengths | viacharacter.org/surveys/takesurvey |
| 12 | Attachment Style | attachmentstyletest.com |
| 14 | Love Languages | 5lovelanguages.com/quizzes |
| 16 | Dark Triad (IDRlabs) | idrlabs.com/dark-triad/test.php |
| 18 | Moral Foundations | yourmorals.org |
| 20 | CliftonStrengths | gallup.com/cliftonstrengths |

Results are stored in `quizzes/results.json` and used to recalibrate the profile dimensions.

---

## Session flow

1. Claude loads `bank.json`, `master-profile.json`, and `answered.json`
2. Asks how many questions you want
3. Selects questions — filters cooldowns, avoids over-sampled axes, enforces max 2 per category
4. Presents questions one at a time conversationally
5. After the last question: synthesizes a session narrative
6. Updates `master-profile.json`, `tracking/answered.json`, and writes `sessions/session-{date}-{hex}.json`
7. Writes Markdown exports to `exports/`
8. Suggests any due online tests

**Special commands:**
- `"show my profile"` — displays current profile without running questions
- `"export my profile"` — writes a full profile export to `exports/`
- `"add quiz results"` — updates profile with online test results

---

## Standard Notes export

| File | Standard Notes title |
|---|---|
| `exports/master-profile.md` | `WisdomBuilder — Master Profile` *(replace each session)* |
| `exports/session-N-YYYY-MM-DD.md` | `WisdomBuilder — Session N — YYYY-MM-DD` *(keep all)* |

---

## Contributing

The question bank is the heart of this project. If you write a great question that reveals something real, open a PR with the full schema filled out — dimension, axis, scoring, and notes included.

Questions that don't have complete `reveals` metadata won't be merged — the scoring engine needs it to update the profile correctly.

---

## License

MIT. Build your own version, fork it, change the questions. The point is to know yourself better.
