# Data Engineer Interview: AI Voice Agent in Production

We run AI receptionists for dental clinics. A voice agent answers and places
calls, looks up appointments and books them in the clinic's practice-management
system (PMS). Leads arrive from GoHighLevel (GHL). Clinics use different PMSs,
and we keep adding more.

This exercise is not about how much you build. It is about how you think:
**finding what is wrong, making it simpler, proving it works, measuring what
matters, and knowing when it breaks.**

**Time box: about 3 hours.** Stop at 3 hours and write down what you would do
next. A small, well-reasoned change beats a large one.

---

## The code you are inheriting

A previous engineer wrote this service. It is "working" in production.

```js
// server.js
import express from "express";
import twilio from "twilio";
import { pms } from "./pms.js"; // practice-management system client (stub it)

const app = express();
app.use(express.json());

const client = twilio(process.env.TWILIO_ACCOUNT_SID, process.env.TWILIO_AUTH_TOKEN);
const GHL_API_KEY = "pit-3f9c2e7a-51b0-4d6e-9a1f-REPLACE_ME";

// 1. GHL calls this when a new lead is created. We call the lead right away.
app.post("/webhooks/ghl/lead", async (req, res) => {
  const { contact } = req.body;
  console.log("new lead", JSON.stringify(req.body));

  await client.calls.create({
    to: contact.phone,
    from: process.env.TWILIO_NUMBER,
    twiml: `<Response>
              <Say>Hi ${contact.firstName}, this is Bright Smile Dental calling about your request.</Say>
              <Connect><Stream url="wss://agent.example.com/stream"/></Connect>
            </Response>`,
  });

  res.sendStatus(200);
});

// 2. The voice agent calls this tool during a live call.
//    `args` are produced by the LLM.
app.post("/tools/lookup_appointments", async (req, res) => {
  const { phone } = req.body.args;
  const appts = await pms.findAppointmentsByPhone(phone);
  console.log("lookup", phone, appts);
  res.json(appts);
});

// 3. The voice agent calls this tool to book. The LLM sends e.g. "2026-11-17 15:00".
app.post("/tools/book_appointment", async (req, res) => {
  const { patientId, slot } = req.body.args;
  const appt = await pms.createAppointment(patientId, new Date(slot + "Z"));
  res.json(appt);
});

app.listen(3000);
```

Each request to `/tools/*` also carries `req.body.call`, set by our voice
platform (not by the LLM):

```json
{ "call": { "sid": "CA123", "clinicId": "clinic_42", "callerPhone": "+13055550123", "direction": "inbound" },
  "args": { "phone": "+13055550123" } }
```

---

## Part 1: Debug and fix (about 75 min)

1. **Review the code as if it were a PR to production.** List every problem
   you find, ranked by how much damage it could do, and say why. Think about
   security, patient privacy, reliability under retries and timeouts, and
   correctness.
2. **Fix the top problems** in a runnable Node.js or Python version of this
   service. Stub `pms` and Twilio; do not call real services.
3. **Write tests that prove your fixes.** At least one test should show an
   attack or failure that worked before your fix and fails after it.

## Part 2: Debug a real complaint (about 15 min)

A clinic in Florida (Eastern Time) writes:

> "A patient asked the AI for Tuesday at 3 pm. She showed up at 3 pm, but our
> schedule had her at 10 am."

Logs:

```
2026-11-16T19:03:40Z call=CA881 clinic=clinic_42 tool=book_appointment args={"patientId":"p_88","slot":"2026-11-17 15:00"} status=200
2026-11-16T19:03:41Z pms.createAppointment patient=p_88 start=2026-11-17T15:00:00.000Z
```

What is the root cause, what is the fix, which test would have caught it, and
how many other patients might be affected? How would you find them?

## Part 3: Simplify (about 25 min, written)

For the next phase, an engineer proposed this architecture:

- API gateway, lead-service, call-service, agent-service, booking-service,
  notification-service, analytics-service (7 services, each with its own Postgres)
- Kafka between every service
- A Hadoop cluster for logs and alerting
- Kubernetes for all of it

Current load: **20 clinics, about 150 calls a week. Two engineers.**

- Draw (ASCII is fine) the **simplest design** you would run today.
- What do you delete, and what would break without each piece you keep?
- At what concrete load or team size would you split something out, and what
  would you split first?

## Part 4: Know when it breaks (about 30 min)

Last quarter, five clinics' voice agents silently stopped receiving calls.
Nobody noticed for weeks. A cron job also crashed 34 times in one week and
everyone ignored the alerts.

1. **Build** a check that alerts when a clinic gets **zero calls during
   business hours for 24 hours**, plus one more alert you think matters most.
   Code plus a test is better than a description.
2. **Write** (short): where alerts go, who owns them, and how you keep alerts
   rare enough that people still read them.
3. **One paragraph:** how would you prove a clinic's agent works end to end
   *before* a patient finds out it does not?

## Part 5: Measure the booking rate, and prove a change raised it (about 45 min)

The folder `data/` holds two weeks of fake data for four clinics:

| File | What it is |
|---|---|
| `clinics.csv` | Clinic id, name, which PMS it uses, its time zone, and its id inside that PMS |
| `calls.csv` | One row per call event from our voice platform (`started_at` is UTC) |
| `appointments_dentrix.csv` | Appointment export from two clinics on Dentrix |
| `appointments_opendental.csv` | Appointment export from two clinics on Open Dental. `apt_status`: 1 scheduled, 2 complete, 3 cancelled, 5 broken (no-show) |

Both PMS exports were taken at midnight local time on 2026-11-12 and use the
clinic's local time. They are what real exports look like: they do not agree
with each other or with our call log on formats, ids or meaning.

1. **Build** one number per clinic: **calls → booked by the AI → attended.**
   Code plus a test is better than a spreadsheet. Write down every row you
   excluded or could not match, and why. We care more about which numbers you
   trust than about the numbers themselves.
2. A third PMS is coming next month, with its own format. **Show where that
   difference lives in your code** so the rest of the system never sees it.
   What is the smallest interface every PMS adapter must satisfy?
3. The clinic dashboard will show this booking rate next to marketing data
   (ad spend, website and search traffic). **What would make that dashboard
   lie?** Name the two most likely ways, and how you would catch each one.
4. We want to change the agent's system prompt to raise bookings. **How do you
   prove it raised bookings without breaking other conversations?** What do
   you compare, over how many calls, and how do you roll back?

## Rules

- **AI coding tools are welcome.** In the follow-up call you will walk us
  through your change, and we will ask about any line of it.
- **Fake data only.** Everything in `data/` is invented, including the phone
  numbers (555-01xx). Do not add real patient data, phone numbers or API keys.
- Keep the change small. We read every line; we would rather read 200 good
  ones than 2,000.
- If something is unclear, make a reasonable assumption and write it down.

## Deliverables

1. Your code and tests, with a short way to run them (one command).
2. `NOTES.md`, at most about two pages: your ranked problem list (Part 1), and
   your answers to Parts 2 to 5, including the
   per-clinic numbers from Part 5 and the rows you excluded.
3. **Your AI session logs.** Export the chats or transcripts from whatever
   tools you used (Claude Code, Cursor, Codex, ChatGPT...) and include them
   unedited in an `ai-logs/` folder. Commit as you go rather than in one
   commit at the end. We want to see how you work with these tools: what you
   asked, what you accepted, what you rejected and why. Heavy AI use is fine;
   we are judging the judgment, not the typing.

## Submission

**Do not open a pull request on this repository; it is public.** Create a
**private** repository with your solution and add the GitHub user we send you,
or send a zip by email. Our GitHub user is `sepehrafla`.

**Deadline:** 12 hours after you receive this exercise. Plan on about 3 hours of
work. A 30-minute follow-up call reviews it with you.
