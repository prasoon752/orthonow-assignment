# OrthoNow — GTM, Landing Page & Integration Assignment

Technical screening submission for the Developer role at **Namoza
(Healthcare Growth & Strategy)**.
Client brief: **OrthoNow**, a 9-clinic orthopaedic group across
Bengaluru, Hyderabad and Chennai.

## Project Overview

OrthoNow's existing 5-year-old WordPress site has no GTM, GA4 tracking
limited to page views, no CRM integration, and a 2.1% landing page
conversion rate against a 6–8% industry benchmark. This submission
addresses all three parts of the assignment: a complete GTM/GA4 event
schema with booking-funnel tracking (Task 1), a framework-free landing
page built for a Bengaluru-specific knee-pain/back-pain campaign
(Task 2), and a simple, well-reasoned integration design connecting the
form to HubSpot, WhatsApp, and Google Ads (Task 3).

## Features

- Full GTM event taxonomy covering booking funnel, click-to-call,
  WhatsApp, content downloads, location pages, and scroll depth
- Step-by-step booking funnel JSON payloads with GA4 Funnel Exploration
  configuration
- Single-file, dependency-free (no frameworks) responsive landing page
  with an accessible, validated lead form
- `dataLayer` event firing on successful form submission, with no page
  reload (SPA-style thank-you state)
- Integration design covering HubSpot deduplication by phone
  number, WhatsApp confirmation timing, and what happens if an API call
  fails

## Technology Used

- HTML5, CSS3 (custom properties, CSS Grid/Flexbox), vanilla JavaScript
- Google Tag Manager, GA4, Google Ads Conversion Tracking (designed, not
  live-deployed)
- HubSpot CRM API, Karix WhatsApp Business API (architecture only —
  documented in Task 3)

## Folder Structure

```
orthonow-assignment/
├── README.md            ← this file
├── task1.md              ← GTM event schema + booking funnel JSON
├── task3.md               ← integration design (HubSpot/WhatsApp/Ads)
├── index.html             ← landing page (Task 2)
└── screenshots/
    └── pagespeed-screenshot.png   ← placeholder for PageSpeed Mobile result
```

- **`task1.md`** — the full GTM event table, JSON payloads for every
  booking funnel step, and the GA4/Google Ads configuration rationale.
- **`task3.md`** — a 300–400 word write-up on how the form data would
  flow from the landing page to HubSpot, to WhatsApp, and to Google Ads,
  including how duplicate leads are handled and what happens if a step
  fails.
- **`index.html`** — self-contained landing page: open directly in a
  browser, no build step or dependencies required.
- **`screenshots/`** — holds the PageSpeed Insights Mobile screenshot
  once the page is deployed to a real URL (PageSpeed requires a public
  URL, not a local file, so this is generated post-deployment).

## How to Run

No build tools or package manager required.

```bash
# Option 1 — just open the file
open index.html

# Option 2 — serve locally (recommended for testing the WhatsApp/tel links)
python3 -m http.server 8000
# then visit http://localhost:8000
```

## Screenshots

`screenshots/pagespeed-screenshot.png` — to be added after deploying
`index.html` to a public URL and running it through PageSpeed Insights
Mobile. The page was built with PageSpeed Mobile 90+ as an explicit
constraint (no frameworks, no external font loading blocking render,
no layout-shift-causing images, inline critical CSS).

## Assignment Coverage

| Task | Status | File |
|---|---|---|
| Task 1 — GTM Event Schema & Funnel | ✅ Complete | `task1.md` |
| Task 2 — Landing Page | ✅ Complete | `index.html` |
| Task 3 — Integration Design | ✅ Complete | `task3.md` |

## Future Improvements

- Connect `submitLead()` to a real backend endpoint instead of
  simulating a successful submission (kept frontend-only here, per the
  assignment's scope)
- Look into server-side tagging so conversions aren't fully dependent on
  the browser-side dataLayer (some browsers/ad blockers can block this)
- A/B test the hero headline and form field count (name+phone vs.
  phone-only) once real traffic is live
- Add Hindi/Kannada language toggle for broader Bengaluru reach
