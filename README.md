# AI Lead Generation Pipeline (n8n + Apollo + Clay + HubSpot)

An end-to-end outbound pipeline that sources leads, scores them against an ICP with an LLM, enriches only the qualified ones, generates personalised outreach, and syncs everything into HubSpot.

Built in n8n. Two workflows, connected by an async webhook loop.

---

## Why it is built this way

The design goal was to spend enrichment credits and CRM records only on leads worth having. Most outbound stacks enrich everything and filter later, which is expensive and pollutes the CRM.

Three decisions do most of the work:

**Score before enriching.** An LLM scores each lead 0 to 10 against the ICP rubric inside n8n, where the prompt is cheap to run and version-controlled. Only leads scoring 5 or above are pushed to Clay. Clay credits are never spent on leads that would fail qualification anyway.

**Async handoff instead of polling.** Clay owns the email waterfall and firmographic enrichment. When it finishes, its HTTP API column POSTs the enriched row back to a webhook in Workflow 02. n8n never sits waiting on enrichment, and Clay stays the single source of truth for enrichment logic.

**Gate again on verified email.** Only leads with a found and verified email get outreach copy generated and a HubSpot record. Unreachable contacts never enter the CRM, so pipeline reporting stays honest.

---

## Architecture

```
Workflow 01: Apollo to Clay
  Schedule trigger
    -> read page cursor from n8n Data Table (runs resume where they left off)
    -> Apollo People Search API
    -> split into individual people
    -> clean profile payload for the LLM
    -> LLM scoring agent (structured output, auto-fix on)
    -> gate: score >= 5
    -> POST qualified leads to Clay webhook source
    -> advance page cursor

Workflow 02: Clay to HubSpot
  Webhook receives enriched row from Clay
    -> gate: verified email present?
       no  -> respond "skipped", stop
       yes -> LLM generates outreach package
              (profile summary, cold email, LinkedIn message, call script)
           -> upsert into HubSpot with score, tier and rationale as custom properties
           -> respond "synced"
```

---

## Engineering details worth noting

**Stateful pagination.** The Apollo page cursor is persisted in an n8n Data Table rather than held in memory, so a failed or restarted run resumes from the correct page instead of re-processing leads and burning API calls.

**Structured output with auto-fix.** Both LLM agents use a JSON schema output parser with `autoFix` enabled and retries configured. If the model returns malformed JSON, it is re-prompted rather than crashing the run or writing garbage downstream.

**Error handling on the AI steps.** The scoring and outreach agents are set to `continueErrorOutput` with retries, so one bad lead does not halt the batch.

**Both branches respond.** The webhook responds on the skip path as well as the success path, so Clay always receives a status instead of timing out on leads that were filtered.

**Sourcing via API, not scraping.** Apollo's People Search replaced an earlier Google CSE plus scraper setup. One API call returns search results and profile data together, removing scraping fragility and pagination workarounds.

---

## Stack

| Layer | Tool |
|---|---|
| Orchestration | n8n |
| Sourcing | Apollo People Search API |
| Scoring and copy generation | OpenAI (GPT-4.1-mini) via n8n LangChain nodes |
| Enrichment and email waterfall | Clay |
| CRM | HubSpot |
| State | n8n Data Tables |

---

## Running it yourself

1. Import both JSON files into your n8n instance.
2. Create credentials for Apollo, OpenAI, Clay and HubSpot. Each node has a placeholder where the credential ID goes.
3. Replace `YOUR_CLAY_WEBHOOK_ID` in Workflow 01 with your own Clay webhook source URL.
4. In Clay, add an HTTP API column that POSTs enriched rows to the Workflow 02 webhook URL.
5. Create the custom properties in HubSpot: `lead_score`, `lead_tier`, `scoring_rationale`, `profile_summary`, `generated_email_subject`, `generated_email_body`, `linkedin_message`, `call_script`.
6. Set the schedule trigger to your preferred cadence and activate.

The ICP rubric and outreach prompts are inside the agent nodes. Rewrite them for your own offer before running.

---

## Note

Credentials, instance identifiers and live webhook URLs have been stripped from these exports. Replace the placeholders with your own before use.
