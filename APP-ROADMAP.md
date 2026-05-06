# Self-Improvement App — Project Roadmap & Strategy

---

## 1. Framework Decision: Next.js + Capacitor vs React Native

This is the most important architectural decision you'll make, so it deserves a honest breakdown.

### Option A: Next.js + Capacitor (Stay in your comfort zone)

Capacitor is a tool made by the Ionic team that wraps your web app in a native shell and lets you publish it to the App Store and Play Store. You write one Next.js app and Capacitor handles the native container.

**What you gain:**

- One codebase for web and mobile
- You stay in Next.js, TypeScript, and CSS — no new framework to learn
- Faster path to a working v1
- Access to native APIs (camera, push notifications, haptics) via Capacitor plugins
- Expo-style convenience without leaving the web ecosystem

**What you give up:**

- Animations and transitions will feel slightly "webby" unless you invest heavily in Framer Motion or similar
- Scroll behavior on iOS in particular can feel off without extra tuning
- Some native UI patterns (swipe-to-dismiss, native date pickers, etc.) require extra work
- The App Store review process can be finicky about web-wrapped apps — not a dealbreaker, but a known friction point

**Best for:** Solo developers who want to ship fast and iterate. For a v1 of a habit tracker, this is a very reasonable path.

---

### Option B: React Native with Expo

React Native renders actual native components — not a web view. The UI feels genuinely native because it is. Expo is a toolchain on top of React Native that handles builds, updates, and device access in a developer-friendly way.

**What you gain:**

- Genuinely native feel — scrolling, animations, gestures all behave exactly as users expect on iOS and Android
- Better performance for animation-heavy gamification (level-up screens, streak celebrations, etc.)
- Access to the full React Native ecosystem (React Navigation, Reanimated, etc.)
- Expo's Over-the-Air updates mean you can push bug fixes without going through the App Store review

**What you need to learn:**

- Styling is not CSS — it uses a Flexbox-based StyleSheet with no cascading, no class names
- Navigation is done with React Navigation or Expo Router, not Next.js routing
- You lose `next/image`, server components, and API routes — your backend logic lives entirely in Supabase or a separate API
- Building for both platforms requires some platform-specific handling

**Transition path if you go this route:**

1. Start with **Expo Router** — it mirrors Next.js file-based routing closely and will feel the most familiar
2. Learn **NativeWind** — it brings Tailwind-style utility classes to React Native, which dramatically reduces the styling learning curve
3. Keep Supabase as your backend — it works identically in React Native
4. Your TypeScript types and business logic are fully transferable

---

### Recommendation

For a solo developer shipping a v1: **start with Next.js + Capacitor**. Get the product right — the features, the gamification loops, the retention hooks. Once you have real users giving feedback, you'll know whether the native feel is a genuine problem worth solving. If it is, migrating the UI to React Native is easier than building the wrong product twice. The Supabase layer won't change at all.

---

## 2. Supabase Schema Foundation

Since you're still learning Supabase, here's a schema designed to scale with the features you've described. Build these tables in phases — don't try to create all of them upfront.

### Phase 1 Tables (Build first)

```sql
-- Extends Supabase auth.users automatically
profiles (
  id uuid references auth.users primary key,
  username text unique,
  display_name text,
  avatar_url text,
  xp integer default 0,
  level integer default 1,
  streak_count integer default 0,
  longest_streak integer default 0,
  last_active_date date,
  is_premium boolean default false,
  premium_expires_at timestamptz,
  trial_started_at timestamptz,
  created_at timestamptz default now()
)

habits (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references profiles(id) on delete cascade,
  title text,
  category text check (category in ('self-improvement', 'exercise', 'memorization', 'diet')),
  frequency text check (frequency in ('daily', 'weekly', 'custom')),
  target_count integer default 1,
  xp_reward integer default 10,
  is_premium_content boolean default false,
  created_at timestamptz default now()
)

habit_logs (
  id uuid primary key default gen_random_uuid(),
  habit_id uuid references habits(id) on delete cascade,
  user_id uuid references profiles(id) on delete cascade,
  completed_at timestamptz default now(),
  notes text
)
```

### Phase 2 Tables (Add when building gamification)

```sql
achievements (
  id uuid primary key default gen_random_uuid(),
  title text,
  description text,
  icon_url text,
  xp_reward integer,
  condition_type text,  -- 'streak', 'total_completions', 'category_count', etc.
  condition_value integer
)

user_achievements (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references profiles(id) on delete cascade,
  achievement_id uuid references achievements(id),
  earned_at timestamptz default now()
)

streak_freezes (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references profiles(id) on delete cascade,
  used_on_date date,
  created_at timestamptz default now()
)
```

### Phase 3 Tables (Add when building social)

```sql
friendships (
  id uuid primary key default gen_random_uuid(),
  requester_id uuid references profiles(id),
  addressee_id uuid references profiles(id),
  status text check (status in ('pending', 'accepted', 'blocked')),
  created_at timestamptz default now(),
  unique(requester_id, addressee_id)
)

challenges (
  id uuid primary key default gen_random_uuid(),
  created_by uuid references profiles(id),
  title text,
  category text,
  start_date date,
  end_date date,
  is_public boolean default false
)

challenge_participants (
  challenge_id uuid references challenges(id),
  user_id uuid references profiles(id),
  joined_at timestamptz default now(),
  primary key (challenge_id, user_id)
)
```

### Key Supabase Concepts to Learn Now

**Row Level Security (RLS)** is the most important concept to internalize. Every table should have RLS enabled so that users can only read and write their own data. A basic policy looks like this:

```sql
create policy "Users can only see their own habits"
  on habits for select
  using (auth.uid() = user_id);
```

**Supabase Edge Functions** will be useful later for things like streak calculations run server-side, webhook handling from Stripe, and sending push notifications on a schedule.

**Realtime** is Supabase's WebSocket feature — you'll use this for the social/leaderboard features so scores update live without polling.

---

## 3. Full Roadmap by Phase

### Phase 0 — Foundation (Weeks 1–2)

- Initialize Next.js project with TypeScript and Tailwind
- Set up Supabase project, connect to app, configure environment variables
- Implement Supabase Auth (email/password + Google OAuth at minimum)
- Build the profiles table and create a profile on signup via database trigger
- Set up Capacitor for mobile builds, test on a simulator
- Configure GitHub repository with basic CI (lint + type check on push)

### Phase 1 — Core Habit Loop (Weeks 3–6)

This is the most important phase. Nothing else matters if the core loop isn't satisfying.

- Build the four category screens (self-improvement, exercise, memorization, diet)
- Create habit creation flow with frequency options
- Build the daily check-in screen — this is your most-used screen, invest in it
- Implement streak calculation logic (server-side via Edge Function or client-side with careful validation)
- Display current streak prominently — it needs to feel like something worth protecting
- Basic XP system: completing a habit awards XP, XP accumulates toward levels
- Simple level-up moment with animation — this is your first "delight" moment

### Phase 2 — Gamification Layer (Weeks 7–10)

- Achievement system: define 20–30 achievements across categories (first 7-day streak, complete 50 workouts, etc.)
- Achievement unlock animation and notification
- Streak Freeze item: users earn or buy these to protect their streak when they miss a day
- XP history and level progression screen
- Weekly summary card showing the week's completions, XP earned, and streaks held

### Phase 3 — Social & Competition (Weeks 11–14)

- Friend system: search by username, send/accept requests
- Friend leaderboard: weekly XP ranking among friends
- Public challenges: users can create or join a 7/14/30-day challenge in a category
- Challenge leaderboard and completion tracking
- Activity feed showing friends' recent achievements (keep this lightweight — avoid becoming a social network)

### Phase 4 — Monetization (Weeks 15–17)

- Integrate **Stripe** for subscription management
- Define premium content gates: curated workout plans, diet templates, advanced memorization decks
- Implement 14-day trial logic: set `trial_started_at` on signup, gate premium content based on trial window
- Build the paywall/upgrade screen — this needs to clearly communicate value
- Use **Stripe Customer Portal** so users can manage their own subscriptions without you building that UI
- Handle webhook events from Stripe to update `is_premium` and `premium_expires_at` in Supabase

### Phase 5 — Retention Infrastructure (Weeks 18–20)

- Push notifications via **OneSignal** or **Firebase Cloud Messaging**: daily reminder at user's preferred time, streak-at-risk warning (haven't checked in by 8pm), achievement unlocked
- Onboarding flow for new users: choose 2–3 starter habits, explain the streak system, show what premium looks like
- Re-engagement email sequence via **Resend** or **Postmark**: Day 3 after signup, Day 7, and "we miss you" at Day 14 of inactivity
- App Store and Play Store submission

### Phase 6 — Ongoing (Post-launch)

- A/B test onboarding variations
- Add new premium content packs based on user demand
- Monthly themed challenges to re-engage churned users
- Desktop web version (this is now trivial since you're already in Next.js)

---

## 4. Full Tools Ecosystem

### Core Stack

| Tool                 | Purpose                                     |
| -------------------- | ------------------------------------------- |
| Next.js + TypeScript | Frontend framework                          |
| Supabase             | Database, auth, realtime, edge functions    |
| Capacitor            | Native mobile wrapper                       |
| Tailwind CSS         | Styling                                     |
| Framer Motion        | Animations (level-ups, achievement unlocks) |

### Payments

| Tool                   | Purpose                                |
| ---------------------- | -------------------------------------- |
| Stripe                 | Subscription billing, trial management |
| Stripe Customer Portal | Self-serve subscription management     |

### Notifications & Communication

| Tool      | Purpose                                    |
| --------- | ------------------------------------------ |
| OneSignal | Push notifications (free tier is generous) |
| Resend    | Transactional email (clean API, great DX)  |

### Analytics & Monitoring

| Tool    | Purpose                                                      |
| ------- | ------------------------------------------------------------ |
| PostHog | Product analytics, funnels, session recording, feature flags |
| Sentry  | Error monitoring and crash reporting                         |

PostHog deserves special mention — it's open source, has a generous free tier, and gives you feature flags which are invaluable for A/B testing retention changes without redeploying.

### Development & Deployment

| Tool                          | Purpose                                 |
| ----------------------------- | --------------------------------------- |
| Vercel                        | Next.js hosting (seamless, zero config) |
| GitHub Actions                | CI: lint, type check, test on every PR  |
| Expo EAS (if switching to RN) | Mobile build service                    |

### Content Management (Optional but useful)

| Tool                 | Purpose                                                  |
| -------------------- | -------------------------------------------------------- |
| Sanity or Contentful | Manage premium workout/diet content without code deploys |

This lets you add new premium content — workout plans, meal templates, memorization decks — through a CMS interface rather than database migrations.

---

## 5. Retention: What Actually Works Beyond Week 2

The honest truth is that most habit apps lose 70–80% of users in the first two weeks. These are the mechanics that meaningfully fight churn:

**Make the streak feel precious.** The streak counter should be the most visible number in the app. When someone is at a 14-day streak, they will think about the app before they go to bed. Duolingo's entire retention strategy is built on this psychology. Add a "streak at risk" push notification that fires at 9pm if the user hasn't checked in — this single feature has documented retention impact.

**Streak Freezes as earned items, not purchased ones.** Let users earn freeze tokens by hitting milestone streaks (7 days, 30 days, etc.). This makes them feel like a reward and teaches the mechanic organically. You can also include 1-2 freeze tokens in the premium trial to show their value.

**Celebrate the small wins, not just the big ones.** A confetti animation on Day 3 means as much as one on Day 30 for a new user. Every completion should feel acknowledged. Haptic feedback on mobile is a small touch that makes completions feel satisfying and real.

**Weekly summaries that feel personal.** A Sunday evening notification that says "You completed 18 habits this week and hit a 12-day streak — your best yet" takes 10 seconds to read and makes people feel seen. This is low-effort to build and high-impact on retention.

**Social accountability is underrated for this category.** When a friend completes a challenge before you do, it creates a competitive itch. Keep the social layer lightweight (a leaderboard and challenges is enough) — you don't want to build Twitter, but peer pressure is one of the strongest behavior change tools that exists.

**The trial needs to show value, not just unlock features.** During the 14-day premium trial, surface the best premium content proactively. Don't make users go hunting for what they're getting. At day 12 of the trial, send a reminder of what they've used and what they'd lose at expiry.

**Personalized difficulty matters.** If every habit is too easy, users feel no progression. If too hard, they quit. Build in a difficulty setting per habit and suggest adjustment if the user is completing something 100% of the time (too easy) or less than 50% (too hard).

---

## 6. Long-Term Maintenance Practices

**Keep your schema migrations version-controlled.** Supabase has a CLI that lets you track schema changes as files in your repo. Use it from day one — this will save you significant pain when you need to replicate the database for staging or debugging.

**Separate staging and production Supabase projects.** Create two projects: one for development/testing and one for production. Never test schema changes on real user data.

**Write TypeScript types for your Supabase tables.** Supabase can auto-generate TypeScript types from your schema with one CLI command (`supabase gen types typescript`). Run this after every schema change and commit the output. This gives you full type safety across your entire data layer.

**Define your premium content in a CMS, not in the database.** Workout plans and diet templates change frequently. If they live in Supabase tables, every update is a database migration. If they live in a headless CMS, a non-technical collaborator (or future you) can update them without touching code.

**Set up error alerts from day one.** Sentry will email you when something breaks in production. Knowing about a crash before a user reports it is the difference between a one-star review and a quiet fix.

**Monitor your key retention metrics weekly:** Day 1, Day 7, and Day 30 retention rates, streak distribution (what percentage of users have an active streak), and trial-to-paid conversion rate. PostHog makes all of this trackable with funnels you define once.

---

## 7. Recommended Build Order (Summary)

1. Auth + profiles + Supabase foundation
2. One category (start with Exercise — it's the most motivating for users)
3. Habit creation, daily check-in, streak tracking
4. XP + leveling
5. Achievements
6. Add remaining three categories
7. Social features
8. Monetization + trial
9. Push notifications + email re-engagement
10. App Store submission
11. CMS for premium content
12. Desktop web (you already have it — just make it responsive)

---

_Built with: Next.js + TypeScript + Supabase + Capacitor_
_Last updated: April 2026_
