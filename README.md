# Multi-Agent-Calendar-Coordination-System


**An autonomous agent that finds a time your friends are all free, figures out something you'd all actually enjoy, and books it — without a group chat.**

> **Status: early prototype.** Design is complete; Phase 0 is in progress.
---

## The problem

Four people want to hang out. Someone floats a weekend. Two reply, one is away, the suggestion drifts, and three weeks later nobody has done anything. The coordination cost is higher than the activity is worth.

## What this does

Friends keep a dedicated **"Free to hangout"** calendar and drop blocks on it when they're genuinely free. Once a week the agent:

1. Reads everyone's availability — *free/busy only, no event details*
2. Intersects it into candidate time slots
3. Matches those slots against everyone's wish lists to propose something concrete
4. Puts a tentative hold on everyone's calendar and sends real invites
5. Renegotiates when someone can't make it — trimming the window, shifting the day, or going ahead without them
6. Asks each person to authorise their own share, then books

---

## How it works

```
   Friends' "Free to hangout" calendars
                  │  freebusy.query — start/end times only
                  ▼
        ┌─────────────────────┐
        │  Ingestor           │  the only component with calendar credentials
        └──────────┬──────────┘
                   ▼
        ┌─────────────────────┐
        │  Planner (LangGraph)│
        │   intersect  → code │  ← pure interval maths, unit-tested
        │   retrieve   → code │  ← activity catalogue + venue APIs
        │   score      → code │  ← maximin coverage + fairness debt
        │   propose    → LLM  │  ← Claude picks from an enumerated menu
        │   validate   → code │  ← rejects, repairs, max 2 rounds
        └──────────┬──────────┘
                   ▼
        ┌─────────────────────┐
        │  Step Functions     │  the multi-day saga
        │   hold → RSVP gate  │
        │   → negotiate       │  ← trim / shift / swap / drop / reschedule
        │   → consent gate    │
        │   → book → settle   │
        └─────────────────────┘
```


---

## Privacy

This reads people's calendars, so the bar is high.

- **Sensitive data is never fetched, not stripped later.** `freebusy.query` returns bare `{start, end}` intervals. Event titles, guests and locations are never retrieved.
- **Narrowest possible permission.** Friends share a dedicated calendar at *"See only free/busy (hide details)"*, or grant `calendar.app.created`, which can only touch calendars the app itself created. The app never gets access to their real calendar.
- **Friends grant no write access at all.** The service owns the hangout event and invites them; they RSVP in their own calendar.
- **Prompts carry no personal data.** The model sees opaque slot IDs and wish-list tags — never names, emails or timestamps. Traces are therefore PII-free by construction.
- **Real timestamps exist in exactly two places:** one DynamoDB item, and the event on Google's servers.

---

## Tech stack

**Reasoning** — Claude Sonnet (planning, negotiation) · Claude Haiku (parsing, copy) · LangGraph · LangSmith
**Orchestration** — AWS Step Functions · Lambda · EventBridge Scheduler · SQS FIFO
**Data** — DynamoDB (single-table, TTL, conditional writes) · Secrets Manager + KMS
**Interfaces** — API Gateway · SES · S3 + CloudFront
**External** — Google Calendar API 
**Language** — Python 3.12 · Pydantic · pytest · AWS CDK

---

## License

MIT — see [LICENSE](LICENSE).
