# Field Log — Setup Guide (one-time, ~15 minutes)

Your private tracker that syncs across all your devices. Data is protected by
your email login — only you can read or write it.

## Step 1 — Create the free database (Supabase)

1. Go to https://supabase.com → Sign up (free) → "New project"
2. Name it anything (e.g. fieldlog), pick a region (Mumbai is closest), set any database password
3. Wait ~1 minute for the project to spin up

## Step 2 — Create the table

In your Supabase project, open **SQL Editor**, paste this, and click **Run**:

```sql
create table fieldlog (
  user_id uuid primary key references auth.users(id) on delete cascade,
  data jsonb not null default '{}',
  updated_at timestamptz not null default now()
);

alter table fieldlog enable row level security;

create policy "own data only"
  on fieldlog for all
  using (auth.uid() = user_id)
  with check (auth.uid() = user_id);
```

This is what makes it private: the "row level security" policy means each
logged-in user can only ever see their own row. Even with the app's public
keys, nobody can read your data.

## Step 3 — Get your two keys

In Supabase: **Settings → API**. Copy:
- **Project URL** (looks like https://abcdefgh.supabase.co)
- **anon public** key (a long string)

## Step 4 — Paste them into the app

Open `index.html` in any text editor. Near the top of the `<script>` section:

```js
const SUPABASE_URL = "PASTE_YOUR_SUPABASE_URL_HERE";
const SUPABASE_ANON_KEY = "PASTE_YOUR_SUPABASE_ANON_KEY_HERE";
```

Replace the placeholders with your two values. Save.
(The anon key is designed to be public — it's safe in the file. Your data
is protected by the login + the security policy from Step 2, not by the key.)

## Step 5 — Deploy on GitHub Pages

1. Create a GitHub repo → upload ALL files in this folder
   (index.html, manifest.json, sw.js, icon-192.png, icon-512.png)
2. Repo → Settings → Pages → Source: "Deploy from a branch" → main → Save
3. Your app is live at: https://yourusername.github.io/yourreponame

## Step 6 — Sign in & install on each device

1. Open the URL → enter your email → tap the sign-in link from your inbox
   (you stay signed in after that — this is once per device)
2. Browser menu → **Add to Home screen**

Done. Add a task on your phone, open your laptop — it's there.

## Allowed sign-in link (important)

In Supabase: **Authentication → URL Configuration → Site URL** — set it to
your GitHub Pages URL (https://yourusername.github.io/yourreponame/).
This makes the email sign-in link land back on your app.

## Troubleshooting

- "sync failed" badge → check the two keys are pasted correctly
- Sign-in email link opens a broken page → fix the Site URL (above)
- App won't install to home screen → make sure you opened the
  github.io URL in Chrome/Safari, not inside another app

## Team Board upgrade (SQL #2)

Run this in Supabase SQL Editor to enable the collaborative Board tab
(copy from "create table" to the final line, no ``` marks):

create table teams (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  owner uuid not null references auth.users(id),
  created_at timestamptz default now()
);

create table team_members (
  team_id uuid references teams(id) on delete cascade,
  email text not null,
  user_id uuid,
  added_at timestamptz default now(),
  primary key (team_id, email)
);

create table board_tasks (
  id uuid primary key default gen_random_uuid(),
  team_id uuid references teams(id) on delete cascade,
  title text not null,
  status text not null default 'Backlog',
  assignee text,
  due date,
  created_by text,
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

create or replace function is_member(t uuid) returns boolean
language sql security definer set search_path = public as $$
  select exists(
    select 1 from team_members
    where team_id = t
      and (user_id = auth.uid() or lower(email) = lower(coalesce(auth.jwt()->>'email','')))
  );
$$;

alter table teams enable row level security;
alter table team_members enable row level security;
alter table board_tasks enable row level security;

create policy "members read teams" on teams for select using (owner = auth.uid() or is_member(id));
create policy "create team" on teams for insert with check (owner = auth.uid());
create policy "owner update team" on teams for update using (owner = auth.uid());
create policy "owner delete team" on teams for delete using (owner = auth.uid());

create policy "members read members" on team_members for select
  using (is_member(team_id) or exists(select 1 from teams where id = team_id and owner = auth.uid()));
create policy "members add members" on team_members for insert
  with check (is_member(team_id) or exists(select 1 from teams where id = team_id and owner = auth.uid()));
create policy "self claim" on team_members for update
  using (lower(email) = lower(coalesce(auth.jwt()->>'email','')));
create policy "members remove" on team_members for delete using (is_member(team_id));

create policy "members read tasks" on board_tasks for select using (is_member(team_id));
create policy "members write tasks" on board_tasks for insert with check (is_member(team_id));
create policy "members update tasks" on board_tasks for update using (is_member(team_id));
create policy "members delete tasks" on board_tasks for delete using (is_member(team_id));

alter publication supabase_realtime add table board_tasks;
