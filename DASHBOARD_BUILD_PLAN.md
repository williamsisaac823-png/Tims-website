# Tim's Dashboard — Full Build Plan

Written 2026-07-16 as a pre-build spec so a future session can execute this
straight through without re-deriving the architecture. If Isaac says "go,"
work through the Build Order at the bottom top to bottom.

## Context / why this exists

The public site (`index.html`, this repo) currently has a Reviews section and
a Blog page, both editable from `Dashboard.html` — but neither dashboard
editor actually persists anywhere real. `saveReviews()` and `saveBlog()` both
just write to that browser's `localStorage`, and the public site reads from
hardcoded JS arrays baked into the HTML. Nothing Tim edits reaches an actual
visitor. This plan replaces that with a real backend so Tim can manage his
own site content from any device, and changes show up for everyone.

## Ownership (already decided, see [[tims-website-backend-plan]] memory)

- **Domain registrar, Vercel, Supabase**: owned by Tim (`whitetimone@gmail.com`),
  Isaac added as a collaborator/developer on each. Tim owns his own business
  assets and data; Isaac isn't a single point of failure.
- **Code repos**: stay under Isaac's GitHub (`williamsisaac823-png`), same as
  the current `Tims-website` repo — consistent with how this project has
  always worked, and Tim doesn't need to learn GitHub.
- **Two repos total**: `Tims-website` (existing, public static site, untouched
  architecture) + a new repo for the dashboard app (name TBD, suggest
  `tim-white-dashboard`).

## Architecture

```
Visitor's browser
   │
   ├─ loads public site from GitHub Pages (static HTML, same as today)
   │     └─ small new JS snippet fetches published reviews/blog posts
   │        directly from Supabase's REST API (public anon key, safe —
   │        RLS only allows reading rows where published = true)
   │     └─ contact form POSTs to a Vercel serverless function (not
   │        straight to Supabase — keeps validation + secrets server-side)
   │
   └─ Tim logs into the dashboard app (hosted on Vercel, separate repo)
         └─ Supabase Auth (email/password + "Sign in with Google")
         └─ reads/writes leads, reviews, blog_posts tables directly
            (his session is authenticated, RLS grants him full access)

Vercel serverless functions
   └─ contact-form-submit: validates input → inserts into `leads` →
      sends Tim a notification + sends the customer a confirmation
      (email via Resend or similar; SMS to Tim only, via carrier
      email-to-SMS gateway, free — texting customers needs a paid
      SMS API like Twilio, decide later if wanted)

Supabase (Postgres + Auth)
   └─ single source of truth for leads, reviews, blog_posts
   └─ Row Level Security enforced on every table (see below)
```

## Database schema

```sql
create table leads (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  company text,
  email text not null,
  phone text,
  message text,
  machine_interest text,
  status text not null default 'new', -- new | contacted | closed
  created_at timestamptz not null default now()
);

create table reviews (
  id uuid primary key default gen_random_uuid(),
  quote text not null,
  name text not null,
  role text,
  rating int not null default 5,
  published boolean not null default false,
  created_at timestamptz not null default now()
);

create table blog_posts (
  id uuid primary key default gen_random_uuid(),
  title text not null,
  slug text not null unique,
  cover_image_url text,
  body text not null, -- markdown or simple section-based JSON, match "magazine style" layout Isaac already built in Claude Design
  published boolean not null default false,
  published_at timestamptz,
  created_at timestamptz not null default now()
);
```

## Row Level Security (critical — do not skip)

- `leads`: public role can `INSERT` only (the contact form technically could
  write directly, but we're routing through the serverless function instead
  for validation/spam-control — RLS is still the backstop). No public
  `SELECT`/`UPDATE`/`DELETE`. Authenticated (Tim) role: full access.
- `reviews`: public can `SELECT` where `published = true` only. Authenticated:
  full access (insert/update/delete, toggle published).
- `blog_posts`: same pattern as reviews — public `SELECT` where
  `published = true`, authenticated gets full access.
- Service role key (bypasses RLS entirely) only ever used inside the Vercel
  serverless function's server-side code — never shipped to any browser bundle.

## Auth

- Supabase Auth, email/password as the default.
- Add Google as an OAuth provider (Isaac's request) — needs a Google Cloud
  OAuth client ID/secret. Lower stakes than the domain/Vercel/Supabase
  ownership call — fine to set up under whichever Google account is most
  convenient at the time, since it's just a credential, not a data store.
- Only one real user for now (Tim). No public sign-up flow needed on the
  dashboard — Tim is the sole account, created directly in Supabase.

## Public site changes needed

1. Replace the hardcoded `REVIEWS` array in `index.html` with a small fetch
   on page load: `GET {supabase_url}/rest/v1/reviews?published=eq.true`
   (using the public anon key — safe, RLS limits what it can return).
2. Same pattern for the Blog page's post list/detail.
3. Contact form's submit handler changes from whatever it does today to a
   `POST` to the new Vercel function's URL.
4. These are the only public-site changes — no other part of the static
   bundle changes, and none of the v3.x fix checklist in `NOTES.md` is
   affected by this (still reapply that checklist on any *new* Claude
   Design export, same as always).

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

- [ ] RLS enabled and tested on all three tables (try an anonymous request
      against each table and confirm it's blocked/limited as designed)
- [ ] Service role key present only in Vercel's server-side environment
      variables, never in any client-shipped code
- [ ] Contact form submits to the serverless function, not straight to Supabase
- [ ] Basic spam protection on the contact form (honeypot field minimum;
      Cloudflare Turnstile if we want stronger protection)
- [ ] Rate limiting on the lead-submission endpoint
- [ ] All secrets (.env) confirmed absent from git history before first push
- [ ] Run the `security-review` skill against the dashboard repo and the
      serverless function before considering this "real"

## Build order (execute top to bottom when told to go)

1. Scaffold the new dashboard repo (Next.js, or whatever framework fits —
   decide at build time) under Isaac's GitHub, connected to Tim's Vercel project.
2. Wire up the Supabase client + environment variables.
3. Build the login page: email/password first, Google sign-in second.
4. Create the three tables + RLS policies in Supabase (via SQL editor or a
   migration file committed to the repo).
5. Build the Leads screen (list + status update) — highest immediate value,
   build first.
6. Build the Reviews editor (add/edit/delete/publish toggle) + add the
   fetch snippet to the public site's Reviews section.
7. Build the Blog editor (matching the "magazine style" layout Isaac already
   designed in Claude Design) + add the fetch snippet to the public Blog page.
8. Build the contact-form serverless function (validate → insert lead →
   notify Tim → confirm to customer) + update the public site's form to
   call it.
9. Add Google Sign-in to the login page.
10. Run through the full security checklist above.
11. Deploy, click through every flow end-to-end as a real visitor + as Tim,
    confirm nothing regressed on the public site.
12. Hand off: confirm Tim can log in, edit a review, edit a post, and see a
    real lead come through, on his own device, unassisted.

## Open decisions still pending (not blockers, just not yet answered)

- Public site ↔ metalforming-usa.com connection: waiting on Tim/company
  (cross-links vs. subdomain vs. independent domain — see backend-plan memory)
- Whether to add paid SMS-to-customer notifications (Twilio) later, or keep
  SMS as Tim's free internal alert only
- Final domain name pick for the public site (shortlist below)

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
