# Task 01 — GTM Event Schema (OrthoNow)

This schema covers every tracked interaction on the OrthoNow site so the performance
marketing team has full-funnel visibility before paid campaigns go live.

**A note on implementation approach before the table:** GTM's built-in triggers (Click,
Form Submission, Scroll Depth, Page View) work natively for single-action elements —
buttons, links, standard `<form>` submits. They do **not** work for a JS-driven
multi-step form where "step 2" isn't a new page load, just a DOM state change inside
the same form. GTM has no visibility into JS state unless something tells it a state
changed. So for the booking form specifically, the **front-end developer** has to push
a `dataLayer.push()` at each step transition, and GTM listens for those pushes with a
**Custom Event trigger**. This is called out explicitly in the funnel section below.

---

## 1. Full Event Schema

| Event Name | Trigger Type (GTM) | Key Parameters | Feeds Into (GA4) |
|---|---|---|---|
| `booking_step1_complete` | Custom Event (dataLayer push, fired by FE dev on step 1→2 transition) | `step_number`, `clinic_location`, `specialty` | Funnel Exploration (step 1), Booking Funnel Audience |
| `booking_step2_complete` | Custom Event (dataLayer push, fired by FE dev on step 2→3 transition) | `step_number`, `preferred_date`, `has_phone_number` (boolean, not the raw number — see PII note) | Funnel Exploration (step 2) |
| `booking_confirmed` | Custom Event (dataLayer push, fired by FE dev on final confirm) | `step_number`, `clinic_location`, `specialty`, `booking_id` | Key conversion event; Funnel Exploration (step 3); **primary Google Ads conversion import** |
| `call_now_click` | Click Trigger (Just Links / All Elements, filtered on `tel:` href or `data-cta="call-now"`) | `page_location`, `clinic_location` (blank on homepage), `click_text` | Engagement report; "High Intent — No Form" audience |
| `whatsapp_click` | Click Trigger (filtered on href containing `wa.me`) | `page_location`, `widget_state` (open/floating), `clinic_location` | Engagement report; retargeting audience for non-converters |
| `patient_guide_form_submit` | Form Submission trigger (native `<form>` — this one *is* a real page-level form, so built-in trigger works) | `form_id`, `page_location`, `lead_source` | Lead gen conversion (secondary); feeds nurture audience |
| `patient_guide_download` | Custom Event fired by FE only after the gated form succeeds and the actual PDF link is served (prevents crediting the download when the form silently fails) | `file_name`, `form_id`, `page_location` | Content engagement report |
| `clinic_page_view` | Trigger: Page View, condition Page Path matches `/clinics/*` (one trigger, not 9 separate ones — location is captured as a parameter, not a separate event/trigger per page) | `clinic_name`, `clinic_city`, `page_location` | Clinic-level traffic report; location-based remarketing lists |
| `blog_scroll_depth` | Scroll Depth trigger (GTM built-in, thresholds 25/50/75/90%) | `percent_scrolled`, `page_location`, `article_title` | Content engagement / "Engaged Reader" audience for retargeting |

**PII note on `booking_step2_complete`:** name and raw phone number should **not** be
pushed into the dataLayer as event parameters — GA4 terms of service prohibit sending
PII, and it also becomes a data-leak surface since dataLayer is client-side and
inspectable. Only send booleans/metadata (`has_phone_number: true`) at this stage. The
actual contact data goes straight from the form to the backend/CRM over a server call,
not through GTM.

---

## 2. Booking Funnel — Step-Level Drop-off Tracking

### Why native GTM triggers don't work here
The booking form is a single-page, JS-controlled 3-step flow — there's no URL change
and no native form submission between steps 1 and 2. A GTM "Form Submission" or "Page
View" trigger has nothing to listen to at those transitions. The only way GTM sees a
step change is if the front-end code explicitly tells it, via `window.dataLayer.push()`,
at the exact moment a user completes a step and the UI advances.

**So the ownership split is:** the front-end dev writes and fires the `dataLayer.push()`
calls in the form's step-transition logic. I (as the GTM implementer) then build Custom
Event triggers in GTM that listen for those specific event names and map them to GA4
events. I don't write application code that lives in the form component — I hand the
FE dev a spec (exact event name, exact parameter keys, and *when* in the code each push
should fire) and they wire it into the step-change handler.

### DataLayer push spec (what I'd brief the FE dev to implement)

**Step 1 → Step 2** (fires when user selects clinic + specialty and clicks "Next"):
```json
{
  "event": "booking_step1_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty selected}}"
}
```

**Step 2 → Step 3** (fires when user enters name/phone/date and clicks "Next" —
note no raw PII in the payload):
```json
{
  "event": "booking_step2_complete",
  "step_number": 2,
  "step_name": "contact_details_entered",
  "clinic_location": "{{clinic name}}",
  "preferred_date": "{{preferred date}}",
  "has_valid_phone": true
}
```

**Step 3 → Confirmation** (fires only after the backend confirms the booking was
actually created — not on button click — so we don't count failed submissions as
conversions):
```json
{
  "event": "booking_confirmed",
  "step_number": 3,
  "step_name": "booking_confirmed",
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty selected}}",
  "booking_id": "{{backend booking reference}}"
}
```

### Surfacing drop-off in GA4 Funnel Exploration
1. Create a new **Funnel Exploration** in GA4, using an **open funnel** (not closed) —
   we want to see everyone who entered step 1, not just users who arrived from one
   specific entry point, since traffic hits this form from multiple campaigns.
2. Steps configured as: `booking_step1_complete` → `booking_step2_complete` →
   `booking_confirmed`.
3. Turn on **"Show elapsed time"** to see how long users sit on step 2 (contact
   details) before dropping — this is usually the highest-friction step for
   healthcare forms because it's where phone number trust concerns kick in.
4. Use the built-in **breakdown dimension** on `clinic_location` at each step to see
   if drop-off is uneven across the 9 clinics (could indicate a location-specific
   issue, e.g. one clinic's specialty list is broken).
5. The step 2→3 drop-off rate becomes the number I'd flag first in weekly reporting —
   it's the step where we're asking for the most trust (phone number) with the least
   payoff shown yet.

---

## 3. Google Ads Conversion Import — Which Event, and Why

**Import `booking_confirmed` as the primary Google Ads conversion action.**

Not `call_now_click` and not `whatsapp_click`, even though both are tempting because
they're higher-volume and easier to fire. Reasoning:

- **`call_now_click`** only tells us a tap happened — it doesn't tell us a call
  connected, how long it lasted, or whether it resulted in a booking. Importing this
  as a conversion would train Google's bidding algorithm to optimize toward tap volume,
  which is gameable by low-intent traffic (people browsing on mobile who tap
  accidentally) and would actively degrade lead quality over time.
- **`whatsapp_click`** has the same problem — it fires on opening the chat widget, not
  on any actual enquiry being made inside WhatsApp. GTM/GA4 has no visibility into
  what happens after the `wa.me` link opens.
- **`booking_confirmed`** is the one event in this schema that's gated behind an actual
  backend confirmation (not just a UI click), meaning it only fires when a real
  appointment record exists. It's the closest proxy to revenue we have, and it's the
  event Google Ads Smart Bidding should be optimizing toward — otherwise we're feeding
  the algorithm noise instead of signal.

`patient_guide_form_submit` is a reasonable **secondary/micro-conversion** to import
later once we have volume, for top-of-funnel bid adjustments, but it shouldn't be the
primary signal — downloading a guide is a much weaker intent signal than completing a
booking.
