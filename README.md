# Pubky Decentralized Indexer Anti-Spam: Phased Proposal

Status: draft for team discussion (revised 2026-11-25).
How to participate: open an Issue to comment, or edit this file and submit a Pull Request to propose changes directly.
References: [Ballast proposal](https://github.com/pubky/ballast/blob/main/docs/proposal.md)

## Problem

Anyone can generate a Pubky key for free and run a homeserver. As indexing decentralizes, every indexer must stay fast, bounded in storage, and resistant to spam without relying on a single central gatekeeper. The hardest edge case is an honest newcomer on a new or self-hosted homeserver with zero follows who wants to reach strangers.

Two indexer shapes hit this problem differently:

- **Pull (full) indexer:** chooses which homeservers and users to read, then decides what to store and what to surface.
- **Scoped indexer + inbox:** indexes only one user's circle (follows plus N hops). Because it never polls outsiders, it needs a way to learn about replies, mentions, reposts, or follows from outside the circle. Today's Light Nexus PR stack (`#1175`) bridges these by polling a big Nexus. Replacing that bridge later requires an open homeserver inbox ("ping me to index this interaction"), which is an unauthenticated write surface anyone can flood.

## Core Principle

**Gate both ingestion/storage and display/reach.**

We cannot store everything forever and filter only at display time. Unbounded ingestion lets a single noisy account exhaust Neo4j, Redis, disk, and media transcoding, or stall co-hosted users. Instead:

1. **Ingestion and storage** are bounded by per-user and per-homeserver resource budgets (fair queuing, rate caps, and retention limits).
2. **Display and reach** are governed by the social graph (Web of Trust) and viewer controls.

## Current Baseline (What Actually Exists Today)

Before designing new anti-spam layers, here is where the codebase actually stands:

- **`pubky-nexus` external watcher:** `monitored_homeservers_limit` (default 50) is a global cap on total third-party homeservers polled per cycle, not a per-server quota, and `key_based_events_limit` (default 50, max 100) caps events per user per run.
- **`pubky-nexus` primary homeserver and storage:** reads a single sequential `/events` cursor (`events_limit` 50 in default config, max 1000) with no per-user isolation, no run budget, and no homeserver backoff (`nexus-watcher/src/service/runner/homeserver.rs`, `nexus-watcher/src/service/indexer/homeserver.rs`). There are zero per-user write/tag quotas, Neo4j resource tags are merged unconditionally, PR `#1189`'s `Event:ResourceChanges` Redis list has no trim/maxlen cap, and foreign/resource-tag moderation in `nexus-watcher/src/events/moderation.rs` is currently a no-op.
- **`pubky-nexus` trust and homeserver ordering:** `[trust_rank]` (Personalized PageRank from configured seeds) and `hide_unranked_authors` exist in code, and external homeservers sort by aggregate hosted trust then active user count. However, `seed` ships empty and filtering ships off, so there are zero active homeserver tiers or scoring today (unseeded ordering falls back to user count, letting Sybil servers outrank small self-hosted servers).
- **`pubky-homegate` and `pubky-homeserver`:** Homegate verifies via SMS (Prelude), Lightning (`phoenixd`), Google, Apple, and IP, and returns an opaque (not blind-signed) one-time signup token for a single configured homeserver without seeing the user's Pubky key at issuance. However, the homeserver records which key redeemed each token (`mark_token_used(&token, &used_by)`), so an operator running both services can already link verification to key. Homegate has no signing keys, no portable attestations, and no revocation system, and `pubky-homeserver` has no inbox endpoint yet.

## Phased Priority Ladder

Build in order of real risk. Do not build speculative anti-abuse infrastructure before the underlying attack exists.

| Priority | When to Build | What Ships | Explicitly Not Built Yet |
|---|---|---|---|
| **Priority 1: Build Now** | Now (real gaps today) | Per-user/HS ingestion quotas & bounded queues; seeded WoT reach & viewer controls; Scoped Inbox v1 (free tier + first-contact + flood-safe held queue); dedicated lane for legitimate bots/curators | Homeserver tiering/scoring, Homegate attestations, PoW stamps |
| **Priority 2: Triggered** | Only when external-HS or open-inbox spam actually appears | WoT-weighted homeserver standing (without breaking self-hosting); pluggable recipient-chosen inbox admission signals (WoT; day-one paid signal: small non-refundable Lightning fee to recipient's homeserver operator; direct recipient-wallet payment later once online-receive/custody is solved; optional blind-signed/unlinkable attestations) | SHA-256d miner-marketplace brokerage, cross-indexer reputation protocols |
| **Priority 3: Deferred** | Only if real attack data proves Priority 1 and 2 fail | Ballast SHA-256d stamps via the pluggable inbox/admission slot; cross-indexer reputation sharing / federated blocklists | Everything in this tier stays deferred |

---

### Priority 1: Build Now (Problems We Actually Have Today)

1. **Ingestion and storage protection in Nexus**
   - Add per-user fair queuing and event/tag quotas on both primary and external homeservers. Today, when the external watcher exhausts `KEY_BASED_RUN_BUDGET` (8 minutes), it backs off the entire homeserver for up to 1 hour, throttling innocent co-hosted users alongside a single noisy tenant. Per-user isolation stops one flooder from stalling the shared cursor or triggering whole-homeserver backoff.
   - Bound internal queues and storage: cap/trim change feeds (such as `#1189`'s Redis list), enforce per-author storage/tag budgets, prune or cap unfollowed/demoted noise (Light Nexus PR `#1173` stops polling demoted users but retains their already-indexed content), and close the foreign/resource-tag moderation gap in `nexus-watcher/src/events/moderation.rs`.
2. **Social graph (WoT) and viewer controls**
   - Configure and enable `[trust_rank]` Personalized PageRank for discovery and external homeserver polling priority (preventing empty-seed Sybil servers from crowding out real homeservers under `monitored_homeservers_limit`).
   - Support follow-distance reach and vouching by follow (using PageRank attenuation or rate-limited vouching rather than raw binary single-follow unlocks, so one compromised account cannot launder a spam farm).
   - Route stranger replies and mentions beyond N hops into a collapsed "people you don't know" bucket, alongside viewer mute, block, and configurable stranger thresholds.
3. **Scoped inbox v1 (when moving past `#1175`'s big-Nexus bridge)**
   - **Free tier first:** allow N pings per hour per inbox with zero friction so normal users never hit a wall.
   - **First contact only:** once the recipient replies or follows back, the sender enters the recipient's graph and the inbox no longer gates them.
   - **Flood-resistant held queue:** excess or untrusted pings land in a capped held queue, but do **not** use naive FIFO ("oldest dropped"), which lets one flooder trivially flush honest strangers out of the queue. Use per-sender/per-subnet caps and stratified buckets (or weighted eviction), plus strict SSRF and timeout guards before fetching a stranger's homeserver.
4. **Explicit lane for legitimate high-volume bots and curators**
   - Bots and curators (such as resource seeders) legitimately produce bursty, link-heavy traffic. Give them opt-in, isolated per-key quotas and graph-based standing so they do not trip naive burst/link-flood filters, stall human streams, or drag down co-hosted users on the same homeserver.

---

### Priority 2: Build Only When Triggered (Real External-HS or Inbox Spam)

Ship these only when observed spam on third-party homeservers or open inboxes outgrows Priority 1.

1. **Homeserver standing without the email oligopoly trap**
   - Email solved spam with IP/domain reputation, which centralized deliverability around Gmail and Microsoft and made self-hosting painful. Pubky exists for credible exit, so new self-hosted homeservers must not be trapped in permanent purgatory.
   - Isolate and cap individual abusive keys **before** downgrading their homeserver; reserve server-level downgrades for systemic multi-key abuse.
   - Weight mute and abuse reports strictly by WoT / TrustRank so coordinated mobs cannot brigade rival or small self-hosted servers.
   - Keep a fast, frictionless vouch path so a single trusted follow (or small cluster of follows) lifts an honest self-hosted user immediately, independent of their homeserver's age.
2. **Pluggable, recipient-chosen inbox admission signals (above the free tier)**
   - Because a one-time per-key credential lets one verified spammer ping every inbox once, inbox gating above the free tier must be checked per contact (or paired with strong per-contact cost/limits). The recipient's homeserver policy chooses which signals it accepts:
     1. **Graph / WoT distance (free):** sender is within N hops of the recipient or followed by a trusted key. No third party required, though zero-follow strangers have none.
     2. **Direct per-contact Lightning payment (day-one paid signal: homeserver anti-spam fee; recipient-wallet payment later):** on day one when Priority 2 triggers, ship a small non-refundable Lightning anti-spam fee paid directly to the recipient's homeserver operator. This works immediately on existing online server infrastructure (such as `phoenixd`), requires no new wallet setup for users, and avoids holding funds on their behalf. Paying the recipient's own wallet directly follows later once its architectural blocker is solved: receiving Lightning directly requires either an always-online receiving wallet for the recipient (e.g. an LSP / async-receive setup) or opt-in homeserver custody.
     3. **Privacy-preserving signup/reputation attestations (optional, e.g. Homegate):** only adopt if built with blind signatures (Privacy Pass style) or another unlinkable construction and short epoch lifetimes, and only if clients and homeservers avoid locking everyone onto a single default gatekeeper. Today, Homegate issues opaque one-time signup tokens that only an operator running both Homegate and the homeserver can correlate (`mark_token_used(&token, &used_by)`). Publishing standard key-bound signed attestations on a user's homeserver would regress privacy by exposing that verification-to-key link publicly to everyone, while also recreating a central gatekeeper by default bias.
   - Note: lightweight client-side PoW is excluded as a standalone anti-spam option; it is only a trivial rate-limit speed bump and effectively free for botnets.

---

### Priority 3: Explicitly Deferred (Problems We Do Not Have Yet)

1. **Ballast SHA-256d miner-marketplace stamps**
   - **Fair representation:** [Ballast](https://github.com/pubky/ballast/blob/main/docs/proposal.md) proposes an issuer-less Sybil cost floor designed to sit alongside free tiers and WoT. Users pay Bitcoin miners to compute SHA-256d work committed to a 32-byte subject (a public key, or a domain-tagged hash such as `hash(domain || sender || recipient)` for single-use/per-contact gating). Any verifier checks a proof offline in 3 SHA-256d hashes with zero trusted issuer. (Note: the miner sees the subject being stamped, so purchase privacy depends on the payment method rather than blind signatures.)
   - **Why we are not building it now:** SHA-256d proofs rely on rented hash-rate marketplaces (only two, Braiins Hashpower and NiceHash, sell to arbitrary Stratum pools today) or dedicated mining hardware. Direct single-user rental is slow (~15+ minutes due to marketplace minimum bid and cancellation windows), while fast delivery (~30 seconds for a $1 proof at 100 PH/s) requires operating continuous pooled miner services with automated bidding, idle-hash management, price discovery, and failure/refund handling. A reusable per-key stamp also does not stop per-contact inbox fanout on its own, whereas a small non-refundable Lightning anti-spam fee to the recipient's homeserver operator already prices per-contact admission on day one without miner brokerage.
   - **Door stays open:** because Priority 2 treats inbox admission as a pluggable policy slot, a per-contact Ballast proof can slot in later if real attack data ever proves Priority 1 and Priority 2 insufficient.
2. **Cross-indexer reputation sharing and federated blocklist protocols**
   - Premature until multiple independent indexers run in production with real abuse data to exchange.

---

## Open Questions

### Decide Now (Priority 1)

1. **Ingestion quotas and queue bounds:** What per-user event/tag rates, per-homeserver caps, and Redis/Neo4j retention limits (including `#1189` trimming and Light Nexus demotion pruning) should ship first?
2. **Bot and curator lane:** How does a pull indexer identify and isolate legitimate high-volume bots (e.g. opt-in curator allowlist, dedicated quota class, or TrustRank-scaled limits)?
3. **Seeded TrustRank defaults:** Who bootstraps the default `[trust_rank]` seed set on Synonym's Nexus, and what minimum score or probation window opens global discovery?
4. **Stranger replies UX:** What is the default hop limit N for inline replies/mentions vs. the collapsed stranger bucket, and can viewers adjust it?
5. **Scoped inbox v1 mechanics:** What is the free-tier limit N (per recipient, per sender, or both), what held-queue size and anti-flush eviction rule do we use, and when do we build the homeserver inbox to replace `#1175`?

### Defer Until Triggered (Priority 2 and 3)

1. **Direct Lightning inbox payment spec:** What default sat amount to charge for the day-one non-refundable homeserver anti-spam fee, and how to later add direct recipient-wallet payments (always-online recipient wallet / LSP async-receive vs. opt-in homeserver custody).
2. **Blind-signed Homegate attestations:** If needed later, what blind-token format and epoch length preserve unlinkability without exposing verification-to-key links publicly or building a central revocation registry, and how do we prevent single-issuer default lock-in?
3. **Well-known storage path:** Where on a user's homeserver should optional proofs or attestations live so indexers and inboxes can discover them consistently?
4. **Ballast trigger threshold:** What concrete inbox or pull-indexer attack breaks Priority 1 and Priority 2 in practice and would justify operating SHA-256d miner-pool infrastructure?
