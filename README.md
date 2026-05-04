# StudyAI — Multi-AI Study Platform

A full-stack study platform that integrates 5 AI engines (ChatGPT, Perplexity, Claude, Turbo, Wolfram) with Stripe subscriptions and token-based usage tracking.

---

## Architecture

```
User → Frontend (React) → Backend (Express) → AI APIs
                                    ↓
                              SQLite Database (usage, users)
                                    ↓
                                Stripe (payments + webhooks)
```

---

## Quick Start

### 1. Backend Setup

```bash
cd backend
npm install
cp .env.example .env
# Fill in your API keys in .env
npm run dev
```

### 2. Frontend Setup

The frontend is a React component (`studyai-frontend.jsx`). To run it locally:

```bash
npx create-react-app studyai-frontend
cd studyai-frontend
# Replace src/App.js with the contents of studyai-frontend.jsx
# Add to index.html <head>:
#   <link href="https://fonts.googleapis.com/css2?family=Poppins:wght@300;400;500;600;700;800&display=swap" rel="stylesheet">
npm start
```

---

## API Keys Required

| Service     | Get your key at                                  | Env Variable          |
| ----------- | ------------------------------------------------ | --------------------- |
| OpenAI      | https://platform.openai.com/api-keys             | `OPENAI_API_KEY`      |
| Perplexity  | https://www.perplexity.ai/settings/api            | `PERPLEXITY_API_KEY`  |
| Anthropic   | https://console.anthropic.com/                   | `ANTHROPIC_API_KEY`   |
| Wolfram     | https://developer.wolframalpha.com/              | `WOLFRAM_APP_ID`      |
| Stripe      | https://dashboard.stripe.com/apikeys             | `STRIPE_SECRET_KEY`   |

---

## Stripe Setup

### 1. Create the Product

In Stripe Dashboard → Products → Add Product:
- **Name:** Student AI Plan
- Create **two prices**:
  - **Student:** $9/month (recurring) → copy the `price_xxx` ID into `STRIPE_STUDENT_PRICE_ID`
  - **Pro:** $29/month (recurring) → copy the `price_xxx` ID into `STRIPE_PRO_PRICE_ID`

### 2. Set Up the Webhook

In Stripe Dashboard → Developers → Webhooks → Add endpoint:
- **Endpoint URL:** `https://your-domain.com/api/stripe/webhook`
- **Events to listen for:**
  - `invoice.paid`
  - `customer.subscription.updated`
  - `customer.subscription.deleted`
- Copy the **Signing Secret** (`whsec_...`) into `STRIPE_WEBHOOK_SECRET`

For local testing use the Stripe CLI:
```bash
stripe listen --forward-to localhost:4000/api/stripe/webhook
```

### 3. How Payments Work

1. User clicks "Subscribe" → Frontend calls `POST /api/stripe/create-checkout`
2. Backend creates a Stripe Checkout Session and returns the URL
3. User completes payment on Stripe's hosted page
4. Stripe fires `invoice.paid` webhook to your server
5. Backend receives webhook → resets `tokens_used = 0` and sets new `token_quota`
6. User is now on their new plan with full tokens

---

## Subscription Plans

| Plan    | Price   | Tokens/Month | Max per Prompt |
| ------- | ------- | ------------ | -------------- |
| Free    | $0      | 500          | 500            |
| Student | $9/mo   | 25,000       | 500            |
| Pro     | $29/mo  | 100,000      | 500            |

---

## API Endpoints

### Auth
| Method | Path             | Description       | Auth |
| ------ | ---------------- | ----------------- | ---- |
| POST   | `/api/auth/signup` | Create account    | No   |
| POST   | `/api/auth/login`  | Log in            | No   |
| GET    | `/api/auth/me`     | Get current user  | Yes  |

### AI Tools
| Method | Path              | AI Engine   | Purpose                        |
| ------ | ----------------- | ----------- | ------------------------------ |
| POST   | `/api/ai/chatgpt`   | GPT-5 Mini  | General study Q&A              |
| POST   | `/api/ai/perplexity` | Perplexity  | Research with verified sources  |
| POST   | `/api/ai/claude`     | Claude      | Writing assistance             |
| POST   | `/api/ai/turbo`      | GPT-4 Turbo | Flashcards & quizzes           |
| POST   | `/api/ai/wolfram`    | Wolfram Alpha | Math & problem solving       |

All AI endpoints require `Authorization: Bearer <token>` header and accept `{ "message": "..." }` in the body.

### Stripe
| Method | Path                          | Description              | Auth |
| ------ | ----------------------------- | ------------------------ | ---- |
| POST   | `/api/stripe/create-checkout`  | Start payment flow       | Yes  |
| GET    | `/api/stripe/subscription`     | Get subscription status  | Yes  |
| POST   | `/api/stripe/customer-portal`  | Manage subscription      | Yes  |
| POST   | `/api/stripe/webhook`          | Stripe webhook receiver  | No*  |

*Webhook is verified by Stripe signature, not JWT.

---

## Database Schema

```sql
users:
  id, email, password_hash, name,
  stripe_customer_id, subscription_status,
  token_quota, tokens_used,
  current_period_end, created_at, updated_at

usage_logs:
  id, user_id, ai_tool, tokens_used,
  prompt, created_at
```

---

## Token System

- Each AI request deducts tokens from the user's monthly quota
- Maximum 500 tokens per individual prompt
- When `tokens_used >= token_quota`, further AI requests are blocked
- The `invoice.paid` webhook resets `tokens_used` to 0 each billing cycle
- Token count is shown live in the bottom-left corner of the frontend

---

## Deployment

### Backend (e.g. Railway, Render, Fly.io)

1. Push the `backend/` folder to your hosting provider
2. Set all environment variables from `.env.example`
3. Make sure the Stripe webhook URL points to your live domain

### Frontend (e.g. Vercel, Netlify)

1. Wrap the React component in a standard React app
2. Update `API_BASE` in the frontend to point to your deployed backend URL
3. Deploy

---

## File Structure

```
studyai/
├── backend/
│   ├── server.js              # Express entry point
│   ├── package.json           # Dependencies
│   ├── .env.example           # Environment template
│   ├── config/
│   │   └── db.js              # SQLite setup & schema
│   ├── middleware/
│   │   ├── auth.js            # JWT verification
│   │   └── tokens.js          # Token limit enforcement
│   └── routes/
│       ├── auth.js            # Signup / Login / Me
│       ├── ai.js              # AI proxy routes (5 engines)
│       └── stripe.js          # Checkout + webhook
└── studyai-frontend.jsx       # React frontend (artifact)
```
