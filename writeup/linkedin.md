# LinkedIn post

You don't have to be job-hunting to want to know your market — what's out there, what it pays, where
you'd actually fit. Staying current on the landscape (and your own worth) is good professional
hygiene. The catch is that checking is tedious, so we don't — until we're already job-hunting and
starting cold.

So I automated mine. Most company career pages are just a skin over one of three ATS (Greenhouse,
Lever, Ashby), and all three hand you the jobs as unauthenticated JSON. No scraping — you poll the
same API the page itself calls.

An n8n workflow, every 6 hours:
🔌 pulls those ATS APIs + a broad remote-jobs feed (every company, all categories)
🌍 keeps only global / pan-EU / my-own-country roles — drops "remote-in-one-other-country" noise
🤖 scores each posting 0–100 against my CV with a local LLM
✉️ emails me only the 80%+ matches — silent when nothing clears the bar
🗃️ dedup + state in n8n's own data table, no external database

The shift that made it work: stop narrowing the search, start widening it. Cheap keyword + location
filters keep the candidate set (and the LLM cost) small — then the model is the bar at the door,
gating on real fit. Cast a wide net; let the 80% line decide what reaches you.

Code + write-up (MIT): github.com/janos-gyorgy/ats-job-poller

#n8n #devops #automation #homelab #llm
