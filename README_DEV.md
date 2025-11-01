# 🦷 Dental Inventory App — Developer README

## Overview
The Dental Inventory App is a Supabase-powered web application for small dental clinics to manage stock of clinical consumables, with built-in barcode scanning, expiry tracking, and transaction logging.

This version adds:
- ✅ Google + Magic Link sign-in  
- ✅ Profile & role (admin/user)  
- ✅ Item catalogue (with photos and expiry)  
- ✅ Transactions (`IN`, `OUT`, `ADJUST`)  
- ✅ Real-time stock and expiry reporting via database views  
- ✅ Admin-only CSV import/export  
- ✅ Debug console for Supabase permission diagnostics  

---

## Supabase Configuration

### 1. Project Keys
In `index.html`, update:

```js
const SUPABASE_URL  = 'https://sgjoadsefpcixvmthtfy.supabase.co';
const SUPABASE_ANON = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InNnam9hZHNlZnBjaXh2bXRodGZ5Iiwicm9sZSI6ImFub24iLCJpYXQiOjE3NTkwMDIyMjYsImV4cCI6MjA3NDU3ODIyNn0.UcmAuHzEtKZIBs0pGkcLyRIYcLzCA4tNfuuDkUVLpU8';
const REDIRECT_URL  = 'https://dentalcount.github.io/dental-inventory/';


Important: use your project’s anon key only — never the service_role key.

2. Database Schema

Run the following in Supabase SQL Editor.

a) Profiles
create table if not exists public.profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  email text unique,
  role text default 'user',
  org_id uuid,
  created_at timestamptz default now()
);

alter table public.profiles enable row level security;

drop policy if exists "profiles read own" on public.profiles;
drop policy if exists "profiles write own" on public.profiles;
drop policy if exists "profiles admin" on public.profiles;

create policy "profiles read own"
  on public.profiles
  for select
  using (auth.uid() = id);

create policy "profiles write own"
  on public.profiles
  for update
  using (auth.uid() = id)
  with check (auth.uid() = id);

create policy "profiles admin"
  on public.profiles
  for all
  using (exists (select 1 from public.profiles p where p.id = auth.uid() and p.role = 'admin'))
  with check (exists (select 1 from public.profiles p where p.id = auth.uid() and p.role = 'admin'));


To assign admin role:

update public.profiles set role='admin' where email='jonathan.l.jiang@gmail.com';

b) Catalogue
create table if not exists public.catalogue (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references public.profiles(id) on delete cascade,
  barcode text not null,
  name text,
  unit text,
  pack_size text,
  expiry_date date,
  photo_url text,
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

create unique index if not exists catalogue_barcode_uidx on public.catalogue(barcode);

alter table public.catalogue enable row level security;

drop policy if exists "cat read" on public.catalogue;
drop policy if exists "cat write" on public.catalogue;
drop policy if exists "cat admin" on public.catalogue;

create policy "cat read"
  on public.catalogue
  for select
  using (auth.uid() = user_id);

create policy "cat write"
  on public.catalogue
  for insert
  with check (auth.uid() = user_id);

create policy "cat admin"
  on public.catalogue
  for all
  using (exists (select 1 from public.profiles p where p.id = auth.uid() and p.role='admin'))
  with check (exists (select 1 from public.profiles p where p.id = auth.uid() and p.role='admin'));

c) Inventory Transactions
-- Enum type
do $$
begin
  if not exists (select 1 from pg_type where typname = 'movement_kind') then
    create type public.movement_kind as enum ('IN','OUT','ADJUST');
  end if;
end $$;

-- Transaction table
create table if not exists public.inv_transactions (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references public.profiles(id) on delete cascade,
  item_id uuid not null references public.catalogue(id) on delete cascade,
  kind public.movement_kind not null,
  qty numeric(12,3) not null,
  note text,
  lot_code text,
  expiry_date date,
  unit_cost numeric(12,2),
  created_at timestamptz not null default now()
);

create index if not exists inv_transactions_item_idx   on public.inv_transactions(item_id);
create index if not exists inv_transactions_user_idx   on public.inv_transactions(user_id);
create index if not exists inv_transactions_expiry_idx on public.inv_transactions(expiry_date);

alter table public.inv_transactions enable row level security;

drop policy if exists "trx owner read" on public.inv_transactions;
drop policy if exists "trx owner write" on public.inv_transactions;
drop policy if exists "trx owner update" on public.inv_transactions;
drop policy if exists "trx admin all" on public.inv_transactions;

create policy "trx owner read"
  on public.inv_transactions
  for select
  using (auth.uid() = user_id);

create policy "trx owner write"
  on public.inv_transactions
  for insert
  with check (auth.uid() = user_id);

create policy "trx owner update"
  on public.inv_transactions
  for update
  using (auth.uid() = user_id)
  with check (auth.uid() = user_id);

create policy "trx admin all"
  on public.inv_transactions
  using (exists (select 1 from public.profiles p where p.id = auth.uid() and p.role = 'admin'))
  with check (exists (select 1 from public.profiles p where p.id = auth.uid() and p.role = 'admin'));

d) Derived Views
-- Signed qty view
drop view if exists public.v_inv_transactions_signed cascade;

create view public.v_inv_transactions_signed as
select
  t.*,
  case
    when t.kind = 'IN'     then +t.qty
    when t.kind = 'OUT'    then -t.qty
    when t.kind = 'ADJUST' then  t.qty
  end as qty_signed
from public.inv_transactions t;

grant select on public.v_inv_transactions_signed to anon, authenticated;

-- Stock per item
drop view if exists public.v_stock_per_item cascade;

create view public.v_stock_per_item as
select
  c.id as item_id,
  c.barcode,
  c.name,
  c.unit,
  c.pack_size,
  c.photo_url,
  c.user_id,
  p.email as owner_email,
  coalesce(sum(v.qty_signed), 0) as stock_qty
from public.catalogue c
left join public.v_inv_transactions_signed v on v.item_id = c.id
left join public.profiles p on p.id = c.user_id
group by c.id, c.barcode, c.name, c.unit, c.pack_size, c.photo_url, c.user_id, p.email;

grant select on public.v_stock_per_item to anon, authenticated;

-- Stock per lot (for expiry tracking)
drop view if exists public.v_stock_per_lot cascade;

create view public.v_stock_per_lot as
select
  c.id as item_id,
  c.barcode,
  c.name,
  c.unit,
  c.pack_size,
  c.user_id,
  p.email as owner_email,
  v.expiry_date,
  v.lot_code,
  coalesce(sum(v.qty_signed), 0) as stock_qty
from public.catalogue c
left join public.v_inv_transactions_signed v on v.item_id = c.id
left join public.profiles p on p.id = c.user_id
group by c.id, c.barcode, c.name, c.unit, c.pack_size, c.user_id, p.email, v.expiry_date, v.lot_code
having coalesce(sum(v.qty_signed), 0) <> 0
order by v.expiry_date nulls last;

grant select on public.v_stock_per_lot to anon, authenticated;

3. Storage Bucket

Create a bucket named product_photos in Storage with public access for item photos.

4. Local Testing

You can run the app directly from GitHub Pages or locally:

npx serve .


Open http://localhost:3000
 or your GitHub Pages URL.

Use ?debug=1 at the end of the URL to see detailed debug output.

5. Roles and Access
Role	Description	Access
User	Default after sign-in	Sees and edits only their own catalogue and transactions
Admin	Manually promoted via SQL	Can see all data, run CSV import/export

To promote a user:

update public.profiles set role='admin' where email='jonathan.l.jiang@gmail.com';

6. Core Tables Summary
Table	Purpose
profiles	user record + role
catalogue	master list of items (1 row per barcode)
inv_transactions	immutable log of stock movements
v_inv_transactions_signed	adds signed quantity field
v_stock_per_item	total stock per item
v_stock_per_lot	stock by expiry and lot
7. Next Development Goals

 Offline/PWA caching

 Bulk photo upload

 Usage analytics by month

 Low-stock alerts via email/webhook

 QR label generation for stock bins

Author: Dr Jonathan Jiang
Supabase Project: sgjoadsefpcixvmthtfy
Updated: 2025-11-02
Frontend: Single-page HTML + Supabase JS v2
