# ATS Job Poller (n8n)

A self-hosted [n8n](https://n8n.io) workflow that polls the **public** job-board APIs of a fixed
list of companies on a schedule, filters for the roles and locations you care about, scores each
new posting against **your** profile with an LLM, deduplicates against an n8n Data Table, and emails
you a single digest **sorted best-match-first**.

It replaces manually refreshing 30+ career pages. One editable config node, no external database,
no scraping — just the ATS APIs those companies already expose.

```
Every 6h ─┐
Manual ───┴→ Config → Fetch & normalize → Filter roles → Keep only new
              → Build score prompt → Score (LLM) → Build digest → Send email → Explode rows → Mark seen
```

## Why it exists

Most company career pages are just a skin over one of three ATS (applicant tracking systems), and
all three expose an unauthenticated JSON endpoint:

| ATS | Endpoint |
|---|---|
| Greenhouse | `https://boards-api.greenhouse.io/v1/boards/{token}/jobs?content=true` |
| Lever | `https://api.lever.co/v0/postings/{token}?mode=json` |
| Ashby | `https://api.ashbyhq.com/posting-api/job-board/{token}?includeCompensation=true` |

So instead of refreshing pages, you poll the APIs, normalize the three shapes into one, and let a
workflow do the watching.

## What it does

- **Fetches** every enabled company once per run, each in its own try/catch so one dead source can't
  kill the batch. A failing source surfaces as a `⚠️ N sources failing` footer in the email — so a
  company silently switching ATS doesn't just vanish from your results.
- **Filters** by role keyword **and** a location policy designed for remote/EU job-hunting: keep
  globally-remote, pan-EU (EMEA/Europe/EU/EEA), or your home country — and **drop single-country
  remote** (a "Spain-remote" role needs Spain residency). Ambiguous locations are kept on purpose.
- **Dedups** against an n8n **Data Table** (`rowNotExists`) — no external database, state lives in n8n.
- **Scores** every new posting 0–100 against an editable profile, in **one batched LLM call** per run
  (OpenAI-compatible; defaults to NVIDIA NIM). The email is the **full** list sorted by score — no
  cutoff, you decide.
- **Notifies, then marks seen** — the insert happens *after* a successful email, so a failed send
  retries next run instead of silently dropping a posting.

Email line: `92% — Company — Senior Platform Engineer (Remote, EMEA) — strong k8s/GitOps match — link`

## Setup

1. **Import** [`workflow/ats-job-poller.json`](workflow/ats-job-poller.json) into n8n (Workflows →
   Import from File). Requires n8n with **Data Tables** (n8n ≥ 1.x with the feature enabled).
2. **Create a Data Table** named `seen_jobs` with string columns:
   `external_id, company, title, location, url`. The two Data Table nodes reference it by name.
3. **SMTP credential** — create one and select it on the **Send Email** node. `From`/`To` come from
   the Config node.
4. **LLM endpoint** — the **Score (LLM)** node POSTs to `Config.llmUrl` (default: NVIDIA NIM public
   API). Add an **HTTP Header Auth** credential (`Authorization: Bearer <your-key>`) on the node, or
   point `llmUrl` at a self-hosted NIM/Ollama and remove the auth.
5. **Prime run (once)** — disable *Send Email*, run once via the Manual trigger to fill `seen_jobs`
   without emailing, then re-enable. (Otherwise your first email lists every currently-open match.)
6. **Activate.** Runs every 6h; emails only new matches, best-scored first; silent when nothing's new.

## Configuration

Everything lives in the **Config** node, one object:

- `companies[]` — `{ company, ats_type, token, enabled }`. `ats_type` is `greenhouse | lever | ashby`.
  Find a company's `token` by loading its careers page, watching the network tab for a request to one
  of the three hosts above, and taking the slug — or just try the slug against each endpoint.
- `candidateProfile` — the rubric the LLM scores against. **Replace the placeholder with your own.**
- `roleKeywords` — the candidate gate (keeps LLM cost to one call on a small set). Relax for breadth.
- `locationRegion` / `locationHome` / `locationDeny` — the location policy. Region/home win over deny.
- `llmUrl` / `llmModel` — any OpenAI-compatible chat endpoint.
- `pollHours` — change here **and** on the *Every 6h* Schedule Trigger.

## A note on resolving companies

Not every company is on one of the three ATS, and the token isn't always the company name. When this
was built, roughly half of an initial wishlist (HashiCorp, SUSE, Aiven, Hugging Face, …) **404'd** on
all three — they're on Workday / SuccessFactors / Teamtailor / custom systems. The shortlist here is
the set that actually resolves. Adding more is a config edit, not a code change.

## Notes

- Public ATS endpoints, used as intended (these power the companies' own career pages). Be polite —
  6h is plenty.
- The LLM only sees public job descriptions. No personal data leaves your n8n instance except the
  profile *you* write into the config.

## License

MIT — see [LICENSE](LICENSE).
