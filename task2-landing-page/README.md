# Task 02 — Landing Page

`index.html` is a single self-contained file — open it directly in any browser, no
server or build step needed.

## How to verify the dataLayer push (for the Loom)
1. Open `index.html`, open DevTools console.
2. Type `window.dataLayer` and hit enter → should log an empty array on page load
   (confirms the event is **not** firing on load).
3. Fill in a name and a valid 10-digit mobile number, click **"Book my
   consultation."**
4. Type `window.dataLayer` again → the `consultation_form_submitted` event object
   should now be in the array, with `form_location`, `page_path`, and
   `campaign_name` params.
5. Confirm the UI swapped to the thank-you state without a page reload (check
   Network tab — no new document request).

## Why it's built this way (performance)
- **No web fonts, no images, no external CSS/JS.** This is the actual lever for a
  90+ Mobile PageSpeed score — most points get lost to font/image round trips and
  render-blocking requests, not "unoptimized code." Everything here is system fonts
  + hand-written inline SVG.
- Single file, ~17KB total, zero network dependencies beyond the HTML document
  itself.
- Form input font-size is 16px specifically to prevent iOS Safari's auto-zoom on
  focus, which otherwise causes a layout jump (bad for CLS/UX on mobile).
- JS is a single deferred block at the end of `<body>`, does nothing until the user
  submits the form — no work competes with initial render.

## Deployment for the PageSpeed screenshot
PageSpeed Insights needs a **live URL** — it can't score a local file. Quickest path:
1. Push this repo to GitHub.
2. Enable **GitHub Pages** on the repo (Settings → Pages → deploy from branch,
   root or `/task2-landing-page`).
3. Run the published URL through [PageSpeed Insights](https://pagespeed.web.dev/),
   select **Mobile**, and save the score screenshot into this folder as
   `pagespeed-score.png`.

*(Screenshot not included here since this build environment has no live hosting —
add it after deploying to GitHub Pages, per the step above, before final submission.)*

## Design decisions (conversion architecture)
- **Form is above the fold**, not buried after marketing copy — this is a bottom-
  funnel campaign landing page (branded "Book a Consultation"), so the visitor
  already has intent; the form shouldn't make them scroll to act.
- **3 fields (name, phone, clinic)** — the brief asked for a 2-field minimal form,
  but Task 3's HubSpot integration expects `clinic_preference` on every contact
  record, and there's no reliable way to infer that server-side. A native
  `<select>` with clinics grouped by city (Bengaluru / Hyderabad / Chennai) adds
  one tap on mobile, not a new page or extra typing — worth flagging as a
  deliberate deviation in the Loom rather than a missed requirement.
- **Trust elements placed after the form, not before it**: clinic count, specialist
  experience, and rating reinforce the decision for people who scroll past without
  converting immediately, rather than delaying the CTA.
- **Signature "desk → pain → diagnosis" strip** speaks directly to the stated
  audience (28–50, desk-bound professionals with knee/back pain) instead of generic
  stock hero imagery — it's the one deliberate visual risk on an otherwise
  restrained page.
