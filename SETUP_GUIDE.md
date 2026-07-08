# UMBRA — Backend Setup (free, ~15 minutes)

The app works instantly in **solo mode** (plans, diet, tracker, XP — all saved in the browser).
This guide switches on **accounts + community chat** using Supabase's free tier (50,000 monthly users).

## 1. Create the Supabase project (5 min)
1. Go to **supabase.com** → Start your project → sign up free (GitHub login is easiest)
2. New project → name it `risemaxx`, set a strong database password, pick a region near your users
3. Wait ~2 min for it to spin up

## 2. Create the database tables (2 min)
In your project: **SQL Editor → New query** → paste ALL of this → Run:

```sql
-- User profiles (username, XP, level)
create table profiles (
  id uuid primary key references auth.users on delete cascade,
  username text unique,
  xp int default 0,
  level int default 1,
  created_at timestamptz default now()
);
alter table profiles enable row level security;
create policy "profiles readable" on profiles for select using (true);
create policy "insert own profile" on profiles for insert with check (auth.uid() = id);
create policy "update own profile" on profiles for update using (auth.uid() = id);

-- Chat messages
create table messages (
  id bigint generated always as identity primary key,
  created_at timestamptz default now(),
  channel text not null,
  user_id uuid not null references auth.users on delete cascade,
  username text,
  level int,
  content text not null check (char_length(content) <= 500)
);
alter table messages enable row level security;
create policy "messages readable" on messages for select using (true);
create policy "send as yourself" on messages for insert with check (auth.uid() = user_id);

-- Reports (moderation queue)
create table reports (
  id bigint generated always as identity primary key,
  created_at timestamptz default now(),
  message_id bigint,
  reporter uuid,
  reason text
);
alter table reports enable row level security;
create policy "anyone signed-in can report" on reports for insert with check (auth.uid() = reporter);

-- Live chat updates
alter publication supabase_realtime add table messages;
```

## 3. Connect the app (1 min)
1. Supabase → **Settings → API** → copy the **Project URL** and the **anon public** key
2. Open `index.html`, find the CONFIG block at the top of the `<script>`:
   ```js
   const SUPABASE_URL = "";
   const SUPABASE_ANON_KEY = "";
   ```
3. Paste your values between the quotes. Save. Redeploy (drag folder to Netlify again).

Email/password sign-up now works (Supabase sends confirmation emails automatically).

## 4. Google sign-in (5 min, free)
1. Supabase → **Authentication → Providers → Google** → toggle on. Copy the "callback URL" it shows.
2. Go to **console.cloud.google.com** → new project → APIs & Services → OAuth consent screen (External, fill in app name + your email) → Credentials → Create credentials → OAuth Client ID → Web application
3. Add the Supabase callback URL under "Authorized redirect URIs"
4. Copy the Client ID + Secret back into the Supabase Google provider form → Save

## 5. Discord sign-in (3 min, free)
1. Supabase → Authentication → Providers → **Discord** → toggle on, copy callback URL
2. **discord.com/developers/applications** → New Application → OAuth2 → add the callback URL as a Redirect
3. Copy Client ID + Secret into Supabase → Save

## Apple sign-in (later)
Requires the Apple Developer Program ($99/year). The app already shows "coming later" — flip it on the same way via Supabase's Apple provider when you're ready to pay for it.

## 6. Netlify redirect URL (1 min — important)
Supabase → Authentication → **URL Configuration** → set Site URL to your live domain
(e.g. `https://risemaxx.com` or your `*.netlify.app` URL). OAuth logins bounce back to this address.

## Moderation — take this seriously
Your users will include insecure teenagers. That's who you built this for, so protect them:
- The app auto-blocks blackpill phrases and has a report button; reports land in the `reports` table (Supabase → Table Editor)
- Check reports weekly minimum; delete rows from `messages` to remove bad content
- Write 2–3 trusted friends in as moderators before you promote the site on TikTok, not after it blows up
- If someone posts about self-harm, don't just delete it — reply with crisis resources (e.g. 988 in the US)

## Cost summary
Domain ~$10/yr · Netlify free · Supabase free · Google/Discord OAuth free.
**Total: ~$10/yr** until you're big enough that upgrading is a good problem to have.
