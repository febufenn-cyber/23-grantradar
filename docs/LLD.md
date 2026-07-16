# LLD — grantradar

## Architecture (request flow)

```
Cron (Worker `scheduled` export, e.g. "0 3 * * *" — nightly, off-peak)
   |-- fetch NIH RePORTER API (zero-auth)   -----> normalize -> upsert grants
   |-- fetch NSF Awards/Funding API (zero-auth) -->            (source-scoped
   |-- fetch Grants.gov Search API (zero-auth) -->             on external_id)
   |-- CORDIS: v1 = one-off/periodic bulk CSV import (script, not cron) until
   |           a registered CORDIS key is approved for live pulls
   v
Supabase Postgres (grants table, full-text index) <-- all reads are LOCAL,
                                                        never a live proxy to
                                                        the source APIs

Browser (magic-link auth)
   |
   |  GET /api/grants/match?q=...            (sync, Postgres tsvector search)
   v
Worker  --------------------------------------------> researcher sees matches
   |
   |  POST /api/draft-aims {grant_id, research_summary}
   |-- consume_draft_quota(user_id)  -> Supabase RPC (monthly quota, not a
   |                                     one-shot credit — subscription product)
   |-- buildDraftAims() -> Anthropic (claude-sonnet-4-6)     [sync, single call]
   |-- insert draft_aims row
   v
Browser shows draft + "not submission-ready" disclaimer
```

## Data model (Supabase Postgres, RLS on)

```sql
create table public.grants (
  id uuid primary key default gen_random_uuid(),
  source text not null check (source in ('nih','nsf','grants_gov','cordis')),
  external_id text not null,
  title text not null,
  agency text,
  deadline date,
  amount_max numeric,
  abstract text,
  url text,
  search tsvector generated always as
    (to_tsvector('english', coalesce(title,'') || ' ' || coalesce(abstract,''))) stored,
  raw jsonb,                       -- full source record, for debugging/re-normalization
  indexed_at timestamptz not null default now(),
  unique (source, external_id)
);
create index grants_search_idx on public.grants using gin (search);
create index grants_deadline_idx on public.grants (deadline);
-- RLS: grants is reference data, not user-owned. Enable RLS, single policy:
alter table public.grants enable row level security;
create policy "authenticated read" on public.grants for select
  using (auth.role() = 'authenticated');
-- All writes are service-role only (cron + CSV import script), no client writes.

create table public.researcher_profiles (
  user_id uuid primary key references auth.users(id) on delete cascade,
  institution text,
  keywords text[],
  research_summary text,
  plan text not null default 'trial' check (plan in ('trial','active','cancelled')),
  draft_quota int not null default 3,        -- resets monthly
  quota_period_start date not null default date_trunc('month', now()),
  created_at timestamptz not null default now()
);
alter table public.researcher_profiles enable row level security;
create policy "read own profile" on public.researcher_profiles for select
  using (auth.uid() = user_id);

create table public.draft_aims (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  grant_id uuid not null references public.grants(id),
  research_summary text not null,
  draft_text text,
  model text, input_tokens int, output_tokens int,
  status text not null default 'pending' check (status in ('pending','done','failed')),
  error text,
  created_at timestamptz not null default now()
);
alter table public.draft_aims enable row level security;
create policy "read own drafts" on public.draft_aims for select using (auth.uid() = user_id);

create or replace function public.consume_draft_quota(p_user uuid) returns boolean
language plpgsql security definer set search_path = public as $$
declare v_period date := date_trunc('month', now());
begin
  update researcher_profiles set draft_quota = 5, quota_period_start = v_period
    where user_id = p_user and quota_period_start < v_period;  -- monthly reset
  update researcher_profiles set draft_quota = draft_quota - 1
    where user_id = p_user and draft_quota > 0;
  return found;
end $$;
revoke execute on function public.consume_draft_quota(uuid) from public, anon, authenticated;
grant execute on function public.consume_draft_quota(uuid) to service_role;
```

No `spend_credit`/`refund_credit` per the reference — this is a flat-subscription product, so quota is monthly (`consume_draft_quota`) rather than a one-free-credit trial balance.

## API routes

| Route | Method | Auth | Behavior | Failure modes |
|---|---|---|---|---|
| `/config.js` | GET | none | Public Supabase URL/anon key | — |
| `/api/grants/match` | GET | Supabase JWT | `plainto_tsquery` search over `grants.search`, filtered by keywords/deadline | empty query (400) |
| `/api/draft-aims` | POST | Supabase JWT | Consumes monthly quota, calls Claude, stores draft | quota exhausted (402), Claude failure -> `failed`, no quota refund (subscription, not a paid credit) |
| `/api/drafts/:id` | GET | Supabase JWT (own row) | Returns a stored draft | not found (404) |
| `/api/me` | GET | Supabase JWT | Profile + remaining quota | — |

Cron (not an HTTP route): nightly indexer for NIH/NSF/Grants.gov; CORDIS via a manual/periodic `scripts/import-cordis-csv.ts` until the API key lands.

## LLM strategy

- Provider: **Anthropic claude-sonnet-4-6**, plain text/light-JSON output (no need for the strict report schema used elsewhere — this is prose, not a structured report): `{draft_text: string, alignment_notes: string}`.
- Prompt: grant title+abstract + researcher's own `research_summary` -> a first-draft specific-aims section, explicitly instructed to flag where it's guessing at scope.
- Cost estimate: ~1.5k input + ~1k output tokens/draft ≈ $0.02 (≈ Rs2) — trivial against Rs999/mo, quota (5/mo default) exists to bound abuse, not cost.
- Sync vs async: both the match query (Postgres full-text search) and the draft call (single Claude completion, no PDF/multi-step pipeline) comfortably finish in seconds — **fully synchronous**, no VPS/callback needed anywhere in this product. The only async thing here is the cron indexer, which is a scheduled job, not a per-request pattern.

## Frontend pages

`index.html` (research profile setup: institution, keywords, summary), `matches.html` (search/browse indexed grants), `grant.html` (single grant + "Draft Aims" button + quota remaining), `account.html` (plan/quota), `tos.html` (disclaimer: drafts are a starting point, not grant-writing advice; NIH PI emails are never surfaced for outreach).

## Error handling + quota flow

Quota decremented atomically via `consume_draft_quota` before the Claude call; on Claude failure the `draft_aims` row is marked `failed` with the error shown to the user, but **quota is not refunded** — same tolerance model as a phone plan's monthly minutes, appropriate for a flat-subscription product (unlike per-operation credits in the reference, where every spend must be refundable).

## Integrations and launch gates

- NIH RePORTER, NSF, Grants.gov: zero-auth, no launch gate.
- **CORDIS registered API key is the one launch-blocking external dependency for live EU-grant coverage.** v1 ships with CSV-seeded CORDIS data (accurate as of import date, refreshed manually) so launch isn't blocked on key approval; live CORDIS cron indexing is added once the key is granted.
- No OAuth anywhere in this product.

## Security notes

- RLS: `researcher_profiles` and `draft_aims` are user-owned (`auth.uid() = user_id`); `grants` is shared reference data, readable by any authenticated user, writable only by service role.
- Secrets: `ANTHROPIC_API_KEY`, `SUPABASE_SERVICE_ROLE_KEY` via `wrangler secret put`; CORDIS key (once issued) likewise.
- Input validation on `research_summary` (length cap) before it's interpolated into the Claude prompt.
- NIH PI email fields, if present in `raw jsonb`, are never exposed via any API route or UI — enforced by only ever selecting the normalized columns, never `raw`, in client-facing queries.

## Out of scope for v1

Live CORDIS API integration (until key approved), embeddings/semantic matching (v1 is full-text search only), any outreach/email feature touching PI contact data, full-proposal drafting beyond the aims section, collaborator/co-PI matching, deadline reminder notifications, multi-user/lab-wide accounts.
