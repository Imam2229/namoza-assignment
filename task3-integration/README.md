# Task 03 — Integration Design (OrthoNow → HubSpot → WhatsApp → Google Ads)

## Architecture

The form posts to a small **serverless middleware function** (a single AWS Lambda
or Vercel function behind an API Gateway route) that owns the whole downstream
flow — not HubSpot's native embed, and not Zapier/Make.

**Flow:** Form submit → middleware → (1) HubSpot Contacts Search API, query by
`phone` → match found → update via `PATCH`; no match → create via `POST` (Name,
Phone, Clinic Preference, Source = "Google Ads — Consultation Landing Page", Lead
Status = "New Enquiry") → (2) same function calls Karix's WhatsApp API directly to
send the confirmation template → (3) the Google Ads conversion fires client-side,
at the same moment the GTM `consultation_form_submitted` event fires, so a slow
HubSpot call never delays the conversion signal.

**Why a direct API call over Forms API/native embed, Zapier, or Make:** the
WhatsApp confirmation has a hard 2-minute SLA. Zapier/Make are largely
polling-based on standard tiers, adding latency before a workflow even starts —
that alone can eat the SLA. A direct API call from purpose-built middleware gives
sub-second control over ordering and retries. Native embed is ruled out since it
can't run the phone-dedup logic below.

**The phone dedup trap:** HubSpot's default contact dedup key is **email**, not
phone — and this form collects no email. Two patients submitting the same phone
with different names would create two separate contacts on a stock HubSpot form.
The middleware searches by phone *before* deciding create vs. update, treating
phone as the de facto unique key. On a repeat submission with a different name,
the existing contact is updated (name overwritten, new clinic preference logged
as an activity note) rather than duplicated — most likely a typo or shared
household phone.

## Biggest failure point + fallback
The middleware function is the single point of failure — if it's down or
HubSpot/Karix time out, the lead is lost silently. Fallback: it writes the raw
submission to a durable queue (SQS/DB row) *synchronously, before* calling any
external API. Downstream calls retry with backoff; after N failures, it alerts
ops via Slack, so no lead is lost even if HubSpot or Karix is down.

## Monitoring the 2-minute SLA
Log the form-submit timestamp and Karix's delivery-webhook timestamp; alert if
the delta exceeds ~90 seconds. Likely break points: an unapproved/expired
WhatsApp message template (Meta requires pre-approval for business-initiated
messages), malformed numbers missing the country code, or Karix rate-limiting
during traffic spikes.
