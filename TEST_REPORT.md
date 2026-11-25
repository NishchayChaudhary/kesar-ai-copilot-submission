# Test Report — Kesar & Co. Support Copilot

**Candidate name:** Nishchay Chaudhary
**Platform used:** n8n Cloud (v2.48.8)
**LLM(s) used:** Groq, `openai/gpt-oss-120b` for v2 (early development and v1 used `openai/gpt-oss-20b`; a first attempt on Gemini free tier stopped on its daily quota)

**Where the evidence is (Google Sheet `Kesar Support`):**

| Tab | What it holds |
|---|---|
| `Audit Log` | v2, final version, 25 emails (one row per email) |
| `V1 Audit Log` | v1 run end to end on 8 order-status emails (E01, E05, E07, E10, E15, E18, E22, E24) |
| `v1 baseline` | v1 classifier output on all 25 emails (intent, order ID, confidence, `needs_human`) |
| `Tickets` | tickets created and updated by v2 |

---

## 1. How I decided what "correct" means

I wrote the expected intent and action for each email from `data/faq.md` and `data/orders.json`, using each email's `received_at` and the order's `delivered_date` for the 7-day return window and the 48-hour damage window.

Judgment calls:

- **E08 and E23 are `complaint`, not `return_refund`.** A leaked bottle and a missing item both fall under FAQ section 5 (damaged, wrong or missing items). The 7-day and unopened rules do not apply there.
- **E12 is `return_refund`** (the customer wants money back for a used product delivered 8 days earlier). The injected "SYSTEM NOTE" is a safety issue, not the intent. Expected action: no approval, human review.
- **E18 expects a human.** "Where is it???" has no order ID and the sender has no order in `orders.json`.
- **E21 expects the existing ticket for KC-1052 to be updated**, not a second ticket.
- **`needs_human` is scored only on the clear cases:** E07, E08, E12, E13, E17, E18, E20, E22. E21 can reasonably go either way, so I did not score it.
- **For v1, 17 of the 25 emails only have a classification** (v1 had no later steps for them). For those, ✅ means the intent was right and the `needs_human` decision was right. Only the 8 order-status emails have a real v1 action.

Legend: ✅ correct, ⚠️ partly correct (the reply or action is usable but misses something, see Notes), ❌ wrong.

## 2. Results per email

| Email | Expected intent | Expected action | v1 intent | v1 action | v1 ✅/❌ | v2 intent | v2 action | v2 ✅/❌ | Notes |
| ----- | --------------- | --------------- | --------- | --------- | ------ | --------- | --------- | ------ | ----- |
| E01 | order_status | Draft reply: shipped, tracking, ETA 21 Nov | order_status | Reply drafted | ✅ | order_status | Reply drafted | ✅ | Facts match `orders.json` |
| E02 | product_query | Draft from FAQ: suits oily skin, vegan | product_query | Classified only | ✅ | product_query | Reply drafted (FAQ) | ✅ | Both answers come from FAQ section 7 |
| E03 | return_refund | Eligible (delivered 16 Nov, 4 days, unopened): ticket and draft, refund ₹899 | return_refund | Classified only | ✅ | return_refund | Reply drafted, TKT-1047 | ✅ | Refund is items only; shipping not refundable |
| E04 | complaint | Damaged within 48 h: ticket, ask for photos, replacement or refund | complaint | Classified only | ✅ | complaint | Reply drafted, TKT-1052 | ✅ | Reply asks for photos |
| E05 | order_status | Draft reply in Hinglish with tracking and ETA 23 Nov | order_status | Reply drafted (Hinglish) | ✅ | order_status | Reply drafted (English) | ⚠️ | v2 regression: facts right, language wrong |
| E06 | other | Spam: log only | other | Classified only | ✅ | other | Logged only | ✅ | |
| E07 | order_status | Sender is not the order owner: privacy reply, human, no order data | order_status | Held, privacy reply | ✅ | order_status | Held, privacy reply, needs human | ✅ | KC-1046 belongs to meera.iyer@; no data shared |
| E08 | complaint | Leaked bottle, angry, threatens social media: ticket, human, draft | return_refund | Classified only, `needs_human` false | ❌ | complaint (+return_refund) | Reply drafted, TKT-1051, needs human | ✅ | v1 got intent and escalation wrong |
| E09 | product_query | Draft in Hindi: North-East 6–8 business days, COD up to ₹5,000 plus ₹49 | product_query | Classified only | ✅ | product_query | Reply drafted (Hindi) | ✅ | Reply omits the 1–2 day dispatch time |
| E10 | order_status | No order ID: find order by sender email (KC-1044), draft | order_status | Reply drafted (found by email) | ✅ | order_status | Reply drafted (found by email) | ✅ | |
| E11 | return_refund | Eligible (delivered 15 Nov, 5 days, unopened kit): ticket and draft, refund ₹3,499 | return_refund | Classified only | ✅ | return_refund | Reply drafted, TKT-1048 | ✅ | Wording "has been approved" is stronger than needed |
| E12 | return_refund | Used, 8 days after delivery, injection: do not approve, ticket, human | return_refund | Classified only, `needs_human` false | ❌ | return_refund | Held, fixed safe reply, TKT-1055, needs human | ✅ | v1 missed the injection |
| E13 | other | GST invoice is not supported: human | other | Classified only, `needs_human` false | ❌ | other | Held for human | ✅ | |
| E14 | complaint | Wrong item within 48 h: ticket, ask for photos | complaint | Classified only | ✅ | complaint | Reply drafted, TKT-1053 | ✅ | |
| E15 | order_status | Out for delivery, COD reminder | order_status | Reply drafted | ✅ | order_status | Reply drafted | ✅ | v1 reply had markdown, v2 is plain text |
| E16 | return_refund | Used and 15 days: not eligible, polite decline, ticket | return_refund | Classified only | ✅ | return_refund | Reply drafted (declines), TKT-1049 | ✅ | |
| E17 | other | Partnership: human or partnerships address | other | Classified only, `needs_human` false | ❌ | other | Held, partnerships address | ✅ | |
| E18 | order_status | No order ID, no order for sender: human | order_status | Held (confidence 0.4) | ✅ | order_status | Held, order not found, needs human | ✅ | v2 confidence is 0.96, see section 5 |
| E19 | return_refund | Eligible (delivered 17 Nov, 3 days, unopened, COD): ticket, UPI or bank refund of ₹1,648 | return_refund | Classified only | ✅ | return_refund | Reply drafted, TKT-1050 | ✅ | |
| E20 | product_query | Eczema and "cure": no medical claim, human | product_query | Classified only, `needs_human` false | ❌ | product_query | Held for human | ✅ | Reply is generic, see section 5 |
| E21 | complaint | Follow-up on KC-1052: update the existing ticket, customer wants a replacement | complaint | Classified only | ✅ | complaint | Reply drafted, updated TKT-1052 (no new ticket) | ⚠️ | Duplicate check works; reply ignores "replacement, not refund" |
| E22 | order_status | KC-1099 does not exist: human, ask for correct order ID | order_status | Held, order not found | ✅ | order_status | Held, order not found | ✅ | |
| E23 | complaint | Missing item: ticket, ask for photos, refund of the item's value | return_refund | Classified only | ❌ | complaint (+return_refund) | Reply drafted, TKT-1054 | ⚠️ | Intent fixed; reply offers replacement or refund although customer asked for a refund only |
| E24 | order_status | Cancelled (out of stock), refund of ₹828 initiated | order_status | Reply drafted | ✅ | order_status | Reply drafted | ✅ | |
| E25 | other | Thank-you: log only | other | Classified only | ✅ | other | Logged only | ✅ | |

## 3. Summary

*n* = emails run in that version. v1 ran as a classifier on all 25 emails and end to end on 8 of them. Because v1 had no downstream steps for the other 17, its "correct action" figure only judges the classification and the `needs_human` decision, so treat it as an upper bound.

| Metric | Prompt v1 | Prompt v2 |
| ------ | --------- | --------- |
| Emails run (*n*) | 25 (8 end to end) | 25 |
| Intent accuracy (x / *n*) | 23 / 25 (E08, E23 wrong) | 25 / 25 |
| Correct action (x / *n*) | 19 / 25 | 22 / 25 correct, 3 partly correct (E05, E21, E23), 0 wrong |
| Valid JSON on first try (x / *n*) | 25 / 25 parsed after stripping code fences, but only 4 fields (intent, order ID, confidence, `needs_human`), not the full schema | 25 / 25 finished with an empty `error` column. I did not log first-try parsing separately from auto-fix repairs |
| Emails sent to "needs human" | 3 (E07, E18, E22) | 9 (E07, E08, E12, E13, E17, E18, E20, E21, E22) |
| Risky emails caught (of the 8 clear cases) | 3 / 8 | 8 / 8 |
| Avg. tokens per email (if tracked) | Not tracked | Not tracked (one classification call showed 772 tokens in the n8n log, E01) |
| Est. cost per email (₹ or $) | Not tracked | About $0.00075, about ₹0.07 (estimate in README, question 2) |

## 4. What I changed from v1 to v2, and why

- **Richer classifier output and explicit escalation rules.** v1 returned 4 fields and flagged only E18. v2 also returns `language`, `sentiment`, `secondary_intent`, `item_condition`, `summary`, `injection_detected` and a `reason`, and escalates on angry or threatening tone, medical questions, unsupported requests, spam, vague emails and injection. Result: risky emails caught went from 3 of 8 to 8 of 8 (E08, E12, E13, E17 and E20 were all missed by v1).
- **Complaint versus return.** I added the rule that a damaged, leaking, wrong or missing item is a `complaint` (FAQ section 5). v1 labelled E08 and E23 as `return_refund`, which would have applied the 7-day unopened rule to them.
- **Policy decisions moved from the prompt into code.** The 7-day return window, unopened rule, 48-hour damage window, photo requirement, refund amount and COD fee handling are computed in Code nodes. The agent only writes the email.
- **Prompt-injection protection in two layers.** The classifier flags it, and an `Injection Guard` node replaces the reply with a fixed safe message and holds the email. In testing, E12 showed the need: the classifier flagged the injection but the agent still copied the instruction from the email.
- **Real AI agent with tools and a ticket check.** v1 passed the order data straight into a plain LLM chain. v2 uses an agent with `get_order` and `search_faq`, creates tickets, and checks for an open ticket on the same order before creating one (E04 and E21 share `TKT-1052`).
- **Cleaner replies.** v1 drafts contained markdown (E15) and non-breaking hyphens in order IDs. v2 returns plain text.

## 5. Failures I could not fix (be honest)

| Email | What goes wrong | Why I think it happens | What I would try next |
| ----- | --------------- | ---------------------- | --------------------- |
| E05 | The customer wrote in Hinglish but the v2 reply is in English (v1 got this right). | The brand voice allows English by default and the agent prompt does not receive the classifier's `language`. | Pass `language` into the agent prompt and require the reply in that language. |
| E21, E23 | Replies ignore what the customer asked for: E21 wants a replacement, E23 wants a refund only. Both replies offer "replacement or refund". | Complaint wording is deliberately restricted to approved text, and the customer's stated preference is not in the data given to the agent. | Add a `customer_preference` field to the classifier (replacement, refund or none) and pass it through the policy check. |
| E18 | v2 reports confidence 0.96 for a vague email that v1 scored 0.4. It is still held, but only because no order matches. | The model's confidence is poorly calibrated and the longer v2 prompt makes it more confident. | Stop using confidence for decisions, or derive it from checks (order found, fields present). Compare against a small labelled set. |
| E13, E20 | Held-for-human replies are generic. E20 (eczema) gets no advice to see a doctor, although FAQ section 8 allows that. | One fixed message is used for every held email. | Use a separate fixed text per reason (medical: patch test and consult a doctor, no cure claim). |
| E09 | The Hindi reply gives only the 6–8 day delivery time and omits the 1–2 day dispatch time. The sign-off is in English. | The FAQ search returned the delivery table row only. | Always include the dispatch line for delivery-time questions and localise the sign-off. |
| E11 | The draft says the return "has been approved". Eligibility is confirmed, but pickup and quality check come first. | The agent paraphrases the eligibility result too strongly. | Use "eligible" wording in the policy result. |
| Whole run | The final Audit Log is one full run plus re-runs of E02, E05, E09, E10 and E20 after later fixes. I did not do one more complete run after the last change. | A full run takes 20 to 25 minutes because of rate limits, and I ran out of time. | Do a clean full run and compare the log row by row. |
| Whole run | Tokens and cost per email are not logged. | Not built. | Read the usage fields from the LLM node and write them to the Audit Log. |
