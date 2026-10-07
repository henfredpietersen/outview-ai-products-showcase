# Engineering notes

The design decisions I'd walk through in a technical interview, with the trade-off behind each.

## 1. Graceful degradation is the default

| Failure | What happens instead |
|---|---|
| Primary LLM provider errors or hits a quota | Same prompt retried on the fallback provider |
| Every provider unreachable | Answer built directly from the retrieved site content |
| Semantic search returns nothing / no embedding API | Keyword retrieval, which needs no external API |
| Booking request hits a provider failure | Booking details saved as a lead for the owner |
| Licence server unreachable | 7-day grace period; afterwards only admin screens lock, never visitor chat |
| Host lacks PHP `imap` or `openssl` | Email sync shows "not available"; encryption becomes a no-op, with no fatal errors |

**Why:** these products run on other people's cheap shared hosting with free-tier API keys. A support channel that silently dies loses the customer's leads, which is worse than a slightly worse answer.

## 2. One source of truth per rule

- One **capability map** decides tier access for 37 features; nothing else compares plan names
- One **chat handler** serves the web widget and all five messaging channels, so a fix lands everywhere
- One **segment → SQL builder** with a field/operator whitelist; no screen builds its own SQL from user filters
- One method marks an invoice paid, which makes duplicate gateway webhooks idempotent

## 3. Keep decision logic pure so it can be tested

Logic such as the flow-builder state machine, offline-answer excerpting, source-health diagnostics, brand-voice prompt building, Slack signature checks (which take "now" as a parameter) and automation wait maths is written without WordPress calls, so it can be unit-tested without a live site or WordPress mocks. The CRM's React admin has 37 Vitest test files.

## 4. Verify everything that comes in from outside

| Inbound | Check |
|---|---|
| Slack Events API | v0 HMAC signature with timestamp |
| Meta (WhatsApp, Messenger, Instagram), Telegram, SMS | Signature/secret verification with constant-time comparison |
| Paystack | HMAC-SHA512 of the raw body |
| Yoco | HMAC-SHA256 webhook signature over id, timestamp and body |
| PayFast | Ordered-parameter signature, then a server-to-server validation call |
| Public widget REST routes | WordPress REST nonce, plus per-conversation rate limiting |

## 5. Don't spend API quota where a rule will do

The intent tags in analytics come from a keyword classifier, not an LLM: instant, free and good enough for a breakdown chart. CRM AI insights run only when a rep clicks. Embeddings are created at index time, not per question.

## 6. Loose coupling between my own products

The CRM pulls chatbot leads by reading the chatbot's table and calling the chatbot's own public status API, rather than patching the chatbot to emit events. Either plugin can be updated, or removed, without breaking the other.

## 7. Built for the South African market

IMAP over Gmail-only OAuth (local hosting providers), PayFast/Paystack/Yoco, BulkSMS, WhatsApp as a first-class channel, POPIA erasure flows, all 11 official languages in the chatbot, and free-tier LLMs (Gemini, Groq) as recommended defaults to keep running costs near zero.

## What I'd improve next

- An offline retrieval evaluation set tracked across releases (Hit@k, MRR), as in [policy-assistant](https://github.com/henfredpietersen/policy-assistant)
- Hybrid retrieval that blends keyword and vector scores instead of falling back from one to the other
- CI that runs the PHP unit tests and the Vitest suite on every push
