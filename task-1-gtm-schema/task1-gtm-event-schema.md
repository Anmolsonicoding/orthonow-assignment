# Task 01 — GTM Event Schema: OrthoNow

## Full Event Schema

| Event Name | Trigger Type (GTM) | Key Parameters | GA4 Report / Audience It Feeds |
|---|---|---|---|
| `booking_step_complete` | Custom Event trigger, listens for `event: booking_step_complete` pushed by front-end dev on each step's "Next/Confirm" action | `step_number` (1/2/3), `step_name`, `clinic_location`, `specialty` (step 1 only) | Funnel Exploration (booking funnel), drop-off audience per step |
| `booking_confirmed` | Custom Event trigger on final step success push | `clinic_location`, `specialty`, `preferred_date`, `lead_source` | Conversions report; **imported into Google Ads** (see below); "Converted Bookers" audience |
| `call_now_click` | Click trigger, Click ID/Class = `[data-cta="call-now"]`, fires on All Elements click | `page_location`, `clinic_location` (blank on homepage), `cta_position` (header/footer/sticky) | Engagement report; remarketing audience "Called but didn't book" |
| `whatsapp_chat_open` | Click trigger on the floating widget element (`#whatsapp-widget`) | `page_location`, `clinic_location`, `entry_point` (floating/inline) | Engagement report; cross-channel attribution for WhatsApp-originated leads |
| `guide_form_submit` | Custom Event trigger fired by front-end on gating-form submit (before download starts) | `form_id`, `name_provided` (bool), `phone_provided` (bool) | Lead-gen funnel; "Guide Leads" audience for nurture remarketing |
| `patient_guide_download` | Custom Event trigger fired once the PDF actually begins downloading (separate from form submit, since submit ≠ successful download) | `file_name`, `clinic_location` (if known), `link_url` | Content engagement report; confirms file actually reached the user |
| `clinic_location_view` | Custom Event trigger, fired on page load of any of the 9 clinic pages (via dataLayer variable set per template, not URL pattern matching — Bengaluru/Hyderabad/Chennai pages may share URL structures) | `clinic_name`, `city`, `specialty_list` | Pages & Screens report broken out by clinic; location-level performance comparison |
| `blog_scroll_depth` | Scroll Depth trigger, vertical thresholds 25/50/75/90% | `percent_scrolled`, `article_title`, `time_on_page_at_trigger` | Engagement report; content depth audience for blog-to-booking remarketing |

**Note on `clinic_location_view`:** GA4's automatic `page_view` event already fires here, but it won't carry clinic-specific dimensions unless they're pushed explicitly. This custom event exists specifically to attach `clinic_name`/`city` as event params — without it you can't break down performance by clinic in GA4 without scraping the URL path, which is fragile if the CMS ever restructures URLs.

---

## Booking Form Funnel Tracking (3-Step)

**The core issue:** GTM cannot natively detect "step 2 of a JS-driven multi-step form" the way it can detect a click or a URL change. There's no DOM event for "user is now on logical step 2" unless the front-end explicitly tells GTM that via `dataLayer.push()`. This has to be briefed to and implemented by the front-end developer — GTM only *listens*, it doesn't *detect* form state.

### What fires at each step

**Step 1 — Location + specialty selected** (fires when user clicks "Next" after valid selection):
```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty selected}}"
}
```

**Step 2 — Contact details entered** (fires when user clicks "Next" after name/phone/date validate):
```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "contact_details_entered",
  "clinic_location": "{{clinic name}}",
  "preferred_date": "{{preferred date}}"
}
```

**Step 3 — Booking confirmed** (fires on successful submission, after backend confirms the booking — not just on button click, to avoid logging failed submissions as conversions):
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

### GTM setup
- One **Custom Event trigger** matching `event equals booking_step_complete`, firing a single GA4 Event tag that maps `step_number` and `step_name` as event parameters (rather than building 3 separate triggers/tags — keeps it maintainable).
- A second Custom Event trigger on `booking_confirmed`, mapped to its own GA4 Event tag, since this one also needs to fire the Google Ads conversion tag.
- All dataLayer variables (`step_number`, `clinic_location`, etc.) registered as GTM Data Layer Variables, then mapped to GA4 event parameters in the tag config — **and** registered as custom dimensions in GA4 (Admin → Custom Definitions), or they won't be queryable in Explorations.

### Surfacing drop-off in GA4 Funnel Exploration
Build an **open funnel** (not closed) in Explore:
1. Step 1: `booking_step_complete` where `step_number = 1`
2. Step 2: `booking_step_complete` where `step_number = 2`
3. Step 3: `booking_confirmed`

Open funnel (not closed) because some users may re-enter the booking flow mid-session from a different entry point, and we want to see true step-to-step drop-off rather than only users who started at step 1 in that exact session. Breaking the funnel down by `clinic_location` as a secondary dimension shows whether drop-off is concentrated at specific clinics (e.g., a clinic with no available slots at step 2).

---

## Conversion Action to Import into Google Ads

**Import `booking_confirmed`, not `booking_step_complete`, `call_now_click`, or `guide_form_submit`.**

Why this one over the others:
- It's the closest GA4 event to an actual qualified lead — Google Ads' bidding algorithms (Target CPA/ROAS) optimize toward whatever conversion is imported, so importing a soft signal like `call_now_click` (which includes accidental taps, repeat clicks, and people who call but never book) would have the algorithm chase volume over quality.
- `guide_form_submit` is tempting because it's higher-volume, but it's a top-of-funnel nurture signal, not a transaction — optimizing spend toward it would inflate cheap, low-intent leads.
- `booking_confirmed` only fires after backend confirmation, so it's resistant to junk/bot submissions making it into the conversion count, which matters for protecting Smart Bidding signal quality from day one of paid campaigns going live.
