# AI Appointment Booking Bot — Telegram + Google Calendar (n8n)

Customers book appointments by sending a normal chat message to a Telegram bot ("can I get a consultation next Tuesday at 3?"). Claude extracts the date, time, and purpose; the workflow checks live Google Calendar availability, books the slot if it's free, and replies in chat instantly — confirmation or a request for another time.

Built with [n8n](https://n8n.io), Claude (Anthropic), Telegram Bot API, and Google Calendar.

![Workflow canvas](docs/workflow-screenshot.png)

## The problem this solves

For many businesses in Asia, customers live in chat apps — not email. This bot gives them instant, 24/7 self-service booking in the app they already use, with zero double-bookings because every request is checked against the live calendar before confirming.

## How it works

```
Telegram Trigger (new message)
        │
        ▼
   AI Agent ──── Anthropic Chat Model (Claude)
        │   └─── Structured Output Parser
        │        { startDateTime, durationMinutes, purpose }
        ▼
Google Calendar · Get availability  →  { available: true / false }
        │
        ▼
   IF available?
     ├─ true ──▶ Create calendar event ──▶ Telegram: ✅ Booked!
     └─ false ─▶ Telegram: 😕 slot taken, send another time
```

## Design decisions

- **Same booking engine as the [email version](https://github.com/tazahein/gmail-ai-appointment-booking), different front door.** Only the trigger and reply nodes changed — the AI extraction, availability check, and booking logic are identical. This channel-swap pattern means adding LINE or WhatsApp later is a small job, not a rebuild.
- **Timezone handled explicitly.** The prompt injects today's date via `$now` and requires ISO 8601 output with the `+07:00` offset (Asia/Bangkok), so "tomorrow" and "next Tuesday" resolve correctly instead of drifting by a day.
- **Replies go back to the same chat dynamically** — `chat.id` is read from the incoming message, so the bot serves any number of customers with one workflow.
- **Availability gates the booking.** The event is only created on the IF-true path; double-booking is structurally impossible.
- **Structured output, not free text.** The parser enforces `{startDateTime, durationMinutes, purpose}`, with a 30-minute default duration — downstream nodes never have to parse prose.

## Setup

**You'll need:** an n8n instance, a Telegram bot (create one with [@BotFather](https://t.me/BotFather)), a Google Calendar (OAuth2), and an Anthropic API key.

1. Import `telegram-ai-appointment-booking.json` into n8n.
2. Attach your credentials: Telegram Bot API (token from BotFather), Google Calendar OAuth2, Anthropic.
3. Replace `YOUR_CALENDAR_EMAIL@gmail.com` in both Calendar nodes with your calendar (or re-pick from the dropdown).
4. If you're not in UTC+7, update the offset in the AI Agent prompt and check your Google Calendar's own timezone setting.
5. Activate, open your bot in Telegram, and send: "book me a consultation tomorrow at 3pm".

## Example exchange

> **Customer:** can i come in friday around 10am for a consultation?
> **Bot:** ✅ Booked! Your Consultation is confirmed for 2026-07-03T10:00:00+07:00. See you then 🎉

---

*Part of a series of production-style AI automation projects — more at [github.com/tazahein](https://github.com/tazahein).*
