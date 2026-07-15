# Aperture — Supabase setup guide

This app needs a **free Supabase project** to store accounts and photos for real — shared across everyone who opens the link, no server of your own to run. Takes about 10 minutes.

## 1. Create the project

Go to https://supabase.com → **Start your project** → New project (any name, any region) → wait ~2 minutes while it provisions.

## 2. Turn off email confirmation (so signup logs people in instantly)

**Authentication → Providers → Email** → turn **OFF** "Confirm email". (You can turn this back on later once you're using a real email provider — while it's on, `auth.signUp()` doesn't return an active session until the user clicks a confirmation link, which this app's flow doesn't currently wait for.)

## 3. Create the schema

Go to **SQL Editor → New query**, paste everything below, then click **Run**.

```sql
create extension if not exists "pgcrypto";

create table profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  username text unique not null,
  full_name text,
  bio text default '',
  is_admin boolean not null default false,
  saved_photo_ids uuid[] not null default '{}',
  created_at timestamptz not null default now()
);

create table photos (
  id uuid primary key default gen_random_uuid(),
  author_id uuid not null references profiles(id) on delete cascade,
  author_username text not null,
  author_full_name text,
  image_url text not null,
  image_storage_path text not null,
  caption text default '',
  location text default '',
  camera text default '',
  lens text default '',
  likes_count int not null default 0,
  dislikes_count int not null default 0,
  has_camera_exif boolean not null default false,
  attested_original boolean not null default false,
  challenge_tag text,
  license_price int default 25,
  created_at timestamptz not null default now()
);

create table reactions (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references profiles(id) on delete cascade,
  photo_id uuid not null references photos(id) on delete cascade,
  type text not null check (type in ('like','dislike')),
  unique(user_id, photo_id)
);

create table reports (
  id uuid primary key default gen_random_uuid(),
  photo_id uuid not null references photos(id) on delete cascade,
  reporter_id uuid not null references profiles(id) on delete cascade,
  reporter_username text not null,
  reason text default '',
  created_at timestamptz not null default now(),
  unique(photo_id, reporter_id)
);

create table follows (
  follower_id uuid not null references profiles(id) on delete cascade,
  following_id uuid not null references profiles(id) on delete cascade,
  created_at timestamptz not null default now(),
  primary key (follower_id, following_id)
);

create table comments (
  id uuid primary key default gen_random_uuid(),
  photo_id uuid not null references photos(id) on delete cascade,
  author_id uuid not null references profiles(id) on delete cascade,
  author_username text not null,
  author_full_name text,
  body text not null,
  created_at timestamptz not null default now()
);

-- Row Level Security

alter table profiles enable row level security;
create policy "Public profiles are viewable by everyone" on profiles for select using (true);
create policy "Users can insert their own profile" on profiles for insert with check (auth.uid() = id);
create policy "Users can update their own profile" on profiles for update using (auth.uid() = id);

alter table photos enable row level security;
create policy "Photos are viewable by everyone" on photos for select using (true);
create policy "Users can insert their own photos" on photos for insert with check (auth.uid() = author_id);
create policy "Owners can update their own photos" on photos for update using (
  auth.uid() = author_id
  or exists (select 1 from profiles where id = auth.uid() and is_admin = true)
);
create policy "Owners and admins can delete photos" on photos for delete using (
  auth.uid() = author_id or exists (select 1 from profiles where id = auth.uid() and is_admin = true)
);

alter table reactions enable row level security;
create policy "Users can view their own reactions" on reactions for select using (auth.uid() = user_id);

alter table reports enable row level security;
create policy "Admins can view reports" on reports for select using (
  exists (select 1 from profiles where id = auth.uid() and is_admin = true)
);
create policy "Signed-in users can create reports" on reports for insert with check (auth.uid() = reporter_id);
create policy "Admins can delete reports" on reports for delete using (
  exists (select 1 from profiles where id = auth.uid() and is_admin = true)
);

alter table follows enable row level security;
create policy "Follows are viewable by everyone" on follows for select using (true);
create policy "Users can follow on their own behalf" on follows for insert with check (auth.uid() = follower_id);
create policy "Users can unfollow on their own behalf" on follows for delete using (auth.uid() = follower_id);

alter table comments enable row level security;
create policy "Comments are viewable by everyone" on comments for select using (true);
create policy "Signed-in users can comment" on comments for insert with check (auth.uid() = author_id);
create policy "Authors and admins can delete comments" on comments for delete using (
  auth.uid() = author_id or exists (select 1 from profiles where id = auth.uid() and is_admin = true)
);

-- Profile photos

alter table profiles add column if not exists avatar_url text;

-- Direct messages

create table messages (
  id uuid primary key default gen_random_uuid(),
  sender_id uuid not null references profiles(id) on delete cascade,
  recipient_id uuid not null references profiles(id) on delete cascade,
  body text not null,
  created_at timestamptz not null default now()
);

alter table messages enable row level security;
create policy "Users can view their own conversations" on messages for select using (
  auth.uid() = sender_id or auth.uid() = recipient_id
);
create policy "Users can send messages as themselves" on messages for insert with check (auth.uid() = sender_id);
create policy "Senders can delete their own messages" on messages for delete using (auth.uid() = sender_id);

-- Required for real-time delivery and real-time delete propagation in chat
alter table messages replica identity full;
alter publication supabase_realtime add table messages;

-- Atomic like/dislike toggle (runs server-side so counts can never drift out of sync)

create or replace function toggle_reaction(p_photo_id uuid, p_type text)
returns void
language plpgsql
security definer
as $$
declare
  existing_type text;
begin
  select type into existing_type from reactions where user_id = auth.uid() and photo_id = p_photo_id;

  if existing_type = p_type then
    delete from reactions where user_id = auth.uid() and photo_id = p_photo_id;
    if p_type = 'like' then
      update photos set likes_count = greatest(0, likes_count - 1) where id = p_photo_id;
    else
      update photos set dislikes_count = greatest(0, dislikes_count - 1) where id = p_photo_id;
    end if;
  elsif existing_type is not null then
    update reactions set type = p_type where user_id = auth.uid() and photo_id = p_photo_id;
    if p_type = 'like' then
      update photos set likes_count = likes_count + 1, dislikes_count = greatest(0, dislikes_count - 1) where id = p_photo_id;
    else
      update photos set dislikes_count = dislikes_count + 1, likes_count = greatest(0, likes_count - 1) where id = p_photo_id;
    end if;
  else
    insert into reactions (user_id, photo_id, type) values (auth.uid(), p_photo_id, p_type);
    if p_type = 'like' then
      update photos set likes_count = likes_count + 1 where id = p_photo_id;
    else
      update photos set dislikes_count = dislikes_count + 1 where id = p_photo_id;
    end if;
  end if;
end;
$$;
```

## 4. Create the storage buckets

**Storage → Create a new bucket** → name it exactly `photos` → toggle **Public bucket** ON → Create.

Repeat: **Storage → Create a new bucket** → name it exactly `avatars` → toggle **Public bucket** ON → Create.

## 5. Set storage policies

Back in **SQL Editor → New query**, paste and run:

```sql
create policy "Users can upload their own photos"
on storage.objects for insert
with check (
  bucket_id = 'photos' and auth.role() = 'authenticated'
  and (storage.foldername(name))[1] = auth.uid()::text
);

create policy "Owners and admins can delete their photos"
on storage.objects for delete
using (
  bucket_id = 'photos' and (
    (storage.foldername(name))[1] = auth.uid()::text
    or exists (select 1 from profiles where id = auth.uid() and is_admin = true)
  )
);

create policy "Users can upload their own avatar"
on storage.objects for insert
with check (
  bucket_id = 'avatars' and auth.role() = 'authenticated'
  and (storage.foldername(name))[1] = auth.uid()::text
);

create policy "Users can replace their own avatar"
on storage.objects for update
using (bucket_id = 'avatars' and (storage.foldername(name))[1] = auth.uid()::text);

create policy "Users can delete their own avatar"
on storage.objects for delete
using (bucket_id = 'avatars' and (storage.foldername(name))[1] = auth.uid()::text);
```

## 6. Get your config

**Project Settings → API** → copy the **Project URL** and the **anon public** key. Paste both into `index.html`, near the top of the `<script type="module">` section, replacing the `PASTE_YOUR_...` placeholders.

## 7. Deploy to GitHub Pages

1. In your GitHub repo, delete everything currently there.
2. Upload the `index.html` file (with your config pasted in) to the **root** of the repo.
3. **Settings → Pages** → Source: "Deploy from a branch" → branch `main` → folder `/ (root)` → Save.
4. Wait 1–2 minutes, then visit your Pages URL.

## How admin works

The **first person to sign up** on your live site is automatically made an admin. Sign up yourself first before sharing the link if you want to be the admin.

## What's in this version

Auth, feed (with an Everyone/Following toggle), explore (with verified-only and gear filters), trending, a weekly challenge with leaderboard, profiles with follow/followers and a connections modal, profile photo upload, saved collections, comments, real-time direct messaging with message delete, photo reporting, an admin moderation panel, and a license-purchase mockup (UI only, no real payments wired up).

## Known limitations

- **Camera verification** is a real EXIF check plus a self-attestation requirement — meaningfully raises the bar, but isn't tamper-proof against a determined bad actor.
- **License purchases** are a UI mockup only — wiring up real payments would need Stripe (or similar) added on top.
- **Chat requires Realtime to be enabled** for the `messages` table (the `alter publication supabase_realtime add table messages;` line in the schema above handles this) — without it, sending still works, but you won't see the other person's messages appear live without reloading.
- **Free tier limits**: Supabase's free plan covers 500MB database + 1GB storage + 50K monthly active auth users — plenty for testing and a modest live audience. The paid Pro plan is a flat $25/month when you outgrow it.
