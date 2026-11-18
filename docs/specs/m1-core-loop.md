# M1: Core loop

Goal: add Norwegian words by hand and review them with SM-2 spaced repetition. No external APIs.
At the end of M1 the app is usable every day, locally.

## Out of scope

Dictionary lookups (M2), LLM sentences (M3), deployment and CI (M4), stats (M5), auth, multiple users.

## Data model

### Word

| Field         | Type             | Rules                                                  |
|---------------|------------------|--------------------------------------------------------|
| Id            | Guid             |                                                        |
| Lemma         | string           | Required, trimmed, max 100 chars. Example: `bok`       |
| PartOfSpeech  | enum             | Noun, Verb, Adjective, Adverb, Phrase, Other           |
| Gender        | enum, nullable   | Masculine, Feminine, Neuter. Only allowed for nouns    |
| Translation   | string           | Required, max 200 chars. English or French, free text  |
| Notes         | string, nullable | Max 1000 chars                                         |
| CreatedAt     | DateTimeOffset   | UTC                                                    |

Lemma + PartOfSpeech is unique (case-insensitive). `bok` the noun can exist once.

### Card

Creating a word creates two cards, one per direction.

| Field        | Type           | Notes                                       |
|--------------|----------------|---------------------------------------------|
| Id           | Guid           |                                             |
| WordId       | Guid           | FK, cascade delete                          |
| Direction    | enum           | NorwegianToTranslation, TranslationToNorwegian |
| Repetitions  | int            | Starts at 0                                 |
| EaseFactor   | double         | Starts at 2.5, never below 1.3              |
| IntervalDays | int            | Starts at 0                                 |
| DueAt        | DateTimeOffset | Starts at creation time (due immediately)   |

### ReviewLog

One row per review. Not shown in M1, but needed for stats in M5.

| Field              | Type           |
|--------------------|----------------|
| Id                 | Guid           |
| CardId             | Guid           |
| Grade              | enum           |
| ReviewedAt         | DateTimeOffset |
| IntervalDaysBefore | int            |
| IntervalDaysAfter  | int            |
| EaseFactorBefore   | double         |
| EaseFactorAfter    | double         |

## Scheduling (SM-2)

The UI shows four buttons. Each maps to an SM-2 quality score `q`:

| Grade | q |
|-------|---|
| Again | 1 |
| Hard  | 3 |
| Good  | 4 |
| Easy  | 5 |

When a card is reviewed at time `now`:

If `q < 3` (Again):
- `Repetitions = 0`
- `IntervalDays = 1`
- `EaseFactor` unchanged

If `q >= 3`:
- `IntervalDays` = 1 if `Repetitions == 0`, 6 if `Repetitions == 1`, otherwise `round(IntervalDays * EaseFactor)` using the ease factor from before this review
- `Repetitions += 1`
- `EaseFactor = EaseFactor + (0.1 - (5 - q) * (0.08 + (5 - q) * 0.02))`, then clamp to a minimum of 1.3

In all cases: `DueAt = now + IntervalDays days`.

Rounding is to the nearest integer, midpoint away from zero.

The scheduler is a pure function in Core: it takes the current card state, a grade and `now`, and returns the new state. No database access.

### Scheduler test cases

| # | Starting state (reps, interval, EF) | Grade | Expected (reps, interval, EF) |
|---|-------------------------------------|-------|-------------------------------|
| 1 | 0, 0, 2.5                           | Good  | 1, 1, 2.5                     |
| 2 | 1, 1, 2.5                           | Good  | 2, 6, 2.5                     |
| 3 | 2, 6, 2.5                           | Good  | 3, 15, 2.5                    |
| 4 | 0, 0, 2.5                           | Easy  | 1, 1, 2.6                     |
| 5 | 0, 0, 2.5                           | Hard  | 1, 1, 2.36                    |
| 6 | 3, 15, 2.5                          | Again | 0, 1, 2.5                     |
| 7 | 2, 6, 2.36                          | Hard  | 3, 14, 2.22                   |
| 8 | 5, 30, 1.35                         | Hard  | 6, 41, 1.3 (clamped)          |

Plus: `DueAt` equals `now + interval` in each case (use a fake `TimeProvider`).

Compare doubles with a tolerance (for example 0.0001).

## API

| Method | Route                     | Behavior                                                           |
|--------|---------------------------|--------------------------------------------------------------------|
| POST   | /api/words                | Create word + its two cards. 201 with the word. 409 on duplicate   |
| GET    | /api/words                | List words, newest first. Optional `?search=` on lemma/translation |
| GET    | /api/words/{id}           | 404 if missing                                                     |
| PUT    | /api/words/{id}           | Update fields. Scheduling state of cards is untouched              |
| DELETE | /api/words/{id}           | Deletes word, its cards and their review logs. 204                 |
| GET    | /api/reviews/due?limit=20 | Cards with `DueAt <= now`, oldest due first, with word data        |
| POST   | /api/cards/{id}/review    | Body `{ "grade": "Good" }`. Applies scheduler, writes a ReviewLog, returns the updated card |

Validation errors return 400 with ProblemDetails. Gender on a non-noun is a validation error.

## Frontend

Two pages:

**Words**
- Form: lemma, part of speech, gender (only visible when part of speech is Noun), translation, notes.
- List below the form with search. Edit and delete per row.

**Review**
- Shows the count of due cards.
- Shows the front of one card (lemma with en/ei/et for nouns, or the translation, depending on direction).
- "Show answer" reveals the back, then the four grade buttons appear.
- After grading, the next card loads. When nothing is due: "Ingen kort igjen i dag" plus when the next card is due.

No styling library needed. Plain CSS is fine.

## Tasks (in order)

1. Solution skeleton: the projects from CLAUDE.md, Docker Compose with Postgres, empty React app. Builds and runs.
2. Scheduler in Core, test first, using the table above.
3. Entities, DbContext, first migration.
4. Word endpoints + integration tests (create, duplicate 409, gender validation, delete cascade).
5. Review endpoints + integration tests (due query respects `now`, review writes a log).
6. Words page.
7. Review page.
8. README: what it is, how to run it, screenshot.

## Done when

- `dotnet test` passes, including all 8 scheduler cases.
- I can add 10 words, review them, and see them come back on the right days (checked by moving the fake clock in tests, not by waiting).
- Fresh clone + the README commands gets the app running.
