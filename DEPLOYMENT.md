# FileCraft — Vercel Deployment Guide

## What's inside `filecraft-vercel.zip`

```
filecraft-vercel/
├── vercel.json          ← Vercel routing config
├── .env.example         ← All required environment variables (template)
├── package.json         ← Root package (tells Vercel this is Node 20)
├── public/              ← Built frontend (React + Vite static files)
│   ├── index.html
│   └── assets/
└── api/
    └── index.mjs        ← Express backend (Vercel serverless function)
        pino-*.mjs       ← Pino logger workers
```

---

## Step 1 — Set up a PostgreSQL database

You need a hosted PostgreSQL database. Free options:
- **Neon** → https://neon.tech (recommended, generous free tier)
- **Supabase** → https://supabase.com
- **Railway** → https://railway.app

After creating a database, copy the **connection string** (looks like `postgresql://user:pass@host/db`).

Then push the database schema by running this SQL in your database console,
**or** use `drizzle-kit push` if you have the source code locally:

```bash
pnpm --filter @workspace/db run push
```

---

## Step 2 — Create a Clerk application

1. Go to https://dashboard.clerk.com and create a new application
2. Choose your sign-in options (Email, Google, GitHub, etc.)
3. Go to **API Keys** and copy:
   - **Publishable key** → `pk_live_...`
   - **Secret key** → `sk_live_...`

---

## Step 3 — Deploy to Vercel

### Option A: Vercel Dashboard (drag & drop)

1. Go to https://vercel.com/new
2. Click **"Import a folder"** or drag & drop the `filecraft-vercel` folder
3. Vercel will detect the `vercel.json` automatically

### Option B: Vercel CLI

```bash
# Install Vercel CLI (once)
npm install -g vercel

# Inside the filecraft-vercel/ folder:
vercel deploy --prod
```

---

## Step 4 — Set environment variables in Vercel

In your Vercel project → **Settings → Environment Variables**, add:

| Variable | Value |
|---|---|
| `DATABASE_URL` | Your PostgreSQL connection string |
| `CLERK_PUBLISHABLE_KEY` | `pk_live_...` from Clerk |
| `CLERK_SECRET_KEY` | `sk_live_...` from Clerk |
| `VITE_CLERK_PUBLISHABLE_KEY` | Same as `CLERK_PUBLISHABLE_KEY` |
| `ADMIN_USER_IDS` | Your Clerk user ID (find it after first sign-in at `/api/users/me`) |
| `SESSION_SECRET` | Any random 32+ character string |

---

## Step 5 — Seed the database (first deploy only)

Run the seed SQL in your database console (Neon / Supabase SQL editor):

```sql
INSERT INTO tools (slug, name, description, category, category_slug, icon, color, accepted_file_types, max_file_size_guest, max_file_size_user, max_files_guest, max_files_user, is_enabled, usage_count)
VALUES
  ('pdf-merge','Merge PDF','Combine multiple PDF files into one document.','PDF Tools','pdf','FilePlus2','#E84545',ARRAY['.pdf'],10,100,2,10,true,1243),
  ('pdf-compress','Compress PDF','Reduce PDF file size while preserving quality.','PDF Tools','pdf','FileDown','#E84545',ARRAY['.pdf'],10,100,2,10,true,987),
  ('pdf-split','Split PDF','Extract pages or split a PDF into multiple files.','PDF Tools','pdf','Scissors','#E84545',ARRAY['.pdf'],10,100,1,5,true,654),
  ('image-compress','Compress Image','Reduce image file size without visible quality loss.','Image Tools','image','ImageIcon','#3B82F6',ARRAY['.jpg','.jpeg','.png','.webp'],10,100,2,10,true,2187),
  ('image-resize','Resize Image','Scale images to any dimensions while maintaining aspect ratio.','Image Tools','image','Maximize2','#3B82F6',ARRAY['.jpg','.jpeg','.png','.webp'],10,100,2,10,true,1456),
  ('image-convert-jpg','Convert to JPG','Convert PNG, WebP, and other formats to JPG.','Image Tools','image','ArrowRightLeft','#3B82F6',ARRAY['.png','.webp','.gif','.bmp'],10,100,2,10,true,1102),
  ('image-convert-png','Convert to PNG','Convert JPG, WebP and other formats to PNG with transparency.','Image Tools','image','ArrowRightLeft','#3B82F6',ARRAY['.jpg','.jpeg','.webp','.gif'],10,100,2,10,true,876),
  ('image-convert-webp','Convert to WebP','Convert images to WebP for smaller file sizes on the web.','Image Tools','image','Globe','#3B82F6',ARRAY['.jpg','.jpeg','.png'],10,100,2,10,true,743),
  ('doc-generic','Document Processor','Process and convert common document formats.','Document Tools','document','FileText','#10B981',ARRAY['.docx','.doc','.txt'],10,50,1,5,true,412)
ON CONFLICT (slug) DO NOTHING;
```

---

## Step 6 — Set your custom domain

In Vercel project → **Settings → Domains**, add your domain and follow the DNS instructions.

---

## Notes

- Uploaded files are stored in `/tmp` (ephemeral). Files are processed and immediately available for download for 1 hour, then cleaned up. This works on Vercel's serverless runtime.
- The cleanup cron runs on each cold start — on Vercel this is best-effort.
- If you need persistent file storage across long sessions, consider integrating Vercel Blob or AWS S3.
