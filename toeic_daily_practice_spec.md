# TOEIC Daily Practice App — Implementation Specification

## 1. Purpose

Build a private, responsive web application for daily TOEIC-style English practice. The application must work well in desktop and mobile browsers, and be installable as a Progressive Web App (PWA).

The product is a personal study tool, not an official TOEIC service. It must generate **original questions in a TOEIC-like format** and must not reproduce or claim to use official ETS questions, passages, or scoring.

## 2. Learning Profile and Product Goals

### 2.1 Initial settings

| Setting | Value |
| --- | --- |
| Reading starting band | TOEIC LR 700–750 |
| Long-term Reading goal | TOEIC LR 750 |
| Writing target | Combined S&W target: 250 (Writing-focused in this app) |
| Daily study time | Approximately 15–25 minutes |
| Primary Reading emphasis | Part 5 grammar, vocabulary, collocations, and preposition traps |

### 2.2 Reading topics to emphasize

Prioritize the following topics, especially for Part 5:

- Parts of speech and word form.
- Verb tense, voice, and verb form.
- Subject–verb agreement.
- Conjunctions versus prepositions.
- Preposition usage and collocations.
- Relative clauses and reduced clauses.
- Comparisons and conditionals.
- Business vocabulary and near-synonym discrimination.

### 2.3 Product outcomes

The user must be able to:

1. Open the app on a PC or smartphone and solve the daily practice set.
2. Receive immediate, deterministic grading for Reading answers.
3. Submit Writing answers and receive rubric-based automatic feedback.
4. Review history, weak skills, completed days, and trends.
5. Receive slightly harder questions after sustained strong performance, per skill rather than globally.

## 3. Scope

### 3.1 Version 1 (required)

- Authentication for one or more users.
- One daily set per user and study date.
- Reading Part 5, Part 6, and Part 7 questions.
- One Writing task per day.
- Immediate Reading grading and explanations.
- AI-assisted Writing scoring with actionable feedback.
- Study history and weak-skill summaries.
- Adaptive difficulty by skill.
- PWA manifest, responsive layout, and offline shell caching.

### 3.2 Out of scope for Version 1

- Listening audio generation and Listening questions.
- Speaking recording/scoring.
- Official TOEIC score prediction or official score equivalence claims.
- Social features, leaderboards, or sharing.
- Multi-user classroom administration.

## 4. Recommended Technical Architecture

| Layer | Technology | Responsibility |
| --- | --- | --- |
| Front end | React + TypeScript + Vite | Responsive UI, PWA, client-side interaction |
| UI styling | CSS modules or Tailwind CSS | Accessible, responsive visual system |
| API and scheduled work | Cloudflare Workers | HTTP APIs, scheduled daily-set generation, authorization |
| Database | Cloudflare D1 | Relational persistence for users, questions, attempts, and skill state |
| Authentication | Cloudflare-compatible OAuth provider (e.g. Google OAuth) | Stable user identity across PC and smartphone |
| AI provider | OpenAI Responses API | Structured question generation and Writing evaluation |
| Secrets | Worker secrets / platform environment variables | API keys and OAuth credentials; never expose keys to the browser |

Use a single Worker application for the API and scheduled job in Version 1. Use Cloudflare Cron Trigger at **19:00 UTC**, which is **04:00 JST** on the following study date.

## 5. User Experience

### 5.1 Primary screens

| Screen | Required content and behavior |
| --- | --- |
| Today | Current set, section switcher, question progress, timer, answer controls |
| Reading result | Correct/incorrect state, correct answer, concise explanation, save-for-review control |
| Writing task | Prompt, required content points, word count, editable text area, submit control |
| Writing result | Rubric scores, total score, feedback, corrected/improved example, retry option |
| History | Daily completion, Reading accuracy, Writing scores, streak, and skill trends |
| Review | Previously missed or bookmarked questions, grouped by skill |
| Settings | Target band, daily generation time display, and account settings |

### 5.2 Daily set composition

Start with this default composition. The counts must be configurable by server-side configuration.

| Section | Default quantity | Target time | Notes |
| --- | ---: | ---: | --- |
| Reading Part 5 | 10 questions | 8 min | Four-choice grammar/vocabulary questions |
| Reading Part 6 | 1 passage / 4 questions | 4 min | Grammar, context, and sentence insertion |
| Reading Part 7 | 1 short business passage / 3 questions | 4 min | Email, notice, or short document |
| Writing | 1 task | 8–10 min | Email response by default; opinion task available as a rotation |

The initial experience should visibly label the practice band as **700–750**. Do not display an unsupported conversion such as “your official TOEIC score is X.”

### 5.3 Interaction requirements

- Reading: a user selects one option and presses **Check answer**. The app then locks the answer, shows correct/incorrect styling, correct answer, and explanation.
- Writing: the user writes and submits an answer. Display a loading state while evaluation is pending. Save both the raw response and evaluation result.
- Progress: persist each answer immediately; page refresh must not lose completed answers.
- Mobile: no horizontal scrolling at 320 CSS pixels wide. Main body text must be at least 16px.
- Accessibility: keyboard-accessible choices and buttons, visible focus states, and semantic labels.

## 6. Adaptive Difficulty

### 6.1 Skill model

Maintain a separate `ability` value for each user and skill. Initial ability is `5.0` on a 1.0–10.0 scale.

Required Reading skills:

- `part5_word_form`
- `part5_verb_form`
- `part5_preposition`
- `part5_conjunction`
- `part5_vocabulary`
- `part6_context`
- `part7_reading_comprehension`

Required Writing skills:

- `writing_task_achievement`
- `writing_organization`
- `writing_vocabulary_grammar`

### 6.2 Adjustment rules

After each completed daily set, calculate accuracy for each skill using the latest 20 graded items for that skill.

| Condition | Adjustment |
| --- | --- |
| Accuracy >= 85% for two consecutive completed sets | `ability += 0.25` |
| Accuracy from 60% to 84% | no change |
| Accuracy < 60% | `ability -= 0.25` |

Clamp ability to `1.0..10.0`. Change at most once per skill per daily set. For Writing, calculate the percentage from the rubric total.

The next generated item for a skill must use the user’s current ability rounded to the nearest 0.5. A daily set must contain at least 70% questions at the current level and up to 30% from adjacent levels for variety.

## 7. Question Generation and Quality Controls

### 7.1 Generation timing

At the scheduled time, generate the next study date’s set for every active user. A request to `GET /api/daily-sets/:date` must generate a missing set on demand as a fallback, using the same idempotency key.

### 7.2 Generation contract

Call the AI provider from the Worker only. Require strict structured output (JSON Schema) and validate it again in application code before storing it.

Every Reading item must include:

```json
{
  "section": "part5",
  "skill": "part5_preposition",
  "difficulty": 5.0,
  "prompt": "The marketing team will finalize the schedule ______ the client approves the proposed budget.",
  "choices": [
    { "id": "A", "text": "unless" },
    { "id": "B", "text": "once" },
    { "id": "C", "text": "despite" },
    { "id": "D", "text": "during" }
  ],
  "correctChoiceId": "B",
  "explanation": "...",
  "tags": ["conjunction", "business"]
}
```

Every Writing task must include:

```json
{
  "taskType": "email_response",
  "difficulty": 5.0,
  "prompt": "...",
  "requiredPoints": ["state when the report will be sent", "add one useful detail"],
  "recommendedWordRange": { "min": 30, "max": 50 },
  "rubric": ["task_achievement", "organization", "vocabulary_grammar"]
}
```

### 7.3 Prompt constraints

The generation prompt must require:

- Original, fictional business contexts only.
- No copyrighted or official TOEIC source material.
- Exactly one unambiguous correct answer for a Reading item.
- Plausible distractors with only one grammatically and contextually correct option.
- A short, accurate explanation written for a Japanese learner.
- Difficulty appropriate to the requested skill and level.
- No culturally insensitive, unsafe, or personal-data-based content.

Validate Reading content after generation. Reject and regenerate an item when:

- It has duplicate choices.
- The correct option is missing.
- It has more or fewer than four choices for Part 5/6/7 multiple-choice items.
- Required fields are missing.
- The same normalized prompt was used by the same user within the previous 180 days.

Generation failures must not block study: use the most recent unused, pre-generated fallback question for the same skill and difficulty.

## 8. Grading

### 8.1 Reading grading

Reading grading must be deterministic in application code:

```text
isCorrect = submittedChoiceId === question.correctChoiceId
```

Do not call an AI model to determine whether a multiple-choice answer is correct. Store the submitted answer, answer timestamp, result, and elapsed time.

### 8.2 Writing grading

The model must return structured evaluation data with this rubric:

| Dimension | Maximum | Definition |
| --- | ---: | --- |
| Task achievement | 3 | Addresses each required point and is understandable |
| Organization | 2 | Clear greeting/body/closing or logical paragraph structure |
| Vocabulary and grammar | 3 | Accurate and appropriate language for the task |
| Total | 8 | Sum of dimensions |

Required evaluation payload:

```json
{
  "scores": {
    "taskAchievement": 0,
    "organization": 0,
    "vocabularyGrammar": 0,
    "total": 0
  },
  "strengths": ["..."],
  "improvements": ["..."],
  "correctedVersion": "...",
  "briefJapaneseFeedback": "..."
}
```

The evaluator must not rewrite the learner’s answer without preserving the original separately. Feedback must be constructive and concise. Explicitly state when a score is an **app practice score**, not an official TOEIC result.

## 9. Data Model

Use D1 migrations. Store timestamps in UTC ISO 8601 format and calculate `study_date` in JST.

### 9.1 Tables

| Table | Key fields |
| --- | --- |
| `users` | `id`, `email`, `display_name`, `timezone`, `created_at` |
| `user_settings` | `user_id`, `reading_target_band`, `sw_target`, `daily_generation_hour_jst` |
| `skills` | `id`, `name`, `section` |
| `user_skill_state` | `user_id`, `skill_id`, `ability`, `updated_at` |
| `daily_sets` | `id`, `user_id`, `study_date`, `status`, `generated_at`, `completed_at` |
| `questions` | `id`, `section`, `skill_id`, `difficulty`, `content_json`, `content_hash`, `created_at` |
| `daily_set_items` | `id`, `daily_set_id`, `question_id`, `position` |
| `reading_attempts` | `id`, `daily_set_item_id`, `submitted_choice_id`, `is_correct`, `elapsed_seconds`, `submitted_at` |
| `writing_attempts` | `id`, `daily_set_item_id`, `response_text`, `word_count`, `evaluation_json`, `submitted_at`, `evaluated_at` |
| `review_bookmarks` | `user_id`, `question_id`, `created_at` |

Enforce `UNIQUE(user_id, study_date)` on `daily_sets`. Enforce one completed attempt per item in Version 1, except that Writing may retain multiple revisions with an incrementing `attempt_number`.

## 10. API Contract

All endpoints require an authenticated user unless stated otherwise.

| Method | Endpoint | Behavior |
| --- | --- | --- |
| `GET` | `/api/daily-sets/today` | Return today’s set and existing attempts; create it only if missing |
| `GET` | `/api/daily-sets/:studyDate` | Return a past or current set owned by the user |
| `POST` | `/api/reading-attempts` | Save and grade one Reading answer; return result and explanation |
| `POST` | `/api/writing-attempts` | Save response, queue/evaluate it, and return evaluation |
| `POST` | `/api/review-bookmarks` | Add or remove a review bookmark |
| `GET` | `/api/history` | Return date-based summary and streak information |
| `GET` | `/api/skills` | Return per-skill accuracy, current ability, and recent trend |
| `PUT` | `/api/settings` | Update target band and non-sensitive preferences |

Use a request idempotency key for `POST /api/reading-attempts` and `POST /api/writing-attempts` to prevent duplicate records on mobile retries.

## 11. Front-End Requirements

### 11.1 Visual direction

Use a focused study interface rather than a marketing landing page:

- Dark navy application header with a compact streak display.
- One prominent question card.
- Clear tags for section, target band, and skill.
- Choice buttons large enough for touch input.
- Green correct and red incorrect states with text labels; do not rely on color alone.
- Avoid unnecessary charts on the Today screen.

### 11.2 PWA

- Include `manifest.webmanifest` with app name, icons, theme color, and standalone display mode.
- Use a service worker to cache the application shell and static assets.
- Do not cache authenticated API responses containing study content unless keyed and safely invalidated for the logged-in user.
- If offline, show the last available daily set as read-only and clearly label that submission requires a connection.

## 12. Security and Privacy

- Keep AI and OAuth secrets in server-side platform secrets only.
- Authorize every record lookup by authenticated `user_id`.
- Do not put answer keys in client payloads before the Reading answer is submitted.
- Rate-limit generation and Writing evaluation endpoints per user.
- Log only operational metadata; do not log full Writing text in general request logs.
- Provide a user action to delete their study history in a future version; design the schema so deletion by `user_id` is possible.

## 13. Observability and Failure Handling

- Log generation job start/end, number of generated sets, rejected items, fallback use, and AI latency/cost metadata.
- Persist a job status for each generated set so failed jobs can be retried safely.
- If Writing evaluation fails, preserve the submitted text and show “Evaluation is pending. Retry evaluation.”
- Never silently generate a second daily set for the same user/date.

## 14. Implementation Plan

1. Create the React/TypeScript PWA shell, authentication, D1 schema, and API routing.
2. Implement daily set storage and deterministic Reading grading using seeded development questions.
3. Implement Today, Reading result, History, and Review screens.
4. Add structured AI generation with validation and fallback questions.
5. Add structured Writing evaluation, feedback UI, and retry handling.
6. Implement Cron-triggered generation and adaptive difficulty updates.
7. Add PWA offline shell behavior, accessibility review, responsive QA, and production monitoring.

## 15. Acceptance Criteria

The implementation is complete when all of the following are true:

- A signed-in user can open the same account from both desktop and smartphone and see the same set and history.
- At 04:00 JST, the next day’s set is generated automatically; an on-demand fallback also works.
- The default set contains 10 Part 5 questions, one Part 6 passage, one Part 7 passage, and one Writing task.
- Reading answers are immediately and correctly graded without an AI grading call.
- Writing submission returns structured rubric feedback and an improved example.
- A completed answer persists across a page refresh.
- History shows completion, accuracy, Writing practice scores, streak, and skill-level trends.
- Sustained >=85% performance raises only the relevant skill difficulty in 0.25 increments.
- The UI is usable at 320px wide and with keyboard navigation.
- No API secret or answer key is exposed before the user submits an answer.

