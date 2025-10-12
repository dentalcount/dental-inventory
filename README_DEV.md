# 🦷 Dental Inventory App

### Lightweight Cloud-Based Inventory for Dental Clinics  
**Built with Supabase + GitHub Pages**

---

## 🌐 Overview

This web app lets each clinic securely track:
- **Stock-in / Stock-out transactions**  
- **Minimum reorder levels**  
- **Supplier details**  
- **Staff initials and unit price on purchase**

Each user or clinic logs in with Google or email/password — all data is **isolated via Row-Level Security (RLS)** in Supabase.

**Live site:**  
👉 [https://dentalcount.github.io/dental-inventory/](https://dentalcount.github.io/dental-inventory/)

---

## 👩‍⚕️ Clinic User Guide

### 1. Sign-In
- Click **Sign in with Google** (recommended).  
- Each account’s data is isolated from others.  
- If “Sign out” stops responding, clear cache/cookies and reload.

### 2. Scanning / Logging
- Open the **Scan** tab.  
- Use:
  - a **Bluetooth barcode scanner**, or  
  - tap **Use Camera (beta)** to scan with phone.  
- Choose **Receive (IN)** or **Use (OUT)**.  
- When receiving, enter **Unit Price** — it saves as *last purchase price*.  
- Press **Add Transaction** (or auto-add when scanned).

### 3. Catalog
- Each barcode is automatically added if new.  
- Update fields like:
  - Item name  
  - Pack size  
  - Minimum stock  
  - Supplier  
- The table shows:
  - On-hand (live count)  
  - Last purchase price

### 4. Reorder Sheet
- Items where *On-hand < Min Stock* appear here automatically.  
- Click **Print Reorder Sheet** to export to PDF or email supplier.

### 5. Transactions
- Shows all recent IN / OUT logs with timestamps and staff initials.

---

# ⚙️ Developer Documentation (`README_DEV`)

This section explains database, schema, and Supabase setup.  
Keep for future debugging or cloning to another project.

---

## 🧱 1. Database Schema

Run this SQL in **Supabase SQL Editor**:

```sql
create extension if not exists "uuid-ossp";

create table if not exists public.items (
  id bigserial primary key,
  user_id uuid references auth.users(id) on delete cascade,
  barcode text not null,
  item_name text,
  pack_size text,
  min_stock integer default 0,
  supplier text,
  last_purchase_price numeric,
  inserted_at timestamptz default now(),
  constraint items_unique_per_user unique(user_id, barcode)
);

create table if not exists public.transactions (
  id bigserial primary key,
  user_id uuid references auth.users(id) on delete cascade,
  action text check (action in ('IN','OUT')),
  barcode text not null,
  quantity integer not null check (quantity > 0),
  unit_price numeric,
  staff text,
  ts timestamptz default now()
);

create index if not exists idx_items_user on items(user_id);
create index if not exists idx_tx_user on transactions(user_id);
# Dental Inventory – Developer Setup Guide

Version: **1.0.5**  
Frontend: **Vanilla JS (index.html)**  
Backend: **Supabase (PostgreSQL + Auth + RLS)**  
Host: **GitHub Pages**

---

## ⚙️ 1. Supabase Setup

### Create project
1. Go to https://app.supabase.com
2. Create new project (e.g. `dental-inventory`)
3. Copy the following to your frontend config:
   ```js
   const SUPABASE_URL  = 'https://sgjoadsefpcixvmthtfy.supabase.co';
   const SUPABASE_ANON = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InNnam9hZHNlZnBjaXh2bXRodGZ5Iiwicm9sZSI6ImFub24iLCJpYXQiOjE3NTkwMDIyMjYsImV4cCI6MjA3NDU3ODIyNn0.UcmAuHzEtKZIBs0pGkcLyRIYcLzCA4tNfuuDkUVLpU8';

alter table public.items enable row level security;
alter table public.transactions enable row level security;

-- ITEMS
create policy "read own items" on public.items
for select using (auth.uid() = user_id);

create policy "insert own items" on public.items
for insert with check (auth.uid() = user_id);

create policy "update own items" on public.items
for update using (auth.uid() = user_id);

create policy "delete own items" on public.items
for delete using (auth.uid() = user_id);

-- TRANSACTIONS
create policy "read own transactions" on public.transactions
for select using (auth.uid() = user_id);

create policy "insert own transactions" on public.transactions
for insert with check (auth.uid() = user_id);

create policy "update own transactions" on public.transactions
for update using (auth.uid() = user_id);

create policy "delete own transactions" on public.transactions
for delete using (auth.uid() = user_id);

const SUPABASE_URL  = 'https://sgjoadsefpcixvmthtfy.supabase.co';
const SUPABASE_ANON = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InNnam9hZHNlZnBjaXh2bXRodGZ5Iiwicm9sZSI6ImFub24iLCJpYXQiOjE3NTkwMDIyMjYsImV4cCI6MjA3NDU3ODIyNn0.UcmAuHzEtKZIBs0pGkcLyRIYcLzCA4tNfuuDkUVLpU8';

http://localhost:5173
http://127.0.0.1:5173

https://sgjoadsefpcixvmthtfy.supabase.co/auth/v1/callback

npm install -g supabase
supabase start

npx serve .
# or
python3 -m http.server 5173

localStorage.clear(); sessionStorage.clear(); location.reload();

(async()=>{
  console.log("Checking Supabase connectivity...");
  const sess=await sb.auth.getSession();
  console.log("Session:",sess);
  const sel=await sb.from('items').select('*').limit(1);
  console.log("Select:",sel);
})();

