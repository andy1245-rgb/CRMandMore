# CRMAndMore Voice + Prospector — v1 Specification

**Status:** Draft
**Date:** 2026-06-22
**Author:** Andre + spec collaborator
**Geographic scope:** Canada only (Ontario pilot: Mississauga first)

---

## 1. Vision

CRMAndMore is an agency platform that replaces GoHighLevel's pricing tax with an integration-first hub. **v1 is a wedge product**, not the full platform: an AI voice calling layer + contractor prospector that plugs into GoHighLevel, built for a single agency partner (Andre's friend) who already runs GHL for contractor clients.

The full CRMAndMore vision (Frappe/ERPNext per-client sites, package tiers, dental/PHIPA lane, spend ledger as differentiator) is documented in later phases. v1 exists to **generate revenue and prove the voice + prospecting wedge** before building the broader platform.

### Why this wedge

- The agency partner already has GHL clients today — direct revenue path
- GHL's native Voice AI is US-only, expensive at scale, and has rough turn detection
- The partner also needs to find contractor businesses to sell *to* (agency prospecting)
- LiveKit Turn Detector v1 (released June 2026) makes conversational AI calling viable for sales
- Canada-only scope keeps telephony + compliance simple for v1

### Two modes, one product

| Mode | Who is called | Goal | Compliance profile |
|------|---------------|------|--------------------|
| **A — Agency prospecting** | Contractor businesses (B2B) | Book discovery calls to sell GHL/marketing services | B2B — exempt from National DNCL, must still register + identify |
| **B — Client speed-to-lead** | Homeowners who filled a form (B2C) | Book estimate appointments for the contractor client | Consumer — requires consent or existing relationship; form-fill = consent |

---

## 2. v1 Scope

### In scope (v1)

- Google Places prospector: find contractors by trade + city, filter, enrich
- AI outbound voice calling via LiveKit Agents + Turn Detector v1 + Twilio Canada
- GoHighLevel integration: OAuth, pull/push contacts, webhook triggers, post-call sync
- Campaign queue with rate limiting + CRTC calling-hour enforcement
- Spend tracker: per-call cost attribution (Twilio + LLM + LiveKit)
- Call recordings + transcripts + outcomes
- Simple dashboard (agency-level + per-client views)
- Two script templates (agency prospecting, client speed-to-lead)
- Canada-only telephony (Ontario pilot: Mississauga)

### Explicitly out of scope (v1)

- US numbers / TCPA compliance
- Dental / PHIPA tier
- Full CRM (Frappe/ERPNext)
- Funnel/website builder
- Meta Lead Ads sync
- Email/SMS marketing campaigns
- White-label / multi-agency self-serve signup
- True cold-list consumer calling without consent (deferred until consent tooling exists)
- Inbound voice handling

### Success criteria (90 days)

1. Partner can search contractors in Mississauga by trade
2. AI calls a contractor prospect with natural turn-taking (no mid-sentence cutoffs)
3. Call outcome syncs to GHL (note + tag + pipeline stage)
4. Partner demos voice to 1 contractor client → first paying client
5. Spend tracker shows real cost per call vs. what partner charges client

---

## 3. Architecture

### System overview

```
┌─────────────────────────────────────────────────────────────┐
│  CRMAndMore Hub (FastAPI, DigitalOcean Toronto)             │
│                                                             │
│  ┌──────────────┐   ┌──────────────┐   ┌───────────────┐   │
│  │  Prospector   │   │  Campaign    │   │  Spend Ledger │   │
│  │  (Places API) │──▶│  Queue +     │──▶│  (per-call    │   │
│  │  search/filter│   │  Scheduler   │   │  cost attrib) │   │
│  └──────────────┘   └──────┬───────┘   └───────────────┘   │
│                            │                               │
│  ┌──────────────┐   ┌──────▼───────┐   ┌───────────────┐   │
│  │  GHL Sync     │◀──│  Voice Engine│──▶│  Call Logs +  │   │
│  │  (OAuth, API, │   │  (LiveKit    │   │  Transcripts  │   │
│  │  webhooks)    │   │  Agent worker)│   │  + Recordings │   │
│  └──────────────┘   └──────┬───────┘   └───────────────┘   │
│                            │                               │
│  ┌──────────────┐          │          ┌───────────────┐    │
│  │  Compliance   │          │          │  Dashboard    │    │
│  │  (hours, DNCL,│          │          │  (agency +    │    │
│  │  consent)     │          │          │  per-client)  │    │
│  └──────────────┘          │          └───────────────┘    │
└────────────────────────────┼──────────────────────────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
        Twilio CA       LiveKit Cloud    GoHighLevel API
        (SIP trunk,     (Agent runtime,  (OAuth v2,
         outbound)       Turn Det v1)     contacts,
                                          webhooks)
```

### Components

| Component | Responsibility | Tech |
|-----------|---------------|------|
| **Hub API** | REST surface, auth, orchestration, dashboard backend | FastAPI (Python) |
| **Prospector** | Google Places Text Search + Place Details, filtering, enrichment | Google Places API (New) |
| **Campaign Queue** | Batch call scheduling, rate limits, calling-hour enforcement | DB-backed queue + APScheduler (in-process). No Redis in v1 — upgrade to Celery + Redis in v2 when volume demands |
| **Voice Engine** | LiveKit Agent worker — outbound calls, scripts, Turn Detector v1 | LiveKit Agents SDK (Python) |
| **GHL Sync** | OAuth, contact pull/push, webhook receiver, post-call updates | GoHighLevel REST API v2 |
| **Compliance** | Calling-hour checks, DNCL scrub (consumer mode), consent verification | Internal service + DNCL cache |
| **Spend Ledger** | Per-call cost: Twilio minutes + LLM tokens + LiveKit minutes | DB ledger + provider usage APIs |
| **Call Logs** | Recordings, transcripts, outcomes, duration | Storage (DO Spaces) + Postgres |
| **Dashboard** | Agency + per-client views, campaign management, spend reports | FastAPI templates (Jinja) or minimal SPA |

### Data residency (PIPEDA compliance)

Canada's PIPEDA doesn't require data to stay in Canada, but requires adequate protection for data that crosses borders. Here's where each data flow goes:

| Data | Provider | Processing location | PIPEDA note |
|------|----------|-------------------|-------------|
| Call audio (real-time) | Twilio + LiveKit Cloud | Twilio: Canada/US; LiveKit: us-east (Virginia) | Voice data transits through US servers |
| STT (speech-to-text) | Deepgram | US (api.deepgram.com) or EU (api.eu.deepgram.com) | Use EU endpoint for marginally better privacy posture |
| LLM (conversation) | OpenAI | US | Business data (contractor names, call content) transits to US |
| TTS (text-to-speech) | Cartesia | US | Text transits to US |
| Recordings + transcripts | DO Spaces (Toronto) | Canada | Stored in Canada |
| Contact data + PII | PostgreSQL (DO Toronto) | Canada | Stored in Canada |
| GHL sync | GoHighLevel | US (GHL is US-hosted) | Contact data already in GHL (US) |

**v1 approach:** All data at rest is in Canada (Postgres + DO Spaces Toronto). Real-time processing transits through US (LiveKit, Deepgram, OpenAI, Cartesia). This is PIPEDA-compliant as long as:
1. Privacy policy discloses which providers process data and where
2. Providers have adequate security (all are SOC2 compliant)
3. No PHI/health data in v1 (dental/PHIPA is v3+)

**v2 upgrade path:** Self-host LiveKit on DO Toronto to keep real-time voice data in Canada. Switch Deepgram to EU endpoint. This gets most data flows within Canada/EU, with only OpenAI remaining in US (no Canadian LLM provider at scale yet).

### Tech stack decisions

| Layer | Choice | Rationale |
|-------|--------|-----------|
| Language | Python 3.12+ | LiveKit Python SDK is most mature; one language for hub + agent |
| Hub framework | FastAPI | Async, typed, fast, great for API + lightweight dashboard |
| Voice agent | LiveKit Agents SDK (Python) 1.6.1+ | Native Turn Detector v1 support, telephony via SIP |
| Turn detection | LiveKit Turn Detector v1 (Cloud) | Free on LiveKit Cloud, state-of-the-art, default in SDK |
| STT | Deepgram Nova-3 | Fast, multilingual. **No Canada endpoint** — EU endpoint available (api.eu.deepgram.com), US default. SOC2/HIPAA/GDPR compliant. For v1 (no PHI), US endpoint acceptable — document in privacy policy |
| LLM | OpenAI GPT-4o (via API) | Reliable, well-supported in LiveKit; data transits to US — document in privacy policy |
| TTS | Cartesia Sonic-3.5 | ~$0.02/min (8x cheaper than ElevenLabs), 40ms TTFB. See TTS comparison in §6 |
| Telephony | Twilio Canada (SIP trunk → LiveKit) | Canadian DIDs, local presence, SIP trunking docs exist |
| Voice runtime | LiveKit Cloud (us-east region) | **No Canada region** — us-east (Virginia) is closest to Toronto. For v1, acceptable. v2: self-host on DO Toronto for full data residency |
| Database | PostgreSQL (DigitalOcean Managed) | Relational, reliable, Toronto region |
| Object storage | DigitalOcean Spaces (Toronto) | Call recordings, transcripts |
| Hosting | DigitalOcean Toronto (TOR1) | Cheapest Canada residency, simple, student-budget friendly |
| LiveKit | LiveKit Cloud | Free v1 turn detector, no infra, auto-scaling |
| Maps | Google Places API (New) | ToS-compliant for commercial product, excellent Canada coverage |
| Queue/scheduler | APScheduler (v1, in-process, no Redis) → Celery + Redis (v2) | Start simple with Postgres-backed queue. No Redis dependency in v1 |

### Data residency

All v1 data stays in Canada:
- Hub + DB + object storage: DigitalOcean Toronto (TOR1)
- LiveKit Cloud: confirm region selection (LiveKit Cloud has region options)
- Twilio: Canadian numbers, calls routed via Canadian infrastructure where possible
- OpenAI: API calls transit to US — document this in privacy policy (no PHI in v1, only business contact data)
- Deepgram: confirm data processing region; use Canadian/region-pinned if available

---

## 4. Data Model

### Entities

```
Agency (1) ──< Client (N) ──< Campaign (N) ──< Call (N)
                   │
                   ├──< GhlConnection (1)
                   ├──< Contact (N)
                   └──< SpendEntry (N)

ProspectList (N) ──< Prospect (N) ──> Contact (0..1)
```

### Tables

#### `agencies`
| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| name | text | Agency name (partner's agency) |
| ghl_marketplace_client_id | text nullable | GHL OAuth app client ID |
| ghl_marketplace_client_secret | text nullable (encrypted) | GHL OAuth app client secret |
| twilio_account_sid | text nullable | Twilio account for this agency |
| twilio_auth_token | text nullable (encrypted) | Twilio auth token |
| dncl_ran | text nullable | National DNCL Registration Account Number |
| created_at | timestamptz | |

*v1: single agency row. Schema supports multi-agency for v2.*

#### `users`
Authenticatable users who can log into the hub dashboard.

| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| agency_id | UUID FK → agencies | |
| email | text unique | Login email |
| password_hash | text | bcrypt/argon2 hash |
| role | enum | `admin` \| `agent` \| `viewer` |
| first_name | text | |
| last_name | text | |
| is_active | bool | |
| created_at | timestamptz | |

*v1: one admin user (the partner). `agent` and `viewer` roles for v2.*

#### `clients`
A "client" = one of the partner's contractor clients (or the partner's own prospecting operation).

| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| agency_id | UUID FK → agencies | |
| name | text | e.g. "Joe's HVAC" or "Agency Prospecting" |
| type | enum | `contractor_client` \| `agency_prospecting` |
| ghl_location_id | text | GHL sub-account/location ID |
| twilio_from_number | text | Canadian DID assigned to this client |
| created_at | timestamptz | |

#### `ghl_connections`
OAuth connection to a GHL sub-account.

| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| client_id | UUID FK → clients | |
| access_token | text (encrypted) | |
| refresh_token | text (encrypted) | |
| token_expires_at | timestamptz | |
| scopes | text[] | |
| created_at | timestamptz | |

#### `contacts`
Unified contact record. Synced from GHL or created from prospector.

| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| client_id | UUID FK → clients | |
| ghl_contact_id | text nullable | GHL contact ID if synced |
| phone | text | E.164 format |
| phone_type | enum | `business` \| `mobile` \| `home` \| `unknown` |
| first_name | text nullable | |
| last_name | text nullable | |
| business_name | text nullable | For B2B prospects |
| source | enum | `ghl` \| `prospector` \| `manual` |
| consent_status | enum | `unknown` \| `express` \| `existing_relationship` \| `none` |
| consent_source | text nullable | e.g. "form_fill:website", "prior_customer" |
| dncl_registered | bool nullable | True if on National DNCL (consumer mode only) |
| metadata | jsonb | Trade, city, rating, website status, etc. |
| created_at | timestamptz | |
| updated_at | timestamptz | |

#### `prospect_lists`
A saved search result set from the prospector.

| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| client_id | UUID FK → clients | |
| name | text | e.g. "HVAC Mississauga — no website" |
| search_params | jsonb | Trade, city, filters applied |
| created_at | timestamptz | |

#### `prospects`
Individual contractor businesses found via Google Places.

| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| prospect_list_id | UUID FK → prospect_lists | |
| contact_id | UUID FK → contacts nullable | Once promoted to contact |
| place_id | text | Google Places ID |
| business_name | text | |
| phone | text nullable | E.164 |
| address | text nullable | |
| city | text | |
| province | text | Default ON |
| rating | float nullable | |
| review_count | int nullable | |
| website | text nullable | |
| has_website | bool | Derived — key filter for agency pitch |
| maps_url | text | |
| status | enum | `new` \| `queued` \| `called` \| `converted` \| `skipped` |
| created_at | timestamptz | |

#### `campaigns`
| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| client_id | UUID FK → clients | |
| name | text | |
| mode | enum | `agency_prospecting` \| `client_speed_to_lead` |
| script_template_id | UUID FK → script_templates | |
| status | enum | `draft` \| `scheduled` \| `running` \| `paused` \| `completed` |
| calling_hours_start | time | Default 09:00 (CRTC minimum) |
| calling_hours_end | time | Default 21:30 — treated as MAX; compliance module enforces actual CRTC schedule (21:30 weekdays, 18:00 weekends) |
| max_concurrent_calls | int | Rate limit |
| daily_call_cap | int nullable | |
| max_call_duration_sec | int | Default 300 (5 min hard cap per call) |
| started_at | timestamptz nullable | |
| completed_at | timestamptz nullable | |

#### `calls`
| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| campaign_id | UUID FK → campaigns | |
| contact_id | UUID FK → contacts | |
| client_id | UUID FK → clients | |
| status | enum | `queued` \| `dialing` \| `in_progress` \| `completed` \| `failed` \| `no_answer` \| `skipped_compliance` |
| scheduled_at | timestamptz | |
| started_at | timestamptz nullable | |
| ended_at | timestamptz nullable | |
| duration_sec | int nullable | |
| outcome | enum nullable | `booked` \| `interested` \| `not_interested` \| `voicemail` \| `callback_requested` \| `no_answer` \| `failed` |
| voicemail_detected | bool | True if transcript indicates answering machine |
| cost_total_cents | int | Denormalized sum of all spend_entries for this call (updated as entries are written) |
| transcript | text nullable | Full call transcript |
| transcript_url | text nullable | DO Spaces URL if stored separately |
| recording_url | text nullable | DO Spaces URL |
| ghl_note_posted | bool | |
| ghl_tags_applied | text[] | |
| ghl_stage_moved | text nullable | Pipeline stage if moved |
| livekit_room_id | text nullable | |
| twilio_call_sid | text nullable | |
| created_at | timestamptz | |

#### `spend_entries`
Per-call cost attribution.

| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| call_id | UUID FK → calls | |
| client_id | UUID FK → clients | |
| provider | enum | `twilio` \| `openai` \| `livekit` \| `deepgram` \| `cartesia` \| `google_places` |
| cost_cents | int | Cost in CAD cents |
| units | numeric | Minutes, tokens, requests |
| unit_type | enum | `minute` \| `token` \| `request` \| `search` |
| billed_at | timestamptz | |
| raw_usage | jsonb | Provider response detail |

#### `script_templates`
| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| agency_id | UUID FK → agencies | Global in v1 (single agency), but scoped for v2-readiness |
| name | text | e.g. "Agency Prospecting — Discovery Call" |
| mode | enum | `agency_prospecting` \| `client_speed_to_lead` |
| system_prompt | text | LLM system prompt |
| greeting | text | Opening line template (with variables) |
| variables | jsonb | Available template vars (business_name, city, service, etc.) |
| booking_calendar_id | text nullable | GHL calendar for booking |
| is_active | bool | |

#### `compliance_log`
Audit trail for compliance decisions.

| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| call_id | UUID FK → calls nullable | |
| contact_id | UUID FK → contacts | |
| action | enum | `calling_hour_check` \| `dncl_scrub` \| `consent_check` \| `call_blocked` \| `call_allowed` |
| result | text | Pass/fail + reason |
| checked_at | timestamptz | |
```

---

## 5. Prospector Module

Finds contractor businesses on Google Maps for the partner to call (Mode A — agency prospecting).

### Search flow

```
User selects trade + city + filters
        │
        ▼
Google Places Text Search (New) API
   query: "{trade} contractor {city} Ontario"
   field mask includes: phone, website, rating, reviews
        │
        ▼
Results: place_id + name + address + phone + website + rating + reviews
   (single API call returns all needed fields)
        │
        ▼
Filter: has_phone, has_website, rating, review_count
        │
        ▼
Save to prospect_list + prospects table
        │
        ▼
Display in dashboard → [Call] [Skip] [Add to campaign]
```

### Google Places API usage

**Endpoint:** `https://places.googleapis.com/v1/places:searchText` (Text Search New)

**Optimized approach:** Include phone + website fields directly in the Text Search field mask. This uses the Enterprise SKU (higher per-request cost) but eliminates the need for separate Place Details calls — a 20x cost reduction vs. the naive approach.

**Text Search request (single call, all fields):**
```python
import httpx

async def search_contractors(trade: str, city: str, api_key: str, page_token: str = None):
    url = "https://places.googleapis.com/v1/places:searchText"
    # Field mask includes contact data (phone, website) — triggers Enterprise SKU
    # but avoids N separate Place Details calls
    headers = {
        "Content-Type": "application/json",
        "X-Goog-Api-Key": api_key,
        "X-Goog-FieldMask": (
            "places.id,places.displayName,places.formattedAddress,"
            "places.nationalPhoneNumber,places.internationalPhoneNumber,"
            "places.websiteUri,places.googleMapsUri,"
            "places.rating,places.userRatingCount,"
            "nextPageToken"
        )
    }
    body = {
        "textQuery": f"{trade} contractor {city} Ontario Canada",
        "languageCode": "en",
        "regionCode": "CA",
        "pageSize": 20
    }
    if page_token:
        body["pageToken"] = page_token

    async with httpx.AsyncClient() as client:
        resp = await client.post(url, headers=headers, json=body)
        return resp.json()
```

**Pagination:** Text Search returns 20 results per page. Use `nextPageToken` to fetch additional pages. For Mississauga pilot, one page (20 results) is likely sufficient per trade.

```python
async def search_all_contractors(trade, city, api_key, max_pages=3):
    all_results = []
    page_token = None
    for _ in range(max_pages):
        data = await search_contractors(trade, city, api_key, page_token)
        all_results.extend(data.get("places", []))
        page_token = data.get("nextPageToken")
        if not page_token:
            break
    return all_results
```

### Field masks (cost control)

Including contact data fields (`nationalPhoneNumber`, `websiteUri`) in the Text Search field mask triggers the **Enterprise SKU** billing tier. However, this is dramatically cheaper than making separate Place Details calls:

| Approach | API calls | Cost per 20 prospects |
|----------|-----------|----------------------|
| Text Search (Essentials) + 20× Place Details (Enterprise) | 21 calls | ~$0.83 |
| **Text Search (Enterprise, all fields in one call)** | **1 call** | **~$0.040** |

**Cost estimate:**
- Text Search (Enterprise, with contact data): ~$0.040/request (20 results)
- Mississauga pilot budget: 5 trades × 1 search = **~$0.20 total**
- Full Ontario expansion (5 trades × 10 cities): ~$2.00 total

### Trade query templates

Pre-built search queries per trade (Ontario-focused):

| Trade | Text query |
|-------|-----------|
| HVAC | `HVAC contractor {city} Ontario` |
| Plumbing | `plumber {city} Ontario` |
| Roofing | `roofing contractor {city} Ontario` |
| Electrical | `electrician {city} Ontario` |
| General | `general contractor {city} Ontario` |

### Filters

Applied client-side after Place Details fetch:

| Filter | Purpose | Default |
|--------|---------|---------|
| Has phone | Can't call without it | ✅ Required |
| No website | Agency pitch gold ("I'll build you a site") | Optional, default on |
| Rating < 4.0 | Struggling businesses = receptive | Optional |
| Reviews < 20 | New/low-visibility businesses | Optional |
| Already in CRM | Skip duplicates | ✅ Required |

### Prospector UI (dashboard)

```
┌─ Find Contractors ─────────────────────────────────────┐
│  Trade:     [HVAC ▼]  [Plumbing] [Roofing] [Electrical]│
│  City:      [Mississauga, ON    ]                      │
│  Filters:   ☑ Has phone  ☑ No website                 │
│             ☐ Rating < 4.0  ☐ Reviews < 20            │
│                                                        │
│  [ Search ]                                            │
├────────────────────────────────────────────────────────┤
│  Results: 47 contractors                               │
│  ┌──────────────────────────────────────────────────┐  │
│  │ Joe's HVAC          ☎ 905-555-0123  ⭐ 4.2 (31) │  │
│  │ No website · Mississauga          [Call] [Skip]  │  │
│  ├──────────────────────────────────────────────────┤  │
│  │ Arctic Air Ltd      ☎ 905-555-0198  ⭐ 3.8 (12) │  │
│  │ No website · Mississauga          [Call] [Skip]  │  │
│  └──────────────────────────────────────────────────┘  │
│                                                        │
│  [ Add all to campaign ]  [ Export to GHL ]            │
└────────────────────────────────────────────────────────┘
```

### Deduplication

Before saving a prospect, check:
1. Same `place_id` already exists in any prospect list → skip
2. Same `phone` (normalized E.164) exists in `contacts` table → mark as duplicate, offer to link

### Alternative: Apify Google Maps Scraper

For richer data at comparable cost, Apify offers Google Maps scrapers that extract 30+ fields per business.

| Aspect | Google Places API (optimized) | Apify Google Maps Scraper |
|--------|-------------------------------|--------------------------|
| Cost per 1,000 results | ~$2.00 | ~$2.50 (cheapest: $1.00) |
| Phone + website | ✅ | ✅ |
| `claimedBusiness` flag | ❌ | ✅ (unclaimed = prime outreach target) |
| Categories, popular times | ❌ | ✅ (30+ fields) |
| Email enrichment | ❌ | ✅ (some actors) |
| ToS compliance | ✅ Official API | ⚠️ Scraping — gray area |
| API key needed | Google Maps Platform | Apify account |
| Pagination | Manual (nextPageToken) | Built-in (maxResults param) |

**Decision (June 2026): Use Google Places API for v1.** Google's Terms of Service explicitly prohibit scraping. For a commercial product that the partner is selling to clients, ToS compliance is non-negotiable — a client could sue if the tool they're paying for uses scraped data. The Places API is ToS-safe and the optimized Text Search approach costs about the same (~$2/1,000 vs ~$2.50/1,000).

The `claimedBusiness` flag from Apify would be nice for targeting unclaimed Google Business Profiles, but we can approximate this by checking if the business has a website (no website = likely unclaimed/less sophisticated = good prospect). The Places API already returns `websiteUri`.

**Apify integration (available as fallback if needed):**
```python
async def search_contractors_apify(trade: str, city: str, api_token: str):
    url = "https://api.apify.com/v2/acts/solidcode~google-maps-scraper-2-5-per-1-000-results/runs"
    headers = {"Authorization": f"Bearer {api_token}"}
    body = {
        "searchQueries": [f"{trade} contractor {city} Ontario"],
        "maxResults": 100,
        "countryCode": "CA",
        "language": "en"
    }
    # Start actor run, poll for results
    ...
```

---

## 6. Voice Engine

The core differentiator. AI outbound calls with natural turn-taking via LiveKit Turn Detector v1.

### Architecture

```
Campaign Queue
     │
     ▼  (picks next eligible call)
Voice Engine Worker (LiveKit Agent)
     │
     ├── Creates LiveKit room
     ├── Creates SIP participant (Twilio → outbound call)
     ├── AgentSession with:
     │     • TurnDetector v1 (LiveKit Cloud, free)
     │     • STT: Deepgram Nova-3
     │     • LLM: OpenAI GPT-4o
     │     • TTS: Cartesia Sonic-3.5 (v1 choice — see TTS comparison below)
     │     • System prompt from script_template
     │
     ▼  (conversation happens)
Call ends →
     ├── Transcript captured
     ├── Outcome classified (LLM post-call analysis)
     ├── Recording saved to DO Spaces
     ├── Spend entries written (Twilio + OpenAI + LiveKit + Deepgram + TTS)
     └── GHL sync triggered (note + tags + pipeline stage)
```

### LiveKit Agent setup (Python)

```python
from livekit.agents import Agent, AgentSession, JobContext, WorkerOptions, cli
from livekit.agents import TurnHandlingOptions, inference
from livekit.plugins import deepgram, openai, cartesia

class ContractorAgent(Agent):
    def __init__(self, script: str, contact_context: dict):
        super().__init__(instructions=script)
        self.context = contact_context

    async def on_enter(self):
        # Greeting uses contact context (business name, city, service)
        greeting = self._build_greeting()
        await self.session.generate_reply(instructions=greeting)

    def _build_greeting(self) -> str:
        biz = self.context.get("business_name", "there")
        city = self.context.get("city", "your area")
        return (
            f"Greet the person at {biz} in {city}. "
            f"Say you're calling from [Agency Name] about helping "
            f"contractors get more booked jobs. Ask if they have 30 seconds."
        )


async def entrypoint(ctx: JobContext):
    # Contact context passed from campaign queue
    contact = ctx.job.metadata  # phone, business_name, city, script, etc.

    session = AgentSession(
        turn_handling=TurnHandlingOptions(
            turn_detection=inference.TurnDetector(),  # v1 on LiveKit Cloud
        ),
        stt=deepgram.STT(model="nova-3", language="en"),
        llm=openai.LLM(model="gpt-4o"),
        tts=cartesia.TTS(voice="friendly-contractor-voice"),
    )

    agent = ContractorAgent(
        script=contact["system_prompt"],
        contact_context=contact,
    )

    # Outbound call via SIP (Twilio trunk)
    await ctx.create_sip_participant(
        sip_trunk_id=contact["sip_trunk_id"],
        call_to=contact["phone"],
        room=ctx.room,
    )

    await session.start(room=ctx.room, agent=agent)


if __name__ == "__main__":
    cli.run_app(WorkerOptions(entrypoint_fnc=entrypoint))
```

### TTS provider selection: Cartesia (v1)

Researched and decided June 2026. Cartesia Sonic-3.5 is the v1 TTS provider.

| Metric | Cartesia Sonic-3.5 | ElevenLabs Flash v2.5 |
|--------|-------------------|----------------------|
| **TTFB (latency)** | ~40ms (Turbo variant) | ~75ms |
| **Cost per minute** | ~$0.02/min (Startup tier: $37/mo, 1,667 min) | ~$0.17/min (Pro tier: $99/mo, 600 min) |
| **Cost ratio** | 1x | **8x more expensive** |
| **Voice quality** | Natural, conversational | Ultra-realistic, best cloning |
| **Concurrency (Startup)** | 5 concurrent requests | 5 (Pro tier) |
| **Enterprise compliance** | DPAs and BAAs available | DPAs, BAAs (HIPAA) available |

**Why Cartesia for v1:**
- Cold calling is utility TTS, not premium voice content — the voice needs to be natural enough to hold a conversation, not indistinguishable from human
- At 500 calls/month × 3 min avg = 1,500 TTS minutes: Cartesia costs ~$37/mo vs ElevenLabs ~$249/mo
- Lower latency (40ms vs 75ms) means more natural turn-taking with Turn Detector v1
- Cartesia also offers STT (Ink-2) and Voice Agents (Line) if we want to consolidate providers later

**When to switch to ElevenLabs:** If the partner's clients specifically request ultra-realistic voices or voice cloning (e.g., cloning the contractor's own voice for their speed-to-lead calls). That's a v2 upsell feature.

### Voicemail detection (software-based)

**Problem:** Twilio's Answering Machine Detection (AMD) does NOT work with SIP trunking — it only works with Programmable Voice calls. Since LiveKit uses SIP trunking via Twilio, AMD is unavailable.

**v1 solution:** STT-based voicemail detection. After the call connects, the agent listens to the first utterance. If the transcript matches voicemail patterns, the agent leaves a brief message and hangs up.

```python
import re

VOICEMAIL_PATTERNS = [
    r"leave a message",
    r"after (the|this) (beep|tone)",
    r"you'?ve reached",
    r"(please )?leave your name",
    r"mailbox is full",
    r"not available",
    r"unable to take (your|this) call",
    r"recorded after",
    r"at the (sound|tone) of",
    r"call back later",
]

def is_voicemail(first_utterance: str) -> bool:
    """Check if the first utterance matches voicemail patterns."""
    text = first_utterance.lower().strip()
    if len(text) < 3:
        return False
    return any(re.search(pattern, text) for pattern in VOICEMAIL_PATTERNS)
```

**Agent behavior on voicemail detection:**
1. Agent greets as normal
2. STT captures first utterance from the callee
3. If `is_voicemail()` returns True:
   - Agent delivers a short voicemail message (configurable per script)
   - Agent ends the call
   - Outcome classified as `voicemail`
4. If False: proceed with normal conversation

```python
class ContractorAgent(Agent):
    async def on_user_turn_completed(self, turn_text: str):
        # Check first utterance for voicemail
        if self._is_first_turn and is_voicemail(turn_text):
            await self.session.generate_reply(
                instructions="This is a voicemail. Leave a brief message: "
                "'Hi, this is [Agent] from [Agency]. We help contractors "
                "get more booked jobs. Give us a call back at [number]. Thanks!'"
            )
            await asyncio.sleep(8)  # let the message play
            await self.session.end_call()
            return
        # Normal conversation flow
        ...
```

**Limitation:** This is heuristic, not perfect. A human saying "you've reached John" could trigger a false positive. Mitigate by requiring multiple pattern matches or checking for a beep/silence after the greeting. Refine during pilot.

### Max call duration cap

Every call has a hard duration limit (default 300 seconds / 5 minutes, configurable per campaign via `max_call_duration_sec`). The agent gracefully ends the call when the limit is reached:

```python
async def enforce_duration_cap(session: AgentSession, max_sec: int):
    await asyncio.sleep(max_sec)
    await session.generate_reply(
        instructions="We're running long. Politely wrap up the conversation "
        "and end the call within 15 seconds."
    )
    await asyncio.sleep(15)
    await session.end_call()
```

### Call transfer to human (v1.1 — deferred)

**Not in v1.** The Twilio SIP trunk setup mentions "Enable PSTN transfer" but the agent code does not implement live call transfer. For Mode B emergencies ("no heat, flooding, no power"), the script instructs the agent to flag the call as urgent and offer a callback — the contractor is notified via GHL, not via live transfer.

v1.1 will implement `transfer_sip_participant` for live escalation to the contractor's on-call number. Requires SIP REFER enabled on the Twilio trunk.

Turn Detector v1 is the **default** when running on LiveKit Cloud — no explicit config needed. To pin it explicitly:

```python
session = AgentSession(
    turn_handling=TurnHandlingOptions(
        turn_detection=inference.TurnDetector(version="v1"),
    ),
    # ...
)
```

**Key behaviors:**
- Audio-based: fuses semantic + acoustic signals, no transcript dependency
- 14 languages supported (English primary for v1)
- Free on LiveKit Cloud (no per-inference charge)
- Auto-fallback to v1-mini (CPU, open-weight) if Cloud unavailable
- `unlikely_threshold` tunable per language (lower = more eager, higher = more patient)

### Outbound call flow (SIP via Twilio)

1. **Hub** creates a call record (status: `queued`)
2. **Campaign scheduler** picks the call, checks compliance (hours, consent)
3. **Hub** dispatches a LiveKit job with contact metadata
4. **LiveKit Agent worker** starts, creates room
5. **`create_sip_participant`** sends SIP INVITE to Twilio trunk → Twilio dials the number
6. **Call connects** → agent greets → conversation proceeds with Turn Detector v1
7. **Call ends** → webhook from LiveKit → hub captures transcript + metadata
8. **Hub** classifies outcome, writes spend entries, triggers GHL sync

### Twilio SIP trunk setup

**Twilio side:**
- Create SIP Trunk (`my-trunk.pstn.twilio.com`)
- Configure origination URI → LiveKit SIP endpoint
- Enable PSTN transfer (for call transfers if needed)
- Purchase Canadian DID(s) as caller ID

**LiveKit side:**
- Register Twilio trunk as outbound SIP trunk
- Use trunk ID in `create_sip_participant`

### Script templates (v1)

#### Script A — Agency Prospecting (Mode A)

**Goal:** Book 15-min discovery call
**Called:** Contractor business owners (B2B)

```
System prompt:
You are a friendly, professional outbound caller representing [Agency Name],
a marketing agency that helps HVAC/plumbing/roofing contractors in Ontario
get more booked jobs through Google and Facebook ads.

Rules:
- Be concise. This is a cold B2B call — respect their time.
- Identify yourself and the purpose immediately.
- Don't be pushy. If they're busy, offer to call back.
- Your goal is to book a 15-minute discovery call, NOT to sell on this call.
- If they're interested, collect: best time to call back, email (optional).
- If not interested, thank them and end politely.
- If they ask what it costs, say "it depends on your goals — that's what
  the discovery call is for."
- Never make up specific results or guarantees.

Context:
- Business: {business_name}
- City: {city}
- Trade: {trade}
```

#### Script B — Client Speed-to-Lead (Mode B)

**Goal:** Book estimate appointment for contractor client
**Called:** Homeowners who submitted a form (consent established)

```
System prompt:
You are calling on behalf of {contractor_client_name}, a {trade} company
in {city}. The person you're calling just submitted a form on their website
requesting a quote for {service_requested}.

Rules:
- Identify yourself as calling from {contractor_client_name} immediately.
- Reference the form submission ("You just requested a quote for...").
- Your goal is to book a free estimate appointment.
- If they mention an emergency (no heat, flooding, no power), flag as urgent
  and offer immediate callback from the contractor.
- Collect: preferred time for estimate, address (if needed).
- If they're no longer interested, thank them and end.
- Be warm and helpful — this is a warm lead, not a cold call.

Context:
- Contractor: {contractor_client_name}
- Service: {service_requested}
- Lead source: {lead_source} (website form)
- Calendar ID: {ghl_calendar_id}
```

### Outcome classification

Post-call, the hub runs a lightweight LLM classification on the transcript:

```python
async def classify_outcome(transcript: str) -> str:
    prompt = f"""Analyze this call transcript and classify the outcome.
    Return exactly one of: booked, interested, not_interested, voicemail,
    callback_requested, no_answer, failed

    Transcript:
    {transcript[:4000]}
    """
    # ... LLM call
    return outcome
```

### Call recording + transcript storage

- **Recording:** LiveKit Egress records the room → `egress_started`/`egress_ended` webhooks fire → file saved to DO Spaces (Toronto)
- **Transcript:** Built from Deepgram STT results during the call → stored in `calls.transcript` + DO Spaces backup
- **Retention:** 90 days default, configurable per client

### LiveKit webhook receiver

LiveKit Cloud sends webhooks for room and participant events. The hub receives these to track call lifecycle.

**Configuration:** Set webhook URL in LiveKit Cloud dashboard (Settings → Webhooks). LiveKit signs webhooks with a JWT in the `Authorization` header (content-type: `application/webhook+json`).

**Endpoint:** `POST /webhooks/livekit`

```python
from livekit.api import WebhookReceiver

receiver = WebhookReceiver(LIVEKIT_API_KEY, LIVEKIT_API_SECRET)

@app.post("/webhooks/livekit")
async def livekit_webhook(request: Request):
    # LiveKit sends application/webhook+json — need raw body
    body = await request.body()
    auth_header = request.headers.get("Authorization", "")

    try:
        event = receiver.receive(body.decode(), auth_header)
    except Exception:
        raise HTTPException(401, "Invalid LiveKit signature")

    event_name = event.event
    room_name = event.room.name if event.room else None

    if event_name == "room_finished":
        # Call ended — trigger post-call processing
        call = await find_call_by_livekit_room(room_name)
        if call and call.status == "in_progress":
            await process_call_end(call)

    elif event_name == "participant_left":
        # SIP participant (callee) left
        call = await find_call_by_livekit_room(room_name)
        if call and call.status == "in_progress":
            await process_call_end(call)

    elif event_name == "egress_ended":
        # Recording finished — get file URL
        await save_recording_url(event.egress_info)

    return {"status": "received"}
```

**Key webhook events used:**

| Event | Trigger | Hub action |
|-------|---------|------------|
| `room_finished` | Room closes (call ended) | Process call end: classify outcome, write spend, sync GHL |
| `participant_left` | Callee hangs up | Same as above (earlier signal) |
| `egress_started` | Recording begins | Log recording start |
| `egress_ended` | Recording file ready | Save recording URL to DO Spaces + `calls.recording_url` |

---

## 7. GoHighLevel Integration

The partner keeps GHL as the CRM. CRMAndMore syncs contacts, triggers calls from GHL workflows, and pushes call outcomes back.

### OAuth 2.0 flow

```
1. Partner clicks "Connect GHL" in dashboard
2. Redirect to GHL OAuth:
   https://marketplace.gohighlevel.com/oauth/authorize
     ?response_type=code
     &client_id={GHL_CLIENT_ID}
     &redirect_uri={HUB_URL}/ghl/callback
     &scope=contacts.write contacts.read conversations.write
            calendars.write opportunities.write
3. GHL redirects back with ?code=...
4. Hub exchanges code for access_token + refresh_token
   POST https://services.leadconnectorhq.com/oauth/token
5. Tokens stored (encrypted) in ghl_connections
6. Auto-refresh before expiry (see token refresh below)
```

**App type:** Marketplace app (for distribution) or Private Integration (faster for v1 pilot). Start with Private Integration, publish to Marketplace when scaling.

**Scopes:** Configured in the OAuth app's advanced settings. Scopes are locked once the app is live — define them in draft. Required scopes: `contacts.write`, `contacts.read`, `conversations.write`, `calendars.write`, `opportunities.write`.

### API base

`https://services.leadconnectorhq.com`

**Rate limit:** 100 requests per 10-second burst per token.

### Token refresh

GHL access tokens expire (typically 1 hour). The hub must refresh before expiry:

```python
import time
from datetime import datetime, timezone

async def ensure_valid_token(conn: GhlConnection) -> str:
    """Refresh token if expiring within 5 minutes."""
    buffer = 300  # 5 minutes
    now = datetime.now(timezone.utc)

    if conn.token_expires_at and (conn.token_expires_at.timestamp() - now.timestamp()) > buffer:
        return conn.access_token  # still valid

    # Refresh
    async with httpx.AsyncClient() as client:
        resp = await client.post(
            "https://services.leadconnectorhq.com/oauth/token",
            json={
                "client_id": GHL_CLIENT_ID,
                "client_secret": GHL_CLIENT_SECRET,
                "grant_type": "refresh_token",
                "refresh_token": conn.refresh_token,
            }
        )
        data = resp.json()
        # Update stored tokens
        await update_ghl_connection(conn.id, {
            "access_token": data["access_token"],
            "refresh_token": data.get("refresh_token", conn.refresh_token),
            "token_expires_at": datetime.fromtimestamp(
                time.time() + data["expires_in"], tz=timezone.utc
            ),
        })
        return data["access_token"]
```

### Contact sync

**Pull contacts from GHL** (for client campaigns):
```python
async def fetch_ghl_contacts(token: str, location_id: str):
    url = "https://services.leadconnectorhq.com/contacts/"
    headers = {"Authorization": f"Bearer {token}", "Version": "2021-07-28"}
    params = {"locationId": location_id}
    # Paginate through results
    ...
```

**Push prospect to GHL** (Mode A — create contact from prospector):

GHL custom fields are passed as an array of `{id, value}` objects. Field IDs must be pre-created in GHL's Custom Fields settings and their IDs stored in the hub's config.

```python
async def create_ghl_contact(token: str, location_id: str, prospect: dict, custom_field_ids: dict):
    url = "https://services.leadconnectorhq.com/contacts/"
    headers = {
        "Authorization": f"Bearer {token}",
        "Version": "2021-07-28",
        "Content-Type": "application/json"
    }
    body = {
        "locationId": location_id,
        "firstName": prospect.get("first_name"),
        "lastName": prospect.get("last_name"),
        "name": prospect.get("business_name"),
        "phone": prospect["phone"],
        "tags": ["agency-prospect", f"trade:{prospect['trade']}"],
        "customFields": [
            {"id": custom_field_ids["website_status"],
             "value": "no_website" if not prospect["has_website"] else "has_website"},
            {"id": custom_field_ids["google_rating"],
             "value": str(prospect.get("rating", ""))},
            {"id": custom_field_ids["city"],
             "value": prospect["city"]},
        ]
    }
    ...
```

**Note:** `custom_field_ids` must be fetched from GHL's Custom Fields API (`GET /custom-fields/`) and mapped to field names. Store this mapping in the hub config per location.

### Webhook integration — two mechanisms

GHL provides two distinct webhook mechanisms. The spec uses both for different purposes:

| Mechanism | Use case | Signing | Payload |
|-----------|----------|---------|---------|
| **Workflow Outbound Webhook** | Mode B: trigger calls from GHL workflows (form submit, tag added) | Shared secret in URL or custom header | Flat contact object with all fields |
| **OAuth App Webhook** | v2: contact lifecycle events (create/update/delete) | Ed25519 signature (`X-GHL-Signature`) | `{type, timestamp, webhookId, data}` |

#### Mechanism 1: Workflow Outbound Webhook (used in v1)

The partner configures a webhook action inside a GHL workflow. When the trigger fires, GHL sends a POST with the full contact object + trigger-dependent data.

**Payload structure** (flat object, fields vary by trigger):
```json
{
  "first_name": "John",
  "last_name": "Doe",
  "full_name": "John Doe",
  "email": "john@example.com",
  "phone": "+14165551234",
  "tags": ["lead", "website-form"],
  "company_name": "",
  "contact_source": "Website Form",
  "location": {
    "id": "loc_abc123",
    "name": "Joe's HVAC",
    "city": "Mississauga",
    "state": "ON"
  },
  "calendar": {
    "id": "cal_xyz",
    "startTime": "2026-06-22T14:00:00",
    "endTime": "2026-06-22T15:00:00"
  }
}
```

**Endpoint:** `POST /webhooks/ghl-workflow`

Security: include a shared secret as a query parameter or custom header that the partner configures in the GHL workflow webhook action.

```python
@app.post("/webhooks/ghl-workflow")
async def ghl_workflow_webhook(
    request: Request,
    x_crm_secret: str = Header(None, alias="X-CRM-Secret")
):
    # Verify shared secret (configured by partner in GHL workflow)
    if x_crm_secret != GHL_WEBHOOK_SHARED_SECRET:
        raise HTTPException(401, "Invalid shared secret")

    payload = await request.json()

    # Extract contact data from flat payload
    phone = payload.get("phone")
    location_id = payload.get("location", {}).get("id")

    if not phone:
        return {"status": "ignored", "reason": "no phone"}

    # Find the client by GHL location ID
    client = await find_client_by_ghl_location(location_id)
    if not client:
        return {"status": "ignored", "reason": "unknown location"}

    # Mode B: speed-to-lead — queue a call
    contact = await upsert_contact(
        client_id=client.id,
        phone=phone,
        first_name=payload.get("first_name"),
        last_name=payload.get("last_name"),
        source="ghl",
        consent_status="express",
        consent_source="form_fill:website",
    )
    await queue_speed_to_lead_call(contact, client)

    return {"status": "received"}
```

#### Mechanism 2: OAuth App Webhook (v2 — documented for future)

OAuth app webhooks are subscribed in the marketplace dashboard. These use Ed25519 signature verification.

**Signature verification (Ed25519 — NOT HMAC):**

```python
from cryptography.hazmat.primitives.asymmetric.ed25519 import Ed25519PublicKey
import base64

GHL_PUBLIC_KEY_PEM = b"""-----BEGIN PUBLIC KEY-----
MCowBQYDK2VwAyEAi2HR1srL4o18O8BRa7gVJY7G7bupbN3H9AwJrHCDiOg=
-----END PUBLIC KEY-----"""

def verify_ghl_signature(body: bytes, signature: str) -> bool:
    """Verify X-GHL-Signature using Ed25519 public key."""
    if not signature or signature == "N/A":
        return False
    try:
        public_key = serialization.load_pem_public_key(GHL_PUBLIC_KEY_PEM)
        sig_bytes = base64.b64decode(signature)
        public_key.verify(sig_bytes, body)  # Ed25519 verify
        return True
    except Exception:
        return False
```

**Legacy fallback** (`X-WH-Signature`, RSA-SHA256) — deprecated July 1, 2026. Support during transition only.

**OAuth app webhook payload:**
```json
{
  "type": "ContactCreate",
  "timestamp": "2026-06-22T14:35:00.000Z",
  "webhookId": "evt_abc123",
  "data": {
    "firstName": "John",
    "lastName": "Doe",
    "email": "john@example.com",
    "phone": "+14165551234"
  }
}
```

**Important:** OAuth app webhook event names use PascalCase (`ContactCreate`, `ContactUpdate`, `ContactTag`). These are different from the workflow webhook (which sends a flat payload with no `type` field).

### Post-call sync (push outcomes back to GHL)

After a call completes, push the outcome to GHL:

```python
async def sync_call_to_ghl(call: Call, ghl_conn: GhlConnection):
    # 1. Add note to contact
    await add_ghl_note(
        token=ghl_conn.access_token,
        contact_id=call.contact.ghl_contact_id,
        body=format_call_note(call)
    )

    # 2. Apply tags based on outcome
    tags = outcome_to_tags(call.outcome)
    await add_ghl_tags(
        token=ghl_conn.access_token,
        contact_id=call.contact.ghl_contact_id,
        tags=tags
    )

    # 3. Move pipeline stage (if applicable)
    if call.outcome == "booked":
        await move_ghl_opportunity_stage(
            token=ghl_conn.access_token,
            contact_id=call.contact.ghl_contact_id,
            stage="Estimate Scheduled"  # or "Discovery Call Booked"
        )
```

**Outcome → tag mapping:**

| Outcome | Tags applied |
|---------|-------------|
| booked | `ai-called`, `ai-booked`, `hot-lead` |
| interested | `ai-called`, `ai-interested`, `follow-up` |
| not_interested | `ai-called`, `ai-not-interested` |
| callback_requested | `ai-called`, `callback-requested` |
| voicemail | `ai-called`, `voicemail-left` |
| no_answer | `ai-called`, `no-answer` |

### GHL workflow setup (partner configures in GHL)

For Mode B (speed-to-lead), the partner creates a GHL workflow:

```
Trigger: Form submitted (website lead form)
Action 1: Wait 60 seconds
Action 2: Webhook → {HUB_URL}/webhooks/ghl
          (payload includes contactId, form data, campaign_id)
```

This keeps GHL as the trigger source while CRMAndMore handles the call.

---

## 8. Compliance Module

Canada-only telemarketing compliance under CRTC Unsolicited Telecommunications Rules (UTRs). This is a first-class module, not an afterthought — non-compliance carries financial penalties up to $1,500/violation.

### CRTC rules summary (verified June 2026)

| Rule | Requirement | v1 implementation |
|------|------------|-------------------|
| **Calling hours** | Mon–Fri: 9:00 AM–9:30 PM; Sat–Sun: 10:00 AM–6:00 PM (recipient local time) | Campaign scheduler enforces; default 9:00–21:30 weekdays |
| **National DNCL registration** | All telemarketers must register with DNCL, even for exempt calls | Document partner's RAN; store in agency config |
| **National DNCL scrub (consumer)** | Consumer numbers on DNCL cannot be called unless consent/exemption | Mode B: scrub before dialing; form-fill = express consent |
| **B2B exemption** | Calls to business numbers exempt from DNCL Rules (Parts III/IV still apply) | Mode A: B2B, no DNCL scrub needed; still identify + state purpose |
| **Caller identification** | Must identify yourself + purpose at call start | Enforced in script system prompts |
| **Internal DNCL** | Must maintain own do-not-call list; honor within 14 days | `consent_status = none` + internal blocklist table |
| **Vicarious liability** | Client of telemarketer is liable for violations | Partner (agency) is the client — document in partner agreement |
| **ADAD rules** | Automatic Dialing-Announcing Devices (one-way prerecorded) heavily restricted | **Actively under CRTC review** — see ADAD section below. v1 approach: proactive AI disclosure + follow ADAD consent rules as precaution |

### ADAD classification — CRTC Notice of Consultation 2026-132

**This is the most important compliance question for the product.** Researched June 2026.

**Current ADAD definition (UTRs Part I):**
> "ADAD" means any automatic equipment incorporating the capability of storing or producing telecommunications numbers used alone or in conjunction with other equipment to convey a **pre-recorded or synthesized voice message** to a telecommunications number.

**The problem:** The definition says "pre-recorded or synthesized voice message." Conversational AI dynamically generates responses in real-time — it's not delivering a pre-recorded message. However, "synthesized voice message" could be interpreted to cover AI-generated speech. The definition is ambiguous for conversational AI.

**CRTC is actively reviewing this.** On June 11, 2026, the CRTC published [Notice of Consultation CRTC 2026-132](https://crtc.gc.ca/eng/archive/2026/2026-132.htm) — "Review of the Unsolicited Telecommunications Rules." Key questions:

- **Q2:** "Is the [ADAD] definition sufficient to capture software, applications, or technologies that use synthesized voices, recordings, artificial intelligence, or other methods of non-human generated voice messages?"
- **Q10:** "Should the identification requirements be expanded to require telemarketers to tell consumers that the call is using this sort of technology (i.e., it is not a live person making the call and speaking with the consumer at the start of the call)?"
- **Q11:** Should UTRs apply to personal-use AI calls (e.g., calling a restaurant to make a reservation)?

**Deadlines:**
- Intervention submissions: **July 27, 2026**
- Reply submissions: **August 11, 2026**
- Ruling: Unknown (likely fall 2026)

**v1 compliance strategy (precautionary):**

Until the CRTC rules, we operate as if conversational AI **may** be classified as an ADAD. This means:

1. **AI disclosure at call start** — The agent identifies itself as an AI assistant in the opening greeting. This proactively complies with potential Q10 requirements and builds trust.

```python
def _build_greeting(self) -> str:
    biz = self.context.get("business_name", "there")
    city = self.context.get("city", "your area")
    return (
        f"Hi, I'm calling from [Agency Name] using an AI assistant. "
        f"I'm reaching out to {biz} in {city} about helping "
        f"contractors get more booked jobs. Do you have 30 seconds?"
    )
```

2. **Follow ADAD consent rules for Mode B (B2C)** — Only call consumers with express consent (form fill) or existing business relationship. This is already the v1 policy.

3. **Follow ADAD identification rules** — Agent states: (a) who it's calling on behalf of, (b) purpose of the call, (c) callback number or email. This matches Part IV, section 4(d) requirements.

4. **Mode A (B2B) is lower risk** — B2B calls are exempt from DNCL Rules. ADAD rules technically still apply, but the CRTC's Q11 suggests they're considering exempting legitimate business-purpose AI calls. B2B cold calling with AI disclosure is the safest entry point.

5. **Monitor the consultation** — Submit an intervention if beneficial (the user is a Canadian student building a Canadian product — stakeholder perspective is relevant). Track the ruling outcome.

**Still needed:** A brief consultation with a Canadian telecom lawyer to confirm this precautionary approach is sufficient. The CRTC consultation document itself is the best reference to bring — it shows the regulator hasn't decided yet, which means the legal risk is ambiguous but manageable with proactive disclosure.

### Calling hours enforcement

```python
from datetime import datetime, time

CRTC_HOURS = {
    "weekday": (time(9, 0), time(21, 30)),
    "weekend": (time(10, 0), time(18, 0)),
}

def can_call_now(contact_timezone: str = "America/Toronto") -> tuple[bool, str]:
    """Check if current time is within CRTC calling hours."""
    now = datetime.now(ZoneInfo(contact_timezone))
    is_weekend = now.weekday() >= 5  # Sat=5, Sun=6

    hours = CRTC_HOURS["weekend"] if is_weekend else CRTC_HOURS["weekday"]
    current = now.time()

    if hours[0] <= current <= hours[1]:
        return True, "within calling hours"
    return False, f"outside calling hours ({hours[0]}-{hours[1]} {contact_timezone})"
```

### Consent verification (Mode B — consumer calls)

Before dialing a homeowner, verify consent:

```python
async def verify_consent(contact: Contact) -> tuple[bool, str]:
    if contact.consent_status == "express":
        return True, "express consent (form fill)"
    elif contact.consent_status == "existing_relationship":
        return True, "existing business relationship"
    elif contact.consent_status == "unknown":
        # Check if on National DNCL
        if contact.dncl_registered:
            return False, "on National DNCL, no consent"
        # No consent, not on DNCL — still risky, block by default
        return False, "no consent established"
    else:  # none
        return False, "explicitly blocked (internal DNCL)"
```

**v1 policy:** Mode B only calls contacts with `consent_status = express` (form fill) or `existing_relationship`. No cold consumer calling in v1.

### National DNCL checking (consumer mode)

The Canadian National DNCL has a **public API** for checking numbers programmatically — no need to download and maintain the full list.

**Setup:**
1. Partner registers as telemarketer at [lnnte-dncl.gc.ca](https://lnnte-dncl.gc.ca/en/Organization/Register-your-organization) (select "Other" as telemarketer identity)
2. Must have a Dun & Bradstreet number (register if not)
3. Email support@lnnte-dncl.gc.ca with RAN, account manager name + email, requesting API access
4. Receive unique API access key

**DNCL check before dialing (Mode B):**

```python
async def check_dncl(phone: str, dncl_api_key: str) -> bool:
    """Returns True if number is on National DNCL."""
    url = f"https://lnnte-dncl.gc.ca/api/CheckNumber/{phone}"
    headers = {"Authorization": f"Bearer {dncl_api_key}"}
    async with httpx.AsyncClient() as client:
        resp = await client.get(url, headers=headers)
        data = resp.json()
        return data.get("isRegistered", False)
```

**Note:** For v1 Mode B (form-fill consent), DNCL scrub is technically optional — express consent overrides DNCL registration. However, checking DNCL is best practice and provides a due-diligence defense. The compliance module checks DNCL and logs the result regardless, but only blocks the call if `consent_status` is not `express`.

**Alternative:** RealValidito API (RapidAPI) for phone validation + line type detection (landline/mobile/VoIP) + US DNC check. Useful for enriching prospect data and filtering invalid numbers before calling. 100 free credits, then pay-as-you-go.

Every compliance decision is logged to `compliance_log`:

```python
async def log_compliance(contact_id, call_id, action, result):
    await db.execute(
        "INSERT INTO compliance_log (contact_id, call_id, action, result, checked_at) "
        "VALUES (:cid, :callid, :action, :result, now())",
        values={...}
    )
```

### Legal review items (before pilot)

- [x] **ADAD classification:** CRTC is actively reviewing (Notice 2026-132, June 2026). v1 uses precautionary approach: AI disclosure + ADAD consent rules. Still need lawyer to confirm approach is sufficient.
- [ ] **Consult telecom lawyer:** Bring CRTC 2026-132 document. Confirm precautionary AI disclosure approach. ~1-2 hour consultation.
- [ ] Partner registers with National DNCL (obtain RAN + API access key)
- [ ] Partner agreement documents vicarious liability
- [ ] Privacy policy discloses: AI calling, AI disclosure at call start, recording, OpenAI data transit to US, Deepgram data transit to US
- [ ] Call recording consent (one-party consent in Canada — confirmed for Ontario under PIPEDA)
- [ ] Retention policy for recordings + transcripts (90 days default)
- [ ] Monitor CRTC 2026-132 ruling (expected fall 2026) — adjust compliance if definition expands

---

## 9. Spend Tracker

The differentiator GHL doesn't do well. Per-call cost attribution across all providers.

### Cost capture

During and after each call, capture usage from each provider:

| Provider | What's tracked | How | Timing |
|----------|---------------|-----|--------|
| Twilio | Call minutes, number rental | Twilio Usage API + call SID lookup | **Delayed** — Usage API lags by hours. Estimate from duration × rate at call end, reconcile later |
| LiveKit | Agent session minutes | LiveKit Cloud usage API | Near real-time |
| OpenAI | LLM tokens (prompt + completion) | Token counts from API response | Real-time (from response) |
| Deepgram | STT audio minutes | Deepgram usage metadata | Real-time (from response) |
| Cartesia/ElevenLabs | TTS characters | API response metadata | Real-time (from response) |
| Google Places | Search requests | Count per prospector search | Real-time (counted) |

### Twilio cost reconciliation

Twilio's Usage API has a delay (hours, not real-time). The spend tracker handles this in two phases:

1. **At call end:** Estimate Twilio cost from `duration_sec × per_minute_rate`. Write as a spend entry with `raw_usage.estimated = true`.
2. **Reconciliation job (hourly):** Query Twilio Usage API for completed calls by `twilio_call_sid`. Update the spend entry with actual cost. Set `raw_usage.estimated = false`.

```python
async def reconcile_twilio_costs():
    """Hourly job: fetch actual Twilio costs for recently completed calls."""
    calls = await get_calls_with_estimated_twilio_cost(hours_back=24)
    for call in calls:
        actual = await fetch_twilio_call_cost(call.twilio_call_sid)
        if actual:
            await update_spend_entry(
                call_id=call.id,
                provider="twilio",
                cost_cents=actual.cost_cents,
                raw_usage={"estimated": False, **actual.details}
            )
            await update_call_cost_total(call.id)
```

### Billing models

The spend tracker tracks **actual cost** regardless of how the partner bills clients. Two billing models supported:

- **Per-minute:** Client billed at $X/min (e.g., $0.25/min). Margin = billed - actual cost.
- **Flat monthly:** Client billed $X/mo for Y included minutes. Overage at $Z/min. Spend tracker shows actual cost vs. flat fee to show margin.

### Spend entry creation

```python
async def record_spend(call_id, client_id, provider, cost_cents, units, unit_type, raw):
    await db.execute(
        "INSERT INTO spend_entries "
        "(call_id, client_id, provider, cost_cents, units, unit_type, billed_at, raw_usage) "
        "VALUES (:cid, :clid, :prov, :cost, :units, :utype, now(), :raw)",
        values={...}
    )
```

### Cost estimation (per-minute all-in)

| Component | Est. cost/min (CAD) |
|-----------|-------------------|
| Twilio outbound (CA) | ~$0.02–0.03 |
| LiveKit Cloud agent | ~$0.02–0.04 |
| OpenAI GPT-4o (tokens) | ~$0.03–0.05 |
| Deepgram STT | ~$0.004 |
| TTS (Cartesia) | ~$0.01–0.02 |
| **Total all-in** | **~$0.08–0.15/min** |

### Margin model

```
Partner charges client:  $0.25/min  OR  $299/mo for 500 min ($0.60/min)
Your cost (all-in):      $0.12/min
Margin:                  $0.13/min  (52%)
```

### Dashboard views

**Per-call view:**
```
Call: Joe's HVAC — 2026-06-22 14:32
Duration: 4m 12s
Outcome: booked

Cost breakdown:
  Twilio:          $0.11  (4.2 min × $0.026)
  LiveKit:         $0.13  (4.2 min × $0.031)
  OpenAI:          $0.18  (12,400 tokens)
  Deepgram:        $0.02  (4.2 min)
  Cartesia:        $0.06  (1,840 chars)
  ─────────────────────────
  Total cost:      $0.50

Billed to client:  $1.05  (4.2 min × $0.25)
Margin:            $0.55  (52%)
```

**Monthly client summary:**
```
Client: Joe's HVAC — June 2026
Calls made: 47
Total minutes: 198
Total cost: $23.76
Billed: $49.50 (198 min × $0.25)
Margin: $25.74
```

---

## 10. API Surface

Hub REST API (FastAPI). All endpoints require Bearer token auth.

### Auth

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/auth/login` | Agency login (email + password) |
| POST | `/auth/token` | Get JWT access token |

### Clients

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/clients` | List clients (agency's clients) |
| POST | `/clients` | Create client (contractor client or agency prospecting) |
| GET | `/clients/{id}` | Get client detail |
| PATCH | `/clients/{id}` | Update client (Twilio number, etc.) |

### GHL Integration

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/clients/{id}/ghl/connect` | Initiate GHL OAuth flow |
| GET | `/ghl/callback` | OAuth callback (exchange code for tokens) |
| GET | `/clients/{id}/ghl/status` | Check GHL connection status |
| POST | `/clients/{id}/ghl/sync-contacts` | Pull contacts from GHL |
| POST | `/webhooks/ghl` | Receive GHL webhook events |

### Prospector

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/prospector/search` | Search contractors (trade + city + filters) |
| GET | `/prospect-lists` | List saved prospect lists |
| POST | `/prospect-lists` | Save search results as prospect list |
| GET | `/prospect-lists/{id}` | Get prospects in a list |
| POST | `/prospect-lists/{id}/export-to-ghl` | Push prospects to GHL as contacts |

### Campaigns

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/campaigns` | List campaigns |
| POST | `/campaigns` | Create campaign |
| GET | `/campaigns/{id}` | Get campaign detail + stats |
| POST | `/campaigns/{id}/start` | Start campaign |
| POST | `/campaigns/{id}/pause` | Pause campaign |
| GET | `/campaigns/{id}/calls` | List calls in campaign |

### Calls

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/calls` | List calls (filterable by client, campaign, date) |
| GET | `/calls/{id}` | Get call detail (transcript, recording, outcome) |
| GET | `/calls/{id}/recording` | Get recording URL (DO Spaces presigned) |
| GET | `/calls/{id}/transcript` | Get full transcript |

### Script Templates

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/script-templates` | List templates |
| POST | `/script-templates` | Create template |
| GET | `/script-templates/{id}` | Get template |
| PATCH | `/script-templates/{id}` | Update template |

### Spend

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/spend` | Spend entries (filterable by client, provider, date) |
| GET | `/spend/summary` | Aggregated spend (per client, per month) |
| GET | `/clients/{id}/spend` | Spend for a specific client |

### Dashboard

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/dashboard/agency` | Agency overview (all clients, total spend, calls) |
| GET | `/dashboard/clients/{id}` | Per-client overview |

### Twilio Numbers

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/twilio/numbers` | List available/purchased Canadian numbers |
| POST | `/twilio/numbers` | Purchase a Canadian DID |
| GET | `/twilio/numbers/{sid}` | Get number details |
| DELETE | `/twilio/numbers/{sid}` | Release a number |

### Compliance

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/compliance/logs` | Compliance audit log (filterable by contact, call, action) |
| GET | `/compliance/dncl-check/{phone}` | Check a number against National DNCL |
| GET | `/compliance/calling-hours` | Check if current time is within CRTC calling hours |

### Webhooks (inbound)

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/webhooks/ghl-workflow` | GHL workflow outbound webhook (Mode B triggers) |
| POST | `/webhooks/livekit` | LiveKit Cloud webhook (room/participant/egress events) |

---

## 11. Deployment & Infrastructure

### Hosting: DigitalOcean Toronto (TOR1)

| Resource | Spec | Est. monthly cost |
|----------|------|------------------|
| App Platform (hub API) | Basic tier, 1GB RAM | ~$10–12 |
| Managed PostgreSQL | 1GB RAM, 10GB disk | ~$15 |
| Spaces (object storage) | 250GB + bandwidth | ~$5 |
| Droplet (LiveKit agent worker) | 2GB RAM, 1 vCPU | ~$12 |
| **Total infra** | | **~$42–44/mo** |

Plus usage-based:
- LiveKit Cloud: per agent-minute
- Twilio: per minute + number rental (~$1/number/mo)
- OpenAI/Deepgram/Cartesia: per usage
- Google Places: per search

### Environment variables

```env
# Database
DATABASE_URL=postgresql://...
# No Redis in v1 — APScheduler runs in-process. Add REDIS_URL in v2 for Celery.

# LiveKit
LIVEKIT_URL=wss://your-project.livekit.cloud
LIVEKIT_API_KEY=...
LIVEKIT_API_SECRET=...
LIVEKIT_SIP_TRUNK_ID=...

# Twilio
TWILIO_ACCOUNT_SID=...
TWILIO_AUTH_TOKEN=...
TWILIO_SIP_TRUNK_DOMAIN=...

# OpenAI
OPENAI_API_KEY=...

# Deepgram
DEEPGRAM_API_KEY=...

# TTS
CARTESIA_API_KEY=...  # or ELEVENLABS_API_KEY

# Google Places
GOOGLE_PLACES_API_KEY=...

# GoHighLevel
GHL_CLIENT_ID=...
GHL_CLIENT_SECRET=...
GHL_WEBHOOK_SHARED_SECRET=...  # for workflow outbound webhooks
GHL_PUBLIC_KEY=...  # Ed25519 public key for OAuth app webhook verification (v2)

# National DNCL
DNCL_API_KEY=...  # from lnnte-dncl.gc.ca API registration

# Apify (optional — alternative to Google Places)
APIFY_API_TOKEN=...

# Hub
HUB_URL=https://app.crmandmore.ca
JWT_SECRET=...
ENCRYPTION_KEY=...  # for GHL tokens
```

### Project structure

```
crmandmore/
├── docs/
│   └── SPEC-v1.md          # this document
├── hub/                    # FastAPI application
│   ├── main.py             # app entry
│   ├── config.py           # settings
│   ├── db.py               # database connection
│   ├── models/             # SQLAlchemy models
│   ├── routers/            # API route handlers
│   │   ├── auth.py
│   │   ├── clients.py
│   │   ├── prospector.py
│   │   ├── campaigns.py
│   │   ├── calls.py
│   │   ├── ghl.py
│   │   └── spend.py
│   ├── services/           # business logic
│   │   ├── ghl_client.py
│   │   ├── places_client.py
│   │   ├── compliance.py
│   │   ├── spend.py
│   │   └── campaign_scheduler.py
│   └── templates/          # dashboard (Jinja or minimal SPA)
├── agent/                  # LiveKit voice agent worker
│   ├── worker.py           # LiveKit agent entrypoint
│   ├── scripts/            # script templates
│   └── outcome.py          # post-call classification
├── migrations/             # Alembic migrations
├── tests/
├── pyproject.toml
└── docker-compose.yml      # local dev
```

---

## 12. Phasing & Roadmap

### v1 — Voice + Prospector Wedge (90 days)

| Weeks | Deliverable | Revenue signal |
|-------|------------|----------------|
| 1–2 | Prospector: Places API + Mississauga trade searches + results UI | Partner searches contractors |
| 3–4 | Voice engine: LiveKit + Turn Det v1 + Twilio CA + Script A | First test call works |
| 5–6 | GHL OAuth + push prospects + post-call sync | Prospects land in GHL |
| 7–8 | Campaign queue, rate limits, calling hours, spend tracker | Batch prospecting campaigns |
| 9–10 | GHL webhook trigger + Script B (client speed-to-lead) | Offer voice to 1 client |
| 11–12 | Polish, recordings dashboard, first paying client | Sell to 2–3 contractor clients |

### v2 — Agency Platform Foundation

- Multi-agency self-serve signup
- White-label portal
- Client billing + rebilling (Stripe)
- Email/SMS campaigns (Resend + Twilio)
- Meta Lead Ads sync
- Expanded Ontario cities (GTA, Ottawa, Hamilton, London)
- US market (TCPA compliance module)

### v3 — Replace GHL Modules

- Frappe CRM per-client sites (provisioning via bench)
- ERPNext invoicing on Growth tier
- Workflow automation (visual builder or n8n)
- Vertical snapshots (contractor, dental)
- Full spend dashboard with margin/rebilling UI

### v4 — Dental / PHIPA Tier

- Canadian-only hosting zone (PHI-safe)
- PHIPA compliance: audit logs, access controls, MFA, encryption
- Dental snapshot: recall automation, appointment booking
- PHIPA-safe voice pipeline (Canadian LLM/STT/TTS or consent framework)
- Partner lane for dental friend

### v5 — Full CRMAndMore Platform

- Package tiers (Starter, Growth, Pro, Enterprise)
- Full ERPNext module selection per package
- CAPI / conversion bridge (closed-loop ad optimization)
- Ad campaign dashboard (Meta + Google aggregation)
- Custom vertical apps

---

## 13. Open Questions

Items resolved during spec review (June 2026):

- [x] **TTS provider:** **Cartesia Sonic-3.5** — ~$0.02/min (8x cheaper than ElevenLabs at $0.17/min), 40ms TTFB. Utility TTS doesn't need ElevenLabs' premium realism. See §6 TTS comparison.
- [x] **ADAD classification:** **CRTC is actively reviewing** (Notice of Consultation 2026-132, published June 11, 2026). Current definition ambiguous for conversational AI. v1 uses precautionary approach: AI disclosure at call start + follow ADAD consent rules. See §8 ADAD section. Still need lawyer consultation.
- [x] **Call recording consent:** Canada is one-party consent under PIPEDA — confirmed for Ontario. Document in privacy policy.
- [x] **Deepgram data region:** **No Canada endpoint.** EU endpoint available (api.eu.deepgram.com). US default. SOC2/HIPAA/GDPR compliant. v1 uses US endpoint (no PHI), document in privacy policy. See §3 data residency table.
- [x] **LiveKit Cloud region:** **No Canada region.** Three regions: us-east (Virginia), eu-central (Frankfurt), ap-south (Mumbai). v1 uses us-east (closest to Toronto). v2: self-host on DO Toronto for full data residency. See §3 data residency table.
- [x] **Apify vs Places API:** **Google Places API for v1.** Google ToS prohibits scraping — ToS compliance is non-negotiable for a commercial product. Cost is comparable (~$2/1,000 vs ~$2.50/1,000). Can approximate `claimedBusiness` by checking `websiteUri` presence.
- [x] **Voicemail detection:** Twilio AMD doesn't work with SIP — using STT-based pattern detection. Heuristic approach with regex patterns on first utterance. Refine during pilot.
- [x] **Redis:** Dropped for v1 — APScheduler runs in-process, Postgres-backed queue.
- [x] **GHL app type:** Private Integration for v1 (fast), Marketplace app for v2.
- [x] **Dashboard approach:** Server-rendered (Jinja) for v1 — simplest, upgrade to SPA if needed.
- [x] **National DNCL:** Use official DNCL API (lnnte-dncl.gc.ca) for real-time checks — no full list download needed.
- [x] **OpenAI data transit:** Documented in privacy policy + data residency table (§3). Business data transits to US, no PHI in v1.
- [x] **GHL custom field IDs:** Documented in §7 — must be pre-created in GHL Custom Fields settings, IDs fetched via `GET /custom-fields/` API, mapped in hub config per location.

Items still open (require user action, not research):

- [ ] **Lawyer consultation:** 1-2 hour consult with Canadian telecom lawyer. Bring CRTC 2026-132 document. Confirm precautionary AI disclosure approach is sufficient. (~$300-500 CAD)
- [ ] **Partner agreement:** Draft and sign before first paying client. Covers revenue share (25-30%), vicarious liability, data ownership.
- [ ] **Partner DNCL registration:** Partner registers as telemarketer at lnnte-dncl.gc.ca, obtains RAN + API access key. Requires Dun & Bradstreet number.
- [ ] **CRTC 2026-132 monitoring:** Intervention deadline July 27, 2026. Ruling expected fall 2026. Adjust compliance if ADAD definition expands to cover AI.
- [ ] **Voicemail pattern refinement:** Test STT-based voicemail detection during pilot, refine regex patterns based on false positives/negatives.

---

## Appendix A: Key Research References

- **LiveKit Turn Detector v1:** https://livekit.com/blog/solving-end-of-turn-detection (June 17, 2026)
- **LiveKit Agents docs:** https://docs.livekit.io/agents/build/turns/turn-detector/
- **LiveKit webhooks:** https://docs.livekit.io/intro/basics/rooms-participants-tracks/webhooks-events/
- **GoHighLevel API:** https://marketplace.gohighlevel.com/docs/
- **GHL webhook integration guide:** https://marketplace.gohighlevel.com/docs/webhook/WebhookIntegrationGuide/index.html
- **GHL workflow outbound webhook:** https://help.gohighlevel.com/support/solutions/articles/155000003299-workflow-action-webhook-outbound-
- **Google Places API billing:** https://developers.google.com/maps/documentation/places/web-service/usage-and-billing
- **Apify Google Maps Scraper:** https://apify.com/solidcode/google-maps-scraper-2-5-per-1-000-results
- **CRTC telemarketing rules:** https://crtc.gc.ca/eng/phone/telemarketing/biz.htm
- **CRTC UT Rules:** https://crtc.gc.ca/eng/trules-reglest.htm
- **CRTC Notice of Consultation 2026-132 (ADAD/AI review):** https://crtc.gc.ca/eng/archive/2026/2026-132.htm
- **Canadian National DNCL API:** https://lnnte-dncl.gc.ca/en/Organization/DNCL_API
- **RealValidito DNC lookup:** https://www.realvalidito.com/dnc-lookup/
- **Twilio AMD (does NOT work with SIP):** https://www.twilio.com/docs/voice/answering-machine-detection-faq-best-practices
- **LiveKit SIP voicemail detection issue:** https://github.com/livekit/sip/issues/403
- **LiveKit + Twilio SIP:** https://docs.livekit.io/telephony/start/providers/twilio/
- **LiveKit outbound calls:** https://docs.livekit.io/agents/quickstarts/outbound-calls/
- **Cartesia pricing:** https://www.cartesia.ai/pricing
- **ElevenLabs pricing:** https://elevenlabs.io/pricing
- **Deepgram data privacy compliance:** https://developers.deepgram.com/trust-security/data-privacy-compliance
- **LiveKit Cloud regions:** https://livekit.com/products/agent-cloud-deployment
- **Apify Google Maps Scraper:** https://apify.com/solidcode/google-maps-scraper-2-5-per-1-000-results

---

*This spec is a living document. Update as decisions are made and pilot feedback comes in.*
