# Task 1 — GTM Event Schema & Booking Funnel Tracking

**Client:** OrthoNow (9 orthopaedic clinics — Bengaluru, Hyderabad, Chennai)
**Author:** [Your Name], Developer Candidate — Namoza

## 0. Context and approach

OrthoNow's current setup (GA4 tracking page views only, no GTM, no CRM
integration) means the business can't easily answer the question that
matters most: *which marketing channel and which page actually led to a
patient booking an appointment?* I tried to design every event around
that question — each one either shows a visitor moving toward a booking,
or helps explain why they didn't.

I used **GTM as the single collection layer** (instead of hard-coding
gtag calls into the WordPress theme) for a few reasons I learned while
researching this: the marketing team can add or adjust tags without a
developer editing theme files, GTM keeps a version history so changes
are traceable, and the same event can be sent to GA4 and Google Ads (and
later other platforms) without touching the page code again.

---

## 1. GTM Event Schema

| Event Name | Business Purpose | GTM Trigger | Event Parameters (min. 3) | GA4 Report | Audience Created | Google Ads Usage |
|---|---|---|---|---|---|---|
| `booking_step_complete` (step 1) | Know how many visitors *start* the booking flow and where they drop | Custom Event trigger listening for `dataLayer` push `event: booking_step_complete`, `step_number: 1` | `step_number`, `step_name` (`location_specialty_selected`), `clinic_location`, `specialty` | Funnel Exploration (step 1 of 4), Events report | "Started Booking, Did Not Finish" (fired step 1, not `booking_success`, in last 7 days) | Not imported directly — feeds remarketing audience for Google Ads RLSA |
| `booking_step_complete` (step 2) | Track mid-funnel drop-off (date/time selection is the highest-friction step in healthcare booking forms) | Same Custom Event trigger, filtered `step_number = 2` | `step_number`, `step_name` (`datetime_selected`), `preferred_date`, `preferred_time_slot` | Funnel Exploration step 2 | "Selected Slot, Did Not Confirm" | Feeds same remarketing pool with a "slot-abandoner" tag for sequential ad messaging |
| `booking_step_complete` (step 3) | Capture intent at the point patient enters contact details — highest-value abandoners | Custom Event trigger, filtered `step_number = 3` | `step_number`, `step_name` (`contact_details_entered`), `lead_phone_provided` (boolean, never raw PII), `clinic_location` | Funnel Exploration step 3 | "Entered Contact Info, No Submit" (priority list for front-desk follow-up calls) | Not imported as a conversion (not a real conversion yet) — used to suppress cold-traffic ads and push warm follow-up creative |
| `booking_success` | **Primary conversion.** Confirms a real appointment request was completed | Custom Event trigger, `event: booking_success` | `step_number: 4`, `booking_id`, `clinic_location`, `specialty`, `appointment_date`, `lead_source` | Conversions report, Funnel Exploration (final step), Attribution report | Excluded from all remarketing (don't re-target converted users) | **Imported to Google Ads as the primary conversion action** (see Section 4) |
| `call_now_click` | Track click-to-call, the dominant conversion path for 45+ patients who prefer calling over forms | Custom Event trigger on click, Trigger Group: Click Trigger filtered by Click ID `contains "call-now"` | `clinic_location`, `phone_number_dialed`, `page_path`, `device_category` | Events report, secondary conversion in Conversions report | "Call Intent Visitors" (for Display remarketing reminding them to call back) | Imported as a **secondary/micro conversion** in Google Ads (lower target value than `booking_success`) |
| `whatsapp_click` | Track WhatsApp widget engagement — a major channel for Indian healthcare leads who prefer chat over forms/calls | Custom Event trigger on click, Click trigger filtered by Click Classes `contains "whatsapp-widget"` | `clinic_location`, `page_path`, `widget_position` (`floating` / `inline`), `device_category` | Events report | "WhatsApp Engaged Visitors" | Imported as **micro conversion**; primarily used as a signal for Smart Bidding, not hard-counted toward CPA targets |
| `patient_guide_download` | Mid-funnel lead-magnet for patients still researching (not ready to book) — captures contact info via gated PDF | Custom Event trigger, `event: patient_guide_download` | `guide_topic` (`knee_pain` / `back_pain` / `post_surgery_care`), `lead_email_provided` (boolean), `traffic_source` | Events report, Engagement report | "Downloaded Guide, Not Booked Yet" (nurture sequence audience) | Imported as **micro conversion** for nurture-stage bidding, never blended into primary CPA |
| `clinic_location_page_view` | Distinguish *which of the 9 clinics* a visitor is researching — essential since OrthoNow runs city-specific campaigns | History Change / Page View trigger filtered `Page Path matches RegEx ^/clinics/` | `clinic_location`, `city`, `referrer_channel`, `page_path` | Custom GA4 report segmented by `clinic_location` dimension | "Bengaluru Clinic Researchers", "Hyderabad Clinic Researchers", "Chennai Clinic Researchers" (for city-level Performance Max campaigns) | Used to build **Customer Match-ready, city-segmented remarketing lists** for Google Ads, not a conversion itself |
| `blog_scroll_depth` | Identify which educational content (knee pain vs back pain articles) actually engages the target audience, to prioritize future SEO/content spend | Scroll Depth trigger at 25/50/75/90% thresholds, restricted to `Page Path contains /blog/` | `scroll_percentage`, `article_topic`, `time_on_page_seconds`, `page_path` | Engagement report, custom Exploration comparing scroll depth by article | "Engaged Blog Readers (75%+)" (top-of-funnel remarketing, cheapest CPM audience) | Not imported to Google Ads as a conversion — used only as a content-engagement signal to brief the content team |

### Why each event exists
I tried to make sure every event answers a real question, not just track for the sake of tracking:
- The four `booking_step_complete` / `booking_success` events exist because OrthoNow's main problem is the 2.1% conversion rate — without step-level data, it's hard to tell whether people are dropping off at "selecting a time slot" (a UX issue) or "entering phone number" (maybe a trust issue). Breaking the funnel into steps turns "fix the conversion rate" into something more specific to look into.
- `call_now_click` and `whatsapp_click` exist because a lot of patients in this market probably call or message instead of filling the form — if that's not tracked, the landing page looks like it's converting worse than it actually is, and Google Ads doesn't get the full picture of what's actually working.
- `patient_guide_download` exists to capture visitors who have pain but aren't ready to book yet — without it, those visitors would just leave and never be followed up with.
- `clinic_location_page_view` exists because OrthoNow has 9 clinics across 3 cities, so it's useful to know which city/clinic a visitor is actually interested in.
- `blog_scroll_depth` exists to see which blog content people actually read, rather than just counting pageviews, which don't tell you if anyone engaged with the content.

### Why each parameter is useful
I kept the parameters limited to what's actually needed to build the GA4 reports and audiences above, rather than collecting extra data "just in case." For example, `clinic_location` shows up on most events because almost every report in this plan needs to be split by clinic. `lead_phone_provided` / `lead_email_provided` are kept as **booleans, not the actual phone/email**, because GA4 and Google Ads aren't supposed to receive personal information directly — only whether the field was filled in.

### Primary vs. micro conversions
- **Primary conversion:** `booking_success` only. This is the one event set as the main conversion action in Google Ads — from what I understand, mixing strong conversions (actual bookings) with weaker ones (clicks, downloads) into one target can confuse Smart Bidding.
- **Micro conversions:** `booking_step_complete` (steps 1–3), `call_now_click`, `whatsapp_click`, `patient_guide_download`, `clinic_location_page_view`, `blog_scroll_depth`. These are marked as **secondary** conversions so Google Ads can still use them as signals without them affecting the main conversion rate OrthoNow would be measured on.

---

## 2. Booking Funnel — Frontend Event JSON

### Step 1 — Location & Specialty Selected
```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "Bengaluru",
  "specialty": "Orthopaedics"
}
```

### Step 2 — Date & Time Selected
```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "datetime_selected",
  "preferred_date": "2026-07-04",
  "preferred_time_slot": "10:30 AM",
  "clinic_location": "Bengaluru"
}
```

### Step 3 — Contact Details Entered
```json
{
  "event": "booking_step_complete",
  "step_number": 3,
  "step_name": "contact_details_entered",
  "lead_phone_provided": true,
  "clinic_location": "Bengaluru",
  "specialty": "Orthopaedics"
}
```

### Step 4 — Booking Success
```json
{
  "event": "booking_success",
  "step_number": 4,
  "booking_id": "ORTHN-20260630-00417",
  "clinic_location": "Bengaluru",
  "specialty": "Orthopaedics",
  "appointment_date": "2026-07-04",
  "lead_source": "Google Ads"
}
```

### Why GTM alone cannot detect these steps
GTM has no native concept of "the user is now on step 2 of a multi-step
form" — a multi-step booking form is almost always a **single-page
application state change** (no full page reload, no new URL), so GTM's
default triggers (Page View, History Change, simple Click) cannot see it.
GTM only reacts to two things: a DOM event it's told to listen for, or a
`dataLayer.push()`. The actual *business logic* — "the user selected a
valid date/time and clicked Next" — lives in application state on the
frontend (React/Vue/vanilla JS form controller), and only the frontend
code knows when that state transition is valid (e.g., it shouldn't fire
`step_number: 2` if the date field is empty). GTM is a listener, not a
decision-maker.

### How frontend developers should implement them
The frontend owns one rule: **push to `dataLayer` only after successful
client-side validation of that step**, never on every click of "Next."
```javascript
function onStepValidated(stepNumber, stepName, extraParams) {
  window.dataLayer = window.dataLayer || [];
  window.dataLayer.push({
    event: "booking_step_complete",
    step_number: stepNumber,
    step_name: stepName,
    ...extraParams
  });
}
```
This is called from the form controller's "advance step" function, after
validation passes and before the UI transitions to the next step — so the
event always reflects a real, valid step completion, keeping GA4 funnel
data trustworthy.

### How GTM listens to them
A single **Custom Event trigger** with Event name `booking_step_complete`
(and a separate one for `booking_success`) is created once in GTM. Inside
the GA4 Event tag, parameters are mapped using GTM's built-in **Data
Layer Variables** (`{{DLV - step_number}}`, `{{DLV - clinic_location}}`,
etc.) rather than hard-coding values — this means if the frontend team
ever adds a new parameter, only the GTM variable needs to be added, not a
new trigger or tag.

### How GA4 Funnel Exploration is configured
In GA4 → Explore → Funnel Exploration: Step 1 = `booking_step_complete`
where `step_number = 1`, Step 2 = same event where `step_number = 2`,
Step 3 = `step_number = 3`, Step 4 = `booking_success`. "Show elapsed
time" and "Trended funnel" are both enabled so the co-founders can see
*when* (which day/week) drop-off increases, not just the aggregate rate —
critical for correlating funnel performance with specific ad campaigns
or site changes.

---

## 3. Google Ads Conversion Recommendation

**Recommended import: `booking_success` only, as a single primary
conversion action with "Include in Conversions" = Yes, counting = "One"
per click (not "Every").**

**Why:** Google Ads' Smart Bidding optimizes toward whatever conversion
action is marked primary. If a micro-conversion like `whatsapp_click`
were also marked primary, the algorithm would start optimizing for
cheap clicks/messages instead of real appointments, which could make the
reported "conversions" look good without actual patient growth.
`call_now_click` and `whatsapp_click` are imported as **secondary**
conversions instead, so Smart Bidding can still use them as a signal
(useful since healthcare research journeys are usually multi-step)
without affecting the main CPA number OrthoNow would be tracking.
