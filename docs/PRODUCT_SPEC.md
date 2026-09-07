# Kindred — Interest-Based Dating App

**Status:** Design / Spec (pre-implementation)
**Author:** Nitish Chauhan (spec drafted with Claude Code)
**Last updated:** 2026-09-07

## 1. Concept

Kindred is a dating app where matching is driven by **shared interests and
compatibility signals**, not by swipe-first appearance judgments (though
photos are still part of a profile). Think "Hinge/OkCupid-style" rather than
pure "Tinder-style": users complete an interest/values quiz on signup, tag
interests from a curated taxonomy, and the app surfaces a small, high-quality
daily set of candidates ranked by a compatibility score. Users can still
like/pass on suggested profiles, but the *ranking* — not raw swiping volume —
is the core mechanic.

### Design principles
- **Quality over volume**: a small daily batch of well-matched candidates
  beats an infinite swipe deck.
- **Explainable matches**: show *why* two people were matched ("You both
  care about hiking and slow mornings"), not just a percentage.
- **Consent and safety by default**: reporting/blocking, photo verification,
  and moderation are first-class, not bolted on later.

## 2. Primary user flows

1. **Onboarding**
   - Sign up (email/phone + password, or OAuth) → verify contact.
   - Build profile: photos, bio, basic facts (age, location, gender,
     orientation/preferences).
   - Interest quiz: pick interests from a taxonomy (e.g. "Hiking",
     "Sci-fi movies", "Cooking") + answer a short set of values/lifestyle
     questions (e.g. "Morning person or night owl?", "Do you want kids?").
   - Set discovery preferences: age range, distance radius, gender(s) of
     interest.

2. **Daily discovery**
   - App computes a compatibility score between the user and candidate
     pool (see §5) and returns a ranked, capped daily batch (e.g. 10–20
     profiles).
   - For each candidate the user sees a profile card with an
     **explanation chip** ("92% match — 4 shared interests, similar
     values on family & travel").
   - User can Like, Pass, or Super Like (limited per day).

3. **Matching**
   - A match is created when both users Like each other (standard) or
     when a Super Like is accepted.
   - Both users are notified; a chat thread opens.

4. **Chat**
   - Real-time messaging (text first; media/GIFs later).
   - Icebreaker prompts drawn from shared interests ("You both liked
     'Photography' — ask about their favorite shot").
   - Unmatching/blocking/reporting available from the thread.

5. **Profile management & safety**
   - Edit profile/interests any time (triggers re-scoring).
   - Report/block a user → hides them from discovery for the reporter
     immediately and flags for moderation review.
   - Photo verification (selfie-vs-profile-photo check) to reduce fake
     accounts.

## 3. Core features (MVP scope)

| Feature | In MVP? | Notes |
|---|---|---|
| Email/phone signup + auth | Yes | JWT-based sessions |
| Profile creation (photos, bio, facts) | Yes | Min 1 photo, max 6 |
| Interest taxonomy + quiz | Yes | ~150 curated interests, 8–10 quiz questions |
| Compatibility scoring & ranked daily batch | Yes | See §5 |
| Like / Pass / Super Like | Yes | Super Likes rate-limited |
| Mutual match creation | Yes | |
| Real-time chat | Yes | WebSocket-based |
| Icebreaker suggestions | Yes | Simple rule-based, not LLM in MVP |
| Report / block | Yes | Required for safety/compliance |
| Photo verification | Stretch | Manual review queue acceptable for MVP |
| Push notifications | Stretch | Email fallback for MVP |
| Video chat | Post-MVP | |
| Paid tiers (boosts, extra super likes) | Post-MVP | |

## 4. Data model

Relational schema (PostgreSQL). Simplified DDL sketch:

```sql
-- Identity & auth
users (
  id UUID PK,
  email TEXT UNIQUE,
  phone TEXT UNIQUE NULL,
  password_hash TEXT,
  created_at TIMESTAMPTZ,
  status TEXT CHECK (status IN ('active','suspended','deleted')),
  last_active_at TIMESTAMPTZ
)

-- Public-facing profile
profiles (
  user_id UUID PK REFERENCES users(id),
  display_name TEXT,
  birthdate DATE,
  gender TEXT,
  orientation TEXT,
  bio TEXT,
  city TEXT,
  lat DOUBLE PRECISION,
  lng DOUBLE PRECISION,
  verified BOOLEAN DEFAULT FALSE,
  updated_at TIMESTAMPTZ
)

photos (
  id UUID PK,
  user_id UUID REFERENCES users(id),
  url TEXT,
  position SMALLINT,
  is_primary BOOLEAN
)

-- Discovery preferences
preferences (
  user_id UUID PK REFERENCES users(id),
  min_age SMALLINT,
  max_age SMALLINT,
  max_distance_km INTEGER,
  interested_in_genders TEXT[]
)

-- Interests taxonomy (curated, admin-managed)
interests (
  id SERIAL PK,
  category TEXT,          -- e.g. 'Outdoors', 'Arts', 'Food'
  label TEXT UNIQUE        -- e.g. 'Hiking'
)

user_interests (
  user_id UUID REFERENCES users(id),
  interest_id INTEGER REFERENCES interests(id),
  PRIMARY KEY (user_id, interest_id)
)

-- Values/lifestyle quiz
quiz_questions (
  id SERIAL PK,
  prompt TEXT,
  question_type TEXT       -- 'scale' | 'single_choice' | 'multi_choice'
)

quiz_answers (
  user_id UUID REFERENCES users(id),
  question_id INTEGER REFERENCES quiz_questions(id),
  answer_value JSONB,
  PRIMARY KEY (user_id, question_id)
)

-- Interaction & matching
swipes (
  id UUID PK,
  actor_id UUID REFERENCES users(id),
  target_id UUID REFERENCES users(id),
  action TEXT CHECK (action IN ('like','pass','super_like')),
  created_at TIMESTAMPTZ,
  UNIQUE (actor_id, target_id)
)

matches (
  id UUID PK,
  user_a_id UUID REFERENCES users(id),
  user_b_id UUID REFERENCES users(id),
  matched_at TIMESTAMPTZ,
  unmatched_at TIMESTAMPTZ NULL,
  UNIQUE (user_a_id, user_b_id)
)

messages (
  id UUID PK,
  match_id UUID REFERENCES matches(id),
  sender_id UUID REFERENCES users(id),
  body TEXT,
  sent_at TIMESTAMPTZ,
  read_at TIMESTAMPTZ NULL
)

-- Trust & safety
reports (
  id UUID PK,
  reporter_id UUID REFERENCES users(id),
  reported_id UUID REFERENCES users(id),
  reason TEXT,
  details TEXT,
  status TEXT CHECK (status IN ('open','reviewing','resolved')),
  created_at TIMESTAMPTZ
)

blocks (
  blocker_id UUID REFERENCES users(id),
  blocked_id UUID REFERENCES users(id),
  created_at TIMESTAMPTZ,
  PRIMARY KEY (blocker_id, blocked_id)
)
```

Indexing notes:
- `swipes(actor_id, target_id)` unique index doubles as the "already
  seen" lookup for excluding candidates from future batches.
- `profiles(lat, lng)` should use PostGIS or a geo index for
  distance-based prefiltering before scoring.
- `matches` should be looked up by either user id — store both directions
  or add a computed/generated column for a canonical ordering.

## 5. Matching algorithm (interest/compatibility-based)

Two-stage pipeline, run as a nightly/periodic batch job per user (not
purely on-request) so the daily set is stable and cheap to serve:

**Stage 1 — Candidate filtering**
- Apply hard filters: age range, gender/orientation preference,
  max distance, exclude already-swiped, exclude blocked/blocking users,
  exclude suspended accounts.

**Stage 2 — Compatibility scoring**
Weighted score combining:

- **Interest overlap** (Jaccard similarity of `user_interests` sets),
  weight ~40%.
- **Quiz/values similarity** (cosine or weighted distance over
  `quiz_answers`, normalizing scale-type answers), weight ~35%.
- **Activity/reciprocity signal** (recently active, historical like-back
  rate for similar profiles), weight ~15%.
- **Distance decay** (closer = slight boost), weight ~10%.

```
score = 0.40 * interest_overlap
      + 0.35 * quiz_similarity
      + 0.15 * reciprocity_signal
      + 0.10 * distance_decay
```

Output: top N candidates (e.g. 15/day) per user, persisted to a
`daily_batches` table so re-opening the app doesn't reshuffle the set,
and so "why matched" explanations (top shared interests + top aligned
quiz answers) can be precomputed once and cached.

This is intentionally a transparent, tunable weighted-sum model for the
MVP — no ML training loop required. A v2 could replace the static
weights with a learned ranking model once there's enough interaction
data (likes/passes/message-response labels) to train on.

## 6. API surface (REST, JSON)

```
POST   /auth/signup
POST   /auth/login
POST   /auth/logout

GET    /me
PATCH  /me
GET    /me/preferences
PUT    /me/preferences

GET    /interests                 (taxonomy list)
GET    /me/interests
PUT    /me/interests               (replace set)

GET    /quiz/questions
GET    /me/quiz-answers
PUT    /me/quiz-answers

POST   /photos                     (upload)
DELETE /photos/:id
PATCH  /photos/:id/position

GET    /discovery/batch            (today's ranked candidates)
POST   /discovery/swipe            { target_id, action }

GET    /matches
GET    /matches/:id/messages
POST   /matches/:id/messages
DELETE /matches/:id                (unmatch)

POST   /reports
POST   /blocks
DELETE /blocks/:id
```

Real-time: a WebSocket channel (`/ws`) per authenticated user for new
matches and incoming messages, backed by the same `messages`/`matches`
tables (no separate message queue needed at MVP scale).

## 7. Tech stack

- **Frontend:** React (Vite), React Router, a component library (or
  Tailwind) for styling, WebSocket client for chat.
- **Backend:** Node.js + Express, REST + WebSocket (e.g. `ws` or
  `socket.io`).
- **Database:** PostgreSQL (+ PostGIS extension for geo queries).
- **Auth:** JWT access/refresh tokens, bcrypt/argon2 password hashing.
- **File storage:** S3-compatible object storage for photos.
- **Background jobs:** a simple job runner (e.g. `node-cron` or a queue
  like BullMQ/Redis) for the nightly compatibility scoring batch.

## 8. Trust, safety & compliance considerations

- Minimum age enforcement (18+) with birthdate verification.
- Report/block flows must be reachable from every surface showing
  another user (profile card, chat thread).
- Manual moderation queue for reports; auto-suspend on repeated reports
  above a threshold pending review.
- Photo moderation (basic automated NSFW screening + manual review for
  flagged content) before a photo is shown to other users.
- Data privacy: location stored at reduced precision for anything shown
  to other users (approximate distance, not exact coordinates);
  location-based features must present accordingly.
- Clear data-deletion path (account deletion removes/anonymizes
  profile, photos, messages within a defined retention window).

## 9. Suggested milestones

1. **M0 — Foundations:** auth, profile CRUD, interest taxonomy admin,
   Postgres schema + migrations.
2. **M1 — Discovery core:** quiz, preferences, candidate filtering,
   scoring batch job, `/discovery/batch` + swipe endpoints.
3. **M2 — Matching & chat:** match creation, REST message history,
   WebSocket real-time delivery.
4. **M3 — Safety:** reporting, blocking, moderation queue, basic photo
   screening.
5. **M4 — Polish:** icebreaker prompts, push/email notifications,
   onboarding UX pass.
6. **Post-MVP:** photo verification, paid tiers, learned ranking model,
   video chat.

## 10. Open questions for you

- Target platforms for v1: web only, or web + native mobile from day one?
- Any existing brand/name preference, or is "Kindred" just a placeholder?
- Do you want moderation to be fully manual at MVP, or is an
  automated NSFW/abuse screening API in scope for v1?
- Any geographic launch market that drives compliance needs (age
  verification laws, data residency, etc.)?
