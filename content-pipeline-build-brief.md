# Build Brief: Make Scenarios and a Separate Blog

**Oxford Infrastructure Partners / Telia Towers**
2026-08-07. Companion to the content pipeline decision doc.

---

## Part A: The two Make scenarios

You need two, not one. The first finds and drafts. The second distributes.

### Scenario 1: Radar (runs once a day)

Make gives each scenario a single trigger, so don't try to hang six RSS triggers off one scenario. Use a schedule trigger and pull the feeds yourself.

| # | Module | What it does |
|---|---|---|
| 1 | Schedule | Fires daily at 7am |
| 2 | Google Sheets: Search Rows | Reads your feed list, one row per feed URL plus a cluster label |
| 3 | Iterator | Splits into one bundle per feed |
| 4 | HTTP: Make a request | GETs the feed |
| 5 | XML: Parse XML | Turns it into fields |
| 6 | Iterator | Splits into one bundle per article |
| 7 | Data Store: Search records | Looks up the article URL to see if you've handled it before |
| 8 | Filter | Stops anything already seen |
| 9 | Filter | Stops anything matching your blocked-entity list |
| 10 | Anthropic Claude | Scores relevance 1 to 5 and drafts both the blog post and the LinkedIn version |
| 11 | JSON: Parse JSON | Splits the response into fields |
| 12 | Filter | Stops anything scoring under 4 |
| 13 | Google Sheets: Add a row | Writes it to your review queue |
| 14 | Data Store: Add a record | Marks the URL as seen |
| 15 | Aggregator, then Email | Sends you one digest instead of fifteen emails |

Three things worth getting right:

**Put both filters before the Claude call.** Steps 8 and 9 come first because filters are free in Make and Claude calls are not. This also means the blocked-entity list is a genuine hard stop, since nothing sensitive ever reaches the drafting step.

**Watch your operations budget.** Make bills per module execution per bundle, and step 7 runs once for every article in every feed, so that's your biggest line. Six feeds at roughly twenty items each works out to about 150 to 200 operations per run, or around 5,000 a month, which sits comfortably inside the $9 Core plan. If you add feeds and the number climbs, the fix is fewer feeds or a tighter Google News query, not a bigger plan.

**Ask Claude for JSON.** Have it return one object with `score`, `blog_draft`, `linkedin_draft`, and `source_url` so step 11 can split it cleanly. If you ask for prose you'll spend an afternoon writing text parsers.

### Scenario 2: Distribute (runs on a trigger)

| # | Module | What it does |
|---|---|---|
| 1 | RSS: Watch RSS feed items | Watches your blog feed |
| 2 | LinkedIn: Create an Organization Post | Posts to the company page |
| 3 | Slack or Email | Sends the personal-profile draft to whoever's posting it |

Make has its own LinkedIn module that posts to company pages, so you can skip Buffer and save the subscription. You connect the page through Make's already-approved LinkedIn app, which means you still don't need your own LinkedIn API access.

Leave scenario 2 switched off for the first week. Run scenario 1, read what lands in the sheet, and tune the prompt until the drafts are worth publishing. Then turn on distribution.

---

## Part B: The separate blog

### Can you attach it to the main domain?

Yes, on a subdomain. Not on a subdirectory.

**A subdomain works.** Something like `insights.oxfordinfrastructurepartners.com` points at GitHub Pages with a single CNAME record, and your main Squarespace site is untouched. Setup runs about twenty minutes plus DNS propagation:

1. In Squarespace, go to `account.squarespace.com/domains`, pick the domain, open DNS Settings.
2. Add one CNAME record. Host is `insights`, data is `yourusername.github.io`. Leave every existing record alone, because the apex still needs to point at Squarespace.
3. In the GitHub repo, open Settings, then Pages, and enter `insights.oxfordinfrastructurepartners.com` as the custom domain. GitHub writes the CNAME file into the repo for you.
4. Wait, then tick Enforce HTTPS. GitHub issues a free Let's Encrypt certificate and renews it automatically. The option can take up to 24 hours to appear, and DNS can take up to 48.

**A subdirectory doesn't.** You can't serve `oxfordinfrastructurepartners.com/insights` from GitHub Pages while Squarespace serves the rest, because Squarespace has no path-based routing or reverse proxy. Doing it anyway means putting Cloudflare in front of your entire site and proxying Squarespace through it, which Squarespace doesn't officially support and which puts your main site at risk to save one URL segment. Don't.

### On the SEO point

You said you don't think you'd get SEO from it, and that's worth correcting, because it changes the maths. A subdomain does get crawled, indexed, and ranked. Google treats it as connected to your main domain but keeps the signals somewhat separate, so a subdirectory would consolidate authority a little better. The gap is real but modest, and it's nothing like zero. If you're publishing two useful posts a week about tower and data center transactions, that content will rank for the long-tail terms whether it sits on a subdomain or not.

### How publishing actually works

This is the part that makes the separate blog worth considering, because it turns your daily copy-paste into a single tap.

Set up a Jekyll site in a GitHub repo. Blog posts are just markdown files in a `_posts` folder. Add a step to scenario 1 that writes the approved draft as a markdown file using Make's HTTP module against the GitHub REST API, specifically `PUT /repos/{owner}/{repo}/contents/{path}`. Make does have a GitHub app, but check which actions it exposes before you rely on it, because the HTTP module against the REST API definitely works and takes about ten minutes to wire up.

Have Make commit to a `drafts` branch and open a pull request rather than committing straight to main. You then get a notification, read the draft on your phone, and tap merge to publish. GitHub rebuilds the site in about a minute. That's your approve-before-publish gate, and it's faster than the Squarespace paste by a wide margin.

Jekyll also generates an RSS feed for free through the `jekyll-feed` plugin, which is what scenario 2 watches.

### What you give up

The Squarespace editor. Anyone who wants to write or fix a post has to edit markdown, either in GitHub's web editor or locally. That's fine for you and probably fine for anyone technical, but it will block a colleague who just wants a normal text box. If that matters, add Decap CMS to the repo. It's free, it gives you a proper editing interface in the browser, and it commits to the same repo underneath.

### Two notes on hosting

GitHub's terms say Pages isn't for running an online business or e-commerce or software as a service. A company insights blog is a normal use and isn't what that clause is aimed at, so you're fine. If you'd rather have no ambiguity at all, Cloudflare Pages and Netlify both do the same job, both have free tiers, both take a custom domain, and neither has that restriction. The git workflow is identical, so you can switch later without rebuilding anything.

### Linking it back to Squarespace

Add a nav item on the main site pointing at the subdomain, and match the blog's styling to the main site so the move doesn't feel like leaving. You can also drop an RSS block on the Squarespace homepage that pulls in the three most recent posts, which keeps fresh headlines on the main domain even though the posts live elsewhere.

---

## What to do this week

Do these in order and stop when you run out of time. Each step is useful on its own.

1. Build the feed list and the blocked-entity list in a Google Sheet. About an hour, and it's the input to everything else.
2. Build scenario 1 and let it fill the review queue for a few days. Publish by hand into Squarespace while you read the output and tune the prompt.
3. Once the drafts are good, decide on the blog. Staying on Squarespace costs you ten minutes a day forever. Moving to a subdomain costs you a day of setup and gives you the merge-to-publish workflow.
4. Build scenario 2 last, whichever blog you land on, because both routes end in an RSS feed and the scenario is the same either way.

The ordering matters because step 2 answers the question that decides step 3. If the drafts turn out to need heavy rewriting every time, the manual Squarespace paste costs you almost nothing extra and you should stay put. If they come out close to publishable, the merge-to-publish workflow is worth the move.
