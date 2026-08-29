# ADR-008 : Retirement of SearXNG from Kiwinet Infrastructure

**Date** : 29 August 2026
**Status** : Accepted
**Context** : Self-hosted SearXNG instance on Scaleway VPS, publicly exposed via `searxng.kiwinet.me`

---

## Problem Statement

SearXNG instance suffered from systematic blocking by major search engine backends (Google, DuckDuckGo, Brave, Startpage) due to IP reputation issues inherent to datacenter hosting, combined with public exposure enabling third-party scraping abuse.

**Search results degraded to StackExchange, GitHub, Wikipedia exclusively** — all services using official APIs rather than HTML scraping.

---

## Root Cause Analysis

Two cumulative factors:

1. **Configuration vulnerability** : rate limiter (`server.limiter`) was disabled in `settings.yml` on a publicly exposed instance without authentication. This enabled unidentified third parties to use the instance as an anonymous scraping proxy, triggering cascading bans on Google/Brave/Startpage/DuckDuckGo.

2. **Structural datacenter IP reputation** : SearXNG official documentation confirms that instances hosted on datacenter IPs (Scaleway, OVH, AWS, etc.) are inherently suspicious to anti-bot systems of these engines — independent of configuration corrections. This is an architectural trade-off of self-hosting a meta-search engine that scrapes third-party services without official API access.

---

## Mitigation Attempts (Retained)

Applied corrections to `homeserver.yaml` and persisted in Git:

- Enabled rate limiter (`server.limiter: true`) with Redis connection
- Configured `trusted_proxies` in `limiter.toml` (`172.19.0.0/16`) to recognize Docker network behind Traefik reverse proxy
- Migrated to new flexible IP on Scaleway (clean reputation slate)

**Result** : Same backends re-blocked within minutes on new IP, confirming blocking is **ASN/hosting-provider level**, not IP-specific.

---

## Options Considered & Rejected

| Option | Rationale for Rejection |
|--------|-------------------------|
| WireGuard tunnel to home Freebox ISP IP | Architectural disproportionality: a service requiring a second VM/tunnel to function normally contradicts engineering simplicity principles |
| Commercial residential proxy | Reintroduces third-party trust boundary (proxy observes search queries), antithetical to SearXNG privacy goal; rejected for consistency with zero-unnecessary-dependencies principle |
| Accept degraded results | Usage became unreliable; cannot recommend tool that returns only StackExchange for most queries |

---

## Decision

**Retire SearXNG from daily active stack.**

Replacement: **Brave Search** — independent index, no Google/Bing dependency, no account required — accessed via privacy-neutral browser containers:
- **GrapheneOS** : Vanadium browser (already in use)
- **Debian desktop** : Chromium official package with privacy policy enforced (`/etc/chromium/policies/managed/privacy.json`): Brave Search pinned as default, sync disabled, telemetry disabled, search suggestions disabled

---

## Implementation

**27 August 2026** :
- Shut down SearXNG on VPS (`docker compose down`)
- Removed DNS record `searxng.kiwinet.me` from Bluehost
- Created archive branch `archive/searxng-retired-20260829` preserving full history
- Removed `searxng/` directory from main branch

**Result** : VPS freed ~152 MB RAM (SearXNG + Redis), now hosts Synapse + kiwinet-web + PostgreSQL only

---

## What Is Retained

SearXNG docker-compose configuration and full deployment history remain accessible in branch `archive/searxng-retired-20260829` for future reference or re-evaluation. Rate limiter fixes (`server.limiter: true`, `trusted_proxies` config) are documented here for potential future use in different network contexts.

---

## Lesson Learned

Complete diagnostic workflow (Docker, network, DNS, IP migration) is reusable if meta-search self-hosting need resurfaces in fundamentally different context (native residential IP, private non-exposed usage) where datacenter anti-bot blocking is not a structural barrier.

---

## Consequences

- ✅ VPS resource freed for core services (Synapse, kiwinet-web, PostgreSQL)
- ✅ Simplified stack: fewer moving parts, fewer dependencies
- ✅ Search privacy maintained via Brave Search (independent index, no tracking required)
- ⚠️ Loss of fully self-hosted search; now dependent on Brave Search infrastructure (trade-off accepted)
