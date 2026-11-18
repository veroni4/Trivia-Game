# Quiz Night: Build Specification

This document is the complete specification for building a trivia web application for a school project. Follow it section by section. Where this document and your own assumptions disagree, this document wins. If something is genuinely unspecified, choose the simplest option that satisfies the requirements and note the choice in a comment in the code.

---

## 1. Project overview

Quiz Night is a multiple-choice trivia game. A user signs in (or plays as a guest), picks one of five categories, answers 10 random questions with a 15-second timer per question, sees a final score, and can save that score to a leaderboard.

The app does not use any generative AI. Questions come from the Open Trivia Database (opentdb.com).

### 1.1 In scope

- Landing page with Start button and sign-in and register forms
- Email and password authentication with Supabase
- Category menu, 10-question games, per-question timer, scoring, summary screen
- Saving scores for logged-in users and a leaderboard filterable by category
- Sound effects and animations
- Dark and light mode
- Responsive layout for desktop and mobile

### 1.2 Out of scope (do not build)

- Any AI feature, including AI explanations, hints, or generated questions
- Local-storage high scores (replaced by the leaderboard)
- Share-result buttons
- Social features, friends, chat, or profile pictures
- Payments, admin dashboards, or question editing
- Offline mode

---

## 2. Technology stack

| Concern | Choice |
|---|---|
| Framework | Next.js (latest stable), App Router |
| Language | TypeScript, strict mode on |
| Styling | Tailwind CSS, with custom colours defined in the Tailwind config |
| Animation | Framer Motion (package `motion`) |
| Sound | Howler.js (package `howler`) |
| Auth and database | Supabase (`@supabase/supabase-js`, `@supabase/ssr`) |
| Question source | Open Trivia Database API (opentdb.com) |
| Package manager | npm |
| Deployment target | Vercel (or any Node host) |

Create the project with:

```
npx create-next-app@latest quiz-night --typescript --tailwind --app --eslint --src-dir
```

Then install: `npm install @supabase/supabase-js @supabase/ssr motion howler` and `npm install -D @types/howler`.

Do not add other UI libraries. Build components with Tailwind.

---

## 3. Architecture

### 3.1 Layers

1. **Browser (client components):** Game flow, timer, sounds, animations, theme toggle, forms.
2. **Server (route handlers and server components):** Fetches questions from opentdb, cleans them, reads the leaderboard, saves scores. The browser never calls opentdb directly.
3. **Supabase:** Authentication and the Postgres database.

### 3.2 Data flow for one game

1. User picks a category on `/menu`. The app navigates to `/play/[category]`.
2. The play page calls `GET /api/questions?category=<slug>`.
3. The route handler gets or creates an opentdb session token (see section 7), fetches 10 questions, decodes them, shuffles each answer list, and returns JSON.
4. The client stores the 10 questions in game state. All answering happens client-side with no network calls.
5. When the 10th question is resolved, the client navigates to `/summary` with the score in the URL query (`?score=7&category=science`).
6. If the user is logged in, the summary page calls `POST /api/scores` with the score. If the user is a guest, the summary shows a "Log in to save your score" prompt instead.

### 3.3 Folder structure

Use this structure. Create every file listed.

```
quiz-night/
  .env.local                      (not committed)
  .env.example
  README.md
  supabase/
    schema.sql                    (the SQL in section 6, saved as a file)
  public/
    sounds/
      correct.mp3
      wrong.mp3
      tick.mp3
      timeout.mp3
      win.mp3
      click.mp3
  src/
    app/
      layout.tsx                  (root layout, theme provider, fonts)
      globals.css
      page.tsx                    (landing page)
      menu/page.tsx               (category selection)
      play/[category]/page.tsx    (game screen)
      summary/page.tsx            (summary screen)
      leaderboard/page.tsx        (leaderboard)
      api/
        questions/route.ts        (GET: fetch and clean questions)
        scores/route.ts           (POST: save score)
      auth/
        callback/route.ts         (Supabase email confirmation callback)
    components/
      AuthPanel.tsx               (login and register form on landing page)
      CategoryCard.tsx
      QuestionCard.tsx
      AnswerButton.tsx
      ProgressBar.tsx
      TimerRing.tsx
      ScoreSummary.tsx
      LeaderboardTable.tsx
      ThemeToggle.tsx
      SoundToggle.tsx
      NavBar.tsx
      Button.tsx
    lib/
      categories.ts               (the 5 categories, IDs, colours, icons)
      opentdb.ts                  (API calls, token handling, decoding)
      shuffle.ts                  (Fisher-Yates shuffle)
      sounds.ts                   (Howler wrapper with mute support)
      game-reducer.ts             (game state reducer)
      supabase/
        client.ts                 (browser client)
        server.ts                 (server client using cookies)
        middleware.ts             (session refresh helper)
    hooks/
      useCountdown.ts
      useSoundSettings.ts
      useTheme.ts
    types/
      trivia.ts                   (shared types)
  src/middleware.ts               (calls the session refresh helper)
  tailwind.config.ts
```

---

## 4. Functional requirements

Each requirement has an ID. Mention the ID in code comments where the behaviour is implemented, so the requirements can be traced.

| ID | Requirement | Priority |
|---|---|---|
| FR-01 | The user can choose one of 5 categories: History, Geography, Science & Nature, Music, General Knowledge. | Must |
| FR-02 | Each game draws 10 random questions from the chosen category. | Must |
| FR-03 | Questions come from opentdb.com, with one category ID per category. | Must |
| FR-04 | HTML entities in questions and answers are decoded before display. | Must |
| FR-05 | Each question shows 4 answers in random order, with exactly one correct answer. | Must |
| FR-06 | One question is shown at a time, with progress such as "Question 3 of 10". | Must |
| FR-07 | After answering, the correct answer turns green and any wrong choice turns red. | Must |
| FR-08 | A "Next" button moves to the next question. On the 10th question the button reads "See results". | Must |
| FR-09 | Each correct answer scores 1 point. The running score is visible during the game. | Must |
| FR-10 | After 10 questions, the summary shows the final score (for example, "7 / 10") and the category. | Must |
| FR-11 | The summary offers "Play again" (same category, new questions) and "Exit to menu". | Must |
| FR-12 | The landing page has a visually appealing design, a "Start playing" button that goes to the menu, and the sign-in and register forms. | Must |
| FR-13 | The menu shows the 5 categories as selectable cards. | Must |
| FR-14 | A new user can register with an email, a password, and a username from the landing page. | Should |
| FR-15 | A returning user can log in. The session persists until logout. | Should |
| FR-16 | A logged-in user's completed game score is saved with the category and date. | Should |
| FR-17 | The leaderboard shows the top 10 scores, with a filter for each category and for "All categories". | Should |
| FR-18 | A logged-in user can log out from the navigation bar. | Should |
| FR-19 | Each question has a 15-second countdown. If time runs out, the question counts as wrong and the correct answer is revealed. | Could (included as an extra) |
| FR-20 | Guests can play a full game, but their score is not saved. The summary explains this and offers a link to log in. | Should |

### 4.1 Behaviour details

- **Timer (FR-19):** Starts when a question appears. When the user answers, the timer stops. When it reaches zero, the app treats the question as wrong, reveals the correct answer, and waits for the user to press Next. The timer does not auto-advance.
- **Guest play (FR-20):** Guests can use every page except saving. The leaderboard is public and visible without logging in.
- **Repeat questions:** Use the opentdb session token so the same question does not repeat within a session.
- **Score cap:** The score is always 0 to 10. The `total` value stored in the database is always 10.

---

## 5. Non-functional requirements

| ID | Requirement | Priority |
|---|---|---|
| NFR-01 | The layout works on desktop (1280px and wider) and mobile (360px and wider). | Must |
| NFR-02 | Answer feedback and transitions respond within 300 ms once questions have loaded. | Should |
| NFR-03 | If the question fetch fails, show a clear message and a "Try again" button. Use the same pattern for failed score saves and leaderboard loads. | Must |
| NFR-04 | Passwords are handled by Supabase Auth and never stored, logged, or displayed in plain text by the app. | Must |
| NFR-05 | No secret keys in client code. Only the Supabase anon key is exposed through `NEXT_PUBLIC_` variables. | Must |
| NFR-06 | Fetch questions once per game, not once per question. Stay within opentdb's limit of about one request every 5 seconds per IP. | Must |
| NFR-07 | The app works in current Chrome, Firefox, Safari, and Edge. | Should |
| NFR-08 | Text meets WCAG AA contrast in both themes. All controls are reachable and usable with a keyboard. Feedback is announced to screen readers with `aria-live`. | Should |
| NFR-09 | Animations and sound respect `prefers-reduced-motion` and the user's sound toggle. | Must |

---

## 6. Database

### 6.1 Schema

Save this as `supabase/schema.sql` and run it in the Supabase SQL editor.

```sql
-- Profiles: one row per user, holds the public username
create table public.profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  username text not null unique check (char_length(username) between 3 and 20),
  created_at timestamptz not null default now()
);

-- Scores: one row per completed game by a logged-in user
create table public.scores (
  id bigint generated always as identity primary key,
  user_id uuid not null references public.profiles(id) on delete cascade,
  category text not null check (category in ('history','geography','science','music','general')),
  score int not null check (score between 0 and 10),
  total int not null default 10 check (total = 10),
  played_at timestamptz not null default now()
);

create index scores_leaderboard_idx on public.scores (category, score desc, played_at asc);

-- Create a profile automatically when a user signs up.
-- The username is passed in as user metadata during sign-up.
create or replace function public.handle_new_user()
returns trigger
language plpgsql
security definer set search_path = ''
as $$
begin
  insert into public.profiles (id, username)
  values (new.id, new.raw_user_meta_data ->> 'username');
  return new;
end;
$$;

create trigger on_auth_user_created
  after insert on auth.users
  for each row execute procedure public.handle_new_user();

-- Row Level Security
alter table public.profiles enable row level security;
alter table public.scores enable row level security;

-- Anyone (including guests) can read profiles and scores for the leaderboard
create policy "profiles are public" on public.profiles
  for select using (true);

create policy "scores are public" on public.scores
  for select using (true);

-- A user can insert a score only for themselves
create policy "users insert own scores" on public.scores
  for insert with check (auth.uid() = user_id);
```

### 6.2 Notes

- The leaderboard query is: select from `scores` joined to `profiles`, filtered by category when one is chosen, ordered by `score desc, played_at asc`, limited to 10 rows. Return `username`, `category`, `score`, and `played_at`.
- Scores are never updated or deleted by the app. Each finished game is a new row.
- Do not expose email addresses anywhere in the UI or in API responses.
- If the trigger fails during sign-up because the username is missing, the sign-up should fail with a clear message. Validate the username on the client and the server before calling Supabase.

---

## 7. Open Trivia Database integration

### 7.1 Endpoints

- Session token: `https://opentdb.com/api_token.php?command=request`
- Questions: `https://opentdb.com/api.php?amount=10&category=<id>&type=multiple&token=<token>&encode=url3986`
- Token reset (when the token is exhausted): `https://opentdb.com/api_token.php?command=reset&token=<token>`

Verify the parameter names and category IDs against https://opentdb.com/api_config.php before you hardcode them. Use `encode=url3986` and decode each string with `decodeURIComponent`. If that parameter does not work as described, decode HTML entities with a small helper instead.

### 7.2 Category map

| Slug | Display name | opentdb ID | Colour |
|---|---|---|---|
| `history` | History | 23 | #C47A2C |
| `geography` | Geography | 22 | #2E9E6A |
| `science` | Science & Nature | 17 | #2F80C1 |
| `music` | Music | 12 | #B0458F |
| `general` | General Knowledge | 9 | #7C5CC4 |

Store this map in `src/lib/categories.ts`. Icons (Tabler outline names are a suggestion): history `hourglass`, geography `world`, science `flask`, music `music`, general `bulb`. Use `lucide-react` if Tabler is not available.

### 7.3 Session token handling

- Keep the token in an httpOnly cookie named `opentdb_token` set by the route handler, valid for 6 hours.
- On each request, use the cookie if it exists. If it does not exist, request a new token.
- If opentdb returns `response_code` 3 (token not found), request a new token and retry once.
- If it returns `response_code` 4 (token exhausted, meaning every question has been used), call the reset command and retry once.
- Space out requests to respect the rate limit. If a request returns `response_code` 5 (rate limit), wait 5 seconds and retry once before returning an error to the client.

### 7.4 Response codes

| Code | Meaning | What the app does |
|---|---|---|
| 0 | Success | Clean and return the questions. |
| 1 | Not enough questions for this category | Show "Not enough questions in this category right now. Try another category." |
| 2 | Invalid parameter | Log the error on the server. Show the generic error message. |
| 3 | Token not found | Request a new token and retry once. |
| 4 | Token exhausted | Reset the token and retry once. |
| 5 | Rate limited | Wait, retry once, then show the generic error message. |

### 7.5 Cleaning the response

For each result:

- Decode the question and every answer.
- Build `options` as `[correct_answer, ...incorrect_answers]`, then shuffle with Fisher-Yates.
- Store `correctIndex` as the index of the correct answer after shuffling.
- Give each question a stable `id` (the index from the response is fine).

Return this shape:

```ts
type Question = {
  id: number;
  question: string;
  options: [string, string, string, string];
  correctIndex: number;
  category: string;
};
```

---

## 8. Game logic

### 8.1 State

Use a reducer in `src/lib/game-reducer.ts`. The state holds:

- `status`: one of `loading`, `ready`, `error`, `finished`
- `questions`: array of 10 `Question`
- `index`: current question index (0 to 9)
- `score`: correct answers so far
- `selected`: the chosen option index, or `null`
- `timedOut`: boolean
- `timeLeft`: seconds remaining (15 at the start of each question)

### 8.2 Actions

- `LOAD_SUCCESS`, `LOAD_FAILURE`
- `ANSWER(optionIndex)`: ignored if an answer is already selected or the question has timed out. Increments score if correct. Plays the correct or wrong sound.
- `TIMEOUT`: ignored if an answer is already selected. Sets `timedOut` to true and plays the timeout sound.
- `NEXT`: ignored unless the user has answered or timed out. Advances `index`, resets `selected`, `timedOut`, and `timeLeft`. If `index` was 9, sets status to `finished` and navigates to the summary.
- `TICK`: decrements `timeLeft` by 1. Plays the tick sound when `timeLeft` is 5 or less.

### 8.3 Timer

Create `useCountdown` in `src/hooks/useCountdown.ts`. It runs one interval per question using `setInterval` with 1-second steps. It must clear the interval on unmount and when the question changes. Use `requestAnimationFrame` only if you need smoother rendering. Do not let the timer keep running after an answer is selected.

### 8.4 Saving the score

- The summary page reads `score` and `category` from the URL.
- Validate both: `score` must be an integer from 0 to 10, and `category` must be one of the five slugs. If either is invalid, redirect to `/menu`.
- If the user is logged in, POST to `/api/scores` once. Guard against double submission (React strict mode runs effects twice in development, so use a ref or check the response before saving again).
- Show "Score saved" on success, or a retry button with the NFR-03 message on failure.

---

## 9. API routes

### 9.1 `GET /api/questions?category=<slug>`

- Validate `category` against the five slugs. Return 400 for anything else.
- Return 200 with `{ questions: Question[] }`, or an error object `{ error: string }` with a suitable status code.
- Never return the opentdb token or raw API response to the client.

### 9.2 `POST /api/scores`

- Requires a logged-in user. Return 401 if there is no session.
- Body: `{ category: string, score: number }`. Validate both.
- Insert into `scores` with `user_id` taken from the session on the server, never from the request body.
- Return 201 with the saved row, or 400 or 500 with an error message.

---

## 10. Authentication

- Use the Supabase email and password methods: `signUp` with `options.data.username`, and `signInWithPassword`.
- Set up `src/middleware.ts` to refresh the session cookie on every request using the helper in `src/lib/supabase/middleware.ts`.
- Use the server client (`src/lib/supabase/server.ts`) in server components and route handlers. Use the browser client in client components.
- The `auth/callback` route exchanges the confirmation code for a session. If you turn off email confirmation in Supabase for development, the route can stay minimal, but keep it.
- Log out calls `signOut` and redirects to `/`.
- Show the username in the nav bar when logged in, and a "Log in" link when not.
- Validation messages: username 3 to 20 characters, letters, numbers, and underscores only. Password at least 8 characters. Show errors inline next to the field, in the danger colour.

---

## 11. Screens

Each screen below states its layout and its behaviour. Build responsively: stack columns on screens narrower than 768px.

### 11.1 Landing page (`/`)

- Full-height section with the navy background.
- Left column: "Quiz Night" logo text in gold, tagline "Test your knowledge!", and a full-width "Start playing" button that links to `/menu`.
- Right column (when logged out): the `AuthPanel` with Log in and Register tabs. Fields: email, password, and username (register only). A submit button in gold. A small text link to switch between tabs.
- Right column (when logged in): a short welcome message with the username and a "Go to menu" button. The auth forms are hidden.
- Decorative flat circles in the corners (solid colours, no gradients, no blur).
- Background: navy `#14213D`. Panel: `#1F3158`.

### 11.2 Menu (`/menu`)

- Top navigation bar on the deep navy `#0E1A30` background: logo on the left, links to "Leaderboard" and "Theme and sound" toggles on the right, plus the username and "Log out" (or "Log in").
- Heading "Choose a category" (18px, weight 500).
- A 2-column grid of category cards. The General Knowledge card spans both columns. Each card has its category colour as the background, an icon at 22px, and the category name at 15px, weight 500, white text. Minimum card height 80px.
- Hover and focus: the card lifts by 4px with a transition of 150ms. No drop shadows; use a 2px outline in the gold accent on keyboard focus.
- Clicking a card plays the click sound and navigates to `/play/<slug>`.

### 11.3 Game screen (`/play/[category]`)

- Top row: category chip (background in the category colour, white text, 12px), then "Question N of 10" and "Score N" on the right in the secondary text colour.
- Progress bar: 10 segments in a row. Completed segments use the accent gold. Upcoming segments use `#2A3D66` in dark mode and `#D9E0EE` in light mode.
- Timer: a ring or bar showing the 15-second countdown, with the number in the centre. It turns red when 5 seconds or fewer remain.
- Question card: panel background, 17px text, weight 500, padding 18px 16px.
- Answers: 2 by 2 grid on desktop and mobile. Each answer button has a letter (A, B, C, D) in a small circle, 52px tall, 10px radius, panel background, 1px border.
  - Before answering: hover lifts the border to the gold accent.
  - After answering: the correct option turns green. A wrong selection turns red. Other options dim.
  - Options are real `<button>` elements. Once answered, they are disabled.
- Feedback line under the answers: "Correct!" or "Not quite. The answer is <text>." or "Time's up. The answer is <text>." Announce it with `aria-live="polite"`.
- Next button: gold, bottom right, appears only after the question is resolved. Label "Next", or "See results" on question 10.
- Animations: the question card slides in from the right over 250ms. A correct answer gives a short scale-up pulse. A wrong answer gives a short horizontal shake. All animations are disabled when `prefers-reduced-motion` is set.
- Sounds: `click` on answer selection, `correct` or `wrong` on the result, `tick` for the final 5 seconds, `timeout` when the timer expires. Sound is muted when the user has switched it off.

### 11.4 Summary (`/summary?score=N&category=slug`)

- Panel on the navy background, centred text.
- "Round complete" in the secondary colour, 14px.
- Large score, 60px, weight 500, gold: "7 / 10".
- Message by range: 9 to 10 "Outstanding!", 7 to 8 "Great run!", 4 to 6 "Not bad, try again.", 0 to 3 "Keep practising."
- Category name below the message.
- Save status line:
  - Logged in and saved: "Score saved to the leaderboard."
  - Logged in and failed: the NFR-03 message with a "Try again" button.
  - Guest: "Log in to save your score." with a link to the landing page.
- Buttons stacked vertically: "Play again" (gold, same category) and "Exit to menu" (outline).
- Win sound on scores of 7 or more, played once on load. Play a confetti-style burst animation made from simple shapes for a score of 10.

### 11.5 Leaderboard (`/leaderboard`)

- Heading "Leaderboard".
- Category filter as pill buttons: "All", then the five categories. The active pill uses gold with navy text.
- Table or list of up to 10 rows. Each row shows rank, username, category, score, and date. The top three ranks use gold, silver (`#C0C7D1`), and bronze (`#CD7F32`) circles with navy text.
- Empty state: "No scores yet. Be the first."
- Loading state: skeleton rows. Error state: NFR-03 message with "Try again".
- The filter updates the URL query (`?category=music`) so the page can be shared.

---

## 12. Colour system and themes

Define these as CSS variables on `:root` in `globals.css`, and map them to Tailwind colours in `tailwind.config.ts`. Use `darkMode: 'class'`.

### 12.1 Dark theme (default)

| Token | Hex | Use |
|---|---|---|
| `--bg` | #14213D | Page background |
| `--bg-deep` | #0E1A30 | Navigation bar, inputs |
| `--surface` | #1F3158 | Cards and panels |
| `--border` | #3A4F7A | Borders and dividers |
| `--accent` | #FCA311 | Primary buttons, progress, score |
| `--accent-text` | #14213D | Text on gold buttons |
| `--text` | #FFFFFF | Primary text |
| `--text-secondary` | #C9D3E6 | Secondary text |
| `--text-muted` | #8A9BB8 | Placeholders |
| `--correct` | #2E9E6A | Correct answer |
| `--wrong` | #D64545 | Wrong answer |
| `--progress-empty` | #2A3D66 | Unfilled progress segments |

### 12.2 Light theme

| Token | Hex | Use |
|---|---|---|
| `--bg` | #F4F6FA | Page background |
| `--bg-deep` | #E6EBF5 | Navigation bar, inputs |
| `--surface` | #FFFFFF | Cards and panels |
| `--border` | #C9D3E6 | Borders and dividers |
| `--accent` | #FCA311 | Gold fills (buttons, progress, bars) |
| `--accent-text` | #14213D | Text on gold buttons |
| `--text` | #14213D | Primary text |
| `--text-secondary` | #4A5878 | Secondary text |
| `--text-muted` | #6B7896 | Placeholders |
| `--correct` | #2E9E6A | Correct answer fill |
| `--wrong` | #D64545 | Wrong answer fill |
| `--progress-empty` | #D9E0EE | Unfilled progress segments |

In light mode, do not use gold as text on white, because the contrast is too low. Use `#B76E00` for gold text, and keep the gold fills for buttons and bars.

### 12.3 Category colours (same in both themes)

History `#C47A2C`, Geography `#2E9E6A`, Science & Nature `#2F80C1`, Music `#B0458F`, General Knowledge `#7C5CC4`. White text on these colours passes contrast for their large labels. Do not change them between themes.

### 12.4 Theme behaviour

- Default to the operating system preference (`prefers-color-scheme`) on the first visit, then remember the user's choice in `localStorage` under `quiznight-theme`. Storing this preference is allowed because it is not game data.
- A `ThemeToggle` in the navigation bar switches between light and dark. Add the `dark` class to `<html>` and avoid a flash of the wrong theme on load by setting the class in a small inline script in the root layout.
- Every colour in components must come from the tokens above. No hardcoded hex values inside components.

---

## 13. Sounds

- Use Howler.js through `src/lib/sounds.ts`. Load each file once and reuse it.
- Sound files go in `public/sounds/`: `click.mp3`, `correct.mp3`, `wrong.mp3`, `tick.mp3`, `timeout.mp3`, `win.mp3`. Keep each file under 50 KB. Use short royalty-free effects from a source with a clear licence, and note the source in the README.
- A `SoundToggle` in the navigation bar turns all sound on or off. Save the setting in `localStorage` under `quiznight-sound`. Default to on.
- Browsers block audio until the user interacts with the page. Start sounds only after the first click, which the Start button provides.
- Respect `prefers-reduced-motion` for animation only. Do not disable sound because of it.

---

## 14. Animations

Use Framer Motion (`motion`). Keep every animation short (under 400ms) and subtle.

- Page transitions: fade and move up 8px, 200ms.
- Category cards: hover lift of 4px, 150ms.
- Question card: slide in from the right, 250ms.
- Correct answer: scale to 1.04 and back, 200ms.
- Wrong answer: horizontal shake of 6px, 300ms.
- Progress segments: fill in with a 300ms transition when completed.
- Summary score: count up from 0 to the final value over 800ms.
- Perfect score (10 of 10): a burst of about 30 small shapes in the gold, green, and category colours. Remove the shapes after 2 seconds.

All of these must be disabled when `prefers-reduced-motion` is set. Use Framer Motion's `useReducedMotion` hook to check this, and render the final state immediately in that case.

---

## 15. Accessibility

- Every interactive element is a `<button>` or `<a>` with a visible focus ring (2px gold outline).
- The timer has an accessible label such as "12 seconds left" and updates at most once per second for screen readers.
- Feedback text uses `aria-live="polite"`. Errors use `role="alert"`.
- Form fields have labels (not just placeholders). Errors are linked with `aria-describedby`.
- Colour is never the only signal. Correct and wrong answers also show a check or cross icon and the words "Correct" or "Wrong".
- The page has one `<h1>` per screen.

---

## 16. Error and empty states

| Situation | What the user sees |
|---|---|
| Question fetch fails | "We couldn't load questions. Check your connection and try again." with a "Try again" button |
| Not enough questions (opentdb code 1) | "Not enough questions in this category right now. Try another category." with a "Back to menu" button |
| Score save fails | "We couldn't save your score." with a "Try again" button |
| Leaderboard fails to load | "We couldn't load the leaderboard." with a "Try again" button |
| Leaderboard is empty | "No scores yet. Be the first." |
| Sign-up fails (username taken) | "That username is taken. Try another." |
| Sign-in fails | "Email or password is incorrect." (Do not reveal which one is wrong.) |
| Session expires during a game | The game continues. Saving will fail with a login prompt. |

---

## 17. Environment and configuration

`.env.example` must contain:

```
NEXT_PUBLIC_SUPABASE_URL=your-project-url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
```

- Do not add a service role key. This app does not need one.
- Add `.env.local` to `.gitignore`. Next.js does this by default, so check it.
- The README must explain how to create a Supabase project, run `supabase/schema.sql`, copy the URL and anon key, and run `npm run dev`.

---

## 18. Testing and acceptance checks

Write these checks as a manual test checklist in the README. Add automated tests only where they are quick to write (the reducer and the cleaning function are good candidates, using Vitest).

### 18.1 Game

- [ ] Each category loads exactly 10 questions.
- [ ] No HTML entities such as `&quot;` or `&#039;` appear on screen.
- [ ] Answers appear in a different order between games.
- [ ] Correct answer turns green, wrong selection turns red, and each question can be answered only once.
- [ ] Score on the summary matches the number of correct answers.
- [ ] Timer counts down from 15, stops on answer, and at zero marks the question wrong and reveals the answer.
- [ ] Timer stops and does not restart if the user switches tabs and returns (check the logic, not just the display).
- [ ] "Play again" loads a new set of questions for the same category.
- [ ] "Exit to menu" returns to `/menu`.

### 18.2 Accounts and leaderboard

- [ ] Registration with a duplicate username shows the correct error.
- [ ] A new account has a profile row (check in the Supabase table editor).
- [ ] A logged-in user's score is saved with the correct category.
- [ ] A guest's score is not saved, and the summary shows the login prompt.
- [ ] Log out clears the session and the nav bar updates.
- [ ] The leaderboard shows the top 10 and the category filter works.
- [ ] A user cannot insert a score for another user (test this with a second account, or confirm the RLS policy).

### 18.3 Look, sound, and motion

- [ ] Layout checked at 360px, 768px, and 1280px widths.
- [ ] Dark and light themes both pass the contrast check in section 15 (use a browser contrast tool).
- [ ] Theme choice persists after reload with no flash of the wrong theme.
- [ ] Sound toggle mutes all sound and the setting persists.
- [ ] With reduced motion enabled in the operating system, animations are off and the final states still display.

### 18.4 Failure cases

- [ ] Turning off Wi-Fi during a game shows the question fetch error when starting a new game, and the in-progress game still works.
- [ ] A broken API response shows the NFR-03 message, not a crash.

---

## 19. Build order

Build in this order so that every step can be tested before the next one starts.

1. Create the Next.js project, Tailwind config with the colour tokens, and the global layout with the theme provider.
2. Build `categories.ts`, `shuffle.ts`, and the `opentdb.ts` helper. Test the cleaning function on a sample response.
3. Build the `/api/questions` route and call it from a test page.
4. Build the game reducer and the `/play/[category]` screen without the timer or sound.
5. Add the timer, the summary page, and "Play again".
6. Build the landing page and the menu.
7. Set up Supabase, run `schema.sql`, and add the auth helpers and middleware.
8. Build the `AuthPanel`, the nav bar, and the logout flow.
9. Build `POST /api/scores` and connect the summary page to it.
10. Build the leaderboard page and the category filter.
11. Add sounds and the sound toggle.
12. Add animations and the reduced-motion handling.
13. Polish the layouts at each breakpoint, check contrast in both themes, and run the checklist in section 18.
14. Write the README.

---

## 20. Deliverables

- The complete source code following the folder structure in section 3.3.
- `supabase/schema.sql`.
- `.env.example`.
- `README.md` with setup steps, the manual test checklist, and sound file credits.
- No leftover test pages, console logs, or unused components.

---

## 21. Coding standards

- TypeScript strict mode with no `any`. Use `unknown` and narrow it when parsing API responses.
- Components are small and do one job. Keep game logic in the reducer and the hooks, not in the JSX.
- Server-only code (anything that uses cookies or the opentdb token) must not be imported from client components.
- Add a short comment referencing the requirement ID (for example `// FR-19: timer`) where a requirement is implemented.
- Use descriptive names. Use sentence case for all visible UI text, with no ALL CAPS and no emoji.
- Use Tailwind classes for styling. Use CSS variables from section 12 for theme colours.
- Do not commit the Supabase project URL or anon key into the source code. Read them from environment variables only.
