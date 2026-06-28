# URL Shortener (TinyURL / Bitly)

**Difficulty:** Foundation
**Category:** Storage-heavy, Read-heavy
**Time Box:** 45 min
**Key Concepts:** Key-value storage, Base62 encoding, caching, redirect semantics, async analytics pipelines, back-of-envelope estimation
**Asked at:** Google, Amazon, Meta, Microsoft, PayPal

---

## Problem Statement

Your company wants to launch a URL shortening service. Marketing teams embed short links in SMS campaigns, printed materials, and social media posts where character count matters and link readability matters more than the full destination URL. Product teams want click analytics so campaign managers can measure engagement.

The product needs to be simple on the surface: a user pastes a long URL, gets back a short one (e.g., `https://sho.rt/x7kP2q`), and anyone who clicks the short link gets redirected to the original. Behind that simplicity is a system that must handle read traffic that dwarfs write traffic by orders of magnitude — every link created may be clicked thousands of times, and some go viral and spike unpredictably.

For context on real-world scale: Bitly processes roughly 256 million link creations per month and 10 billion redirects per month as of January 2026. The redirect path is the hot path — it runs orders of magnitude more frequently than creation, and any latency there directly damages the user experience of whoever clicked the link.

---

## Actors

| Actor | Description | Primary Action |
|-------|-------------|----------------|
| **Creator** | User or API client creating a short link | POST /shorten |
| **Visitor** | End user clicking the short link | GET /:shortCode |
| **Analytics Consumer** | Marketing team reading click reports | GET /analytics/:shortCode |
| **Admin** | Internal tooling / abuse management | DELETE, expire |

---

## Functional Requirements

**Core (MVP):**
- FR1: Given a long URL, generate a unique short code and return the full short URL
- FR2: Given a short code, redirect the visitor to the original long URL
- FR3: Short codes must be unique — two different long URLs must never resolve to the same short code

**Secondary:**
- FR4: Support custom aliases (user-chosen short code, e.g., `/myproduct`)
- FR5: Support expiry — links can be set to expire after a given date or after N clicks
- FR6: Track click events per short code (timestamp, referrer, user-agent, approximate geo)
- FR7: Provide per-link analytics (click count, click-over-time, top referrers)
- FR8: Allow link deletion / deactivation by the owner

---

## Non-Functional Requirements

| Requirement | Target | Rationale |
|-------------|--------|-----------|
| Redirect latency (p99) | < 50 ms | Visitor perceives redirect as part of page load; Bitly targets < 30 ms |
| Availability | 99.99% (< 52 min/year downtime) | Links embedded in printed materials can't be retried |
| Write latency (p99) | < 200 ms | Creation is infrequent; user can tolerate a short wait |
| Short code length | 6–8 characters | Balances URL brevity vs. collision probability |
| Durability | No link loss after creation | A link disappearing after being printed is catastrophic |
| Read:write ratio | ~100:1 | Informs caching strategy — reads dominate completely |
| Analytics delivery | Eventually consistent, < 5 min lag | Async is acceptable; real-time analytics not required |

---

## Capacity Estimation Hints

Use these assumptions to derive your own numbers before looking at the solution guide:

- **Monthly link creations:** 100 million new short URLs/month
- **Monthly redirects:** 10 billion clicks/month
- **Average long URL size:** ~200 bytes; short code metadata record: ~500 bytes total
- **Retention:** Links stored for 5 years by default
- **Peak traffic:** Viral links can spike to 50–100× average redirect rate
- **Geographic distribution:** Global traffic, no single dominant region

Derive: daily write QPS, daily read QPS, peak read QPS, 5-year storage, bandwidth.

---

## Good Clarifying Questions to Ask

### Scale
1. How many new links per day? (drives short code namespace size and write QPS)
2. What is the read-to-write ratio? (drives caching decisions — if 100:1, cache is not optional)
3. Do we need to handle viral spikes, and what's the expected multiplier? (drives peak capacity planning)

### Features
4. Do users need custom aliases? (changes generation strategy — must check for conflicts)
5. Do links expire? If so, on time or on click count? (adds TTL management complexity)
6. What analytics do we need — just click counts, or breakdown by referrer / geo / device? (drives analytics pipeline complexity)

### Consistency
7. If a link is created, does the creator need to immediately be able to share it, or is eventual consistency acceptable? (drives replication topology)
8. Is it acceptable to count a click twice under failure? (drives exactly-once vs at-least-once semantics for analytics)

### Constraints
9. Any compliance requirements — GDPR, CCPA? (affects what data we log in analytics, retention policies)
10. Do we need to detect and block malicious redirect targets (phishing, malware)? (adds URL scanning pipeline)

---

## Key Components (hints)

<details>
<summary>Reveal components</summary>

- **API Service** — handles `/shorten` and `/:shortCode` HTTP endpoints
- **Short Code Generator** — converts a numeric ID or hash into a Base62 string
- **URL Database** — stores `{shortCode → longURL}` mapping durably; key-value access pattern
- **Cache Layer** — sits in front of DB on the read path; essential for redirect latency SLA
- **Analytics Ingestion Pipeline** — async path; click events → queue → analytics store
- **Analytics Query Service** — serves aggregated click data
- **ID Generator Service** (optional) — distributed counter or Snowflake-style unique ID generation
- **CDN / Edge** (advanced) — cache redirect responses at the edge for the hottest links

</details>
