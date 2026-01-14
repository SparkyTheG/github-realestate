# 🚀 Fresh Setup Guide - Railway + Supabase

## Part 1: Railway Environment Variables

### Required Variables (Backend Deployment)

Add these environment variables in your Railway project settings:

```bash
# ===== REQUIRED: OpenAI API Key =====
OPENAI_API_KEY=sk-proj-your-openai-api-key-here

# ===== REQUIRED: Supabase Configuration =====
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_ANON_KEY=your-supabase-anon-key-here

# ===== OPTIONAL: ElevenLabs for Audio Transcription =====
ELEVENLABS_API_KEY=your-elevenlabs-api-key-here

# ===== OPTIONAL: OpenAI Realtime Configuration =====
OPENAI_REALTIME_MODEL=gpt-4o-realtime-preview-2024-12-17
OPENAI_REALTIME_DISABLED=false

# ===== RAILWAY AUTO-SET (No action needed) =====
RAILWAY_GIT_COMMIT_SHA=auto-set-by-railway
PORT=auto-set-by-railway
```

### How to Get These Values:

**OPENAI_API_KEY:**
- Go to https://platform.openai.com/api-keys
- Create a new API key
- Copy it immediately (you won't see it again)

**SUPABASE_URL and SUPABASE_ANON_KEY:**
- Go to your Supabase project dashboard
- Click "Settings" → "API"
- Copy "Project URL" → this is `SUPABASE_URL`
- Copy "anon" key (under "Project API keys") → this is `SUPABASE_ANON_KEY`

**ELEVENLABS_API_KEY** (optional):
- Go to https://elevenlabs.io/
- Sign up/login
- Go to "Profile" → "API Keys"
- Copy your API key

---

## Part 2: Supabase SQL Migrations

Run these SQL scripts **in order** in your Supabase SQL Editor (Dashboard → SQL Editor → New Query).

---

### Migration 1: Create Closer Profiles Table
**File:** `20251119000931_create_closer_profiles_table.sql`

```sql
/*
  # Create Closer Profiles Table

  1. New Tables
    - `closer_profiles`
      - `id` (uuid, primary key)
      - `closer_id` (text, unique identifier for the closer)
      - `name` (text)
      - `photo` (text, URL)
      - `company` (text)
      - `title` (text)
      - `successful_call_transcripts` (jsonb, array of transcript data)
      - `difficult_call_transcripts` (jsonb, array of transcript data)
      - `biggest_frustrations` (jsonb, array of selected frustrations)
      - `core_transformation` (text)
      - `painful_problems` (jsonb, array of problems)
      - `common_symptoms` (jsonb, array of symptoms)
      - `target_audience` (jsonb, demographics, traits, behavior, triggers)
      - `buyer_beliefs` (jsonb, array of beliefs)
      - `differentiation` (jsonb, array of differentiators)
      - `false_beliefs` (jsonb, array of limiting stories)
      - `price_tiers` (jsonb, array of tier objects)
      - `discounts_bonuses` (jsonb, discount and bonus details)
      - `payment_options` (jsonb, array of payment options)
      - `delivery_timeline` (jsonb, timeline details)
      - `pleaser_signals` (jsonb, array of signals)
      - `red_flags` (jsonb, array of red flags)
      - `decision_making_styles` (jsonb, array of styles)
      - `created_at` (timestamptz)
      - `updated_at` (timestamptz)
  
  2. Security
    - Enable RLS on `closer_profiles` table
    - Add policies for authenticated users to manage their profiles
*/

CREATE TABLE IF NOT EXISTS closer_profiles (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  closer_id text UNIQUE NOT NULL,
  name text NOT NULL DEFAULT '',
  photo text DEFAULT '',
  company text DEFAULT '',
  title text DEFAULT '',
  successful_call_transcripts jsonb DEFAULT '[]'::jsonb,
  difficult_call_transcripts jsonb DEFAULT '[]'::jsonb,
  biggest_frustrations jsonb DEFAULT '[]'::jsonb,
  core_transformation text DEFAULT '',
  painful_problems jsonb DEFAULT '[]'::jsonb,
  common_symptoms jsonb DEFAULT '[]'::jsonb,
  target_audience jsonb DEFAULT '{}'::jsonb,
  buyer_beliefs jsonb DEFAULT '[]'::jsonb,
  differentiation jsonb DEFAULT '[]'::jsonb,
  false_beliefs jsonb DEFAULT '[]'::jsonb,
  price_tiers jsonb DEFAULT '[]'::jsonb,
  discounts_bonuses jsonb DEFAULT '{}'::jsonb,
  payment_options jsonb DEFAULT '[]'::jsonb,
  delivery_timeline jsonb DEFAULT '{}'::jsonb,
  pleaser_signals jsonb DEFAULT '[]'::jsonb,
  red_flags jsonb DEFAULT '[]'::jsonb,
  decision_making_styles jsonb DEFAULT '[]'::jsonb,
  created_at timestamptz DEFAULT now(),
  updated_at timestamptz DEFAULT now()
);

ALTER TABLE closer_profiles ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can view all closer profiles"
  ON closer_profiles
  FOR SELECT
  TO authenticated
  USING (true);

CREATE POLICY "Users can insert their own closer profile"
  ON closer_profiles
  FOR INSERT
  TO authenticated
  WITH CHECK (true);

CREATE POLICY "Users can update their own closer profile"
  ON closer_profiles
  FOR UPDATE
  TO authenticated
  USING (true)
  WITH CHECK (true);

CREATE POLICY "Users can delete their own closer profile"
  ON closer_profiles
  FOR DELETE
  TO authenticated
  USING (true);
```

---

### Migration 2: Create Call Debriefs Table
**File:** `20251224065550_create_call_debriefs_table.sql`

```sql
/*
  # Create Call Debriefs Table

  1. New Tables
    - `call_debriefs`
      - `id` (uuid, primary key)
      - `closer_id` (text, identifier for the closer)
      - `prospect_type` (text, the type of prospect)
      - `call_date` (timestamptz, when the call happened)
      
      ## 4-Pillar Live Data (from co-pilot)
      - `lubometer` (integer, 0-100 scale)
      - `urgency` (integer, 0-10 scale)
      - `trust` (integer, 0-10 scale)
      - `authority` (integer, 0-10 scale)
      - `structure` (integer, 0-10 scale)
      
      ## 8-Pillar Debrief Data (added in post-call analysis)
      - `perceived_spread` (integer, 0-10 scale) - P1: Meaningful tension to resolve
      - `urgency_pillar` (integer, 0-10 scale) - P2: Real cost to waiting (can duplicate urgency dial)
      - `decisiveness` (integer, 0-10 scale) - P3: Can choose and follow through
      - `available_money` (integer, 0-10 scale) - P4: Resources structurally accessible
      - `responsibility` (integer, 0-10 scale) - P5: Will own outcomes over time
      - `price_sensitivity` (integer, 0-10 scale) - P6: Money creates emotional friction
      - `trust_pillar` (integer, 0-10 scale) - P7: Trust stable enough for exposure (can duplicate trust dial)
      - `risk_tolerance` (integer, 0-10 scale) - P8: Can live with uncertainty
      
      ## Analysis
      - `truth_index_score` (integer, 0-100 calculated coherence score)
      - `collapse_conditions` (jsonb, array of detected incoherences)
      - `hard_no_go_triggered` (boolean, whether any no-go conditions were met)
      - `notes` (text, closer's notes on the call)
      - `outcome` (text, 'closed', 'no-sale-good', 'brutal')
      - `recording_url` (text, optional link to recording)
      
      - `created_at` (timestamptz)
      - `updated_at` (timestamptz)
  
  2. Security
    - Enable RLS on `call_debriefs` table
    - Add policies for authenticated users to manage their call debriefs
*/

CREATE TABLE IF NOT EXISTS call_debriefs (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  closer_id text NOT NULL DEFAULT '',
  prospect_type text NOT NULL DEFAULT '',
  call_date timestamptz DEFAULT now(),
  
  -- 4-Pillar Live Data
  lubometer integer DEFAULT 50,
  urgency integer DEFAULT 5,
  trust integer DEFAULT 5,
  authority integer DEFAULT 5,
  structure integer DEFAULT 5,
  
  -- 8-Pillar Debrief Data
  perceived_spread integer DEFAULT 5,
  urgency_pillar integer DEFAULT 5,
  decisiveness integer DEFAULT 5,
  available_money integer DEFAULT 5,
  responsibility integer DEFAULT 5,
  price_sensitivity integer DEFAULT 5,
  trust_pillar integer DEFAULT 5,
  risk_tolerance integer DEFAULT 5,
  
  -- Analysis
  truth_index_score integer DEFAULT 50,
  collapse_conditions jsonb DEFAULT '[]'::jsonb,
  hard_no_go_triggered boolean DEFAULT false,
  notes text DEFAULT '',
  outcome text DEFAULT 'pending',
  recording_url text DEFAULT '',
  
  created_at timestamptz DEFAULT now(),
  updated_at timestamptz DEFAULT now()
);

ALTER TABLE call_debriefs ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can view all call debriefs"
  ON call_debriefs
  FOR SELECT
  TO authenticated
  USING (true);

CREATE POLICY "Users can insert call debriefs"
  ON call_debriefs
  FOR INSERT
  TO authenticated
  WITH CHECK (true);

CREATE POLICY "Users can update call debriefs"
  ON call_debriefs
  FOR UPDATE
  TO authenticated
  USING (true)
  WITH CHECK (true);

CREATE POLICY "Users can delete call debriefs"
  ON call_debriefs
  FOR DELETE
  TO authenticated
  USING (true);
```

---

### Migration 3: Add Narrative Debrief Columns
**File:** `20251224070820_add_narrative_debrief_columns.sql`

```sql
/*
  # Add Narrative Debrief Columns
  
  1. Changes
    - Add narrative-based columns for call analysis:
      - `what_went_well` (text) - What the closer did well on the call
      - `what_didnt_go_well` (text) - What could have been handled better
      - `why_outcome` (text) - Why the close happened or didn't happen
      - `system_notes` (text) - Notes for system learning
      - `accuracy_check` (text) - Questions to check system accuracy
    
  2. Notes
    - These columns replace the visible "8 pillar" structure
    - The internal pillar data is still stored for analytics
    - This provides a narrative-driven debrief experience
*/

DO $$
BEGIN
  IF NOT EXISTS (
    SELECT 1 FROM information_schema.columns
    WHERE table_name = 'call_debriefs' AND column_name = 'what_went_well'
  ) THEN
    ALTER TABLE call_debriefs ADD COLUMN what_went_well text DEFAULT '';
  END IF;

  IF NOT EXISTS (
    SELECT 1 FROM information_schema.columns
    WHERE table_name = 'call_debriefs' AND column_name = 'what_didnt_go_well'
  ) THEN
    ALTER TABLE call_debriefs ADD COLUMN what_didnt_go_well text DEFAULT '';
  END IF;

  IF NOT EXISTS (
    SELECT 1 FROM information_schema.columns
    WHERE table_name = 'call_debriefs' AND column_name = 'why_outcome'
  ) THEN
    ALTER TABLE call_debriefs ADD COLUMN why_outcome text DEFAULT '';
  END IF;

  IF NOT EXISTS (
    SELECT 1 FROM information_schema.columns
    WHERE table_name = 'call_debriefs' AND column_name = 'system_notes'
  ) THEN
    ALTER TABLE call_debriefs ADD COLUMN system_notes text DEFAULT '';
  END IF;

  IF NOT EXISTS (
    SELECT 1 FROM information_schema.columns
    WHERE table_name = 'call_debriefs' AND column_name = 'accuracy_check'
  ) THEN
    ALTER TABLE call_debriefs ADD COLUMN accuracy_check text DEFAULT '';
  END IF;
END $$;
```

---

### Migration 4: Create User Settings Table
**File:** `20260103000000_create_user_settings.sql`

```sql
/*
  # Create User Settings Table

  Stores Admin Panel settings per authenticated user.

  1. New Tables
    - `user_settings`
      - `user_id` (uuid, primary key, references auth.users)
      - `settings` (jsonb)
      - `created_at` (timestamptz)
      - `updated_at` (timestamptz)

  2. Security
    - Enable RLS
    - Only the owner (auth.uid()) can read/write their row
*/

create table if not exists public.user_settings (
  user_id uuid primary key references auth.users(id) on delete cascade,
  settings jsonb not null default '{}'::jsonb,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

alter table public.user_settings enable row level security;

drop policy if exists "read own settings" on public.user_settings;
create policy "read own settings"
on public.user_settings
for select
to authenticated
using (auth.uid() = user_id);

drop policy if exists "insert own settings" on public.user_settings;
create policy "insert own settings"
on public.user_settings
for insert
to authenticated
with check (auth.uid() = user_id);

drop policy if exists "update own settings" on public.user_settings;
create policy "update own settings"
on public.user_settings
for update
to authenticated
using (auth.uid() = user_id)
with check (auth.uid() = user_id);
```

---

### Migration 5: Create Call Sessions and Transcripts
**File:** `20260103001000_create_call_sessions_and_transcripts.sql`

```sql
/*
  # Call Sessions + Live Transcript Chunks
  Stores live call sessions and incremental transcript chunks as the call progresses.

  Tables:
    - public.call_sessions
    - public.call_transcript_chunks

  Notes:
    - user_id is the owner (auth.users.id)
    - RLS enforces per-user isolation
*/

create table if not exists public.call_sessions (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  prospect_type text not null default '',
  connection_id text,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  ended_at timestamptz
);

create index if not exists call_sessions_user_id_idx on public.call_sessions(user_id);
create index if not exists call_sessions_updated_at_idx on public.call_sessions(updated_at desc);

create table if not exists public.call_transcript_chunks (
  id bigserial primary key,
  session_id uuid not null references public.call_sessions(id) on delete cascade,
  user_id uuid not null references auth.users(id) on delete cascade,
  created_at timestamptz not null default now(),
  chunk_text text not null,
  chunk_char_count int not null default 0,
  client_ts_ms bigint
);

create index if not exists call_transcript_chunks_session_id_idx on public.call_transcript_chunks(session_id);
create index if not exists call_transcript_chunks_user_id_idx on public.call_transcript_chunks(user_id);
create index if not exists call_transcript_chunks_created_at_idx on public.call_transcript_chunks(created_at desc);

alter table public.call_sessions enable row level security;
alter table public.call_transcript_chunks enable row level security;

-- call_sessions policies
drop policy if exists "call_sessions_select_own" on public.call_sessions;
create policy "call_sessions_select_own"
on public.call_sessions
for select
to authenticated
using (auth.uid() = user_id);

drop policy if exists "call_sessions_insert_own" on public.call_sessions;
create policy "call_sessions_insert_own"
on public.call_sessions
for insert
to authenticated
with check (auth.uid() = user_id);

drop policy if exists "call_sessions_update_own" on public.call_sessions;
create policy "call_sessions_update_own"
on public.call_sessions
for update
to authenticated
using (auth.uid() = user_id)
with check (auth.uid() = user_id);

-- call_transcript_chunks policies
drop policy if exists "call_transcript_chunks_select_own" on public.call_transcript_chunks;
create policy "call_transcript_chunks_select_own"
on public.call_transcript_chunks
for select
to authenticated
using (auth.uid() = user_id);

drop policy if exists "call_transcript_chunks_insert_own" on public.call_transcript_chunks;
create policy "call_transcript_chunks_insert_own"
on public.call_transcript_chunks
for insert
to authenticated
with check (auth.uid() = user_id);
```

---

### Migration 6: Add Email and Speaker Fields
**File:** `20260103002000_add_email_and_speaker_fields.sql`

```sql
/*
  # Add Email + Speaker fields for auditability
  - Allows you to see which email updated settings, and attribute sessions/chunks to email
  - Adds speaker_role to transcript chunks
  - Adds transcript_text snapshot to sessions for easy paragraph reading
*/

-- user_settings: store email for audit/debug (denormalized)
alter table public.user_settings
  add column if not exists user_email text,
  add column if not exists updated_by_email text;

-- call_sessions: store email + rolling transcript snapshot
alter table public.call_sessions
  add column if not exists user_email text,
  add column if not exists transcript_text text not null default '',
  add column if not exists transcript_char_count int not null default 0;

-- call_transcript_chunks: store email + speaker role
alter table public.call_transcript_chunks
  add column if not exists user_email text,
  add column if not exists speaker_role text not null default 'unknown';

-- Optional: basic validation of speaker_role values
do $$
begin
  if not exists (
    select 1 from pg_constraint where conname = 'call_transcript_chunks_speaker_role_check'
  ) then
    alter table public.call_transcript_chunks
      add constraint call_transcript_chunks_speaker_role_check
      check (speaker_role in ('closer','prospect','unknown'));
  end if;
end $$;
```

---

### Migration 7: AI Speaker Detection Optimization
**File:** `20260104000000_ai_speaker_detection_optimization.sql`

```sql
/*
  # AI Speaker Detection - Simplified Schema
  
  Changes:
  1. Removes speaker_confidence column (not needed, slows down AI)
  2. Adds indexes for speaker-based queries
  3. Updates constraint for AI detection values
  
  The transcript_text on call_sessions will be formatted as:
  
  CLOSER: how many months behind are you on payments
  
  PROSPECT: i think about maybe 3 months
  
  CLOSER: what happened that caused you to fall behind
*/

-- Drop speaker_confidence column if it exists (we don't need it)
alter table public.call_transcript_chunks
  drop column if exists speaker_confidence;

-- Add comment for documentation
comment on column public.call_transcript_chunks.speaker_role is 'AI-detected speaker: closer, prospect, or unknown';

-- Drop old constraint if exists (might have old values)
alter table public.call_transcript_chunks
  drop constraint if exists call_transcript_chunks_speaker_role_check;

-- Add updated constraint with valid values for AI detection
alter table public.call_transcript_chunks
  add constraint call_transcript_chunks_speaker_role_check
  check (speaker_role in ('closer', 'prospect', 'unknown'));

-- Create index for efficient queries by speaker role (useful for analytics)
create index if not exists call_transcript_chunks_speaker_role_idx 
  on public.call_transcript_chunks(speaker_role);

-- Create composite index for session + speaker queries
create index if not exists call_transcript_chunks_session_speaker_idx 
  on public.call_transcript_chunks(session_id, speaker_role);
```

---

### Migration 8: Create Call Summaries Table
**File:** `20260105000000_create_call_summaries.sql`

```sql
/*
  # Call Summaries Table
  Stores AI-generated conversation summaries (progressive during call, final when call ends)
  
  Notes:
  - One summary per session (upserted)
  - summary_json contains the full AI analysis
  - is_final indicates if this is the final summary (true) or progressive (false)
  - Removed foreign key constraints to avoid dependency issues
*/

create table if not exists public.call_summaries (
  id uuid primary key default gen_random_uuid(),
  session_id uuid not null unique,
  user_id uuid not null,
  user_email text not null,
  prospect_type text not null default '',
  summary_json jsonb not null,
  is_final boolean not null default false,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create index if not exists call_summaries_session_id_idx on public.call_summaries(session_id);
create index if not exists call_summaries_user_id_idx on public.call_summaries(user_id);
create index if not exists call_summaries_user_email_idx on public.call_summaries(user_email);
create index if not exists call_summaries_is_final_idx on public.call_summaries(is_final);
create index if not exists call_summaries_updated_at_idx on public.call_summaries(updated_at desc);

alter table public.call_summaries enable row level security;

-- RLS Policies
drop policy if exists "call_summaries_select_own" on public.call_summaries;
create policy "call_summaries_select_own"
on public.call_summaries
for select
to authenticated
using (auth.uid() = user_id);

drop policy if exists "call_summaries_insert_own" on public.call_summaries;
create policy "call_summaries_insert_own"
on public.call_summaries
for insert
to authenticated
with check (auth.uid() = user_id);

drop policy if exists "call_summaries_update_own" on public.call_summaries;
create policy "call_summaries_update_own"
on public.call_summaries
for update
to authenticated
using (auth.uid() = user_id)
with check (auth.uid() = user_id);
```

---

## Part 3: Frontend Environment Variables (Local Development)

Create a `.env` file in the `frontend/` directory:

```bash
# WebSocket URL (points to your Railway backend)
VITE_WS_URL=wss://your-railway-app.railway.app/ws

# Supabase Configuration (same as backend)
VITE_SUPABASE_URL=https://your-project.supabase.co
VITE_SUPABASE_ANON_KEY=your-supabase-anon-key-here
```

**Note:** For production deployment (Netlify/Vercel), add these as environment variables in your hosting platform's settings.

---

## ✅ Setup Checklist

- [ ] Created new Supabase project
- [ ] Ran all 8 SQL migrations in order
- [ ] Created new Railway project
- [ ] Added all Railway environment variables
- [ ] Connected Railway to GitHub repository
- [ ] Deployed backend to Railway
- [ ] Created `.env` file in frontend directory
- [ ] Added frontend environment variables
- [ ] Tested local development (`npm run dev`)
- [ ] Verified WebSocket connection
- [ ] Verified Supabase authentication

---

## 🎯 Quick Test

Once everything is set up:

1. **Start frontend:**
   ```bash
   cd frontend
   npm install
   npm run dev
   ```

2. **Open browser:** http://localhost:5173

3. **Check console:**
   - Should see: `WebSocket connected`
   - Should see: Backend build info

4. **Test recording:**
   - Click microphone button
   - Speak into microphone
   - Should see real-time transcript
   - Should see Lubometer updating

---

## 🆘 Troubleshooting

### Backend not connecting:
- Check Railway logs for errors
- Verify `OPENAI_API_KEY` is set correctly
- Verify `SUPABASE_URL` and `SUPABASE_ANON_KEY` are correct

### Database errors:
- Verify all 8 migrations ran successfully
- Check Supabase logs (Dashboard → Logs)
- Verify RLS policies are enabled

### Frontend connection issues:
- Check `VITE_WS_URL` matches your Railway URL
- Verify Railway backend is deployed and running
- Check browser console for WebSocket errors

---

## 📞 Support

If you encounter issues:
1. Check Railway logs: `railway logs`
2. Check Supabase logs: Dashboard → Logs
3. Check browser console: F12 → Console tab
4. Verify all environment variables are correct

---

**Last Updated:** January 14, 2026
