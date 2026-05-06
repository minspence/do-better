# Project Setup Commands

Run these in order from your terminal. Everything below assumes you have Node 18+ and pnpm installed.

---

## 1. Create the Next.js Project

```bash
pnpm create next-app@latest grind-app --typescript --tailwind --eslint --app --src-dir --import-alias "@/*"
cd grind-app
```

> Name it whatever you want — `grind-app` is just a placeholder.

---

## 2. Install Core Dependencies

```bash
# Supabase
pnpm add @supabase/supabase-js @supabase/ssr

# Animations
pnpm add framer-motion

# Icons
pnpm add lucide-react

# Class merging utility (essential for component variants)
pnpm add clsx tailwind-merge

# Date utilities
pnpm add date-fns
```

---

## 3. Install Dev Dependencies

```bash
pnpm add -D @types/node
```

---

## 4. Install Capacitor

```bash
pnpm add @capacitor/core
pnpm add -D @capacitor/cli
pnpm add @capacitor/ios @capacitor/android

# Capacitor plugins you'll need
pnpm add @capacitor/push-notifications @capacitor/haptics @capacitor/status-bar @capacitor/splash-screen
```

---

## 5. Initialize Capacitor

```bash
pnpm exec cap init "Your App Name" "com.yourname.yourapp" --web-dir=out
```

> Replace `"Your App Name"` and `"com.yourname.yourapp"` with your real values.
> The `out` directory is where Next.js puts the static export — we'll configure this below.

---

## 6. Add Mobile Platforms

```bash
pnpm exec cap add ios
pnpm exec cap add android
```

> You'll need Xcode installed for iOS (Mac only) and Android Studio for Android.
> You don't need these to build the web app — add them when you're ready to test on device.

---

## 7. Set Up Supabase CLI (for schema management)

```bash
# Install Supabase CLI globally
pnpm add -D supabase

# Initialize Supabase in your project
pnpm exec supabase init
```

---

## 8. Environment Variables

Create a `.env.local` file in the root of your project:

```env
NEXT_PUBLIC_SUPABASE_URL=your-supabase-project-url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-supabase-anon-key
```

You get these from: **Supabase Dashboard → Your Project → Settings → API**

---

## 9. Copy the Scaffold Files

Copy all files from this scaffold into your project, matching the directory structure exactly.

---

## 10. Development Workflow

```bash
# Run web dev server
pnpm dev

# Build static export (for Capacitor)
pnpm build

# Sync web build to native platforms
pnpm exec cap sync

# Open in Xcode (iOS)
pnpm exec cap open ios

# Open in Android Studio
pnpm exec cap open android

# Generate TypeScript types from Supabase schema (run after schema changes)
pnpm exec supabase gen types typescript --project-id YOUR_PROJECT_ID > src/types/database.types.ts
```

---

## Folder Structure at a Glance

```
src/
├── app/
│   ├── (auth)/          ← Login, signup pages
│   ├── (app)/           ← All protected app pages
│   └── layout.tsx       ← Root layout
├── components/
│   ├── ui/              ← Reusable primitives (Button, Card, etc.)
│   ├── gamification/    ← XP bars, streak counters, badges
│   ├── habits/          ← Habit-specific components
│   └── layout/          ← BottomNav, Header, ProtectedLayout
├── lib/
│   ├── supabase/        ← Supabase client instances
│   ├── hooks/           ← Custom React hooks
│   └── utils/           ← Helper functions (XP calc, streaks, etc.)
└── types/               ← TypeScript types
```
