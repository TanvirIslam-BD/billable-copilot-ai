# Billable Copilot — Dashboard Build Guide

`app.billable.copilotai.live`

The signed-in space where a freelancer keeps their contracts, checks requests, sends change orders, and talks to the copilot about their own money.

---

## 0. The One Architectural Decision That Matters

Your landing page promises *"works inside Claude and ChatGPT."* That promise only holds if the web dashboard and the user's own AI assistant are calling **the same MCP server**. Build it in that order:

```
                  ┌──────────────────────┐
                  │   Billable MCP       │
                  │   (tools + auth)     │
                  └──────────┬───────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
   Web dashboard      User's Claude        User's ChatGPT
   (your chat UI)     (their own app)      (their own app)
```

The MCP server is the product. The dashboard is one client of it — the one you control, the one that renders things nicely, the one that takes payment. Never build dashboard-only logic that the MCP layer can't reach, or the two surfaces will drift apart within a month.

This is the same shape as Money Copilot, so most of your server scaffolding transfers.

---

## 1. Don't Build a Chat-Only Dashboard

The tempting version is one big chat box. It fails, for three reasons:

- A freelancer landing at 9am wants to *see* what's open, not compose a question about it.
- Chat can't be scanned. "Which contracts are near their revision limit" is a glance, not a conversation.
- If the whole product is a chat box, the user correctly concludes they could just use Claude directly and skip your subscription.

**The rule:** deterministic surfaces for state, chat for judgment.

| Deterministic UI | Chat |
|---|---|
| Contract list, revision counters | "Is this request in scope?" |
| Pending change orders + status | "Rewrite that reply, softer" |
| Recovery ledger, monthly totals | "Why did I lose money on the Kessler job?" |
| Client list, payment aging | "Draft a note chasing the March invoice" |

Chat sits in a right-hand rail, always present, aware of whatever the user is looking at.

---

## 2. Data Model

Six tables. Resist adding more until users ask.

```sql
users
  id, email, name, currency, timezone, plan, trial_ends_at, created_at

clients
  id, user_id, name, email, company, created_at

contracts
  id, user_id, client_id, title, status(active|closed|draft)
  raw_text, parsed_scope(jsonb), hourly_rate, project_fee
  revisions_allowed, revisions_used
  start_date, due_date, created_at

requests                      -- every client ask that got checked
  id, user_id, contract_id, source(email|slack|whatsapp|manual|mcp)
  raw_text, received_at
  verdict(in_scope|out_of_scope|ambiguous)
  clause_ref, reasoning, suggested_amount, suggested_days
  status(open|waived|ordered|paid)

change_orders
  id, user_id, request_id, contract_id
  amount, currency, days_added, body_text, pdf_url
  status(draft|sent|accepted|declined|paid)
  sent_at, responded_at

events                        -- the recovery ledger, append-only
  id, user_id, type, amount, meta(jsonb), created_at
```

Two notes worth following:

- **`parsed_scope` as JSONB:** When a contract is uploaded, parse it once into structured scope — deliverables, limits, exclusions, rates — and store it. Every subsequent scope check reads the JSON, not the raw contract. Cheaper, faster, and far more consistent than re-reading a PDF every time.
- **`events` is append-only:** Your headline number ("$1,840 recovered") must be reconstructible from an audit trail, not a mutable counter. When a user disputes the figure — and they will — you need to show the line items.

---

## 3. MCP Tool Surface

Same naming discipline as Money Copilot. Verb-first, plain language, one job each.

### Contracts
- `add_contract` — raw text or file in; parsed scope out
- `list_contracts` — active first, with revisions remaining
- `get_contract` — full parsed scope for one
- `update_contract` — amend scope after an accepted change order
- `close_contract` — project delivered

### The Core Loop
- `check_request` — *the tool that matters.* Request text + contract → verdict, clause reference, reasoning, suggested price, suggested delay
- `list_requests` — filter by contract, verdict, status
- `draft_reply` — verdict → the message the freelancer actually sends; takes a `tone` argument (`warm` | `neutral` | `firm`)

### Change Orders
- `create_change_order` — from a request
- `send_change_order` — email + PDF
- `list_change_orders` — with status
- `mark_change_order` — accepted, declined, paid

### Money
- `get_recovery_report` — monthly: caught, billed, waived, collected
- `get_contract_health` — revisions used, unbilled work, days over
- `list_unpaid` — change orders sent but not paid, aged

### Housekeeping
- `set_defaults` — currency, default rate, default tone
- `export_data` — CSV/JSON, everything

### Two Critical Design Rules for `check_request`
1. It must return **`ambiguous`** as a real verdict, not force a binary. Contracts are vague; pretending otherwise produces confident wrong answers, and one confident wrong answer costs you the user's trust permanently.
2. It must always return `clause_ref` and `reasoning`. The freelancer is about to ask a client for money based on this output — they need to see *why* before they send it.

---

## 4. Screens (5 Total)

Five screens make up the entire product:

### 1. Today (Default Landing)
The one-glance answer. Open requests needing a verdict, change orders awaiting client response, anything aging past 14 days. If everything's clear, say so plainly and show the month's recovered total. An empty state here is good news and should read like it.

### 2. Contracts
Card or row per active contract: client, title, revisions used as a small meter (`2 / 3`), unbilled work to date, days to deadline. Clicking opens the parsed scope with the raw contract behind a toggle — users need to verify your parse was right, especially in week one.

### 3. Requests
The stream. Every check, newest first, with verdict chips. Paste box pinned at the top — this is the highest-frequency action in the product, so it should never be more than one click from anywhere.

### 4. Change Orders
Pipeline view: Draft → Sent → Accepted → Paid. Amounts in monospace, right-aligned. This screen is where the user sees the product paying for itself, so it deserves care.

### 5. Ledger
Monthly recovery report. Caught / billed / waived / collected, plus a per-client breakdown. The line that keeps subscriptions alive: *"Billable found $2,340 this month. You billed $1,900 of it."*

Waived work being visible is deliberate. Seeing what they gave away voluntarily is more motivating than only seeing wins.

---

## 5. The Chat Rail

- **Placement:** Fixed right column at ≥1200px, slide-over sheet below that. Always reachable, never covering the main content on desktop.
- **Context Injection:** Every message carries what the user is currently looking at:

```json
{
  "screen": "contract_detail",
  "contract_id": "ct_8812",
  "visible_state": { "revisions_used": 3, "revisions_allowed": 3, "unbilled": 640 }
}
```

Without this, the user types "is this over budget?" and the copilot has to ask "which one?" — which instantly feels dumber than it is.

- **Streaming:** Token-stream the reply. Show tool calls as they run, as quiet inline chips: `checking contract §2…`. Users trust an assistant more when they can see it doing work rather than pausing mysteriously.
- **Suggested Prompts:** Three per screen, changing with context. On the contract screen:
  - *"What's still unbilled here?"*
  - *"Draft a check-in for Maya"*
  - *"Am I on track for the deadline?"*
  These teach capability without a tutorial.
- **Actions Need Confirmation:** Reading is free; writing is not. Anything that sends an email, creates a change order, or amends a contract renders a confirmation card in the chat with an explicit button. Never let the model send a client-facing message on its own.

---

## 6. Auth and Billing

- **Auth:** Clerk or Supabase Auth. Google OAuth plus magic link — freelancers hate passwords. Skip email/password entirely.
- **MCP Auth:** OAuth 2.1 with PKCE, per the current MCP spec. The user connects Billable inside Claude, approves once, and the token maps to their account. Scope tokens per user, never a shared key.
- **Billing:** Paddle or Lemon Squeezy as merchant of record — handles global VAT and works from Bangladesh without a US entity. Webhook `subscription.updated` → update `users.plan`. Gate on plan at the tool layer, not just the UI, or MCP users bypass your limits entirely.
- **Trial:** 14 days, no card. Track `trial_ends_at`. At day 11, if they've recovered real money, show it: *"Billable found $1,240 for you. Trial ends in 3 days."* That's your single strongest conversion moment in the whole product — build it properly.

---

## 7. Design Continuity

Carry the landing page tokens over, with one inversion.

The marketing site is light with a dark demo console. The **app is the console** — so the working surface goes darker:

```css
--app-bg:      #0D1012;
--app-panel:   #161A1D;
--app-line:    #252A2D;
--app-text:    #E4E9EB;
--app-muted:   #8A9396;
--money:       #0E9F6E;   /* recovered revenue only */
--flag:        #E5484D;   /* out-of-scope only */
```

Same rule as the landing page: green means money recovered, red means out of scope, and neither colour is ever used decoratively. Every amount stays monospace with `tabular-nums` so columns align.

**Stack:** Next.js App Router, Tailwind, shadcn/ui, TanStack Query. Vercel Postgres or Supabase. Boring on purpose — the interesting engineering belongs in the MCP layer.

---

## 8. Build Order

Ship the loop before the polish:

1. **Week 1 — The Spine:** Auth → `add_contract` with parsing → `check_request` → verdict displayed. Nothing else. If `check_request` isn't accurate, no amount of dashboard saves the product.
2. **Week 2 — Closing the Loop:** `draft_reply` → change order creation → PDF → send. Now money can actually move.
3. **Week 3 — Making It Visible:** Today screen, ledger, recovery report. This is retention; without it users forget the product is working.
4. **Week 4 — The Promise:** MCP OAuth, published server, connect flow. Now the landing page headline is true.
5. **Week 5 — The Money:** Paddle, trial logic, plan gating, the day-11 conversion moment.

> *Chat rail comes after week 3. It's an accelerator on top of working surfaces, and building it first tempts you into a chat-only product.*

---

## 9. Where This Will Actually Break

- **Parsing bad contracts:** Half of freelance "contracts" are an email thread or a Notion page. Your parser must handle informal text and, when scope genuinely isn't defined, say so and offer to help define it — rather than inventing structure that isn't there.
- **Ambiguity is the common case:** "Three revisions" — is a bug fix a revision? Your product's real value is a defensible answer plus the clause it came from, not a confident guess.
- **Trust is single-shot:** One wrong out-of-scope flag that a user sends to a client, and they churn and tell people. Bias `check_request` toward `ambiguous` over a wrong confident verdict, especially in the first months.
- **Client data is sensitive:** Contracts contain names, rates, and terms under NDA. No training on user data, encryption at rest, and a working delete. Say it plainly on the pricing page — your competitors already do, and freelancers ask.

---

## 10. The Metric to Watch

Not signups, not DAU. This one:

> **Recovered dollars per active user per month.**

If it's above the subscription price, retention takes care of itself and you can raise prices. If it's below, the product isn't working regardless of how good the dashboard looks — and no feature you add will fix that, only better `check_request` accuracy will.
