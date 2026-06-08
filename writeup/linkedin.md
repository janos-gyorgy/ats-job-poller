# LinkedIn post

I was refreshing 30 career pages looking for remote, EU-eligible platform/SRE roles. So I built a
thing to do it for me — and learned a few things worth sharing.

Most company career pages are just a skin over one of three ATS (Greenhouse, Lever, Ashby), and all
three expose the jobs as unauthenticated JSON. No scraping needed — you poll the same API the page
itself calls.

So: an n8n workflow that every 6 hours →
→ fetches ~30 companies' boards (each in its own try/catch, so one dead source can't kill the run)
→ filters for the roles + locations I actually want
→ scores every new posting 0–100 against my CV with a local LLM (Llama 3.3 70B on NVIDIA NIM)
→ emails me a single digest, sorted best-match-first

Three things I didn't expect:

1) Half my company wishlist 404'd. The token isn't always the company name, and plenty of firms
(HashiCorp, SUSE, Hugging Face…) aren't on the big three ATS at all. Verify before you trust a slug.

2) "Remote — Spain" is not remote for me. The biggest noise-killer was dropping single-country remote
roles and keeping only global / pan-EU / my-own-country. Region beats country.

3) The LLM belongs on top of the filters, not instead of them. Keyword matching can't tell a
Kubernetes "Platform Engineer" from a design-system one. The job description can. So the model ranks;
it doesn't filter. A 40% with a one-line "why" is still signal.

No database (n8n Data Tables for dedup), no scraping, one editable config node. Notify-then-mark-seen
so a failed email retries instead of dropping a job.

Code + write-up (MIT): github.com/janos-gyorgy/ats-job-poller

#n8n #DevOps #automation #homelab #LLM #jobsearch
