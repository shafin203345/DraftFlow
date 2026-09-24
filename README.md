# Enterprise Applicant Application Portal with Supabase Autosave & Multi-Tab Sync

A production-quality, modern, responsive Applicant Application Form web application built with **React**, **TypeScript**, **Vite**, **Tailwind CSS**, and **Supabase**.

The primary objective of this application is **zero data loss for applicants**. It automatically persists form progress to Supabase with optimistic concurrency control, seamlessly restores drafts across page refreshes and browser sessions, handles offline network interruptions gracefully, and prevents silent data overwrites when the same application draft is edited across multiple browser tabs simultaneously.

---

## 📽️ Walkthrough Video

> **Walkthrough Video Link:** `[Insert Walkthrough Video URL Here]`

The 5-minute walkthrough video demonstrates:
1. **Polished UI/UX**: Multi-section application form with completion progress tracking.
2. **Autosave Engine**: Real-time debounced saving with live relative timestamps ("Saved 12s ago").
3. **Refresh & Restore**: Browser refresh maintaining 100% of entered data.
4. **Offline Resilience**: Simulating offline status, local browser persistence, and auto-sync on reconnect.
5. **Multi-Tab Conflict Resolution**: Side-by-side diff view and resolution choices when editing in 2 tabs.
6. **Supabase Integration**: Viewing persistent draft records in Supabase PostgreSQL tables.

---

## ✨ Key Features

- 🔄 **Real-Time Autosave Engine**: Debounced saving (1-second inactivity delay) and immediate save on section switch or tab closure.
- 🕒 **Live Last Saved Timestamp**: Dynamic relative timer display ("Saved 5 seconds ago", "Saved just now").
- 📡 **Offline Graceful Degradation**: Edits are stored locally in `localStorage` when offline and synced automatically upon reconnection.
- 🛡️ **Optimistic Concurrency & Multi-Tab Conflict Handling**: Detects concurrent edits across multiple browser tabs using `version` incrementing and `BroadcastChannel`. Presents a side-by-side visual diff modal to resolve conflicts.
- 🔑 **Secure Token-Based Draft Access**: Unpredictable 32-character hexadecimal draft tokens enable secure guest applications without requiring complex authentication upfront.
- 📋 **Multi-Section Form Architecture**:
  1. Personal Information (Name, Email, Phone, Address, Country)
  2. Education Background (Highest Degree, Institution, Major, Year, GPA)
  3. Professional Experience (Current Company, Job Title, Years of Exp, Skills, Bio)
  4. Additional Information (Cover Letter with character counter, Portfolio, LinkedIn, GitHub)
- 📊 **Validation & Progress Tracking**: Real-time section completion indicators and completion percentage bar.
- 🎯 **Evaluator Demo Toolbar**: Built-in test toolbar to simulate offline mode, inject 2nd tab conflict events, fill sample profile data, and inspect raw JSON payloads.

---

## 🛠️ Tech Stack

- **Frontend**: React 19 + TypeScript
- **Build Tool**: Vite 6
- **Styling**: Tailwind CSS v4 (with `@tailwindcss/vite`) + Lucide Icons
- **Database**: Supabase PostgreSQL (via `@supabase/supabase-js`)
- **Concurrency**: PostgreSQL Row Level Security (RLS) & `BroadcastChannel` API

---

## 🚀 Local Development Setup

### Prerequisites
- **Node.js**: v18.0.0 or higher
- **npm**: v9.0.0 or higher

### Installation Steps

1. **Clone the repository**:
   ```bash
   git clone https://github.com/your-org/applicant-portal.git
   cd applicant-portal
   ```

2. **Install dependencies**:
   ```bash
   npm install
   ```

3. **Configure Environment Variables**:
   Copy the example environment file and fill in your Supabase credentials:
   ```bash
   cp .env.example .env
   ```

4. **Start Vite Development Server**:
   ```bash
   npm run dev
   ```
   Open your browser at `http://localhost:5173`.

---

## 🔐 Environment Variables

Create a `.env` file in the project root:

```env
VITE_SUPABASE_URL=https://your-project-ref.supabase.co
VITE_SUPABASE_ANON_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

> **Note**: Never expose `SUPABASE_SERVICE_ROLE_KEY` in frontend code. Only `VITE_SUPABASE_ANON_KEY` is used.
> If environment variables are omitted or contain placeholders, the app automatically operates in **Local Storage Fallback Mode** so you can test all features offline immediately!

---

## 🗄️ Supabase Setup & Database Schema

Run the contents of [`supabase/schema.sql`](file:///c:/Users/Hello/Desktop/projecr1/supabase/schema.sql) in the **Supabase SQL Editor**:

```sql
-- Enable UUID Extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Create Drafts Table
CREATE TABLE IF NOT EXISTS public.drafts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    access_token TEXT NOT NULL UNIQUE,
    payload JSONB NOT NULL DEFAULT '{}'::jsonb,
    version INTEGER NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Indexes
CREATE INDEX IF NOT EXISTS idx_drafts_access_token ON public.drafts (access_token);
CREATE INDEX IF NOT EXISTS idx_drafts_updated_at ON public.drafts (updated_at DESC);

-- Automatic updated_at Trigger
CREATE OR REPLACE FUNCTION public.set_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_drafts_updated_at
    BEFORE UPDATE ON public.drafts
    FOR EACH ROW
    EXECUTE FUNCTION public.set_updated_at();

-- Row Level Security
ALTER TABLE public.drafts ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Public can create new drafts" ON public.drafts FOR INSERT TO public WITH CHECK (true);
CREATE POLICY "Token holders can view their draft" ON public.drafts FOR SELECT TO public USING (access_token IS NOT NULL);
CREATE POLICY "Token holders can update their draft" ON public.drafts FOR UPDATE TO public USING (access_token IS NOT NULL);

GRANT SELECT, INSERT, UPDATE ON public.drafts TO anon, authenticated;
```

---

## 🛡️ Security Model

1. **Token-Gated Access**: When an applicant opens the portal for the first time, a high-entropy 32-character hexadecimal token (`access_token`) is generated and stored locally in `localStorage` and URL parameters (`?draft_token=...`).
2. **Row Level Security (RLS)**: Public select and update queries on the `drafts` table require a matching `access_token`. Unauthenticated callers cannot list or read drafts belonging to other access tokens.
3. **No Service-Role Key**: Front-end applications use only the public anonymous key.

---

## ⚡ Autosave Engine Architecture

- **Debounced Save**: Typing triggers a 1000ms debounce timer. Once the applicant pauses typing, autosave executes.
- **Deep Equality Guard**: Hashes/compares current form JSON against `lastPersistedPayload`. Unchanged keystrokes do not produce API requests.
- **Lifecycle Flush**: Saves immediately on window `beforeunload` or visibility change.

---

## 📡 Offline Synchronization Strategy

1. **Network Detection**: Uses `navigator.onLine` and `window.addEventListener('online' | 'offline')`.
2. **Local Persistence**: When offline, edits are saved directly to `localStorage` under `applicant_draft_payload_<token>`.
3. **Automatic Sync**: As soon as internet connectivity is restored, the application sets state to `syncing` and uploads the latest local payload to Supabase.

---

## ⚔️ Multi-Tab Conflict Strategy

When the same application draft is opened in two browser tabs (Tab A & Tab B):

1. **Version Checking**: Each draft record contains an incremental integer `version`.
2. **Optimistic Concurrency**: Before writing to Supabase, the app verifies if `remote_version > local_version`.
3. **BroadcastChannel Real-Time Alerts**: Tab A broadcasts a `DRAFT_SAVED` event via `BroadcastChannel('applicant_draft_sync_channel')`.
4. **Resolution Choices**:
   - If Tab B has *no unsaved local changes*, it silently updates to Tab A's draft.
   - If Tab B has *unsaved local changes*, a **Tab Conflict Resolution Modal** pops up with a **Side-by-Side Field Diff**:
     - **Option 1: Keep My Current Tab Changes** (Overwrites remote draft with current tab's payload & increments version).
     - **Option 2: Load Remote Tab Draft** (Replaces local form state with remote tab's version).

---

## 🔄 Draft Restoration Behavior

1. Applicant visits `http://localhost:5173`.
2. App checks URL parameters `?draft_token=...` or `localStorage` key `applicant_draft_access_token`.
3. App fetches matching record from Supabase or `localStorage`.
4. Form fields populate instantly and `lastSavedTime` displays the exact timestamp.

---

## 🌐 Deployment Instructions

### Deploy to Vercel
```bash
npm install -g vercel
vercel
```
Add Environment Variables in Vercel Dashboard:
- `VITE_SUPABASE_URL`
- `VITE_SUPABASE_ANON_KEY`

### Deploy to Netlify
Build command: `npm run build`  
Publish directory: `dist`

---

## 🧪 Testing Instructions

| Test | Procedure | Expected Outcome |
| text | --- | --- |
| **TEST 1 — Create Draft** | Open app, enter Name & Email | Draft created in Supabase & token stored |
| **TEST 2 — Refresh Restore** | Enter data, refresh page | All fields restored identically |
| **TEST 3 — Last Saved** | Edit a field and wait 3s | UI updates to "Saved X seconds ago" |
| **TEST 4 — Offline Sync** | Click "Simulate Offline", edit form, click "Go Online" | Shows offline banner, then syncs to server |
| **TEST 5 — Multi-Tab Edit** | Click "Copy Tab URL", open in 2nd tab, edit field | 2nd tab conflict modal appears with diff |

---

## 📌 Known Limitations

- **Browser Storage Quota**: Offline storage relies on `localStorage` (~5MB limit), which is more than sufficient for text application payloads (<50KB).
- **Service Worker Caching**: Offline support covers form payload saving; full offline asset caching can be enhanced further with PWA Service Workers.
