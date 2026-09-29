# Phase 10 — Security, rate limiting, accessibility, testing, deployment

This phase doesn't add product surface. It audits what Phases 1–9
built and closes the gaps that matter before this goes in front of
real students and teachers. Everything below was found by reading the
actual code, not by re-deriving the spec from scratch — each item says
what was wrong and why the fix is what it is.

No `npm install` / build / lint was run while making these changes —
this container has no network access to the npm registry. Before you
deploy, run:

```bash
npm install
npm run lint
npm run build
```

and fix anything that surfaces (I'd expect nothing — the edits are
narrow and use only patterns already present elsewhere in the
codebase — but I haven't been able to verify it myself, so say so
plainly rather than implying I did).

## 1. Security fixes (the part that actually mattered)

### 1a. `/api/quiz/grade` — missing ownership check (fixed)

The DB-authoritative grading path (`gradeFromDatabase`) read a
`quiz_attempts` row by id and proceeded as long as the row existed and
was readable under RLS. It never checked *whose* attempt it was.

That was fine through Phase 7, when the only RLS `SELECT` policy on
`quiz_attempts` was owner-only. Phase 8 added a second policy so a
teacher can read a student's attempt for an assignment in a class they
own (so `/teacher/submissions/[attemptId]` works). Postgres RLS
policies OR together — so from that point on, `quiz_attempts` was
readable by *either* its owner *or* the relevant teacher, but
`gradeFromDatabase` still treated "readable" as "safe to grade as."

The consequence: the endpoint writes `results` rows stamped with
`user_id: user.id` — the *caller's* id. A teacher who could read a
student's attempt (by design, for review) could also POST that
`attemptId` to `/api/quiz/grade` directly and have grading run and
persist under the teacher's own user id instead of the student's.
Depending on the `results` table's insert policy, that either fails
outright or creates a row the actual student's `results` `SELECT`
policy (owner-only) can't see — either way, not what anyone intended,
and not something the Phase 8 README notes actually prevented; they
just noted that the *UI* doesn't expose a "grade now" button to
teachers. The API had no equivalent guard.

**Fix**: `gradeFromDatabase` now selects `user_id` alongside the rest
of the attempt and returns 403 unless `attempt.user_id === user.id`.
Only the student who took the attempt can trigger its grading — matching
what the README already claimed was true.

### 1b. `/api/quiz/grade` — not idempotent (fixed)

Every call re-ran `gradeAllQuestions` (which calls Gemini for every
short/creative question) even if the attempt was already graded. The
client-side cache in `gradingStorage.ts` masks this on the same
device/session, but a second device, a cleared cache, or a direct API
call would silently re-bill Gemini for a quiz that already has a
score. The code even had a comment claiming results are "stable on
revisit" — the comment was aspirational, not enforced.

**Fix**: if `attempt.status === 'graded'`, the route now reads the
existing `results` rows and returns them, without touching Gemini.
Falls through to normal grading only if the status says graded but no
rows exist (shouldn't happen, but fails open to "grade it" rather than
erroring the student out of ever seeing a result).

### 1c. Unrated cost-control gap: three routes had no rate limiting

`/api/quiz/repair`, `/api/flashcards/generate`, and `/api/quiz/grade`
had zero request throttling through Phase 9 — only `/api/textbook/
analyze` and `/api/quiz/generate` did, and each had its own
copy-pasted version of the same 15-line limiter. Spec section 46
("Cost Control") and section 64 ("Rate Limiting") call this out
explicitly as mandatory for exactly this class of endpoint.

**Fix**: one shared implementation, `src/lib/rateLimit.ts`, with named
per-route presets (`RATE_LIMITS.quizRepair`, etc.) applied to all five
AI-calling routes. Repair and flashcard generation get looser limits
than full quiz generation (cheaper calls), grading gets a moderate
limit even though the DB path is now idempotent (the legacy
client-trusted path still calls Gemini every time, per-request).

**What this fix does not do**: it's still an in-memory `Map`, scoped
to one serverless instance. On Vercel, concurrent cold starts each get
their own counter, so the effective ceiling under real concurrent load
is higher than the configured number, not exactly it. This was true of
the Phase 1–9 limiters too — it's now honestly documented in the file
instead of implied to be more than it is. A real fix is a shared store
(Upstash Redis + `@upstash/ratelimit` is the standard Vercel-native
choice) — and because `checkRateLimit()` is the only thing routes call,
swapping the store is a change to one file, not five.

### 1d. What was *not* changed, on purpose

- **The Phase 5 legacy client-trusted grading path** (`gradeFromClientPayload`,
  used when signed out or Supabase isn't configured) is unchanged. A
  signed-out user can still tamper with their own local answer key
  before submitting it for grading. Closing this means requiring
  sign-in to take any quiz, which the Phase 7 README explicitly
  decided against ("felt like the wrong tradeoff... for a study tool
  that should still work anonymously"). That's a product call, not a
  bug, and it isn't Phase 10's place to reverse it unilaterally — it's
  flagged again here so whoever owns that tradeoff decides consciously
  in production.
- **RLS policies themselves were not modified.** The 001/002/003
  migrations were re-read carefully while tracing the 1a bug above; no
  gap was found in the policies as written — the gap was entirely in
  application code that didn't add an *additional* check on top of
  what RLS allows. (RLS answers "can this row be read/written by this
  caller at all"; it can't answer "should this specific action be
  allowed given that read access exists for a different reason.")

## 2. Accessibility fixes

- `/quiz/take`'s MCQ options (`AnswerArea.tsx`) were plain `<button>`s
  with a visual checkmark for the selected one and no semantic
  grouping. A screen reader announced four unrelated buttons with no
  indication exactly one is a selection state. Now: the container is
  `role="radiogroup"`, each option is `role="radio"` with
  `aria-checked`, and `type="button"` is explicit (defensive — matters
  if this ever ends up inside a `<form>`).
- The short-answer `<label>` said "Your answer" but wasn't associated
  with its `<textarea>` (`htmlFor`/`id`) — a screen reader user
  tabbing to the textarea got no label at all. Fixed.
- Creative A–D answer boxes now carry an `aria-label` naming the part
  and its prompt, since their visual label (a small part-letter badge)
  isn't reliably associated with the textarea either.

**Not done in this pass** (real remaining gaps, listed rather than
hidden): no audit was done of color contrast, focus-visible styling
across the whole app, keyboard-only navigation of the question
navigator grid, or `prefers-reduced-motion` handling for the
animations spec section 52 asks for. Those need a dedicated pass with
an actual screen reader and a contrast checker, not something to claim
fixed from a code read alone.

## 3. Additional bugs found on a second pass

Three more real issues, found by tracing actual data/access paths
rather than re-reading the spec:

### 5a. `/api/export/answer-key` and `/api/export/question-paper` — same class of bug as §1a, more severe

Both routes fetched the quiz row and treated "the query returned a
row" as "this caller may export it." But RLS grants `SELECT` on
`quizzes` to more than the owner:

- `student_select_assigned_quiz` (002) — so a student can read a quiz
  *assigned to them*, to take it.
- `teacher_select_quizzes` (002) — so a teacher can read a quiz
  assigned in their class, to review it.
- `public_select_quiz` (003) — so **any signed-in user** can read
  **any** quiz with `visibility = 'public'`, with no check against
  `share_token` at all (see §3c).

None of those three are "may see the correct answers." Concretely,
before this fix, **a student assigned a quiz could call
`/api/export/answer-key?quizId=<their own assigned quiz>` and download
the answer key before attempting it** — a direct hole in exam
integrity, not a theoretical one. The README even stated the intended
invariant ("a teacher can export their own quizzes, nothing more")
as if RLS already guaranteed it; it didn't.

**Fix**: both routes now additionally select `user_id` and require
`quizRow.user_id === user.id`, returning 404 otherwise (same
not-found-vs-not-owned non-disclosure pattern already used in
`/api/quiz/share`). Exporting is now genuinely owner-only, matching
what was already documented as the intent.

### 5b. `weakTopics.ts` — `quizIds` built in the wrong order

The type comment says `quizIds` is "most recent first," and
`/practice/weak-areas` relies on that — it reads `quizIds[0]` to
decide which quiz/textbook to group a weak topic under and to source
its "Practice these topics" regeneration from. The aggregation loop
walks `listHistory()`, which is already sorted newest-first, but used
`.unshift()` to build `quizIds`. Applying `unshift` while iterating
newest-to-oldest reverses the result: `quizIds[0]` ended up being the
**oldest** quiz that ever touched that topic, not the most recent one.

In practice: if a topic first appeared in an old textbook and later
recurred in a newer one, "Practice these topics" would group and
source the practice quiz from the stale, possibly-no-longer-relevant
textbook instead of the current one.

**Fix**: changed `.unshift()` to `.push()` — since the loop already
visits entries newest-first, pushing preserves that order instead of
inverting it.

### 5c. `share_token` doesn't actually gate anything (found, not fixed — flagging honestly)

`003_quiz_sharing.sql`'s `public_select_quiz` policy is
`using (visibility = 'public')` — it never compares against
`share_token`. The token is generated and put in the `/shared/[token]`
URL, but nothing in the database or application code actually checks
that a reader supplied it. The practical effect: once a quiz is
flipped to public, it's readable (and, before §3a's fix, exportable)
by **any signed-in user who knows or guesses/enumerates its id** —
not only someone who received the share link. "Unshare" does correctly
revoke this (flips `visibility` back to `private`), so the exposure
window is real but bounded to however long a quiz is left public.

This is a genuine gap between the documented design ("anyone signed in
**with the link** can take it") and the implementation ("anyone
signed in, full stop, once it's public"). I did not fix this one — a
correct fix means the RLS policy itself needs to check a token, which
means threading the token through Postgres session context (similar
in spirit to `join_class_by_code`'s `security definer` pattern for
class codes), and I don't have a way to test a Postgres RLS change
against a live database from this environment. Flagging it precisely
instead of shipping an unverified schema change: **before relying on
"public" sharing for anything sensitive, either accept that it means
"discoverable by any signed-in user" (rename the UI copy to say so
honestly), or add a proper token check to the policy and test it
against a real Supabase project.**

## 4. Testing plan

No test suite exists in this codebase (no `jest`/`vitest`/`playwright`
in `package.json`). Given the size of the app, a full suite is a
project of its own — this is the plan for someone doing it, prioritized
by what would actually catch a shipped bug like the ones in §1 and §3:

**Priority 1 — RLS/ownership (most likely place for silent data bugs):**
- A teacher can read but not grade a student's attempt (regression
  test for §1a, specifically).
- A student can't read another student's `quiz_attempts`/`answers`/
  `results` via a guessed/enumerated id.
- A teacher can only see attempts tied to assignments in classes they
  own — not every attempt in the database.
- Grading an already-graded attempt returns identical results without
  a second Gemini call (mock the client and assert the call count).
- A student assigned a quiz gets 404 from `/api/export/answer-key`
  for it; only the quiz's actual creator gets a 200 (regression test
  for §3a — this is the one to run before trusting Exam Mode with
  real students).

**Priority 2 — quiz correctness (spec's own explicit test list, §81):**
- A requested N-question quiz produces exactly N questions after the
  repair/drop pipeline runs.
- Every MCQ has exactly 4 options and exactly 1 correct answer, even
  after Gemini returns a malformed draft (feed the validator a known-bad
  fixture, not just golden-path output).
- Total marks displayed match the sum of the assembled questions'
  marks — server-assigned, never trusted from the model.

**Priority 3 — resilience:**
- Grading with `GEMINI_API_KEY` unset still grades a pure-MCQ quiz
  (already true per the Phase 5 README note — worth pinning as a test
  so it can't regress).
- Rate limiting actually returns 429 after the configured count, on
  each of the five routes now covered.
- Autosave survives a simulated refresh mid-quiz (localStorage
  round-trip).

**Priority 4 — accessibility regression:**
- Automated: run `axe-core` (via `@axe-core/playwright` or similar)
  against `/quiz/take`, `/upload`, and `/quiz/results` as a smoke test
  — cheap to add, catches the class of bug fixed in §2 from recurring.

Suggested stack: Vitest for `src/lib/*` unit tests (validation,
grading, weak-topic aggregation are all pure functions — no reason
these aren't tested already), Playwright for the RLS/flow tests above
since they need a real Supabase project and real HTTP requests.

## 5. Deployment checklist

Before pointing real users at a Vercel deployment:

- [ ] `npm run build` passes with zero errors (unverified here — see
      top of this file).
- [ ] Both `001_init.sql`, `002_teacher_mode.sql`, and
      `003_quiz_sharing.sql` have been run, in that order, against the
      production Supabase project.
- [ ] Environment variables set in Vercel (Production **and** Preview):
      `GEMINI_API_KEY`, `NEXT_PUBLIC_SUPABASE_URL`,
      `NEXT_PUBLIC_SUPABASE_ANON_KEY`.
- [ ] Supabase Auth → URL Configuration → Redirect URLs includes the
      real production domain's `/auth/callback`.
- [ ] Decide, consciously, whether the Phase 5 legacy grading path
      (§1d) is acceptable for launch, or whether quiz-taking should
      require sign-in for this deployment. Not a code change either
      way — a decision to make and document.
- [ ] Replace the in-memory rate limiter (§1c) with a shared store
      before expecting real concurrent traffic — the current one only
      protects a single warm instance.
- [ ] Confirm `GEMINI_API_KEY` does not appear in any `NEXT_PUBLIC_*`
      variable or client bundle (it doesn't, per the `server-only`
      guard in `src/lib/gemini.ts` — worth a final `grep -r
      GEMINI_API_KEY src/app/**/*.tsx` as a sanity check since this is
      exactly the kind of mistake that's invisible until someone
      inspects the deployed bundle).
- [ ] Run the Priority 1 tests above against the production Supabase
      project (a staging project with real RLS, not a mocked one) —
      RLS policy bugs don't show up in a build, only in a live query.
- [ ] Decide a real quota story before this is public — `LIMITS.
      freeTierDailyGenerations` exists in `src/lib/config.ts` but
      nothing currently enforces it; only the per-minute rate limiter
      is wired up.
