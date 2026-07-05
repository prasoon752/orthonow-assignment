# Task 3 — Integration Design: Landing Page → HubSpot → WhatsApp → Google Ads

**Flow:** Landing Page → HubSpot CRM → Karix WhatsApp API → Google Ads Conversion

## How it works

When a patient submits the form, the page sends the lead data (name,
phone, clinic location, and the Google Ads click ID if present) to a
small backend endpoint — I've kept this as a single API function rather
than putting this logic directly in the WordPress site, mainly because
it keeps lead handling and retries separate from an old WordPress
install that's already doing a lot.

That backend endpoint does three things, one after another:

1. **Creates or updates the contact in HubSpot CRM** using the Contacts
   API.
2. **Sends a WhatsApp confirmation message** through the Karix WhatsApp
   API.
3. **Reports the conversion to Google Ads**, using the `gclid` (Google
   Click ID) that was captured when the visitor first landed on the
   page, so the campaign that brought them in gets credit for the lead.

I chose HubSpot because OrthoNow's front-desk staff already need
somewhere to see and follow up on leads across all 9 clinics — a CRM
is the natural fit instead of building a custom database for this.
Karix is a WhatsApp Business API provider that's commonly used for
Indian businesses and has good delivery rates here, which matters since
this is a healthcare client communicating appointment confirmations.

## Request / Response Flow

1. Frontend → Backend: `POST /api/lead` with `{name, phone,
   clinic_location, gclid}`.
2. Backend → HubSpot: checks if a contact with this phone number already
   exists (see deduplication below), then creates or updates the
   contact and gets back a `contact_id`.
3. Backend → Karix: sends a pre-approved WhatsApp template message
   (booking confirmation) to the patient's number.
4. Backend → Google Ads: reports the conversion using the `gclid`, so it
   shows up against the right campaign in Google Ads reporting.
5. Backend → Frontend: once the HubSpot step succeeds, the frontend gets
   a success response and shows the Thank You message. The WhatsApp
   message and Google Ads conversion are sent right after, but the
   patient isn't kept waiting on those — they only need to know their
   appointment request was received.

## Phone-Based Deduplication

By default, HubSpot identifies duplicate contacts using email. This form
doesn't collect email on purpose (one less field on a small mobile
screen, so more people actually finish filling it). So instead, before
creating a new contact, the backend first searches HubSpot for an
existing contact with the same phone number. To make sure `9876543210`
and `+919876543210` aren't treated as two different people, the phone
number is converted to one consistent format (e.g. `+91XXXXXXXXXX`)
before the search happens. If a match is found, the existing contact is
updated instead of a new one being created, so the same patient calling
in from a different campaign doesn't end up with duplicate records.

## What Could Go Wrong, and How I'd Handle It

**Biggest risk point:** the HubSpot API call. Everything after it — the
WhatsApp message and the front-desk follow-up — depends on the lead
actually being saved in the CRM, so if this call fails, the lead could
be lost.

**How I'd handle a failure:**
- If the HubSpot call fails, the backend retries it 2–3 times before
  giving up, since a lot of API failures are temporary (a slow response,
  a brief outage).
- If it still fails after retrying, the lead is logged with all its
  details and flagged for manual follow-up, instead of being silently
  dropped — someone on the OrthoNow team can check this log and add the
  patient to HubSpot themselves.
- Every step (HubSpot, WhatsApp, Google Ads) logs whether it succeeded
  or failed, with a timestamp, so if patients later report they didn't
  get a confirmation message, it's possible to check the logs and see
  exactly where the process broke.

**Making sure WhatsApp goes out within 2 minutes:** the WhatsApp message
is sent right after the HubSpot step succeeds, not after the Google Ads
step — that way it isn't waiting behind a less time-sensitive call. Using
a pre-approved WhatsApp template (rather than a free-form message) also
helps it go out instantly, since template messages don't need to go
through any extra review before sending.
