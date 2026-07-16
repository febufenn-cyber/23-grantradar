# grantradar

Grant discovery + AI application drafting: a nightly cron indexes NIH RePORTER, NSF, Grants.gov, and CORDIS into Supabase, so researchers get matched funding calls and an AI-drafted specific-aims section to start from.

**Status: planned — not yet built (50-SaaS challenge #23)**

## The problem

Researchers spend hours a month manually trawling four different government grant portals for calls that match their work, then more hours writing a first draft of the aims section from scratch for every application.

## Target buyer

University researchers and nonprofits, with an India angle: Indian institutions applying to US (NIH/NSF/Grants.gov) and EU (CORDIS) calls, who don't have a grants office doing this legwork for them.

## Pricing hypothesis

Rs999/mo flat subscription, with a monthly quota on AI-drafted aims to keep LLM cost bounded.

## Stack summary

Cloudflare Worker (TypeScript) with a cron trigger that pre-indexes public grant APIs into Supabase Postgres nightly; researchers query the indexed data (full-text search, no live proxying) and get one sync Claude call to draft aims from a matched grant + their research summary.

## How to continue this build

Nothing is implemented yet. Read `docs/LLD.md` for the architecture and data model, then `docs/PLAN.md` for the ordered TDD task list, then follow `CLAUDE.md` for how this repo relates to the shipped reference implementation.

## Risks / Constraints

- NIH RePORTER, NSF, and Grants.gov APIs are US-government public domain data, zero-auth — no API keys needed for those three.
- The Worker **must pre-index via the nightly cron and never live-proxy** a user's search to these APIs — they have per-IP rate limits that a live-proxied SaaS would blow through immediately.
- NIH RePORTER data includes PI email addresses — **these must not be used for outreach** (no scraping-to-cold-email feature, ever).
- CORDIS requires a registered API key that isn't in hand yet — v1 seeds CORDIS data from a bulk CSV export instead of a live feed; live CORDIS indexing is blocked on key approval.
- AI-drafted aims are a starting draft for the researcher to rewrite, not grant-writing advice or a submission-ready document — this disclaimer must ship on every draft.
