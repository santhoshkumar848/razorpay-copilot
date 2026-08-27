# AI Customer Support Co-Pilot for Razorpay Merchants

A full-stack demo app: a customer-facing chat widget that auto-answers
common payment/refund/settlement questions using the Razorpay API, escalates
anything complex to a human with full context, and a merchant analytics
dashboard showing query trends and auto-resolution rate.

```
razorpay-copilot/
├── backend/     Node.js + Express API (Razorpay + Supabase integration)
└── frontend/    React (Vite) chat widget + dashboard
```

## How it works

1. **Chat** — customer types a question → `POST /api/chat`.
2. **Classify** — `services/queryClassifier.js` tags it as
   `refund_status` / `payment_failed` / `settlement` / `dispute` / `account` / `other`,
   and flags emotionally-charged or repeat-complaint language for escalation.
3. **Auto-reply** — if the category is known and a payment/refund ID is present
   (e.g. `pay_Iu...`, `rfnd_Iu...`), `services/razorpayService.js` calls the real
   Razorpay API and turns the response into a plain-English answer.
4. **Escalate** — anything unresolved (unknown category, API error, or an
   escalation signal) is written to the `escalations` table with the full
   classification + chat context, and optionally POSTed to a webhook
   (Slack/Freshdesk/etc.) if `ESCALATION_WEBHOOK_URL` is set.
5. **Log** — every query (resolved or not) is written to `query_logs`,
   which is the only data source for the dashboard.
6. **Dashboard** — `GET /api/dashboard/summary` aggregates `query_logs`
   into top categories, auto-resolved vs escalated %, and a daily trend line.

## Prerequisites

- Node.js 18+
- A free [Supabase](https://supabase.com) project
- A [Razorpay](https://razorpay.com) account (test mode keys are fine)

## 1. Set up the database

1. Create a free project at supabase.com.
2. Open **SQL Editor** → paste the contents of `backend/db/schema.sql` → **Run**.
3. Copy your **Project URL** and **service_role key** from *Project Settings → API*.

## 2. Backend setup

```bash
cd backend
npm install
cp .env.example .env
# edit .env: fill in RAZORPAY_KEY_ID, RAZORPAY_KEY_SECRET, SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY
npm run dev        # or: npm start
```

The API runs at `http://localhost:5000`. Check `http://localhost:5000/api/health`.

## 3. Frontend setup

```bash
cd frontend
npm install
cp .env.example .env   # VITE_API_URL should point at the backend above
npm run dev
```

Open the URL Vite prints (typically `http://localhost:5173`).

## 4. Try it out

In the **Chat** tab, try:
- `"When will my refund arrive?"` → generic policy answer.
- `"Why did payment pay_Iu123456789 fail?"` → live Razorpay lookup (use a real
  test-mode payment id from your Razorpay dashboard to see a live answer).
- `"This is unacceptable, I've complained three times about my settlement"`
  → escalated to human support with context.

Switch to the **Dashboard** tab to see category breakdown, resolution %, the
daily volume trend, and the open escalation queue update as you chat.

## Notes on going to production

- Put the backend behind HTTPS and never expose `RAZORPAY_KEY_SECRET` or the
  Supabase **service_role** key to the browser — both are backend-only.
- Swap `queryClassifier.js`'s keyword rules for an LLM-based classifier if you
  need to handle more varied phrasing.
- Replace the webhook stub in `escalationService.js` with your real helpdesk
  API (Freshdesk/Zendesk/Slack) call.
- Add authentication so `merchantId` comes from a verified session instead of
  being passed by the client.

