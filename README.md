# Studyforge — Phase 1

AI-powered textbook-to-quiz platform. This is **Phase 1: Core UI** —
no backend wiring yet, on purpose (see "What's real vs. not" below).

## What's built so far

**Phase 1 — Core UI**
- Landing page, dashboard (honest empty states), quiz configuration
  page (full question-mix/marks/difficulty controls, live totals)
- Design system: Button, Card, Badge, Stepper, Segmented, Switch,
  ProgressBar
- Shared types (`src/types`) and config (`src/lib/config.ts`) matching
  the full product's data model

**Phase 2 — Real textbook analysis**
- Upload page: drag/drop, multi-file, camera capture, reorder/rotate/
  delete, plus a **real** (not simulated) client-side quality check —
  brightness + Laplacian-variance blur detection on a downsampled
  canvas — that flags likely-unreadable pages before they're sent
  anywhere
- `POST /api/textbook/analyze` — secure server route. Sends page
  images to Gemini (`ANALYSIS_MODEL` in `src/lib/config.ts`), strips
  markdown fences, and validates the response against a Zod schema
  (`src/schemas/textbook.ts`) before it ever reaches the client. If
  the page count in the response doesn't match the request, or the
  JSON doesn't match the schema, the request is rejected rather than
  passed through
- `GEMINI_API_KEY` is read only inside this route via `server-only` +
  a small wrapper (`src/lib/gemini.ts`) — never bundled client-side.
  Missing the key returns a clear 503, not a crash or a fake result
- A simple in-memory per-IP rate limiter on the analyze route (5
  requests/minute) — a placeholder until Phase 7 auth gives us real
  per-user quotas
- Results view (`AnalysisResults.tsx`) shows the structured output
  per page: chapter/section, summary, key concepts, formulas,
  detected diagrams, and a readability-confidence badge Gemini itself
  reports

**Phase 3 — Quiz generation, validation, repair**
- `POST /api/quiz/generate` — takes the Phase 2 analysis (re-validated
  server-side, not trusted from the client as a raw string) + the
  user's `QuizConfig`, builds a compact grounding text from the
  structured analysis (`src/lib/buildSourceContent.ts`), and prompts
  Gemini (model chosen via `DIFFICULTY_MODEL_MAP`) for exactly the
  requested MCQ/short/creative counts
- Marks and question ids are **assigned server-side** from the user's
  config (`src/lib/quizAssembly.ts`) — never trusted from the model
- A deterministic, rule-based quality checker
  (`src/lib/quizValidation.ts`) — not another AI call — enforces
  exactly-4-options/exactly-1-correct on every MCQ, catches duplicate
  questions/options, and flags incomplete short/creative questions
- Failing questions go through an automatic **repair pipeline**: each
  bad question is regenerated individually (up to 2 attempts) against
  the same source content, grounded by a prompt that states exactly
  what was wrong. If a question still can't be fixed, it's dropped
  from the quiz rather than shown broken — the response says how many
  were dropped and why
- `POST /api/quiz/repair` — the same repair logic exposed standalone,
  used by the preview page's per-question **Regenerate** button (a
  real, working action, not a placeholder)
- Quiz preview page (`/quiz/preview`) shows the generated quiz with
  live marks totals, source-page badges, and working regeneration

## Data flow across pages right now

There's still no database (that's Phase 7), so `src/lib/
textbookSession.ts` passes the analysis → config → quiz through
`sessionStorage` between `/upload` → `/quiz/configure` → `/quiz/
preview`. This is called out explicitly in the code as a stand-in,
not a real persistence layer — closing the tab loses it.

**Phase 4 — Quiz-taking experience**
- `/quiz/take` — real answer UI for all three question types: MCQ with
  an animated selection tick, short-answer textarea, and creative A–D
  with per-part answer boxes and marks shown
- Question navigator grid (answered / unanswered / marked-for-review,
  jump to any question) plus Previous/Next
- **Mark for review** toggle per question
- **Autosave** — every answer writes to `localStorage` keyed by quiz
  id, immediately, via `attemptStorage.ts`. Refresh or close the tab
  and reopen `/quiz/take`: you're offered "Continue your unfinished
  quiz?" before anything is shown
- **Exam mode with a user-set timer** — practice mode has no timer;
  exam mode requires one, chosen from 20/30/60-minute presets *or* a
  custom value (1–300 min) typed in on the configure page. The timer
  is stored as an absolute expiry timestamp (not "minutes remaining"),
  so a refresh can't be used to reset the clock. Hitting zero
  auto-submits
- **Submit confirmation** — warns about unanswered questions with
  Cancel / Review unanswered / Submit anyway, matching spec section 18
  exactly
- Submission is honest about what it isn't: `/quiz/submitted` shows
  exactly what was recorded per question, with AI grading clearly
  marked "wired up in Phase 5" rather than faked

**Phase 5 — AI grading + local Quiz History**
- `POST /api/quiz/grade`:
  - MCQ is graded **deterministically, no AI call** — correct/incorrect
    by comparing the submitted option id to the stored correct id. A
    pure-MCQ quiz grades fully offline, no `GEMINI_API_KEY` needed at
    all (verified — it returns a real score with no key set)
  - Short answers are graded by Gemini against the question's own
    rubric and model answer, grounded in the same source content used
    for generation. Partial credit is explicit, not exact-match
  - Creative A–D parts are graded independently, each against its own
    model answer, in one call per question
  - Every subjective grade carries a **confidence** rating; low
    confidence sets `needsReview: true`, surfaced in the UI as "AI
    grading may need review" rather than presented as certain
  - A failed grading call for one question returns a flagged 0 with
    "please review manually" — it never silently drops or fakes a score
- Results page (`/quiz/results`) — animated score reveal, MCQ/short/
  creative breakdown, per-question correct/partial/wrong review with
  feedback, confidence flags surfaced per question
- **Quiz History** (`/history`, new nav link, and a real "Recent
  quizzes" list + accurate average-score stat on the dashboard) —
  lists every quiz you've submitted on this device, graded or not,
  with a score once graded. Explicitly labeled "stored on this device
  only" — this is `localStorage`, not an account, and the README says
  so rather than implying it syncs anywhere
- Generated quizzes are now archived to `localStorage`
  (`src/lib/quizArchive.ts`) in addition to the per-tab session cache,
  so History links still open correctly after the tab that generated
  them is closed

**Phase 6 — Learning loop**
- **Real weak-topic detection** (`src/lib/weakTopics.ts`) — walks every
  graded quiz in local history, sums marks earned vs. possible per
  topic (using the topic each question was actually tagged with at
  generation time), and flags anything scoring below 70% with at least
  2 marks of evidence (so one lucky/unlucky question can't flag a
  topic). This is pure aggregation of real grading data — nothing
  estimated or seeded
- `/practice/weak-areas` — shows detected weak topics grouped by the
  textbook they came from, with a working "Practice these topics"
  button that regenerates a quiz biased toward exactly those topics
  (`QuizConfig.focusTopics`, threaded through the Phase 3 generation
  prompt). Wired into both the dashboard quick actions and the
  dashboard's "Weak topics" card, which now shows real percentages
  instead of an empty state once you have graded data
- **Flashcards, first pass**: `POST /api/flashcards/generate` produces
  front/back cards grounded in the analyzed textbook (same source-
  grounding discipline as quiz generation — no invented facts). `/
  flashcards` has a flip animation, deck list, and Know it / Almost /
  Don't know review buttons. Review state uses a simple fixed-interval
  schedule (0 / 3 / 7 days) — deliberately not a full spaced-repetition
  algorithm yet, but the storage shape (`src/app/flashcards/
  deckStorage.ts`) is built so a real algorithm (e.g. SM-2) can replace
  just the interval calculation later without changing how cards are
  stored or reviewed, per spec section 28

**Phase 7 — Accounts + database (Supabase/Postgres)**
- `supabase/migrations/001_init.sql` — the full schema (textbooks,
  textbook_pages, quizzes, questions, quiz_attempts, answers, results,
  flashcard_decks, flashcards, flashcard_reviews) with **row-level
  security on every table** — a user can only ever see rows where
  `user_id = auth.uid()`. Run this in the Supabase SQL editor (or
  `supabase db push`) against your own project
- Email/password auth: `/signup`, `/login`, `/auth/callback` (handles
  the email confirmation link), sign-out from the nav. Nav shows the
  signed-in email or a Sign in button, reactively, via
  `supabase.auth.onAuthStateChange`
- `src/proxy.ts` (Next's current name for what used to be
  `middleware.ts`) refreshes the auth session cookie on every request
- **This closes the Phase 5/6 grading trust-boundary gap** — the part
  called out as a known limitation since Phase 5:
  - `/api/textbook/analyze` now persists the textbook + pages to the
    database when signed in, returning a real `textbookId`
  - `/api/quiz/generate` now persists the quiz + questions (the
    authoritative answer key) when signed in, and the quiz's id
    becomes a real database id instead of a local timestamp-based one
  - Submitting a quiz creates a `quiz_attempts` row + `answers` rows
    instead of only writing to `localStorage`
  - `/api/quiz/grade` now has **two paths**: given an `attemptId`, the
    server reads the quiz, correct answers, rubrics, and submitted
    answers **entirely from the database** under the caller's own RLS
    policies — the client's request body is just `{ attemptId }` and
    has zero influence on the answer key. Given a full quiz+answers
    payload instead (no `attemptId`), it falls back to the Phase 5
    client-trusted path exactly as before — used when signed out, or
    when Supabase isn't configured for a deployment
- Everything degrades gracefully with no Supabase env vars set: pages
  render, the nav shows "Sign in", and every feature keeps working
  exactly as it did in Phases 1-6 on `localStorage`/`sessionStorage`
  alone. Verified — every route still returns 200 and the Gemini-key
  check still 503s correctly with Supabase completely unconfigured.

## What's intentionally NOT migrated to the database yet

Quiz History, weak-topic detection, and flashcard decks/reviews still
read from `localStorage` even when signed in — the schema for all of
this exists in `001_init.sql` (`flashcard_decks`, `flashcard_reviews`,
etc.) but the app doesn't write to or read from those tables yet. This
was a deliberate scope cut to get the trust-boundary fix (the more
important gap) shipped first rather than migrating every feature at
once. Wiring History/weak-topics/flashcards to the database — so they
follow a signed-in user across devices — is the natural next slice of
Phase 7 work, not a new phase.

## Running locally

```bash
npm install
cp .env.example .env.local   # nothing required yet for Phase 1
npm run dev
```

Visit `http://localhost:3000`.

## Environment variables

Copy `.env.example` to `.env.local` and set:

```
GEMINI_API_KEY=your-key-here
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
```

- Gemini: https://aistudio.google.com/apikey
- Supabase: create a project at https://supabase.com, then run
  `supabase/migrations/001_init.sql` in its SQL editor before signing
  up — the tables and RLS policies need to exist first.

Without the Supabase variables, the app runs exactly as it did through
Phase 6: fully functional, no accounts, everything on this device only.

## Deploying to Vercel

1. Push this repo to GitHub.
2. Import it in Vercel.
3. In Project Settings → Environment Variables, add `GEMINI_API_KEY`,
   `NEXT_PUBLIC_SUPABASE_URL`, and `NEXT_PUBLIC_SUPABASE_ANON_KEY`
   (Name / Value, applied to Production + Preview).
4. In your Supabase project, add your Vercel domain to Authentication
   → URL Configuration → Redirect URLs (so the email-confirmation
   link's `/auth/callback` redirect works) — e.g.
   `https://your-app.vercel.app/auth/callback`.
5. Deploy.

## Known limitations (expected at this phase)

- No quiz generation or grading yet (Phase 3+).
- No auth, no database — nothing persists across a refresh, so an
  analyzed textbook is gone if you reload the page.
- The rate limiter on `/api/textbook/analyze` is in-memory per
  serverless instance — real per-user quotas need Phase 7 auth plus a
  shared store.

## Known limitations (Phase 7)

- History, weak-topic detection, and flashcards are still
  `localStorage`-only, signed in or not — see "What's intentionally
  NOT migrated" above.
- The legacy client-trusted grading path (used when signed out, or
  Supabase isn't configured) still has the Phase 5 trust-boundary
  caveat by design — closing it for everyone requires requiring
  sign-in, which felt like the wrong tradeoff to make unilaterally
  for a study tool that should still work anonymously.
- No OAuth providers yet (Google/GitHub sign-in) — email/password
  only, as the original spec allows for a first pass ("OAuth providers
  can be added later").
- Deleting a textbook/quiz/account doesn't cascade-clean up yet beyond
  what the DB's `on delete cascade` handles automatically — a proper
  account-deletion flow (spec section 57/58) is still ahead.

**Phase 8 — Teacher Mode**
- `supabase/migrations/002_teacher_mode.sql` — run this after
  `001_init.sql`. Adds `classes`, `class_members`, `assignments`, an
  `assignment_id` column on `quiz_attempts`, and grading-override
  columns (`teacher_marks_awarded`, `teacher_feedback`,
  `overridden_by`, `overridden_at`) on `results`. No new environment
  variables — this is pure schema + RLS on top of the same Supabase
  project.
- **Any signed-in user can become a teacher** by creating a class —
  there's no separate role flag, matching how spec section 36 frames
  Teacher Mode as a capability rather than an account type
- `/teacher` — create a class (gets a random 6-character join code,
  e.g. `7QK4M2`), see your classes and how many students have joined
  each
- `/teacher/classes/[id]` — assign any quiz you've generated to the
  class with an optional due date, and see submission/grading counts
  per assignment
- `/teacher/classes/[id]/assignments/[id]` — every submission for one
  assignment, with its score once graded and a flag for anything the
  AI grader marked low-confidence
- `/teacher/submissions/[attemptId]` — full per-question review:
  question, student's answer, the AI's mark and feedback, and an
  override control. Per spec section 38, overriding **never**
  overwrites the AI's own grade — `marks_awarded`/`feedback` (the AI's)
  and `teacher_marks_awarded`/`teacher_feedback` (the teacher's) are
  stored side by side, so nothing is silently replaced
- `/classes` (student-facing) — join a class by code
  (`join_class_by_code`, a `security definer` Postgres function — see
  the migration's comments for why this is a function and not a
  looser `SELECT` policy on `classes`), see assignments across all
  joined classes, and jump straight into one that hasn't been started
  yet via `/assignments/[id]/start`
- **The RLS problem this phase actually has to solve**: a teacher
  needs to read a student's `quiz_attempts`/`answers`/`results` rows
  for their own assignments, despite `user_id` on those rows being the
  student's, not the teacher's. `001_init.sql`'s owner-only policies
  are left completely untouched — this migration adds *additional*
  narrowly-scoped `SELECT`/`UPDATE` policies (Postgres OR's all
  matching policies together) that only ever grant a teacher access to
  attempts tied to an assignment in a class they own. The reverse
  direction — a student reading the quiz/questions for an assignment,
  which the teacher owns — gets the same treatment
- Grading itself is still only ever triggered by the student opening
  their own `/quiz/results` (closing the Phase 5/7 trust boundary
  requires the `results` row's `user_id` to be the student who took
  the quiz, not whoever is looking at it) — a teacher's submissions
  view only ever *reads* a grade that already exists, it doesn't have
  a "grade now" button that could misattribute that row

## What Phase 8 does not do yet

- Assigning a quiz doesn't notify students — spec section 59's
  notification system is still Phase 9 work. Students find new
  assignments by visiting `/classes`.
- The submission review page shows the student's raw answers next to
  the AI's grade, but doesn't show *why* an MCQ was marked wrong (i.e.
  the correct option isn't called out inline) — a small follow-up, not
  a structural gap.
- `/quiz/results` and `/quiz/submitted` still resolve the quiz/
  submission from this device's `sessionStorage`/local archive first
  (see "Data flow" above, a Phase 1-6 carryover). A student who
  submits an assignment on one device and opens the link on another
  will hit "not found in this session" — the DB has everything needed
  to fix this (quiz + questions + answers are all persisted), it just
  isn't wired as the primary read path yet.
- No teacher ability yet to edit/approve-reject individual questions
  before assigning (spec section 36's "edit questions, approve/reject
  questions" bullet) — a teacher can only assign a quiz as generated.

**Phase 9 (partial) — Export + quiz sharing**
- `supabase/migrations/003_quiz_sharing.sql` — adds `visibility`
  (`private`/`public`, default private) and a unique `share_token` to
  `quizzes`, plus the two additional `SELECT` policies that let a
  signed-in visitor with the link read a public quiz's questions
  without owning it (same pattern as Phase 8's teacher/student
  cross-access policies — additive, existing owner-only policies
  untouched)
- New dependency: `pdf-lib` (added to `package.json` — run
  `npm install` to pick it up). Chosen because it's pure JS with no
  native bindings, so it runs fine in a Vercel serverless function;
  `src/lib/pdf.ts` is a deliberately small text-flow helper (title/
  heading/paragraph/wrap/page-break), not a layout engine — this is a
  printable question paper, not a typeset document
- `GET /api/export/question-paper?quizId=` — PDF of every question,
  no answers. `GET /api/export/answer-key?quizId=` — PDF with correct
  options/model answers/rubrics. Both require sign-in and rely
  entirely on the existing quiz/question RLS policies to decide who
  can export what — a teacher can export their own quizzes, nothing
  more
- `GET /api/export/results?attemptId=` — CSV of a graded attempt's
  per-question marks (AI + teacher override side by side), confidence,
  and feedback. Same RLS-decides-access approach — works for both the
  student who took it and the teacher who assigned it
- `POST /api/quiz/share` — flips one of your own quizzes to a public
  share link (`/shared/[token]`); `action: "unshare"` reverts it.
  Anyone signed in with the link can take it — this is the "shareable"
  case from spec section 39; "class-only" sharing already exists as
  Phase 8's assignments
- Wired into `/history` (Share / Question paper / Answer key on any
  quiz that's actually in the database — local-only Phase 1-6 quizzes
  don't get the controls, since there's nothing on a server to export)
  and into `/quiz/results` + the teacher's submission review page
  (CSV download)

## What Phase 9 does not do yet

- **Notifications** (spec section 59) — assigning a quiz still doesn't
  notify anyone; students find it by visiting `/classes`.
- **Voice** (spec section 35) — no text-to-speech or speech-to-text
  anywhere yet.
- **Diagram-labeling practice** (spec section 32) — not started.
- **PDF results export** (only CSV exists) and **printable quiz**
  (as opposed to the question-paper PDF, which is the same thing) —
  the CSV covers the "results" half of spec section 44's export list;
  a formatted PDF results sheet is a small follow-up on top of the
  same `SimplePdf` helper, not a new capability.
- Export is intentionally **not connector-based** (no Google
  Drive/email attach) — it's a direct download; spec section 44 doesn't
  ask for more than that.

**Phase 10 (partial) — Security hardening, rate limiting, accessibility**

See `PHASE10.md` for the full write-up, testing plan, and deployment
checklist. Summary of what changed:

- **Fixed a real trust-boundary gap in `/api/quiz/grade`**: the
  DB-authoritative grading path checked that an attempt existed and
  was readable, but never checked it belonged to the caller. Since
  Phase 8's teacher policies grant a teacher read access to a
  student's `quiz_attempts` (so they can review submissions), the
  endpoint itself — not just the UI — needed to enforce that only the
  attempt's own student can trigger grading. It now does
  (`attempt.user_id !== user.id` → 403).
- **Grading is now idempotent**: an already-graded attempt returns its
  stored `results` rows instead of re-calling Gemini and re-billing on
  every visit to `/quiz/results`.
- **Rate limiting extended to every AI-calling route.** Through
  Phase 9, only `/api/textbook/analyze` and `/api/quiz/generate` had
  it (as duplicated code); `/api/quiz/repair`, `/api/flashcards/
  generate`, and `/api/quiz/grade` had none. All five now share one
  implementation (`src/lib/rateLimit.ts`) with per-route presets. The
  underlying limitation is unchanged and documented in that file: it's
  still per-serverless-instance memory, not a shared store — a real
  per-user quota needs Upstash or similar, which is a drop-in swap of
  that one file, not a route-by-route rewrite.
- **Accessibility**: MCQ options in `/quiz/take` are now a proper
  `role="radiogroup"`/`role="radio"` set with `aria-checked`, instead
  of unrelated `<button>`s; the short-answer textarea's `<label>` is
  now actually connected via `htmlFor`/`id`; creative-question answer
  boxes get an `aria-label` naming their part and prompt.
- **A second, more severe version of the same ownership bug**, in the
  export routes: `/api/export/answer-key` and `/api/export/question-paper`
  trusted "RLS let me read this quiz row" as "I may export it" — but
  RLS also lets an *assigned student* read a quiz (to take it) and a
  *public-quiz viewer* read it (see below). Before this fix, a student
  could download their own assigned quiz's answer key before taking
  it. Both routes now require actual ownership (`quiz.user_id ===
  caller`), not just read access.
- **`weakTopics.ts` built its `quizIds` list backwards** — documented
  as "most recent first" and relied on that way by `/practice/weak-areas`,
  but a stray `.unshift()` while iterating an already-newest-first list
  silently reversed it, so weak-topic practice could get grounded in a
  stale, superseded textbook instead of the current one. Fixed
  (`.unshift()` → `.push()`).
- **Found, flagged, not fixed**: `share_token` on public quizzes isn't
  actually checked anywhere — the RLS policy only checks
  `visibility = 'public'`, so once a quiz is public it's readable by
  any signed-in user, not just someone with the link. Fixing this
  properly means changing the RLS policy itself, which I didn't want
  to do blind without a database to test it against. Full explanation
  and options in `PHASE10.md` §5c.

What Phase 10 does **not** close (see `PHASE10.md` for the rest):
the Phase 5 legacy client-trusted grading path itself is unchanged —
closing it means requiring sign-in, which is a product decision, not
a bug fix, and the README has flagged it as a known tradeoff since
Phase 7; a load/perf pass under real traffic; and full WCAG audit
beyond the specific gaps found above.
