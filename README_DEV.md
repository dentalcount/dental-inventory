# 🦷 Dental Inventory App — Developer Documentation

**Project:** `sgjoadsefpcixvmthtfy`  
**Live Demo:** [https://dentalcount.github.io/dental-inventory/](https://dentalcount.github.io/dental-inventory/)  
**Backend:** Supabase (Postgres + Storage + Auth)  
**Frontend:** Static HTML (Supabase JS v2, hosted via GitHub Pages)

---

## 🔑 Supabase Configuration

**Project URL:**  
https://sgjoadsefpcixvmthtfy.supabase.co

markdown
Copy code

**Anon Key:**  
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InNnam9hZHNlZnBjaXh2bXRodGZ5Iiwicm9sZSI6ImFub24iLCJpYXQiOjE3NTkwMDIyMjYsImV4cCI6MjA3NDU3ODIyNn0.UcmAuHzEtKZIBs0pGkcLyRIYcLzCA4tNfuuDkUVLpU8

java
Copy code

**Redirect URL (Auth success):**
https://dentalcount.github.io/dental-inventory/

pgsql
Copy code

---

## 📁 Database Schema

### Table: `profiles`
Stores all signed-in users.  
```sql
create table if not exists public.profiles (
  id uuid primary key references auth.users on delete cascade,
  email text unique,
  role text default 'user' check (role in ('user','admin')),
  org_id uuid,
  created_at timestamptz default now()
);
Column	Type	Notes
id	UUID	Matches auth.users.id
email	text	Supabase login email
role	text	'user' or 'admin'
org_id	UUID	Optional grouping
created_at	timestamptz	Auto

Table: catalogue
Stores all inventory items per user.

sql
Copy code
create table if not exists public.catalogue (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references public.profiles(id) on delete cascade,
  barcode text,
  name text,
  unit text,
  pack_size text,
  expiry_date date,
  photo_url text,
  updated_at timestamptz default now()
);
Column	Type	Notes
barcode	text	Unique per user or org
expiry_date	date	Used for re-ordering filter
photo_url	text	Public URL in product_photos

View: v_catalogue_with_owner
Drops and recreates the existing view.

sql
Copy code
drop view if exists public.v_catalogue_with_owner cascade;

create view public.v_catalogue_with_owner as
select
  c.id,
  c.barcode,
  c.name,
  c.unit,
  c.pack_size,
  c.photo_url,
  c.expiry_date,
  c.user_id,
  c.updated_at,
  p.email as owner_email,
  (c.expiry_date::date - current_date) as days_to_expiry
from public.catalogue c
left join public.profiles p on p.id = c.user_id;

grant select on public.v_catalogue_with_owner to anon, authenticated;
🛡️ Row-Level Security
Profiles
sql
Copy code
alter table public.profiles enable row level security;

create policy "Users can see own profile" 
  on public.profiles
  for select using (auth.uid() = id);

create policy "Allow insert self profile"
  on public.profiles
  for insert with check (auth.uid() = id);

create policy "Allow admin full access"
  on public.profiles
  using (exists (select 1 from public.profiles p2 where p2.id = auth.uid() and p2.role = 'admin'));
Catalogue
sql
Copy code
alter table public.catalogue enable row level security;

create policy "Owner read/write own items"
  on public.catalogue
  using (auth.uid() = user_id)
  with check (auth.uid() = user_id);

create policy "Admin can read/write all"
  on public.catalogue
  using (exists (select 1 from public.profiles p2 where p2.id = auth.uid() and p2.role = 'admin'))
  with check (exists (select 1 from public.profiles p2 where p2.id = auth.uid() and p2.role = 'admin'));
🖼️ Storage
Bucket: product_photos
bash
Copy code
# In Supabase Dashboard → Storage → Create bucket
product_photos (public)
Policies:

Public read

Authenticated users can upload into a subfolder named by their UID

👑 Admin Setup
Admins are defined by email.

To grant admin access manually:

sql
Copy code
update public.profiles
set role='admin'
where email='jonathan.l.jiang@gmail.com';
🧩 App Behaviour Overview
Feature	Description
Login	Magic Link or Google OAuth (PKCE)
Main Tabs	“Catalogue” and “Re-Ordering”
Mobile Responsive	Stacks inputs vertically on small screens
Admin Controls	CSV Import visible only to admins
Photo Upload	Direct to product_photos via Supabase Storage
Expiry Date Tracking	Each item has an expiry date (YYYY-MM-DD)
Re-Ordering Tab	Filter items by expiry date range or show expired
Barcode Scanner	Uses BarcodeDetector API (fallback manual entry)
Debug Mode	?debug=1 shows internal logs and RLS probes

🧰 Development Commands
Run locally
Since it’s static HTML:

bash
Copy code
python3 -m http.server 8080
Visit → http://localhost:8080

Build/versioning
Edit inside <script>:

js
Copy code
const BUILD = '2025-10-28T11:10';
and append ?v=${BUILD} to cache-bust external scripts.

🧪 Debugging Tips
Open ?debug=1 to enable the debug console and step-by-step event logs.

Use Supabase “Auth → Users” tab to confirm your Google login email exists.

If buttons don’t work, do a hard refresh or clear localStorage/sessionStorage.

For persistent “RLS blocked” messages, verify that auth.uid() has matching entries in profiles.

🔄 Future Enhancements (ideas)
Multi-org support for shared catalogue between clinics.

Real barcode lookup via OpenFoodFacts or UPC databases.

Offline PWA mode.

Price tracking and supplier benchmark (to integrate future syndicate feature).

💾 Version Log
Date	Change
2025-10-28	Added expiry date & re-ordering tab
2025-10-27	Fixed Google login redirect, debug mode
2025-10-26	Added photo upload + barcode scanning
2025-10-25	Introduced Magic Link login & RLS fixes

Maintainer:
Dr Jonathan Jiang (jonathan.l.jiang@gmail.com)
