# MessMate n8n Pack — Workflows & Integration Guide

**Updated:** 2025-08-13T19:44:18 (IST)  
**Contains:** 6 import‑ready workflows for notifications, payments, waste logs, QR checks, and an AI agent.

---

## What’s inside

| File | Workflow | Trigger | Purpose |
|---|---|---|---|
| `MM_01_booking_created.json` | Booking Created → Push + Slack | **POST** `/webhook/booking-created` | Notify user & team when a booking is made |
| `MM_02_booking_cancelled.json` | Booking Cancelled → Push + Slack | **POST** `/webhook/booking-cancelled` | Notify on cancellations |
| `MM_03_recharge_log.json` | Recharge Log → Verify + Notify | **POST** `/webhook/recharge-log` | Verify Stripe/Razorpay (stub) and notify |
| `MM_04_waste_log.json` | Waste Log → Slack + Echo | **POST** `/webhook/waste-log` | Log daily waste and echo result |
| `MM_05_qr_check.json` | QR Check → Allow/Deny | **POST** `/webhook/qr-check` | Simple gate for plate/entry checks |
| `MM_06_ai_agent.json` | MessMate AI Agent | **POST** `/webhook/agent` | LLM chat that returns structured JSON actions |

> **Timezone:** All cron/time logic defaults to `Asia/Kolkata`.
> **Credentials:** n8n does not export secrets. After import, open each workflow and set values in the **Config** nodes / credential selectors.

---

## Quick start

1. In n8n: **Workflows → Import from file** (import each `.json` you need).  
2. Open the **Config** node and set these values (as applicable):
   - `SLACK_WEBHOOK_URL` — Slack incoming webhook URL
   - `FCM_SERVER_KEY` — Firebase Cloud Messaging (legacy server key)
   - `SIGNING_SECRET` — Optional HMAC for booking webhooks
   - `STRIPE_SECRET_KEY`, `RAZORPAY_BASIC_USER`, `RAZORPAY_BASIC_PASS` — for `/recharge-log`
   - `ACCESS_CODE` — token for `/qr-check`
3. For **MM_06 AI Agent**, open the **LLM (Structured JSON)** node and select your **OpenAI API** credential.
4. Click **Activate** to start listening for webhooks.

---

## Common payload conventions

- **User object** (when present):
  ```json
  { "id": "uuid", "name": "Harseerat Bhatia", "email": "harseerat@example.com", "fcmToken": "xxxx" }
  ```
- **Booking object** (where applicable):
  ```json
  { "mealType": "breakfast|lunch|dinner", "date": "YYYY-MM-DD", "price": 25 }
  ```

**HMAC (optional):** If you set `SIGNING_SECRET`, send an `X-Signature` header with `HMAC-SHA256` of the raw body.

---

## Workflow details & examples

### 1) Booking Created → Push + Slack
**Path:** `/webhook/booking-created`  
**Config:** `SLACK_WEBHOOK_URL`, `FCM_SERVER_KEY`, optional `SIGNING_SECRET`  
**Input JSON:**
```json
{ "user": { "id": "u1", "name": "Harseerat", "email": "h@x.com", "fcmToken": "FCM..." },
  "booking": { "mealType": "lunch", "date": "2025-08-14", "price": 25 } }
```
**Response:** `{ "ok": true }`  
**Side effects:** Sends FCM push and Slack message.

**cURL test:**
```bash
curl -X POST https://<n8n>/webhook/booking-created \
  -H "Content-Type: application/json" \
  -d '{"user":{"id":"u1","name":"Harseerat","email":"h@x.com","fcmToken":"FCM..."}, "booking":{"mealType":"lunch","date":"2025-08-14","price":25}}'
```

---

### 2) Booking Cancelled → Push + Slack
**Path:** `/webhook/booking-cancelled`  
**Config:** same as above  
**Input JSON:**
```json
{ "user": { "id": "u1", "name": "Harseerat", "email": "h@x.com", "fcmToken": "FCM..." },
  "booking": { "mealType": "lunch", "date": "2025-08-14" } }
```
**Response:** `{ "ok": true }`

**cURL test:**
```bash
curl -X POST https://<n8n>/webhook/booking-cancelled \
  -H "Content-Type: application/json" \
  -d '{"user":{"id":"u1","name":"Harseerat","email":"h@x.com","fcmToken":"FCM..."}, "booking":{"mealType":"lunch","date":"2025-08-14"}}'
```

---

### 3) Recharge Log → Verify + Notify
**Path:** `/webhook/recharge-log`  
**Config:** `STRIPE_SECRET_KEY`, `RAZORPAY_BASIC_USER`, `RAZORPAY_BASIC_PASS`, `SLACK_WEBHOOK_URL`, `FCM_SERVER_KEY`  
**Input JSON:**
```json
{ "user": { "id": "u1", "name": "Harseerat", "email": "h@x.com", "fcmToken": "FCM..." },
  "payment": { "provider": "razorpay", "paymentId": "pay_123", "amount": 200 } }
```
**Behavior:** Routes to provider verify step (Stripe/Razorpay), then formats a receipt and sends Slack + FCM.  
**Response:** `{ "ok": true }`

**cURL test (Stripe):**
```bash
curl -X POST https://<n8n>/webhook/recharge-log \
  -H "Content-Type: application/json" \
  -d '{"user":{"id":"u1","name":"Harseerat","email":"h@x.com","fcmToken":"FCM..."}, "payment":{"provider":"stripe","paymentId":"pi_123","amount":200}}'
```

---

### 4) Waste Log → Slack + Echo
**Path:** `/webhook/waste-log`  
**Config:** `SLACK_WEBHOOK_URL`  
**Input JSON:**
```json
{ "date": "2025-08-14", "waste_reduced_kg": 2.3 }
```
**Output:** echoes `{ "date", "waste_reduced_kg", "text" }` and posts to Slack.

**cURL test:**
```bash
curl -X POST https://<n8n>/webhook/waste-log \
  -H "Content-Type: application/json" \
  -d '{"date":"2025-08-14","waste_reduced_kg":2.3}'
```

---

### 5) QR Check → Allow/Deny
**Path:** `/webhook/qr-check`  
**Config:** `ACCESS_CODE` (default `GREEN123`)  
**Input JSON:**
```json
{ "qr": "SOME_QR_STRING_GREEN123", "hasBooking": true }
```
**Response:** `{ "allow": true|false, "reason": "matched|no_match" }`

**cURL test:**
```bash
curl -X POST https://<n8n>/webhook/qr-check \
  -H "Content-Type: application/json" \
  -d '{"qr":"SOME_QR_STRING_GREEN123","hasBooking":true}'
```

---

### 6) MessMate AI Agent (LLM → JSON)
**Path:** `/webhook/agent`  
**Credentials:** Select **OpenAI API** in the LLM node.  
**Input JSON (context optional):**
```json
{ "message": "Can you book dinner for tomorrow?",
  "walletBalance": 450,
  "menu": {"date":"2025-08-14","dinner":["Paneer","Roti"]},
  "bookings": [] }
```
**Response (strict JSON):**
```json
{
  "reply": "I can book dinner for 2025-08-15. Your wallet balance is sufficient.",
  "proposedActions": [
    { "type": "book", "date": "2025-08-15", "mealType": "dinner", "amount": 25 }
  ],
  "metadata": { "confidence": 0.86, "notes": "user_intent=book" }
}
```

**UI integration (React):**
```ts
const res = await fetch("https://<n8n>/webhook/agent", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ message: input, walletBalance, menu, bookings })
});
const data = await res.json(); // { reply, proposedActions, metadata }
```

---

## Security & best practices

- Keep webhook URLs private/unpredictable; add **Basic Auth** or reverse‑proxy auth if possible.
- If the caller is trusted, enable **HMAC (SIGNING_SECRET)** on booking webhooks and send `X-Signature`.
- Never hardcode API keys in the workflow; use n8n **credentials** or env vars.
- Treat `/recharge-log` as a notifier; do **business‑critical debits/credits** in your core backend.
- Add rate limiting on the ingress proxy (e.g., Nginx/Cloudflare).

---

## Troubleshooting

- **401/403 on Slack/FCM** → Recheck `SLACK_WEBHOOK_URL` / `FCM_SERVER_KEY`.
- **Stripe 401** → Confirm `STRIPE_SECRET_KEY`. **Razorpay 401** → fill Basic Auth creds.
- **Agent returns parse_error** → Ensure OpenAI credential is selected; LLM must return **strict JSON**.
- **CORS/Preflight** (browser) → Call via your backend proxy or allow `OPTIONS` on the proxy.

---

## Changelog
- v2: hardened payload formats, consistent responses, clearer Config variables, AI agent JSON parsing guard.

Happy automating!
