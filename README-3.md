# Melp Referral Integration (via Refferq)

This integration only generates a referral code for each Melp user and creates their shareable link. The external tool ([Refferq](https://github.com/refferq/refferq)) takes care of everything else — click tracking, commissions, payouts, and the affiliate dashboard.

---

## What Melp does here

Melp only has three responsibilities:

1. When a user signs up, register them as an affiliate in Refferq so they get a referral code.
2. Give users their shareable link.
3. When a referred user pays, tell Refferq so it can record the commission.

Everything else — commission math, hold periods, payout methods, the affiliate dashboard — lives in Refferq.

---

## Folder structure

```
referral/
├── __init__.py
├── schemas.py          # Pydantic models for what we send/receive from Refferq
├── service.py          # All HTTP calls to Refferq go through here
├── router.py           # FastAPI endpoints: /referral/link and /r/{code}
├── auth_hook.py        # Called from auth/service.py after a user registers
├── billing_hook.py     # Called from billing/webhook.py after a Stripe payment
├── MELP_ENV.example    # Env vars you need to set
├── migrations/
│   └── 0001_add_referral_cols.py   # Adds 2 columns to the users table
└── tests/
    └── test_referral_service.py
```

Drop this in at `backend/app/referral/`.

---

## How the flow works

Someone shares their referral link, e.g. `melp.com/r/ALEX-5832`. A friend clicks it, signs up, and later subscribes. What happens at each step:

**On signup (the person sharing the link):**
Melp calls Refferq to create an affiliate account for that user, and Refferq returns a unique code. We store Refferq's affiliate ID on the Melp user row for later reference.

**When someone clicks the link:**
The `/r/{code}` endpoint validates the code, sends a click event to Refferq's tracking API, sets a 30-day cookie in the browser, then redirects to the signup page. The React signup form reads that cookie and includes the code when the user registers.

**When the referred user pays:**
The Stripe webhook handler calls `notify_referral_conversion()`. It checks whether the paying user signed up with a referral code, and if so, sends a webhook to Refferq with the payment amount. Refferq creates a commission, holds it for 30 days (refund buffer), and credits the affiliate's balance when it matures.

---

## Database changes

Two columns on the existing `users` table:

- `refferq_affiliate_id` — Refferq's ID for this user as an affiliate
- `referral_code_used` — the code this user signed up with (needed later by the billing hook)

No new tables; everything else lives in Refferq's database.

Run the migration:
```bash
alembic upgrade head
```

---

## Setup

**1. Start both services**

`docker-compose.yml` in the root has Refferq and Melp on a shared network:
```bash
docker compose up -d
```
Refferq runs at `http://localhost:3001`, Melp backend at `http://localhost:8000`.

**2. Create a Refferq admin account**

Register at `http://localhost:3001/register`, then promote the account:
```sql
UPDATE users SET role = 'ADMIN', status = 'ACTIVE' WHERE email = 'your@email.com';
```
Then go to Settings → Integrations → API Keys, generate a key, and copy it.

**3. Set env vars**

In Melp's `.env`:
```env
REFFERQ_BASE_URL=http://refferq:3000
REFFERQ_API_KEY=<key from Refferq admin panel>
REFFERQ_WEBHOOK_SECRET=<random string, same value in both Melp and Refferq>
REFFERQ_AFFILIATE_SECRET=<another random string>
MELP_APP_URL=https://melp.com
```

In Refferq's `.env`:
```env
WEBHOOK_SECRET=<same value as REFFERQ_WEBHOOK_SECRET>
```

**4. Register the router in `main.py`**

```python
from app.referral.router import router as referral_router, redirect_router

app.include_router(referral_router, prefix="/referral", tags=["Referral"])
app.include_router(redirect_router, tags=["Referral Redirect"])
```

**5. Hook into auth (after user registration)**

In `backend/app/auth/service.py`, right after the new user is saved:
```python
from app.referral.auth_hook import register_referral_affiliate

await register_referral_affiliate(
    db=db,
    user_id=str(new_user.id),
    user_name=new_user.name,
    user_email=new_user.email,
    referral_code_used=payload.referral_code,
)
```

**6. Hook into billing (on Stripe payment success)**

In `backend/app/billing/webhook.py`, inside the `invoice.payment_succeeded` block:
```python
from app.referral.billing_hook import notify_referral_conversion

await notify_referral_conversion(
    db=db,
    paying_user_id=str(user.id),
    paying_user_email=user.email,
    amount_cents=invoice_amount_cents,
    stripe_payment_intent=payment_intent_id,
)
```

---

## API

**GET /referral/link** (requires auth) — returns the user's code and shareable link:
```json
{
  "referral_code": "ALEX-5832",
  "shareable_link": "https://melp.com/r/ALEX-5832"
}
```

**GET /r/{referral_code}** (public) — sets the attribution cookie and redirects to `/signup`. This is the link users share.

---

## Notes

**Referral calls never block the main flows.** Every call to Refferq is wrapped in try/except. If Refferq is down or errors, registration and payment still succeed. Failures are logged, not raised.

**Webhooks are signed.** Conversion events sent to Refferq include an HMAC signature (`sha256=...`) in the header, which Refferq verifies before acting. This prevents anyone from faking a conversion.

**Users never touch Refferq.** Melp auto-creates a Refferq account per user with a derived password (`sha256(secret + user_id)`). Users only ever see Melp; admins are the only ones who use the Refferq UI.

**Referral codes are validated first.** A code must match `^[A-Za-z0-9-]{3,32}$`. Anything else returns 400 before any redirect or Refferq call.

---

## Tests

```bash
pip install pytest pytest-asyncio httpx
pytest backend/app/referral/tests/ -v
```

No live Refferq instance needed — all HTTP calls are mocked. Tests cover the happy paths, Refferq being unavailable, HMAC signature presence, and code format validation.

---

## Environment variables

See `MELP_ENV.example` for the full list with descriptions.
