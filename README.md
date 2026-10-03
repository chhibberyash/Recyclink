# RecycLink

Scan scrap with AI, get an instant value estimate, get matched to a government-
authorised recycler, get picked up, get paid, get a verifiable digital
certificate. Plus a **Recycler/Driver Dashboard** and an **Admin Panel**.

Stack: React (Vite) + Tailwind v4, Supabase (Postgres + Auth + Storage + Edge
Functions), Gemini 2.5 Flash for scrap identification, pdf-lib for real PDF
certificates with QR verification.

## This copy is already wired to your live project

Everything below has already been applied to your Supabase project `scrap_app`
(`qzkkkwknzhfotkvalkpj`) — you don't need to re-run the SQL unless you're
setting up a fresh project. `.env` already points at it.

## The one thing you still need to do: set the Gemini secret

The `analyze-scrap` function is deployed, but it has no Gemini key yet — that's
why scanning was failing. There is no way to set a function secret from an MCP
integration, so this step needs the Supabase CLI or dashboard:

**Dashboard:** Project → Edge Functions → `analyze-scrap` → Secrets → add
`GEMINI_API_KEY`.

**CLI:**
```
npm install -g supabase
supabase login
supabase link --project-ref qzkkkwknzhfotkvalkpj
supabase secrets set GEMINI_API_KEY=your_new_rotated_key
```

Use a freshly rotated key — the one pasted earlier in chat is compromised.
Get a new one at https://aistudio.google.com/app/apikey.

## What's new in this update

### 1. Recycler / Driver Dashboard (`/recycler`)
- Log in with an account that has `profiles.role = 'recycler'` and a
  `profiles.recycler_id` pointing at a row in `recyclers`.
- **New Requests** tab: Accept or Reject incoming pickups (reject asks for a
  reason, saved to `pickups.rejection_reason`).
- **Active** tab: Mark "On the Way", then "Collected" — collecting opens a form
  to enter the actual verified weight, the final rate, and an optional proof
  photo (uploaded to the `proofs` storage bucket). This computes the final
  amount and creates a `transactions` row for the customer.
- **Awaiting Payment / Completed** tabs to track the rest of the lifecycle.
- Demo login: `greencycle@recyclink.app` / `Recycler@12345`

### 2. Payment page (`/payment/:transactionId`)
- **Cash only** — UPI is shown, greyed out, labelled "Under development".
- Selecting cash moves the transaction to `processing`.
- An admin then marks it `processed` (Admin Panel → Payments to Process),
  which flips the linked pickup to `completed` and unlocks certificate
  generation for that user.

### 3. Real digital certificates (`/certificates`, `generate-certificate` function)
- Once a transaction is `processed`, "My Certificates" shows a **Generate**
  button. This calls the `generate-certificate` Edge Function, which:
  - Builds an actual PDF (via `pdf-lib`) with a unique certificate number
    (`RC-CERT-<year>-<random>`), the user's name, material + weight, amount
    earned, and recycler details.
  - Fetches a QR code (via `api.qrserver.com`) that points at
    `/verify/<certificate_number>` and embeds it in the PDF.
  - Uploads the PDF to the public `certificates` storage bucket and records it
    in the `certificates` table.
- The certificates list has Download and Share buttons, and shows the public
  verify link.
- `/verify/:certNumber` is a public, no-login page — what the QR code opens —
  showing the certificate details so anyone (an employer, a CSR auditor) can
  confirm it's genuine. This is deliberately open to anonymous reads.

### 4. Admin Panel (`/admin`)
- **Pricing tab**: edit every recycler's per-kg buying rate directly; saves to
  `recyclers.buying_rates` and shows up immediately in "Find Recycler".
- **Payments to Process tab**: see every cash payment awaiting confirmation,
  mark as processed with one click.
- **All Pickups tab**: a live read-only feed of every pickup in the system.
- Demo login: `admin@recyclink.app` / `Admin@12345`

### 5. Schema additions
- `profiles.role` (`user` / `recycler` / `admin`) and `profiles.recycler_id`.
- `pickups.status` now includes `accepted`, `rejected`, `awaiting_payment`,
  `completed`; plus `proof_photo_url` and `rejection_reason`.
- `transactions.payment_method` (`cash`/`upi`) and a `processing` → `processed`
  status step.
- `certificates.verify_url`, `recycler_name`, `amount`.
- Storage buckets `proofs` and `certificates` (both public-read).

## Running locally

```
npm install
npm run dev
```

- Customer flow: splash → sign up / log in → scan scrap → find recycler →
  schedule pickup → tracking → (once recycler collects) pay → certificate.
- Recycler flow: log in with a recycler account → `/recycler`.
- Admin flow: log in with an admin account → `/admin`.

## Adding real recyclers/sellers with dashboard logins

1. Insert the seller into `recyclers` (Table Editor or SQL, see `0001_init.sql`
   for the shape).
2. Create their login: Authentication → Users → Add User (set email + password,
   confirm email), which auto-creates a `profiles` row via the trigger.
3. In `profiles`, set that user's `role = 'recycler'` and `recycler_id` to the
   seller's id from step 1.

## Build & deploy

```
npm run build
```

Deploy `dist/` to Vercel/Netlify/Cloudflare Pages with the same
`VITE_SUPABASE_URL` / `VITE_SUPABASE_ANON_KEY` env vars. Also set
`APP_BASE_URL` as a secret on the `generate-certificate` function
(`supabase secrets set APP_BASE_URL=https://your-real-domain.com`) so QR codes
point at your real deployed verify page instead of the placeholder.

## Project structure

```
src/
  pages/            Splash, Login, Home, Scanner, FindRecycler, SchedulePickup,
                     Tracking, Payment, Certificates, VerifyCertificate,
                     Activity, Profile
  pages/recycler/   RecyclerDashboard
  pages/admin/      AdminPanel
  components/       BottomNav, RequireRole
  context/          AuthContext (email/password auth + role helpers)
  lib/              supabaseClient.js, gemini.js
supabase/
  migrations/       0001 schema, 0002 roles/certs/payments, 0003 demo accounts
  functions/        analyze-scrap (Gemini proxy), generate-certificate (PDF+QR)
```

## Still simplified (clear next steps)

- No live map/GPS tracking of the recycler en route — status is manual.
- No push/SMS notifications on status changes.
- UPI payment is a placeholder; wire a real gateway (Razorpay etc.) when ready.
- Certificate PDF design is intentionally simple — easy to restyle in
  `supabase/functions/generate-certificate/index.ts`.
