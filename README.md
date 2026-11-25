# Kesar & Co. Support Copilot

An n8n workflow that reads customer support emails, classifies them with an LLM, looks up the order, checks policy in code, drafts a reply with an AI agent, creates or updates tickets, and logs every email to a Google Sheet. Anything risky goes to a human.

**Candidate:** Nishchay Chaudhary
**Platform:** n8n Cloud (v2.48.8)
**LLM:** Groq, `openai/gpt-oss-120b` (early development and v1 used `openai/gpt-oss-20b`, see Key decisions)
**Scenario date:** 20 November 2026. All date logic uses each email's `received_at`, not the system clock.

| Deliverable | Link |
|---|---|
| Google Sheet (view access) | [Kesar Support](https://docs.google.com/spreadsheets/d/1ND9frVGF9OcPSomBIu_NTEghAllosTrbPEARilzEm3s/edit?usp=sharing) |
| 5-minute demo video | `<ADD LINK>` |
| Workflow export | [`workflow/kesar-copilot-v2.json`](workflow/kesar-copilot-v2.json) |
| Test report | [`TEST_REPORT.md`](TEST_REPORT.md) |

The Sheet has four tabs: `Audit Log` (v2, 25 rows), `Tickets`, `V1 Audit Log` (v1, 8 emails end to end) and `v1 baseline` (v1 classifier output for all 25 emails).

---

## 1. Status at a glance

| Requirement | Status | Notes |
|---|---|---|
| 1. Classify and extract with an LLM (schema-valid JSON) | Done | Structured Output Parser plus a validation Code node with a safe fallback |
| 2. AI agent with at least two tools | Done | `get_order` and `search_faq` |
| 3. Order lookup by order ID, then sender email | Done | E10 has no order ID and is found by sender email |
| 4. Tickets with duplicate check | Done | 9 tickets for 10 emails; E04 and E21 share `TKT-1052` |
| 5. Audit log, one row per email | Done | 25 rows |
| 6. Human in the loop | Done | Rule-based triggers plus confidence below 0.6 |
| 7. v1 vs v2 test | Done, with limits | v1 ran end to end on 8 order-status emails and as a classifier on all 25. See [`TEST_REPORT.md`](TEST_REPORT.md) |

Stretch goals done: Hindi reply (E09), eligibility and policy rules in a Code node, retry on failure with a pacing Wait node.

---

## 2. Architecture

```
Manual trigger
  -> Load Test Emails (Code: fetches test_emails.json, TEST_IDS filter)
  -> Limit Batch
  -> Loop Over Emails (batch size 1)
       -> Classify Email (LLM)            Classifier Model + Classification Schema
       -> Validate Classification (Code)  enum checks, order_id pattern, confidence clamp, hard rules
       -> Route by Intent (Switch)
            |
            |- order_status / return_refund / complaint
            |     -> Fetch Orders (HTTP GET orders.json)
            |     -> Order Lookup and Sender Check (Code)   by order_id, else sender email; sender must match
            |     -> Order Verified? (If)
            |          |- true  -> Policy Check (Code) -> Reply Drafting Agent -> Clean Reply -> Injection Guard
            |          |- false -> Privacy and No-Order Reply (Code, fixed wording)
            |
            |- product_query
            |     -> Product Needs Human? (If)
            |          |- true  -> Hold For Human          (medical, flagged, low confidence)
            |          |- false -> Product Prep -> Reply Drafting Agent (search_faq)
            |
            |- other -> Hold For Human (Code): spam and thank-you are logged only,
            |           partnership enquiries get the partnerships address, the rest go to a human
       -> Merge Branches
       -> Get Open Tickets (Google Sheets) -> Ticket Decision (Code) -> Create or Update? (Switch)
            -> Create Ticket / Update Ticket (Google Sheets)
       -> Log to Audit Log (Google Sheets append)
       -> Rate Limit Wait -> back to Loop Over Emails
```

**Sheets:** `Audit Log` (one row per email) and `Tickets` (one row per order with a return, refund or complaint). Ticket IDs follow the order number, for example `TKT-1052` for order `KC-1052`.

---

## 3. Key decisions

1. **Hard rules in code, wording in the LLM.** Return window (7 days), unopened-only rule, damage reporting window (48 hours), photo requirement, refund amounts, COD fee rules, sender verification and duplicate-ticket detection are all computed in Code nodes. The agent receives the result as facts and only writes the email. This is safer because a model can be talked out of a rule written in a prompt.
2. **Two layers against prompt injection.** The classifier has an `injection_detected` field, and `Validate Classification` forces `needs_human` when it is true. During testing, E12 showed that the classifier flagged the injection but the agent still copied the email's instruction ("your refund of Rs 2,448 has been approved"). I added the `Injection Guard` node, which replaces the reply with a fixed safe message and holds the email for a human.
3. **Privacy check before any order detail is shared.** If the sender email does not match the order's `customer_email` (E07), the workflow sends a fixed privacy reply with no order data.
4. **Restricted wording for complaints.** For damaged, wrong and missing items, the agent may only repeat the facts from `Policy Check`: free replacement or full refund after photo verification, and a team member confirms within 24 business hours. It cannot promise timelines or refunds.
5. **Product questions come only from the FAQ.** If the FAQ has no answer, or the email mentions a skin condition, it goes to a human (E20).
6. **Rate limits shaped the design.** The free Groq tier allows 8,000 tokens per minute and has a daily token cap. I process one email at a time with a Wait node and Retry On Fail. When the daily cap of `gpt-oss-20b` was nearly used up during testing, I moved both models to `gpt-oss-120b`, which had a separate budget. A first attempt on the Gemini free tier stopped on its small daily request limit.
7. **`needs_human` rules.** Angry or threatening tone, medical questions, unsupported requests (GST invoice, partnerships, press), spam, vague emails, prompt injection, sender mismatch, order not found and confidence below 0.6.

---

## 4. Setup

1. **n8n:** use n8n Cloud (free trial) or self-host. Import `workflow/kesar-copilot-v2.json`.
2. **Credentials (create in n8n, never in node fields):**
   - Groq API key, attached to `Classifier Model` and `Agent Model`.
   - Google Sheets OAuth2 (Sign in with Google on n8n Cloud).
3. **Google Sheet:** create a sheet named `Kesar Support` with two tabs, `Audit Log` and `Tickets`, using `templates/audit_log_template.csv` and `templates/tickets_template.csv`. Point the three Google Sheets nodes (`Get Open Tickets`, `Create Ticket` / `Update Ticket`, `Log to Audit Log`) at it. Set sharing to "Anyone with the link can view".
4. **Models:** set both Groq models to `openai/gpt-oss-120b`, temperature 0.
5. **Run:** in `Load Test Emails`, `TEST_IDS = []` runs all 25 emails. Put IDs in the list (for example `['E07','E22']`) to run a subset, and set `Limit Batch` to match.
6. Click **Execute workflow** once. A full run takes about 20 to 25 minutes because of the pacing Wait node. Do not run it twice on the same sheet, or you will get duplicate rows.

---

## 5. Results summary (v2, 25 emails)

| Outcome | Count | Emails |
|---|---|---|
| `reply_drafted` | 16 | E01-E05, E08-E11, E14-E16, E19, E21, E23, E24 |
| `held_for_human` | 7 | E07, E12, E13, E17, E18, E20, E22 |
| `logged_only` | 2 | E06 (spam), E25 (thank-you) |
| `needs_human = TRUE` | 9 | E07, E08, E12, E13, E17, E18, E20, E21, E22 |
| Tickets | 9 | 10 emails linked (E04 and E21 share `TKT-1052`) |

**v1 compared with v2** (details in [`TEST_REPORT.md`](TEST_REPORT.md)):

| | v1 | v2 |
|---|---|---|
| Intent accuracy | 23 / 25 | 25 / 25 |
| Risky emails sent to a human (8 clear cases) | 3 / 8 | 8 / 8 |
| Policy rules | Not built | In Code nodes |
| Reply step | Plain LLM chain | AI agent with 2 tools |

---

## 6. Known limitations (honest list)

- **Model confidence is not calibrated.** v2 reported 0.96 to 0.99 for almost every email. For the vague email E18, v1 gave 0.4 and v2 gives 0.96, so this got worse. The `needs_human` decisions are driven by the rule-based triggers, not by confidence.
- **Escalation replies are generic.** For held emails, the draft only says that the team will follow up (for example E13, E20). It does not, for instance, advise a dermatologist for the eczema email.
- **E05 was written in Hinglish, but the reply is in English.** The brand voice allows English by default, and v1 did reply in Hinglish, so this is a regression. E09 (Hindi) works, but its reply leaves out the 1-2 day dispatch time and has an English sign-off.
- **E21 and E23 replies ignore what the customer asked for.** E21 asks for a replacement and E23 for a refund only, but both replies offer "replacement or refund".
- **E11 wording is too strong.** The draft says the return "has been approved", although pickup and a quality check come first.
- **The final audit log is not a single run.** It comes from one full 25-email run plus later re-runs of E02, E05, E09, E10 and E20 after I added the product-question path and a safer paired-data lookup in `Clean Reply`. I did not do one more complete run after the last change.
- **The tickets sheet shows tickets that my earlier tests created.** Some of them were updated, not created, by the final run. Duplicate detection worked there (no second row, no repeated email ID).
- **Tokens and cost per email are not tracked in the sheet.** The cost numbers below are estimates.
- **Google Sheets as a ticket store** is fine for a demo but not safe under concurrent runs (two emails for the same order could race).
- **v1 only covered the order-status path end to end** (8 emails). Its classifier ran on all 25, so the comparison for the other 17 emails covers classification and the `needs_human` decision only.
- Manual trigger only. I did not build the webhook.

See [`TEST_REPORT.md`](TEST_REPORT.md), section 5, for each failure with a cause and a next step.

---

## 7. Scaling notes

- Replace the manual loop with a webhook or an email trigger, and put a queue in front so bursts do not hit rate limits.
- Use a paid LLM tier. The free tier handles about 3 emails a minute, and 1,000 emails a day needs roughly 40 per hour, with spikes.
- Replace `orders.json` with the real Shopify order API, with the same sender-verification rule.
- Move tickets to a real helpdesk or database with transactions, so duplicate detection cannot race.
- Log tokens, latency and cost per email, and track prompt version so each prompt change can be compared against the last one.
- Add monitoring: alert on parse failures, rate-limit errors and any held-email backlog older than 24 business hours.
- Keep secrets in n8n credentials and export workflows without them.

---

## 8. Product-thinking questions

**1. Auto-send vs. draft**
Today every reply is only a draft in the audit log; nothing is sent. If I enabled auto-send, I would start with two types: order-status replies to a verified sender, and product answers taken word for word from the FAQ. Both use facts already in the data and carry no money or privacy risk. Everything else stays human-approved: returns and refunds, complaints, sender mismatches, injection flags, medical wording, angry customers and unknown orders.
My confidence cut-off is 0.6, but I do not rely on it. v2 reported 0.96 to 0.99 for almost every email, even the vague E18 (v1 gave it 0.4), so the number is poorly calibrated. The rule-based triggers do the real work. For auto-send I would require confidence of at least 0.9 plus every code check passing, after two weeks in draft-only mode to measure how often agents edit drafts.

**2. Cost**
These are estimates, because I did not track tokens per email. A classification call is about 1,000 tokens. About 60% of emails also go through the agent, at roughly 2,500 tokens each (tool calls plus FAQ text). Average: 1,000 + 0.6 x 2,500 = 2,500 tokens per email. I assume a blended price of $0.30 per million tokens for a mid-size open model, to be checked against the current Groq price list.
- Per email: 2,500 x $0.30 / 1,000,000 = $0.00075, about Rs 0.07 at Rs 90 per dollar.
- Per month at 1,000 emails a day: 30,000 emails x 2,500 = 75 million tokens, about $22.50, or roughly Rs 2,000.
The free tier cannot carry this: 8,000 tokens per minute is about 3 emails a minute, and it also has a daily token cap.

**3. Success metric**
Draft acceptance rate: the share of drafts a human sends without edits, tracked per intent. It shows whether the copilot saves real time and exposes weak intents (I expect returns to score lowest). Week-1 target: above 70% on order-status and product replies, with every `needs_human` case reviewed within 24 business hours. I would switch it off, or fall back to log-only, if one reply leaks order details to the wrong person, if a refund or replacement promise goes out that policy does not allow, or if acceptance stays below 40% for two days in a row. The first two are trust failures, and one incident is enough.

**4. Failure experience**
Because replies are drafts, a person sees a wrong draft before the customer does. If one did go out, the customer would see a wrong date or status, an unneeded "we will follow up", or worst of all a refund promise we cannot honour. That is why hard rules sit in code: eligibility, privacy and refund amounts are computed by the workflow, injection-flagged emails get a fixed safe reply (found and fixed during E12 testing), and complaint replies use only approved wording. Damage is limited by the audit log (prompt version and timestamp on every row), a kill switch on the trigger, and a standard correction email that apologises, gives the right facts and names a person who will follow up.

**5. Don't automate**
Final approval of refunds and replacements, including the photo check on damaged, wrong or missing items. The copilot can collect photos, apply the policy and open a High priority ticket, but a person should look at the photos and confirm. It moves money and stock, fake or reused photos are a real fraud pattern, and a wrong approval costs more than a short delay.

---

## 9. How I used AI tools

I used Claude as an AI assistant throughout. It assisted with the n8n setup and the v1 order-status path (the classifier prompt, the Code nodes for parsing, order lookup and sender check, and rate-limit fixes) and with the v2 workflow (the classifier and agent prompts and the Code nodes for validation, policy check, ticket decision and injection guard). I set up and ran the workflow in n8n myself and checked the output in the Google Sheet. Claude also assisted with a final review of all 25 audit-log rows against `orders.json` and `faq.md` (dates, amounts, tracking links, refund figures), and wrote the first drafts of this README and the test report, which I reviewed and edited. I went through the main design decisions, such as keeping rules in code and holding every flagged email for a human, and chose to keep them.

---

## 10. Repository layout

```
.
├── README.md
├── TEST_REPORT.md
├── workflow/
│   └── kesar-copilot-v2.json
└── screenshots/
    ├── workflow.png
    ├── tickets.png
    └── audit_log.png
```
