# AI SDR System

**A phased build of an outbound SDR pipeline: source a lead, qualify it against an ICP rubric, spend enrichment credit only on the ones worth it, write the CRM record, then send, sequence and hand the reply to a human.**

Built on n8n with LLM agents under strict JSON-schema output, Apollo for sourcing, Clay for the email waterfall, and HubSpot as the system of record.

> **Status:** Phase 1 complete and running end to end. Phases 2 and 3 specified, not yet built — see [Roadmap](#roadmap). Client data, credentials, and prompts identifying any client are not in this repository.

---

## The requirement

The business requirement this system is built against, stated in full:

> When a new lead enters the pipeline, research the company, determine whether it fits our ICP, personalise an email, send it, follow up automatically, update the CRM, and notify the sales representative when the lead responds.

Six obligations. Phase 1 delivers four of them. The README says plainly which four, because an SDR system that quietly doesn't send is worse than one that says so.

| # | Obligation | Phase | Status |
| --- | --- | --- | --- |
| 1 | Research the company | 1 | Built |
| 2 | Determine ICP fit | 1 | Built |
| 3 | Personalise an email | 1 | Built |
| 4 | Update the CRM | 1 | Built |
| 5 | Send it, and follow up automatically | 2 | Specified |
| 6 | Notify the rep when the lead responds | 3 | Specified |

---

## Architecture

```
PHASE 1 — SOURCE, QUALIFY, ENRICH, RECORD                              [BUILT]

  Schedule (2d)
       │
       ▼
  ┌──────────────────┐
  │ Get Page Cursor  │  n8n Data Table — runs resume where they left off
  └────────┬─────────┘
           ▼
   ┌───────────────┐   yes   ┌──────────────────────────┐
   │ Pages         ├────────►│ Alert: search exhausted   │
   │ exhausted?    │         └──────────────────────────┘
   └───────┬───────┘ no
           ▼
  ┌─────────────────────┐  err  ┌──────────────────┐
  │ Apollo People Search├──────►│ Log: Apollo Error │
  └────────┬────────────┘       └──────────────────┘
           ▼
  ┌──────────────────────┐
  │ Advance Page Cursor  │  ◄── advances on a successful FETCH, not a successful push
  └────────┬─────────────┘
           ▼
  ┌──────────────────┐   ┌───────────────────┐   ┌──────────────────────────┐
  │ Split Out People ├──►│ Clean Profile     ├──►│ Skip Already-Seen Leads   │
  └──────────────────┘   │ (trim for tokens) │   │ (dedupe on apollo_id)     │
                         └───────────────────┘   └────────────┬─────────────┘
                                                              ▼
                                              ┌────────────────────────────┐  err  ┌────────────────────┐
                                              │ Lead Scoring Agent         ├──────►│ Log: Scoring Error │
                                              │ exclusions → 5-part rubric │       └────────────────────┘
                                              └──────────────┬─────────────┘
                                                    ── Score SOP (strict JSON, auto-fix)
                                                             ▼
                                                    ┌────────────────┐
                                                    │  score >= 5 ?  │
                                                    └───┬────────┬───┘
                                                   no   │        │  yes
                                                        ▼        ▼
                                          ┌─────────────────┐  ┌──────────────────────┐  err  ┌──────────────────┐
                                          │ Log: Rejected   │  │ Push to Clay Webhook ├──────►│ Log: Clay Error  │
                                          │ Lead            │  └──────────┬───────────┘       └──────────────────┘
                                          └─────────────────┘             │
                                                                          ▼
                                                        Clay: email waterfall + firmographics
                                                                          │
                                                          POST /clay-enriched (header-authed)
                                                                          ▼
  ┌──────────────────┐   no email   ┌──────────────────┐
  │ Email Found?     ├─────────────►│ Respond: Skipped │
  └────────┬─────────┘              └──────────────────┘
           ▼ verified email
  ┌────────────────────┐  err  ┌──────────────────────┐
  │ Generate Outreach  ├──────►│ Log: Pipeline Error   │
  │ email · LinkedIn   │       └──────────┬───────────┘
  │ msg · call script  │                  ▼
  └────────┬───────────┘         ┌──────────────────┐
     ── Outreach SOP             │ Respond: Failed  │  HTTP 500 — Clay never hangs
           ▼                     └──────────────────┘
  ┌──────────────────────┐
  │ HubSpot: Upsert      │  score · tier · rationale · copy → custom properties
  └────────┬─────────────┘
           ▼
  ┌──────────────────┐
  │ Respond: Synced  │
  └──────────────────┘


PHASE 2 — SEND & SEQUENCE                                          [SPECIFIED]

  suppression check → send → sequence state table → day 3 / 7 / 14 follow-ups


PHASE 3 — REPLY & HANDOFF                                          [SPECIFIED]

  inbox trigger → classify reply → halt sequence → update lifecycle → Slack the rep
```

---

## Phase 1 — what is built

### Workflow 01 · Source, Score & Gate

**`01-apollo-to-clay.json`**

Apollo People Search replaces a Google CSE + scraper approach: one API call gives search and profile data together, with no pagination dorking and no scraping fragility.

- **Cursor-persisted paging.** The page number lives in an n8n Data Table. Each run reads the last row, asks Apollo for *that* page, and writes the next one immediately after a successful fetch. The cursor deliberately advances on a successful **fetch**, not a successful **push** — otherwise a page where nothing qualifies traps the workflow on the same ten people forever.
- **Cross-execution deduplication** on `apollo_id`, so a re-run never re-scores or re-enriches someone already in the pipeline. This is a cost control as much as a correctness one: Clay credits are not spent twice on the same person.
- **Payload trimming before scoring.** Apollo's person object is cut down to the fields the rubric actually reads. Smaller payload means cheaper tokens and less noise, and less noise means more consistent scores.
- **Hard exclusions run before the rubric.** Over 100 employees, individual contributor, or not software — score zero, stop. Ask a model whether something is a good lead and it will find a reason to say yes; the exclusions are what stop it being agreeable.
- **Five-dimension rubric out of 10**, with a requirement that the rationale cite specific profile facts. A score you cannot audit is a score you cannot trust.
- **The gate sits before enrichment.** Only leads at 5.0 or above are pushed to Clay. Everything below is written to a rejected-leads table, not dropped — the rubric can only be tuned against the leads it turned down.

### Workflow 02 · Enrich Callback, Outreach & CRM

**`02-clay-to-hubspot.json`**

Clay owns the email waterfall and POSTs each enriched row back when it finishes. The callback is asynchronous by design: n8n never blocks waiting on enrichment, and Clay stays the single source of truth for enrichment logic.

- **Gate on a non-empty verified email** before generating anything. An unreachable contact in the CRM pollutes every report downstream of it.
- **Four outreach artifacts** per qualified lead: a profile summary, a cold email, a LinkedIn connection message, and a 30-second call script — generated under hard constraints against inventing a mutual connection, implying prior contact, manufacturing scarcity, or padding a thin profile with invention. Those constraints exist because the first version did all four.
- **HubSpot upsert by email**, carrying score, tier, rationale, generated copy and `apollo_id` as custom properties. The `apollo_id` makes every CRM record traceable back to the pipeline run that created it.

---

## Reliability

The part that separates a demo from something you can leave running.

| Failure | Handling |
| --- | --- |
| Model returns prose instead of JSON | Structured output parser with `autoFix` — reprompts with the schema violation |
| Model returns malformed JSON | Same, bounded at 5 retries |
| Transient API failure (Apollo, Clay, HubSpot) | `retryOnFail`, 3–5 tries |
| Persistent node failure | `onError: continueErrorOutput` routed to an error-log table with the `apollo_id` for replay — the batch continues |
| Lead scores below threshold | Written to a rejected-leads table with its rationale, not silently dropped |
| A page returns nothing qualifying | Cursor still advances; the workflow cannot deadlock on one page |
| Apollo search space exhausted | Email alert to the operator; the run stops instead of looping |
| Same lead appears in a later page or re-run | Deduplicated on `apollo_id` before any spend |
| Clay callback fails mid-pipeline | Every branch reaches a Respond node — an unanswered path would leave Clay hanging until timeout |

A batch of 200 profiles where profile 37 has a mangled education array should still deliver 199 scored leads. That is the standard.

## Security

- **No credentials in the workflow body.** Apollo's `x-api-key` is an n8n Header Auth credential; every export carries credential IDs only.
- **The Clay callback endpoint is header-authenticated.** Without it, anyone who learns the URL can write contacts straight into the client's CRM.
- **Outbound bodies are built as objects**, not string-concatenated JSON. An LLM rationale containing a newline or a quote is not a parse error waiting to happen.

## On data sources

Where profiles come from is a decision with legal weight. This system sources from Apollo under its terms; legitimate alternatives include inbound forms, your own CRM, licensed providers with consent to contact, and opt-in lists you own. Scraping social platforms generally violates their terms of service, and processing personal data for unsolicited commercial contact carries obligations that vary by jurisdiction. The engineering here is the qualification, generation and delivery layer — plug in a source you are entitled to use.

---

## Roadmap

### Phase 2 · Send & Sequence

The system currently writes copy into HubSpot and stops. A human still opens the record and sends. Phase 2 closes that.

- **Suppression check before every send**: already contacted, unsubscribed, competitor domain, existing customer. One list, checked once, no exceptions.
- **Send via a transactional email API**, with a daily cap so a runaway loop cannot burn the sending domain, and a kill switch that pauses all sends without editing a workflow.
- **Sequence state table** keyed on `apollo_id`: `sequence_step`, `next_send_at`, `status`. A scheduled workflow wakes, claims what is due, and sends it.
- **Three touches at day 3, 7 and 14**, each re-checking reply status immediately before firing.
- **Idempotent sends.** The state table is the guard: a lead at step 2 cannot receive step 2 twice, whatever the scheduler does.

### Phase 3 · Reply & Handoff

- **Inbox trigger** scoped to the sending thread.
- **Reply classification** under the same schema-enforced pattern as scoring: interested / not interested / out of office / unsubscribe / other. Out-of-office and auto-responders must not count as replies — a sequence that halts on a vacation bounce is a sequence that quietly stops working in August.
- **Halt the sequence, update the HubSpot lifecycle stage**, and honour unsubscribes at the suppression list.
- **Slack the owning rep** with the reply, the original email, the score and the rationale, so they open the conversation already knowing why this person was contacted.

### Phase 4 · Operations

- Per-run token and cost logging; cost per qualified lead as a reported metric.
- Replay endpoint over the error-log table.
- Weekly operator report: volume sourced, qualification rate, enrichment hit rate, send and reply rates, cost per lead.

---

## Running it

```bash
# 1. Self-host n8n
docker run -it --rm -p 5678:5678 -v n8n_data:/home/node/.n8n n8nio/n8n

# 2. Import 01-apollo-to-clay.json and 02-clay-to-hubspot.json
```

Then fill the placeholders:

| Placeholder | What it is |
| --- | --- |
| `REPLACE_WITH_YOUR_APOLLO_HEADER_AUTH_CREDENTIAL_ID` | n8n Header Auth credential holding `x-api-key` |
| `REPLACE_WITH_YOUR_CLAY_WEBHOOK_URL` | Clay webhook source URL |
| `REPLACE_WITH_YOUR_WEBHOOK_HEADER_AUTH_CREDENTIAL_ID` | Shared secret for the Clay callback |
| `REPLACE_WITH_YOUR_OPENAIAPI_CREDENTIAL_ID` | OpenAI credential |
| `REPLACE_WITH_YOUR_HUBSPOTAPPTOKEN_CREDENTIAL_ID` | HubSpot private app token |
| `REPLACE_WITH_REJECTED_LEADS_TABLE_ID` | Data Table for sub-threshold leads |
| `REPLACE_WITH_ERROR_LOG_TABLE_ID` | Data Table for failed executions |
| `REPLACE_WITH_ALERT_RECIPIENT_EMAIL` | Operator alert address |

You will also need a cursor Data Table with a numeric `start_page` column, seeded with a single row at `1`, and the HubSpot custom properties matching the names in the upsert node.

---

## Repo contents

```
01-apollo-to-clay.json      Workflow 01 — source, score, gate (23 nodes)
02-clay-to-hubspot.json     Workflow 02 — enrich callback, outreach, CRM (12 nodes)
README.md
```

---

**Hinata Kashif** · [LinkedIn](https://www.linkedin.com/in/hinata-kashif) · hinatakashif05@gmail.com
