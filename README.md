# ATS Job Poller (n8n)

A self-hosted [n8n](https://n8n.io) workflow that watches the job market for you. On a schedule it
pulls the **public** job-board APIs of a curated company list **plus a broad remote-jobs feed**,
filters by role and location, scores each new posting against **your** profile with an LLM, and
emails you only the strong matches. No external database, no scraping.

You don't have to be job-hunting to run it — staying current on what's out there (and your own
worth) is just good professional hygiene.

```
Every 6h ─┐
Manual ───┴→ Config → Fetch & normalize → Filter roles → Keep only new → Build score prompt
              → Score (LLM) → Build digest → Has 80%+ match? ─true→ Send email ─┐
                                                            └─false───────────────┴→ Explode rows → Mark seen
```

## Why it works

Most company career pages are just a skin over one of three ATS, and all three expose an
unauthenticated JSON endpoint — so you poll the API the page itself calls, no scraping:

| ATS | Endpoint |
|---|---|
| Greenhouse | `https://boards-api.greenhouse.io/v1/boards/{token}/jobs?content=true` |
| Lever | `https://api.lever.co/v0/postings/{token}?mode=json` |
| Ashby | `https://api.ashbyhq.com/posting-api/job-board/{token}?includeCompensation=true` |

On top of the curated list it also pulls **[Himalayas](https://himalayas.app)** (a broad remote-jobs
feed across thousands of companies) so you're not limited to a hand-maintained list.

## What it does

- **Fetches** every enabled company + the Himalayas feed, each in its own try/catch so one dead
  source can't kill the run. Failures surface as a `⚠️ N sources failing` footer.
- **Filters (cheap, deterministic):** title keyword match + an **allow-only** location gate. Allow-only
  is the point — you list the places you *can* work, and anything naming a place you didn't allow is
  dropped. A deny-list can't enumerate every country; this can't leak one. Global / remote-anywhere
  always passes.
- **Dedups** against an n8n **Data Table** (`rowNotExists`) — no external database.
- **Scores (the LLM):** one small call **per job** (batching all jobs times out on free tiers). The
  model returns `{location_ok, arrangement_ok, score, reason}` — it hard-checks **location**, checks
  **work arrangement** (flexible to the adjacent type, never the opposite), and scores **fit** on
  skills/seniority/domain.
- **Emails only the qualifiers:** `location_ok && arrangement_ok && score ≥ scoreThreshold`, best
  first. Nothing clears the bar → no email. Every new posting is still marked seen, so sub-threshold
  jobs aren't re-scored every run.

Email line: `92% — Company — Senior Platform Engineer (Remote, EMEA) — strong k8s/GitOps match — link`

## Configure it — five fields

Open the **Config** node; the top block is all you touch:

```js
keywords:        ["platform","devops","sre","kubernetes", …]   // title match (the cheap gate)
workArrangement: "remote"            // "remote" | "hybrid" | "onsite"
locationAllow:   ["emea","europe","eu","eea","your country"]    // HARD requirement (allow-only)
scoreThreshold:  80                  // only email roles scoring this or higher
candidateProfile: "…your CV summary…" // what the LLM scores fit against
```

- **`workArrangement`** is flexible to the *adjacent* type but never the opposite: `remote` accepts
  remote+hybrid (never on-site); `onsite` accepts on-site+hybrid (never remote); `hybrid` accepts all.
- **`locationAllow`** are lowercased substring tokens. EU seeker: `["emea","europe","eu","eea","germany"]`.
  US seeker: `["united states","usa","remote us"]`. Global-remote always qualifies regardless.
- **Sources:** `companies[]` (`{company, ats_type, token, enabled}`, `ats_type` ∈ greenhouse|lever|ashby)
  and `himalayas` (`{enabled, pages, perPage}`). Set `himalayas.enabled:false` for a curated-only run,
  or disable companies to lean on the broad feed alone.
- **Engine:** `llmUrl` / `llmModel` (any OpenAI-compatible endpoint), `pollHours`, `maxScorePerRun`
  (caps LLM calls/run), `emailFrom` / `emailTo`.

## Setup

1. **Import** [`workflow/ats-job-poller.json`](workflow/ats-job-poller.json). Requires n8n with **Data Tables**.
2. **Create a Data Table** `seen_jobs` with string columns `external_id, company, title, location, url`
   (the two Data Table nodes reference it by name).
3. **SMTP credential** → select on **Send Email**. From/To come from Config.
4. **LLM credential** → the **Score (LLM)** node defaults to NVIDIA NIM's hosted API; add an
   **HTTP Header Auth** credential, name `Authorization`, value `Bearer <your-key>`. (Self-hosted
   NIM/Ollama? Point `llmUrl` at it and set the node's auth to *None*.)
5. **Prime run (once):** disable *Send Email*, Execute via the Manual trigger to fill `seen_jobs`
   silently, then re-enable.
6. **Activate.** Every 6h; emails only ≥-threshold matches; silent otherwise.

## A note on resolving companies

Not every company is on one of the three ATS, and the token isn't always the name. Roughly half of an
initial wishlist (HashiCorp → Workday, SUSE → SuccessFactors, Aiven → Teamtailor, …) **404'd** on all
three. The list here is the set that resolves; the Himalayas feed covers the long tail. Adding a
company is a config edit, not a code change.

## Notes

- Public ATS endpoints, used as intended. Be polite — 6h is plenty.
- The LLM only sees public job descriptions + the profile *you* write. Nothing else leaves your n8n.

## License

MIT — see [LICENSE](LICENSE).
