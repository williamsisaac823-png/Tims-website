# Tim White's Website — Working Notes

Living notes on how this site works and how to update it. Written after several
rounds of fixes so the next update doesn't have to rediscover any of this.

## What this is

`index.html` and `Dashboard.html` are self-contained exports from a Claude Design
project (design.claude.ai). Each file is a single HTML document (~28MB for
index.html) that bundles the entire page — HTML, CSS, JS, fonts, and every image —
as base64 data. There is no build step and no separate assets folder; everything
GitHub Pages needs to serve is inside these two files.

Live site: https://williamsisaac823-png.github.io/Tims-website/
GitHub Pages config: deploys from `main`, root folder (`/`). Already set up correctly.

## How the bundle actually works (read this before editing either file)

Both files start with a small real `<head>`/`<body>` "loading" shell (~3KB), followed
by a boot script, followed by a **single giant `<script type="__bundler/manifest">`**
(the base64 image/font data) and a **single giant `<script type="__bundler/template">`**
containing the *entire real page* as one long `JSON.stringify`-escaped string.

On load, the boot script:
1. Decodes/decompresses every asset in the manifest into `Blob` objects and builds
   `window.__resources` — a map of original relative path (e.g.
   `"images/products/folders/schechtl-max.png"`) → `blob:` URL.
2. `JSON.parse`s the template string and parses it with `DOMParser`.
3. Swaps it in via `document.documentElement.replaceWith(doc.documentElement)`.

**Practical implications:**
- The template blob is functionally *one line* of text (real newlines are encoded as
  literal `\n` two-character escapes, since it's JSON). Don't use `grep`/`sed`/sed-style
  line-based tools on it — anything that reads "the whole line" will try to dump 20+MB.
  Use `grep -aob` to get a byte offset, then `tail -c +OFFSET file | head -c N` to read
  a small window. Same for edits: find the byte offset, confirm the exact surrounding
  bytes, then do an exact-string `bytes.replace()` in a small Python script (write the
  script to a file and run it — inline `python3 -c` / `perl -e` through several layers
  of shell quoting reliably mangles backslash escapes; this cost real time twice).
- A literal **real newline** accidentally inserted into that JSON string breaks
  `JSON.parse` outright (the whole page goes blank/errors). Always verify with
  `od -c` after editing that region — a real newline shows as a single `\n` token in
  one byte-column; the literal two-char escape shows as separate `\` and `n` columns.
- The readable boot script *above* the manifest/template blobs (roughly the first
  ~200 lines) is normal, real JavaScript source — safe to edit with normal tools
  (Read/Edit), no special escaping needed.

## Known bug: product images can go broken on re-export — expect to refix this

**Symptom:** some product photos show a blue "?" placeholder / gray box instead of
the real photo. Affects `<image-slot src="...">` elements specifically (the product
card image component), not the Home hero or brand-logo `<img>` tags, which are fine.

**Root cause:** the boot script defines a getter/setter override on
`HTMLImageElement.prototype.src` that rewrites a relative path to its `blob:` URL —
but only when something does `element.src = value` (a JS property assignment).
`<image-slot>` isn't even `HTMLImageElement` (browsers treat it as a generic unknown
element), and the app's own template engine fills in `src="{{ r.img }}"` bindings at
render time via `element.setAttribute('src', realPath)` — which **never goes through
that override at all** (setAttribute bypasses JS property setters entirely). So the
raw relative path (e.g. `images/products/folders/schechtl-max.png`) gets set as a
literal attribute, and since that path doesn't exist as a real file on GitHub Pages,
the browser shows a broken image.

Confirmed broken on both desktop Chrome and mobile Safari — not a mobile-only quirk.

**Fix applied (in index.html only — Dashboard.html has no product photos and never
had this problem):** a second, global override on `Element.prototype.setAttribute`
that intercepts any `src` assignment and rewrites it via `window.__resources`, mirroring
the existing property-setter override. It also falls back to a product's base/hero
image when a referenced `angle-N`/`feature-N` variant photo wasn't included in this
particular export's asset bundle (not every machine has extra angle photos — some
exports only bundle the ~78 "essential" images, not the full few-hundred-image library
from the original Claude Design project).

The exact patch (inserted into the embedded template blob, right after the existing
`HTMLImageElement.prototype.src` override, same file/pattern each time):

```js
(function(){try{
  var origSetAttribute = Element.prototype.setAttribute;
  Element.prototype.setAttribute = function(name, value) {
    try {
      if (name === 'src' && value) {
        var R = window.__resources;
        if (R) {
          var k = String(value).replace(/^\.?\//, '');
          if (R[k]) { value = R[k]; }
          else {
            var m = k.match(/^(.*)\/(?:angle|feature)-\d+\.[a-zA-Z0-9]+$/);
            if (m) {
              var prefix = m[1] + '.';
              for (var rk in R) { if (rk.indexOf(prefix) === 0) { value = R[rk]; break; } }
            }
          }
        }
      }
    } catch(e) {}
    return origSetAttribute.call(this, name, value);
  };
}catch(e){}})();
```

**Important:** this is a bug in the Claude Design export tool's generated bootstrap
code, not something specific to this one export. Every time you re-export `Home` from
Claude Design and swap it into `index.html`, this bug will very likely come back and
need to be patched again the same way, until the export tool itself fixes it upstream.
Also reapply the DOMParser-swap fix below for the same reason.

Also carried in the readable boot script (top of file, easy to find via
`grep -n "DOMParser().parseFromString(template"`): a similar issue for any *static*
(non-templated) `<img src="...">` baked directly into the page — those go through
`DOMParser`, which also never triggers the property-setter override. Fix there is a
post-parse loop right after the `DOMParser().parseFromString(...)` call, before
`replaceWith`, doing the same `resourceMap` lookup via `setAttribute`.

## Other fixes carried in the current files (reapply on re-export if needed)

- **Zoom lock**: the viewport meta tag needs `maximum-scale=1, minimum-scale=1,
  user-scalable=no` appended (fresh exports only have `width=device-width,
  initial-scale=1`).
- **Cross-page links**: Home's login/dashboard button does
  `window.location.href = 'Dashboard.dc.html'` in fresh exports — that file doesn't
  exist; it needs to point to `Dashboard.html` (the actual filename in this repo).
  Same the other direction: Dashboard's back/logout action points to `Home.dc.html`,
  needs to be `index.html`.
- Fresh exports already include a solid native `@media (max-width:768px)` mobile
  stylesheet (stacks the hero, kills oversized `min-width`s, scales headings down,
  etc.) — that part doesn't need to be redone, just verify it's still there.

## Workflow for future updates

1. Make the change in the Claude Design project (design.claude.ai), export/download
   the page(s) as offline HTML.
2. Drop the download(s) in the Downloads folder — no need to rename anything, just
   say "use the most recent download(s)."
3. Reapply the fixes above (zoom lock, cross-page links, the two src-rewriting
   patches for index.html) to the fresh export before publishing.
4. **Verify with an actual rendered browser, not just visual inspection of a
   screenshot.** A plain `chrome --headless --screenshot` CLI flag gave a misleading
   (falsely "broken") capture at one point during this work — the real check that
   caught both bugs described above was connecting via Chrome DevTools Protocol
   (`--remote-debugging-port`) and directly measuring `getBoundingClientRect()`,
   `getComputedStyle()`, and `img.naturalWidth`/`image-slot` `src` attributes on the
   live rendered page. Screenshot afterward for a visual sanity check, but don't
   treat it as the source of truth on its own.
5. Commit, push to `main`, and tag the commit (`v3.3`, `v3.4`, ...) so there's a
   version history to point back to. GitHub Pages picks up `main` automatically —
   no separate deploy step.

## Version history

- **v2** — first Claude Design export wired up (Home + Dashboard), fixed the
  Dashboard 404 (was linking to a nonexistent `Dashboard.dc.html`), locked zoom.
- **v3** — refreshed Home/Dashboard exports, same fixes reapplied.
- **v3.1** — mobile layout fixes (phone number was getting cut off, body text was
  overflowing the screen edge on category/product pages). Verified via CDP.
- **v3.2** — fixed broken product images site-wide (the `<image-slot>`/`setAttribute`
  bug described above).
- **v3.3** — copy fixes: hero headline had `who<br>stands` with no real space between
  the words (`<br>` forces a line break but never inserts a space character, so on
  layouts where the break doesn't land there it read as "whostands" jammed together
  — fixed by adding a real space); removed a trailing ", not just sitting on a quote."
  clause from the shared "why buy through Tim" paragraph (appears once in source,
  renders on every machine detail page); fixed a dropped verb ("Jorns conveying
  solutions" → "Jorns offers conveying solutions"), a wrong word ("retreat the bending
  beam" → "retract the bending beam" — mechanical term), and a subject-verb mismatch
  ("Form panels demonstrating compliance" → "Forms panels demonstrating compliance")
  in the JDB/TBS/SSR product copy; removed the visible "Prototype — any password
  works." disclosure line from the Home page's embedded dashboard sign-in preview —
  **note: this only removes the visible text, it does not add real password
  verification. The sign-in still has no real backend auth check; that's a separate,
  bigger task before this dashboard should gate anything actually private.**
- **v3.4** — new Home export (`Tim White - Home2.html` from Downloads) adding a
  standalone FAQ page (nav: Home / Equipment / Brands / Reviews / FAQ / Contact,
  same position on both desktop and mobile menus), 44 Q&As across 9 categories with
  FAQPage schema.org JSON-LD for SEO. As expected on every fresh Claude Design
  export, ALL prior fixes had regressed and were reapplied from scratch: zoom lock,
  the `Dashboard.dc.html` → `Dashboard.html` link, both image-src patches
  (`Element.prototype.setAttribute` override + the pre-`replaceWith` static `<img>`
  cleanup loop — first time both needed inserting fresh into a from-scratch export
  rather than just re-verifying they survived), and all five v3.3 content fixes
  (whostands spacing, the quote-copy clause, the three grammar fixes, the prototype
  disclosure line). **Going forward: treat every fresh export as needing ALL of the
  above reapplied, not just the structural fixes — the content fixes regress too
  since they only ever lived in the exported HTML, never in the Claude Design
  project source itself.** Confirmed the existing `@media (max-width:768px)`
  stylesheet carried over intact, plus this export adds its own FAQ-specific mobile
  CSS (smaller heading, larger accordion tap targets) on top of it.
- **v3.5** — targeted content/bug fixes on top of v3.4, no fresh Claude Design export
  this time (edited the existing v3.4 `index.html` directly, so none of the usual
  regression checklist applied). Changes:
  - **Fixed the mobile FAQ menu bug**: the hamburger menu's FAQ link was wired to
    `sc-camel-on-click="{{ mOpenFaq }}"`, but `mOpenFaq` was never defined in the
    Component's bindings object (every other mobile link — `mGoHome`, `mOpenCatalog`,
    `mOpenBrands`, `mOpenReviews`, `mOpenContact` — has both a markup reference *and*
    a bindings-object definition; `mOpenFaq` only had the markup reference). Clicking
    it silently did nothing. Fixed by adding
    `mOpenFaq: () => { this.openFaq(); this.closeMenu(); }` next to `mOpenReviews` in
    the bindings object. Verified with an actual scripted mobile click via Playwright
    (not just code inspection) that it now navigates to the FAQ page.
  - Added a 4th hero stat tile ("8 / Years in the field") next to the existing
    "World-class brands / Machines to spec / Financing decisions" tiles, using the
    identical tile+divider markup pattern. No prior line stated Tim's personal
    tenure anywhere on the site (the existing "decades of experience" copy refers to
    the machine brands' engineering history, not Tim) — this was new content, not a
    fix, based on Tim's answer (8 years, factory-trained).
  - Sharpened the financing FAQ answer with the real lender and approval time
    ("...financed through Quickturn Financial, with approval typically taking about
    3-4 days...") — previously generic with no lender named.
  - Removed the 3 fabricated testimonials (Marcus D./Ridgeline Roofing, Sara L./Apex
    Sheet Metal, Dave R./North Metal Works) from `REVIEWS`, per Isaac: Tim has zero
    real reviews yet, so fake ones shouldn't be live. `REVIEWS = []` now — both the
    Home "Trusted on the floor" section and the dedicated Reviews page render their
    header with zero cards below (no crash, just visually sparse); real testimonials
    should replace this before the site is shown to real customers.
  - **Did NOT change** the brand roster (New Tech Machinery, Schechtl, Schlebach,
    etc. are all real, already-built-out brands with their own pages/logos/products —
    initially misread one of Tim's answers as a dealer-list correction, caught it by
    re-reading his actual answer before editing anything, no change needed there).
  - Also did NOT touch meta titles/descriptions or the Contact page's `LocalBusiness`
    schema — checked both directly in the file and found them already using the real
    company name, dealer language, and phone number, not placeholders as assumed.
- **v3.6** — Isaac asked for the 3 fake testimonials back (he'll say when to pull them
  for real, once Tim has actual reviews) — restored the original `REVIEWS` array
  exactly as it was in v3.4. Also investigated Dashboard.html's "Customer reviews"
  editor per Isaac's question ("does saving reviews in the dashboard push out to
  where the testimonials show on the homepage?"): **no, it does not.**
  `saveReviews()` in Dashboard.html only does
  `localStorage.setItem('tw_reviews', JSON.stringify(this.state.reviews))` — that's
  browser-local storage on whatever device is signed into the dashboard, not a
  server or database. `index.html` (the real public homepage) never reads that
  `tw_reviews` key at all — its `REVIEWS` array is a separate hardcoded constant.
  So even on the same browser/device, editing and "saving" reviews in the dashboard
  has zero effect on the live public site; Dashboard.html even has its own separate
  hardcoded `DEFAULTREV` fallback with the same kind of placeholder names. Making
  this real would need an actual backend (API + database) shared between both pages
  — bigger, separate task, not done here. Confirmed mobile rendering of the restored
  testimonials (Reviews page + Home "Trusted on the floor" section) and the v3.5
  hero stat tile: no overflow/truncation, tile 4 measures identical width to tile 1.
  Noted one pre-existing (not introduced by v3.5/v3.6) minor legibility quirk: the
  hero background photo is naturally brighter on the right side, so the 2nd column
  of hero stats (previously just "Machines to spec", now also "Years in the field")
  has lower text contrast than the 1st column on mobile — a hero photo/overlay
  design issue, not a CSS bug (confirmed identical computed color/opacity across all
  4 tiles). Flagged to Isaac, not fixed, since it predates this session's changes and
  fixing it well means touching the hero overlay treatment.
