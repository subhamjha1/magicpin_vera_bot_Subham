# magicpin Vera Deterministic Engine

## Approach
This service uses a deterministic rule-based engine rather than a generative model. It maps merchant, customer, category, and trigger context into concise, category-specific outreach messages with a controlled tone and one low-effort CTA.

## Key Features
- Version-controlled context updates for `merchant`, `customer`, `trigger`, and `category` scopes.
- Stateful in-memory store that tracks conversation state and deduplicates processed triggers.
- Deterministic action generation: every `/v1/tick` response is driven by exact dataset values and business rules.
- Category-specific tone: messages adapt voice and urgency based on merchant category metadata.

## Deployment
- `Dockerfile` is configured with `CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "10000"]`.
- Runtime dependencies are captured in `requirements.txt`:
  - `fastapi`
  - `pydantic>=2.0.0`
  - `uvicorn`

## Scoring and Prioritization
- Merchant performance metrics are used to prioritize actions when a trigger indicates a drop in calls, views, or CTR.
- Locality and city context are incorporated into urgency framing, making messages more relevant to nearby customers.
- Actions are built to favour low-friction CTAs and exact merchant data such as offers, city, and owner name.

## Readiness
- `/v1/metadata` returns `team_name: magicpin Vera Engine`.
- `/v1/healthz` returns a robust health payload with status, uptime, and loaded context counts.
- The codebase is ready for a final Git push to Render or GitHub.
