# Data Engineer Interview

We run AI receptionists for dental clinics. Leads arrive from GoHighLevel
(GHL), our voice agent calls them or answers their calls, and books
appointments in the clinic's practice-management system (PMS). Every clinic's
data arrives in a different shape, and we keep adding clinics and PMSs.

We want to know, per clinic and per marketing channel:
**lead → called → booked by the AI → attended.**

This exercise is about how you think: **reading messy data before trusting it,
building a pipeline that is simple and correct, proving the numbers, and running
it in production.**

**Time box: about 3 hours of work.** Stop at 3 hours and write down what you
would do next. A small, well-reasoned pipeline beats a large one.

---

## The data

Everything in `data/` is invented: two weeks for four clinics, plus a sample
from a fifth.

| File | Format | What it is |
|---|---|---|
| `clinics.csv` | CSV | Clinic id, name, PMS, time zone, the clinic's id inside its PMS, and its GHL location id |
| `ghl_leads.jsonl` | JSON lines | Leads from GHL forms and ads. Nested; `attribution` may be missing |
| `calls.csv` | CSV | One row per call event from our voice platform. `started_at` is UTC. `outcome` is what the agent reported |
| `appointments_dentrix.csv` | CSV | Appointment export from the two Dentrix clinics |
| `appointments_opendental.csv` | CSV | Appointment export from the two Open Dental clinics. `apt_status`: 1 scheduled, 2 complete, 3 cancelled, 5 broken (no-show) |
| `new_pms_sample.json` | JSON | A sample export from a new PMS we are adding next month (clinic in Denver). Timestamps are epoch milliseconds |

The PMS exports were taken at midnight local time on 2026-11-12. These files
are what real exports look like: they do not agree with each other on formats,
ids, time zones or meaning. Some rows should not be counted.

---

## Part 1: Build the pipeline (about 90 min)

1. **Load all five sources** into one clean model. Pick the tools you think fit
   (Python, SQL, DuckDB, dbt, pandas... anything that runs locally).
2. **Produce two tables:**
   - per clinic: calls → booked by the AI → appointments due → attended;
   - per clinic and marketing channel (`attribution.medium`): leads → called →
     booked → attended.
3. **Write tests** that would fail if the pipeline counted something wrong.
4. **Write down every row you excluded or could not match, and why.** We care
   more about which numbers you trust than the numbers themselves.

## Part 2: Add a new PMS (about 25 min)

Write the adapter for `new_pms_sample.json`, with a test, so the rest of your
pipeline handles it without changes.

- What is the smallest interface every PMS adapter must satisfy?
- What in this sample would you ask the PMS vendor about before trusting it?

## Part 3: Debug a complaint (about 15 min)

A clinic in Florida (Eastern Time) writes:

> "Your dashboard says your AI booked a patient for Tuesday at 10 am. She came
> in at 3 pm, which is what she asked for."

The booking-tool log:

```
2026-11-16T19:03:40Z call=CA881 clinic=clinic_42 tool=book_appointment args={"patientId":"p_88","slot":"2026-11-17 15:00"} status=200
2026-11-16T19:03:41Z pms.create_appointment patient=p_88 start=2026-11-17T15:00:00+00:00
```

What is the root cause, which test would have caught it, and how would you find
every other affected appointment in the data?

## Part 4: Run it in production (about 30 min, written)

Someone proposed this stack for the pipeline: Kafka, Spark, Airflow, a
Snowflake warehouse and dbt Cloud. Current volume: **about 200 calls and 150
leads a week across 20 clinics. Two engineers.**

1. Draw (ASCII is fine) the **simplest setup** you would run today: where it
   runs, how often, and where the tables live. What would you delete from the
   proposal, and at what volume would you add each piece back?
2. **Reruns and late data:** a PMS export arrives a day late, or twice. How do
   you make reruns safe and avoid double counting?
3. **Feeds that break:** a PMS renames a column, or a clinic's export stops
   arriving. How do you notice before a clinic notices? Where does the alert
   go?
4. **Patient data:** these files hold phone numbers and appointment times.
   What do you keep, hash or drop, and who can see the raw files?

## Part 5: Prove a change worked (about 20 min, written)

1. The dashboard will show your booking rate next to ad spend. **Name the two
   most likely ways it could lie**, and how you would catch each.
2. We change the agent's prompt to raise bookings. **How do you prove it raised
   bookings?** What do you compare, over how many calls, and what would make
   you roll it back?

---

## Rules

- **AI coding tools are welcome.** In the follow-up call you will walk us
  through your work, and we will ask about any line of it.
- **Fake data only.** Everything in `data/` is invented, including the phone
  numbers (555-01xx). Do not add real patient data, phone numbers or API keys.
- Keep it small. We read every line; we would rather read 200 good ones than
  2,000.
- If something is unclear, make a reasonable assumption and write it down.

## Deliverables

1. Your code and tests, with one command to run them and print both tables.
2. `NOTES.md`, at most about two pages: your excluded rows and assumptions
   (Part 1), and your answers to Parts 2 to 5.
3. **AI decisions**, a short section in `NOTES.md`: three to five suggestions
   from your AI tools that you accepted or rejected, and why. Exported chat
   logs are welcome but optional.

## Submission

**Do not open a pull request on this repository; it is public.** Create a
**private** repository with your solution and add the GitHub user we send you,
or send a zip by email to sepi@vdsupport.com.

**Deadline:** 72 hours after you receive this exercise. Plan on about 3 hours of
work. A 60-minute follow-up call reviews it with you, including a small live
change to your pipeline.
