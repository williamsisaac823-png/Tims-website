# Tim's Dashboard — Full Build Plan

Written 2026-07-16, revised same day after switching the stack. Pre-build
spec so a future session can execute this straight through without
re-deriving the architecture. If Isaac says "go," work through the Build
Order at the bottom top to bottom.

**Stack history:** originally planned on Vercel + Supabase. Revised after
verifying both platforms' actual Terms of Service: Vercel's Hobby (free) tier
explicitly prohibits commercial use (confirmed directly in their ToS — "any
Deployment used for the purpose of financial gain of anyone involved... including
a paid employee or consultant writing the code" counts, which this is), and
Supabase's free tier pauses after 7 days of inactivity with no backups,
risky for a live leads database. Switched to **Cloudflare Pages + Workers +
D1 + Access** after verifying (via Cloudflare's own docs/ToS, not a marketing
page) that the free tier has no commercial-use restriction and the request/
storage ceilings are far beyond this project's realistic scale. See
[[tims-website-backend-plan]] memory for the full verification trail.

## Context / why this exists

The public site (`index.html`, this repo) currently has a Reviews section and
a Blog page, both editable from `Dashboard.html` — but neither dashboard
editor actually persists anywhere real. `saveReviews()` and `saveBlog()` both
just write to that browser's `localStorage`, and the public site reads from
hardcoded JS arrays baked into the HTML. Nothing Tim edits reaches an actual
visitor. This plan replaces that with a real backend so Tim can manage his
own site content from any device, and changes show up for everyone.

## Ownership

- **Domain registrar + Cloudflare account**: owned by Tim
  (`whitetimone@gmail.com`), Isaac added as a member on the Cloudflare
  account. Tim owns his own business assets and data; Isaac isn't a single
  point of failure.
- **Code repos**: stay under Isaac's GitHub (`williamsisaac823-png`), same as
  the current `Tims-website` repo.
- **Two repos total**: `Tims-website` (existing, public static site,
  untouched — stays on GitHub Pages, lowest-risk, already working) + a new
  repo for the dashboard app (suggest `tim-white-dashboard`), deployed to
  Cloudflare Pages.

## Why one Cloudflare account can now hold everything

Because Cloudflare's free tier has no commercial-use restriction (verified),
there's no longer a reason to split platforms just to dodge a ToS problem
like there was with Vercel. Practical call: **leave the public site on
GitHub Pages for now** (it already works, zero migration risk) and put the
new dashboard + backend + database + login gate all on the one Cloudflare
account. If Isaac later wants to fully consolidate (move the public site to
Cloudflare Pages too), that's an optional, low-priority cleanup — not needed
for any of this to work.

## Architecture

```
Visitor's browser
   │
   ├─ loads public site from GitHub Pages (static HTML, same as today,
   │  unchanged deploy process — still just git push)
   │     └─ small new JS snippet calls a Cloudflare Worker API endpoint
   │        (e.g. GET https://tw-api.<account>.workers.dev/reviews) which
   │        queries D1 server-side and returns only published rows as JSON.
   │        D1 itself is NEVER exposed directly to the browser — unlike the
   │        Supabase plan, there's no RLS to configure because the database
   │        literally isn't reachable except from inside a Worker.
   │     └─ contact form POSTs to another Worker endpoint (same pattern —
   │        validates input server-side, then writes to D1)
   │
   └─ Tim opens the dashboard (Cloudflare Pages, separate repo)
         └─ Cloudflare Access sits in front of the whole dashboard app —
            nothing loads at all until you sign in. Configured with an
            email allowlist (just whitetimone@gmail.com, add Isaac's too
            if he needs direct access) + Google as the sign-in provider.
            Free for up to 50 users. No custom login code to write at all.
         └─ once past Access, the dashboard's own Worker functions read/
            write leads, reviews, blog_posts in D1 directly (server-side,
            already gated by Access — no additional app-level auth needed)

Cloudflare Workers (backend logic)
   └─ POST /leads: validates input → inserts into `leads` in D1 → sends
      Tim a notification + sends the customer a confirmation (email via
      a provider like Resend; SMS to Tim only via a free carrier
      email-to-SMS gateway — texting customers needs a paid SMS API like
      Twilio, decide later if wanted)
   └─ GET /reviews, GET /blog-posts: public read endpoints, only return
      rows where published = true
   └─ dashboard-only endpoints (behind Access): full CRUD on all three tables

Cloudflare D1 (SQLite-based database)
   └─ single source of truth for leads, reviews, blog_posts
   └─ only ever queried from inside Worker code — never exposed to the
      public internet directly
```

## Database schema (D1 uses SQLite syntax, not Postgres)

```sql
create table leads (
  id text primary key, -- generate with crypto.randomUUID() in the Worker
  name text not null,
  company text,
  email text not null,
  phone text,
  message text,
  machine_interest text,
  status text not null default 'new', -- new | contacted | closed
  created_at text not null default (datetime('now'))
);

create table reviews (
  id text primary key,
  quote text not null,
  name text not null,
  role text,
  rating integer not null default 5,
  published integer not null default 0, -- SQLite has no boolean type; 0/1
  created_at text not null default (datetime('now'))
);

create table blog_posts (
  id text primary key,
  title text not null,
  slug text not null unique,
  cover_image_url text,
  body text not null, -- markdown or simple section-based JSON, match "magazine style" layout Isaac already built in Claude Design
  published integer not null default 0,
  published_at text,
  created_at text not null default (datetime('now'))
);
```

## Access control model (replaces Supabase RLS)

There's no RLS to configure here — the security model is simpler because D1
has no public API of its own. Every rule lives in the Worker code instead:

- Public-facing Worker endpoints only ever run `SELECT ... WHERE published = 1`
  queries, or `INSERT` for the leads endpoint. They never accept a query that
  could read/modify anything else.
- Dashboard-facing Worker endpoints (full CRUD) are only reachable through
  routes that sit behind Cloudflare Access — Access checks the login before
  the request ever reaches the Worker.
- Because of this, there's no separate "RLS policy" step to test — the
  security review below should instead double check that the public endpoints
  truly can't be coaxed into returning unpublished rows or writing outside
  the `leads` table (e.g. via a malformed request).

## Auth

- **Cloudflare Access**, not custom code. Email allowlist (Tim, optionally
  Isaac) + Google as the identity provider — this is exactly the "password
  login plus Sign in with Google" outcome Isaac wanted, but with zero custom
  auth/session code to write or secure.
- Configured entirely in the Cloudflare dashboard UI once the account and
  the dashboard's Pages project exist — a setup step, not a coding task.

## Public site changes needed

1. Replace the hardcoded `REVIEWS` array in `index.html` with a small fetch
   on page load to the Worker's `/reviews` endpoint.
2. Same pattern for the Blog page's post list/detail, via `/blog-posts`.
3. Contact form's submit handler changes to `POST` to the Worker's `/leads`
   endpoint.
4. These are the only public-site changes — no other part of the static
   bundle changes, and none of the v3.x fix checklist in `NOTES.md` is
   affected (still reapply that checklist on any *new* Claude Design export).

## Customer-facing confirmation copy (drafted earlier, ready to use)

**Email:**
> Subject: We've got your request, {{name}} — Tim White will be in touch
>
> Hi {{name}},
>
> Thanks for reaching out. I've received your request and will personally
> follow up within one business day.
>
> Need something sooner? Call or text me directly at (770) 315-0285.
>
> Talk soon,
> Tim White
> Independent Sheet-Metal Machinery Specialist
> Authorized Dealer for MetalForming, LLC

**SMS (to Tim, as the internal alert — not sent to customers unless we later
add a paid SMS API):**
> New lead: {{name}}, {{phone}}, interested in {{machine_interest}}.
> Check dashboard for details.

## Security checklist (verify all before this touches real customer data)

- [ ] Confirm public Worker endpoints genuinely can't read unpublished rows
      or touch tables other than `leads` (test with deliberately malformed
      requests, not just the happy path)
- [ ] Confirm the dashboard's full-CRUD Worker routes are actually unreachable
      without passing Cloudflare Access first (test from a logged-out/incognito session)
- [ ] Basic spam protection on the contact form (honeypot field minimum;
      Cloudflare Turnstile if we want stronger protection — same vendor,
      easy to add)
- [ ] Rate limiting on the `/leads` endpoint
- [ ] All secrets (API keys for the email/SMS provider) stored as Worker
      environment variables/secrets, never committed to git
- [ ] Run the `security-review` skill against the Worker code before
      considering this "real"

## Build order (execute top to bottom when told to go)

1. Scaffold the new dashboard repo under Isaac's GitHub, connected to Tim's
   Cloudflare account via Wrangler (Cloudflare's CLI/deploy tool).
2. Create the D1 database + the three tables (migration file committed to the repo).
3. Set up Cloudflare Access in front of the dashboard's Pages project
   (email allowlist + Google provider) — a dashboard-UI setup step, do this early.
4. Build the public read endpoints (`/reviews`, `/blog-posts`) + wire the
   public site's fetch snippets to them.
5. Build the Leads screen in the dashboard (list + status update) — highest
   immediate value, build first among the dashboard's own screens.
6. Build the Reviews editor (add/edit/delete/publish toggle).
7. Build the Blog editor (matching the "magazine style" layout Isaac already
   designed in Claude Design).
8. Build the `/leads` submission endpoint (validate → insert → notify Tim →
   confirm to customer) + update the public site's contact form to call it.
9. Run through the full security checklist above.
10. Deploy, click through every flow end-to-end as a real visitor + as Tim,
    confirm nothing regressed on the public site.
11. Hand off: confirm Tim can log in via Google, edit a review, edit a post,
    and see a real lead come through, on his own device, unassisted.

## What's realistically doable tonight vs. needs Tim present

**Can be done right now, without Tim at the keyboard:**
- All the actual code: the Worker endpoints, the D1 schema/migration file,
  the dashboard's screens, the public site's fetch snippets. None of this
  needs a live Cloudflare account to *write* — Cloudflare's local dev tool
  (`wrangler dev`) emulates Workers and D1 locally, so it can be built and
  tested without deploying anywhere yet.
- This plan document itself (done).

**Needs Tim's Cloudflare account to exist first (so needs him, this weekend):**
- Actually deploying anything live (`wrangler deploy`) — needs a real
  account to deploy *to*.
- Setting up Cloudflare Access (needs the account + the Pages project to exist).
- Buying the domain, if he wants one now instead of later.

**My recommendation for tonight, if Isaac wants to use the time:** I can
start writing the real Worker code, D1 schema, and dashboard screens now,
test them locally, and have them ready to deploy the moment Tim's account
exists this weekend — rather than just leaving this as a spec. Ask me to
start if that's wanted; otherwise this document is the "tell me to go and
I execute" artifact as originally requested.

## Open decisions still pending (not blockers, just not yet answered)

- Public site ↔ metalforming-usa.com connection: waiting on Tim/company
  (cross-links vs. subdomain vs. independent domain)
- Whether to add paid SMS-to-customer notifications (Twilio) later, or keep
  SMS as Tim's free internal alert only
- Final domain name pick for the public site (shortlist below)
- Whether to eventually consolidate the public site onto Cloudflare Pages
  too (optional, not required for any of the above to work)

## Domain shortlist (all confirmed available via RDAP lookup, 2026-07-16)

- **timwhitemachinery.com** — top pick, matches the site's existing title
- timwhiteequipment.com
- timwhitesheetmetal.com
- metalfabmachines.com
- sheetmetalmachinerytim.com
- buysheetmetalequipment.com
- metalbendingmachines.com
- sheetmetalequipmentdealer.com
- metalshearsforsale.com
- timwhitefolders.com (narrower — only covers folders, not the full catalog)
