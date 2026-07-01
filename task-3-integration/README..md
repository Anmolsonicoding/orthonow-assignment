# Task 03 — Integration Design: Landing Page → HubSpot → WhatsApp → Google Ads

## Architecture

The form does **not** submit directly to HubSpot's client-side Forms API. Instead, on submit it does two things in parallel: (1) pushes `consultation_form_submitted` to `dataLayer`, which GTM listens for and fires the Google Ads conversion tag immediately, client-side — this is the fastest path to get the conversion signal into Ads; and (2) POSTs the form payload to a small serverless function (e.g. a Cloud Function or Vercel Edge Function) that I own and control, which acts as the orchestrator.

I chose a direct server-side function over Zapier/Make or HubSpot's native embed because: HubSpot's native form embed and default Forms API can't run my own dedupe logic before writing the contact; Zapier/Make introduce polling or queue delay that eats into the 2-minute WhatsApp SLA and give me no control over retry behaviour; and the WhatsApp (Karix) API key can't be exposed in a client-side Zapier webhook without extra hops. A direct function call gives me synchronous control over ordering and error handling.

**Order of operations in the backend function:**
1. **Search HubSpot by phone number first** (via the Contacts Search API, filtering on a custom unique property, not email) to check for an existing contact.
2. **Create or update** the contact: Name, Phone, Clinic Preference, Source = "Google Ads – Consultation Landing Page", Lead Status = "New Enquiry". If found, update rather than duplicate.
3. **Call Karix's WhatsApp Business API** to send the confirmation template message, using the HubSpot contact ID as a reference for logging.
4. **Fire a server-side Google Ads conversion (Conversions API)** as a backup to the client-side tag, deduplicated by a unique transaction ID, in case the client-side tag is blocked by an ad blocker or the user closes the tab before GTM fires.

## Phone Deduplication — the trap

HubSpot deduplicates contacts by **email** by default, not phone — and this form collects no email. Without intervention, two submissions from the same number with different names create two separate contacts, splitting the lead's activity history and confusing the clinic's follow-up calls. I'd set phone as a **unique custom property** on the Contacts object (HubSpot supports this), and have the function search on that property before every create. If a second submission arrives with the same phone but a different name, I update the existing contact's name field and append the prior name to a "Previously submitted as" note property, rather than creating a duplicate — so the call team sees both names and isn't blindsided mid-call.

## Biggest Failure Point & Fallback

The Karix WhatsApp call is the most fragile link — third-party API, rate limits, template approval issues. Fallback: retry with exponential backoff within the 2-minute budget; if still failing past ~90 seconds, push a task into HubSpot assigned to a human agent to call the patient directly, and log the failure to a dead-letter queue for review — so the SLA is met by a person even if the automation isn't.

## Monitoring the 2-Minute SLA

Risks: Karix downtime/latency, function cold starts, HubSpot API rate limits. I'd log a timestamp at form-submit and at WhatsApp-sent for every lead, track P95 delivery time on a dashboard, and alert (Slack/PagerDuty) if delivery time or failure rate crosses a threshold, plus a daily reconciliation job comparing HubSpot contacts created against WhatsApp messages confirmed sent.
