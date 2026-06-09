# Know the Market Without Job-Hunting: An LLM-Scored Job Poller in n8n

*Also published at [blog.hippotion.com](https://blog.hippotion.com/posts/ats-job-poller/).*

You don't have to be about to change jobs to want to know the landscape — what's being built, what it
pays, where you'd actually fit. Staying current on the market (and your own worth) is just good
professional hygiene. The trouble is that *checking* is tedious, so most of us don't, until we're
already job-hunting and starting cold. So I automated mine: an [n8n](https://n8n.io) workflow that
polls job boards every six hours, scores each new posting against my profile with an LLM, and emails
me only the strong matches.

## Three APIs cover most of the market

Company career pages look bespoke, but the vast majority run on one of three ATS — and all three hand
you the jobs as unauthenticated JSON:

- **Greenhouse** — `boards-api.greenhouse.io/v1/boards/{token}/jobs?content=true`
- **Lever** — `api.lever.co/v0/postings/{token}?mode=json`
- **Ashby** — `api.ashbyhq.com/posting-api/job-board/{token}?includeCompensation=true`

No scraping. You poll the API the page itself calls, normalize the three shapes into one record, and
you're past the hard part.

## "Resolve the token" is half the battle

The naive assumption — *the token is the company name, and everyone's on one of the three* — is half
right. Probing an initial wishlist, **roughly half 404'd everywhere**: HashiCorp (now IBM → Workday),
SUSE (SuccessFactors), Aiven (Teamtailor), Hugging Face. Ship the ones that resolve; for breadth, add
a broad remote-jobs feed (every company, all categories) on top of the curated list.

## Dedup without a database

n8n's **Data Tables** remember what I've seen: a `seen_jobs` table, an `external_id` namespaced
`{source}:{company}:{id}`, and the `rowNotExists` operation drops anything already recorded. No
external database.

## The location filter is a trap

A first version kept everything that wasn't explicitly US-based, and the inbox filled with *"… — Spain
(Remote)"* and *"… — United Kingdom (Remote)"*. Those aren't remote *for me* — they're remote if you
live there. The fix: keep only globally-remote, pan-EU (EMEA/Europe/EU/EEA), or my own country, and
**drop single-country remote** even within the EU. Biggest noise reduction of anything.

## Widen the net, let the model be the bar

Keyword + location filtering gives a candidate list, but it can't tell a Kubernetes "Platform
Engineer" from a design-system one. The job description can. So each new posting gets scored against
my CV — **one small LLM call per job** (batching them all into one call just times out on free tiers),
returning `{score, reason}`. Model: Llama 3.1 8B via NVIDIA NIM, OpenAI-compatible.

That score is what lets me *widen* the search instead of narrowing it: cast a wide net, let the cheap
filters trim it, and **only email the roles scoring 80%+.** Casting wide is fine when a model is the
bar at the door. Scoring is fail-safe — a hiccuping call just skips that job, and everything gets
marked seen either way, so nothing re-scores forever.

## Takeaways

- The "scrape job boards" problem mostly isn't scraping — it's a few public APIs and a normalizer.
- For personal automation, native dedup state beats a database you have to operate.
- An LLM works best as the **bar at the door**: cheap filters keep the cost small, the model gates on
  real fit — which is what lets you cast a wide net without drowning.

Workflow JSON + setup: see the [repo](https://github.com/janos-gyorgy/ats-job-poller).
