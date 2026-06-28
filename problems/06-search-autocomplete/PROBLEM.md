# Search Autocomplete / Typeahead
**Difficulty:** Intermediate
**Category:** Read-Heavy / Latency-Sensitive
**Time Box:** 45 min
**Key Concepts:** Trie, Top-K, Distributed Prefix Index, Caching, Debounce
**Asked at:** Google, Amazon, Uber, Twitter/X, LinkedIn, Microsoft

## Problem Statement

Your team is building the search bar for a large consumer platform — think a product search on Amazon or the query box on Google. As users type, the system must surface relevant completions within milliseconds. A user who types "iph" should see "iphone 15", "iphone charger", "iphone case" before they finish the word.

The system must handle traffic that spikes heavily during shopping events (Black Friday, product launches). Autocomplete is on the critical path of every search session — if it goes down or becomes slow, users perceive the entire product as broken. Latency, not throughput, is the primary constraint.

Personalization is a future concern but not MVP. For now, completions rank by global popularity — the queries that most users actually submit.

## Actors
| Actor | Description |
|-------|-------------|
| End User | Types queries in a search bar; expects sub-100ms completions |
| Content Indexer | Ingests search logs and updates query popularity scores |
| Admin | Manages blocklist (offensive queries), forced completions |

## Functional Requirements
**Core (MVP):**
- FR1: Return top-5 query completions for any prefix the user types
- FR2: Rank completions by global query frequency (most-searched first)
- FR3: Completions must appear within 100ms of the keystroke (end-to-end, p99)
- FR4: Update the index within 24 hours of a trending query rising in popularity

**Secondary:**
- FR5: Support blocklist — certain queries must never appear as completions
- FR6: Support exact-match boost — promoted queries appear regardless of frequency
- FR7: Personalized completions based on user's own search history (Phase 2)

## Non-Functional Requirements
| Requirement | Target | Rationale |
|-------------|--------|-----------|
| Latency | < 100ms p99 end-to-end | User perception threshold; beyond this, autocomplete feels laggy |
| Availability | 99.99% | Search bar is always visible; outage = perceived product failure |
| Read QPS | 100,000 RPS sustained | 10M DAU × ~10 searches/day × ~10 keystrokes/search ÷ 86,400s |
| Write (index updates) | Eventually consistent, 24h lag acceptable | Popularity shifts slowly; real-time not required |
| Index freshness | Trending queries visible within 24 hours | Black Friday queries must surface same day |

## Capacity Estimation Hints
- DAU: 10 million users
- Average searches per user per day: 10
- Average keystrokes per search before selecting: 10
- Peak multiplier during events: 5×
- Average query length: 20 characters
- Top-K: return top 5 completions per prefix
- Unique queries in index: ~1 billion (Google-scale); ~10 million (mid-size platform)
- Average query size in storage: 30 bytes + 8 bytes frequency = ~38 bytes

## Good Clarifying Questions to Ask

**Scale:**
- How many DAU? How does traffic spike during events (Black Friday, product launches)?
- How many unique queries are in the corpus — millions or billions?
- What is the acceptable tail latency? p99 < 100ms, or stricter?

**Features:**
- Is personalization in scope, or global popularity only?
- Do we need to support multiple languages or just English?
- Is there a blocklist requirement for offensive or brand-unsafe completions?

**Consistency:**
- How quickly must a trending query appear in completions? Real-time or eventual?
- What happens if the autocomplete service is down — silent degradation or error?

**Constraints:**
- How many results per query? Top 5 is standard; anything different?
- Must completions be prefix matches only, or do we need fuzzy/infix matching?

## Key Components
<details><summary>Reveal components</summary>

- **Autocomplete Service**: Stateless API servers that serve prefix lookups
- **Distributed Trie / Prefix Index**: Core data structure holding (prefix → top-K queries)
- **Redis Sorted Sets**: Cache layer; prefix as key, completions as sorted members by score
- **Search Log Aggregator**: Kafka + Spark/Flink pipeline that recomputes top-K per prefix daily
- **Trie Builder**: Offline job that rebuilds the prefix index from aggregated frequencies
- **CDN / Edge Cache**: Caches common prefixes (single letters, "ip", "iph") at the edge
- **Blocklist Service**: Filter layer that strips offensive completions before returning
</details>
