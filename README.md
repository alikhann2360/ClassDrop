# ClassDrop setup (about 10 minutes)

ClassDrop is one static file (`index.html`) plus Supabase. No build step.

## 1. Create Supabase project
supabase.com → New project. Then **Authentication → Users → Add user**: create your CR account (email + password). Turn off public sign-ups under **Authentication → Providers → Email → Allow new users to sign up**.

## 2. Run this SQL (SQL Editor). Replace `cr@example.com` with your CR email.

```sql
create table public.assignments (
  id uuid primary key default gen_random_uuid(),
  student_name text not null check (length(student_name) between 1 and 80),
  roll_number text not null check (length(roll_number) between 1 and 30),
  subject text not null check (length(subject) between 1 and 120),
  assignment_name text not null check (length(assignment_name) between 1 and 120),
  file_name text not null,
  storage_path text not null unique,
  file_size bigint not null check (file_size > 0 and file_size <= 20971520),
  created_at timestamptz not null default now()
);
-- One submission per roll number per assignment (drop this index to allow resubmission)
create unique index one_submission on public.assignments (lower(roll_number), lower(subject), lower(assignment_name));

alter table public.assignments enable row level security;

create policy "students can submit" on public.assignments for insert to anon with check (true);
create policy "cr can read"   on public.assignments for select to authenticated using (auth.jwt()->>'email' = 'cr@example.com');
create policy "cr can delete" on public.assignments for delete to authenticated using (auth.jwt()->>'email' = 'cr@example.com');

-- Private bucket, PDF only, 20 MB
insert into storage.buckets (id, name, public, file_size_limit, allowed_mime_types)
values ('assignments','assignments',false,20971520,array['application/pdf']);

create policy "students can upload" on storage.objects for insert to anon with check (bucket_id = 'assignments');
create policy "students can undo failed upload" on storage.objects for delete to anon using (false);
create policy "cr can read files"   on storage.objects for select to authenticated using (bucket_id='assignments' and auth.jwt()->>'email'='cr@example.com');
create policy "cr can delete files" on storage.objects for delete to authenticated using (bucket_id='assignments' and auth.jwt()->>'email'='cr@example.com');

-- Live dashboard updates
alter publication supabase_realtime add table public.assignments;
```

## 3. Add your keys
**Project Settings → API**: copy the Project URL and the **anon public** key into the CONFIG block at the top of the script in `index.html`. Never put the service-role key anywhere.

## 4. Run / deploy
- Locally: open `index.html`, or `npx serve classdrop`.
- Vercel: push the folder to GitHub → vercel.com → New Project → import it (Framework: Other). Deploy.
- Students use `https://your-site.vercel.app/`. You use `https://your-site.vercel.app/#/admin`.

## Notes
- Subject and assignment are typed in one field, split on " — " or " - ", and become the ZIP folders.
- Set `DEADLINE` in the config to block late uploads in the browser.
