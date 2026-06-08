# Stop Refreshing Career Pages: An LLM-Scored Job Poller in n8n

*Also published at [blog.hippotion.com](https://blog.hippotion.com/posts/ats-job-poller/).*

I was refreshing the same thirty career pages every few days, scanning for remote, EU-eligible
platform/SRE roles. It's exactly the kind of repetitive watching a computer should do. So I gave it
to one — an [n8n](https://n8n.io) workflow that polls company job boards every six hours, keeps the
roles that match, scores each new one against my profile with a local LLM, and emails me a ranked
shortlist.

## Three APIs cover most of the market

Company career pages look bespoke, but underneath, the vast majority run on one of three ATS — and
all three hand you the jobs as unauthenticated JSON:

- **Greenhouse** — `boards-api.greenhouse.io/v1/boards/{token}/jobs?content=true`
- **Lever** — `api.lever.co/v0/postings/{token}?mode=json`
- **Ashby** — `api.ashbyhq.com/posting-api/job-board/{token}?includeCompensation=true`

No scraping, no headless browser. You poll the API the page itself calls, normalize the three shapes
into one record, and you're done with the hard part.

## "Resolve the token" is half the battle

The naive assumption — *the token is the company name, and everyone's on one of the three* — is half
right. When I probed my initial wishlist, **roughly half 404'd everywhere**: HashiCorp (now under IBM
→ Workday), SUSE (SuccessFactors), Aiven (Teamtailor), Hugging Face. The honest move was to ship the
~33 that actually resolve and leave the rest as disabled config stubs.

## Dedup without a database

n8n's **Data Tables** remember which jobs I've seen: a `seen_jobs` table, an `external_id` namespaced
`{ats}:{company}:{id}`, and the `rowNotExists` operation drops anything already recorded. No external
database. And the ordering matters: **notify first, mark seen second** — the insert only happens after
the email sends, so a failed send retries instead of swallowing a posting.

## The location filter is a trap

My first version kept everything that wasn't explicitly US-based, and the inbox filled with *"… —
Spain (Remote)"* and *"… — United Kingdom (Remote)"*. Those aren't remote *for me* — they're remote if
you live there. The fix: keep only globally-remote, pan-EU (EMEA/Europe/EU/EEA), or my own country,
and **drop single-country remote** even within the EU. That cut the noise more than anything else.

## Let an LLM read the actual job

Keyword + location filtering gives you a candidate list, but it can't tell a Kubernetes "Platform
Engineer" from a design-system one. The job description can. So the last step batches every new
posting into **one** call to a local NVIDIA NIM endpoint (Llama 3.3 70B, OpenAI-compatible) with my
CV as the rubric, returning a `{id, score, reason}` per job. The email is the **full** list sorted
best-first — no threshold. The model *ranks*; it doesn't *filter*. A 40% with a one-line "why" is
still signal.

## Takeaways

- The "scrape job boards" problem mostly isn't a scraping problem — it's three public APIs and a
  normalizer.
- For personal automation, native dedup state beats a database you have to operate.
- LLMs shine as a *ranking* layer on top of deterministic filters — not as the filter itself.

Workflow JSON + setup: see the [repo](https://github.com/janos-gyorgy/ats-job-poller).
