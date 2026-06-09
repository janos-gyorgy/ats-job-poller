# LinkedIn post

I was refreshing 30 career pages a week hunting for remote, EU-eligible platform roles. That's a job for a computer, so I gave it to one.

Turns out most company career pages are just a skin over one of three ATS — Greenhouse, Lever, Ashby — and all three hand you the jobs as unauthenticated JSON. No scraping. You poll the same API the page itself calls, normalize the three shapes into one record, and you're past the hard part.

So: an n8n workflow that every 6 hours fetches ~30 companies, filters to the roles and locations I want, scores each new posting against my CV with a local LLM, and emails me a ranked shortlist.

🔌 Three ATS APIs, zero scraping
🌍 Location filter keeps global / pan-EU / my own country — and drops "remote-in-one-other-country" noise
🤖 An LLM scores each role 0–100 with a one-line reason
🗃️ Dedup lives in n8n's own data table — no external database
🔁 Notify-then-mark-seen, so a failed email retries instead of dropping a job

The lesson that stuck: keyword filters can't tell a Kubernetes "Platform Engineer" from a design-system one — but they're the cheap gate that makes one LLM call per job affordable. Deterministic filters first, the model as a ranking layer on top. Never the filter itself.

Code + write-up (MIT): github.com/janos-gyorgy/ats-job-poller

#n8n #devops #automation #homelab #llm
