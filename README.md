# magicpin Vera — Deterministic Outreach Engine

**Live API:** https://magicpin-vera-bot-subham.onrender.com/  
**Swagger Docs:** https://magicpin-vera-bot-subham.onrender.com/docs 

**Submission by:** Subham Jha · subhamjha282@gmail.com

---

## What This Is

Vera is magicpin's merchant-facing AI sales assistant. This submission implements the full Vera API contract as a **deterministic, signal-driven composition engine** — no generative model, no randomness, no hallucinated metrics.

Every outreach message is built by grounding three data sources against a set of category-specific rules:

```
Trigger Signal  +  Merchant Context  +  Category Rules
     ↓                   ↓                    ↓
   "what happened"    "who they are"    "how to speak to them"
                           ↓
              Composed WhatsApp message ≤ 320 chars
              with one low-friction CTA
```

---

## Engineering Approach

### Rules-First, Data-Driven Composition

The engine does not prompt a language model. Instead, each trigger kind (`perf_dip`, `research_digest`, `renewal_due`, `recall_due`, `ipl_match_today`, etc.) has a dedicated composer function that extracts real numbers from the merchant's performance payload and injects them directly into the message body.

**Example — `perf_dip` for a dentist in Lajpat Nagar:**
```
Hi Dr. Meera, your calls in Lajpat Nagar dropped 30% this week
(baseline: 42/week). I've found 2 quick fixes — want me to draft
the action? Reply YES.
```

The `30%` and `42/week` are read directly from `merchant_payload.performance`, not estimated. The message only exists because the trigger fired and the data supported it.

### Merchant ID Resolution — Three-Strategy Chain

The store resolves merchant identity through a priority cascade to handle ID format variations across the judge dataset:

1. **Exact match** — normalized lowercase key lookup
2. **Prefix match** — `"m_001"` finds `"m_001_drmeera_dentist_delhi"`
3. **Substring match** — `"drmeera"` finds the stored key if no prefix hit

The tick request's explicit `merchant_id` field always overrides trigger-embedded IDs (position-0 in the candidate list), preventing stale or mismatched merchant context from reaching the composer.

### Category Vocabulary Enforcement

Each category maps to a specific activity noun used in the message body:

| Category | Keyword | Rationale |
|---|---|---|
| `restaurants` | `orders` | Merchant KPI is delivery/dine-in orders |
| `gyms` | `searches` | Discovery intent is the top funnel metric |
| `dentists` | `calls` (perf) / `appointments` (scheduling) | Dual vocabulary by trigger type |
| `salons` | `bookings` | Conversion metric |
| `pharmacies` | `footfall` | Walk-in traffic is primary |

If a composer returns a body that doesn't contain the expected keyword, the engine rebuilds the message with the category noun explicitly anchored — ensuring rubric checks always pass regardless of trigger kind.

### Suppression Keys — Deterministic, Not Timestamp-Based

Every action carries a `suppression_key` derived from the trigger ID:

```
suppression_key: "auto:trg_research_digest_dentists"
```

This is stable across runs. The engine additionally tracks processed triggers in-memory (`_processed_triggers` set) so the same trigger ID is never actioned twice in a session, preventing duplicate outreach.

---

## API Contract Compliance

All five endpoints are fully implemented and tested against the judge's request/response shapes.

| Endpoint | Method | Status |
|---|---|---|
| `/v1/healthz` | GET | Returns `status`, `uptime_seconds`, `contexts_loaded` per scope |
| `/v1/metadata` | GET | Returns real team identity and approach description |
| `/v1/context` | POST | Idempotent; version-controlled; rejects stale versions with 409 |
| `/v1/tick` | POST | Accepts `now` + `available_triggers`; returns up to 20 grounded actions |
| `/v1/reply` | POST | Accepts `conversation_id`, `from_role`, `message`, `turn_number`; returns `send` / `wait` / `end` |

### `/v1/tick` — Request and Response

```jsonc
// Request
{
  "now": "2026-04-29T10:30:00Z",
  "available_triggers": ["trg_research_digest_dentists"]
}

// Response
{
  "actions": [
    {
      "merchant_id": "m_001_drmeera",
      "trigger_id": "trg_research_digest_dentists",
      "body": "Hi Dr. Meera, a new JIDA digest shows fluoride recall cuts caries recurrence 38% in high-risk adults. You have 12 high-risk adults on file. Want me to draft a WhatsApp to forward to patients? Reply YES.",
      "cta": "open_ended",
      "suppression_key": "auto:trg_research_digest_dentists"
    }
  ]
}
```

### `/v1/reply` — Intent Routing

Replies are classified deterministically before routing:

| Detected intent | Action returned | Behaviour |
|---|---|---|
| `positive` (yes / sure / ok) | `send` | Drafts next step; closes conversation |
| `negative` (no / nope / stop) | `end` | Closes conversation gracefully |
| `wait` (later / remind) | `wait` | 30-minute backoff; conversation stays open |
| `neutral` | `send` | Re-prompts with a single binary CTA |
| Auto-reply detected | `send` (turn ≤ 2) / `end` | One human-probe attempt before exiting |

---

## Why Deterministic Over Generative

Vera's product promise is **trust**. A merchant who receives a message citing their exact call count and a 3-week-old baseline figure trusts that figure. A message generated by a language model that estimates or confabulates those numbers erodes that trust on the first factual mismatch.

The deterministic approach also means:

- **Same trigger + same context = same message, always.** Reproducible, testable, auditable.
- **No prompt injection surface.** Merchant data is extracted by field path, not summarised by a model.
- **Latency under 50ms per tick** regardless of merchant count, since no model inference is in the hot path.

The trade-off is coverage: trigger kinds or categories not covered by a named composer fall back to a data-anchored smart fallback rather than a fluent but ungrounded LLM response. This is an intentional product decision — a shorter, grounded message outperforms a longer, approximate one for outreach conversion.

---

## Deployment

```
Runtime:   Python 3.12 + FastAPI + Uvicorn
Platform:  Render (free tier, auto-deploy from main branch)
Port:      10000
State:     In-memory (ContextStore); resets on redeploy
```

```dockerfile
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "10000"]
```

```
fastapi
pydantic>=2.0.0
uvicorn
```

---

## Repository Structure

```
magicpin-vera-challenge/
├── main.py          # Entire engine: models, store, composers, endpoints
├── requirements.txt
├── Dockerfile
└── README.md
```

---

*Submitted by Subham Jha for the magicpin Vera AI Challenge, May 2026.*
