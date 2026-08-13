# Content Pipeline: Google Alerts to Squarespace to LinkedIn

**Decision doc — Oxford Infrastructure Partners / Telia Towers**
Prepared 2026-08-07

---

## TL;DR

The pipeline you described is automatable at both ends but **not in the middle**. Squarespace has no public API for blog content and no Zapier action, so nothing can write a blog post into Squarespace from the outside. That is a hard wall, not a configuration problem.

The good news: the wall lands exactly where you want a human anyway. Auto-publishing curated news commentary to a LinkedIn company page without review is the highest brand-risk part of this whole idea, so a forced approve-before-publish gate at the Squarespace step is a feature, not a tax.

**Recommended stack:** Make.com + Claude API + Squarespace RSS + Buffer, roughly **$12 to $25/month**, one human touch of about 10 minutes per day.

---

## 1. Hop-by-hop map

| Hop | Automatable? | Mechanism |
|---|---|---|
| Keywords to alerts | Yes | Google Alerts RSS + Google News RSS |
| Alerts to filtered, ranked shortlist | Yes | Make.com scenario, dedupe + relevance filter |
| Shortlist to drafted post | Yes | Claude API step inside Make |
| Draft into Squarespace | **No** | No API. Human paste, ~2 to 3 min/post |
| Squarespace to LinkedIn company page | Yes | Squarespace blog RSS to Buffer or Make |
| Squarespace to personal profiles | Technically yes, **advise no** | See §5 |

### Hop 1: Google Alerts to RSS

Google Alerts has no official API, but the RSS delivery option still works as of 2026. In google.com/alerts, set "Deliver to" = RSS feed and copy the feed URL.

Two cautions, and one upgrade worth making:

- Alerts RSS feeds are text-only (no images) and the URLs occasionally go stale when an alert is edited. Store them somewhere you can re-pull.
- Alert quality is genuinely mediocre for narrow B2B infrastructure terms. Expect noise.
- **Upgrade:** use **Google News RSS** alongside Alerts. The format is `https://news.google.com/rss/search?q=YOUR+QUERY` and it accepts full search operators, so you can write far tighter queries (`"tower company" AND (divestment OR carve-out)`, `site:` filters, date windows). It is more reliable and more controllable than Alerts. Run both; Alerts catches the long tail, News RSS carries the precision.

Suggested keyword clusters: tower cos and TowerCo transactions, data center and edge buildout, Telia / Telia Towers, your named competitors and partners, plus deal-flow terms (divestment, sale-leaseback, carve-out, fiber M&A) in your target geographies.

### Hop 2: The Squarespace wall

Confirmed across the Squarespace developer docs, the Zapier integration, and Squarespace's own forums:

- Squarespace's public APIs cover **commerce only**: orders, products, inventory, transactions, plus webhooks. There is no content, page, or blog write endpoint.
- The **Zapier integration has exactly one trigger (form submission) and zero actions.** You cannot create a Squarespace blog post from Zapier.
- There is **no RSS import.** Squarespace's RSS blocks *display* external feeds in a widget; they do not create native posts. Blog import only supports migrations from other blog platforms (WordPress, Blogger, Tumblr, etc.), not a live feed.

Which leaves three honest options:

1. **Human paste (recommended).** AI writes the draft into a review queue; a person pastes and publishes. Squarespace's native post scheduler then handles timing.
2. **Browser automation** against the Squarespace editor. Do not do this. It is brittle against UI changes, and driving the editor with a script sits badly against Squarespace's terms. Not worth it for two posts a week.
3. **Move the blog off Squarespace** to an API-capable CMS (Ghost, WordPress, Webflow) on a subdomain such as `insights.oxfordinfrastructurepartners.com`. This unlocks genuine end-to-end automation. Covered as Option B in §3.

### Hop 3: Squarespace to LinkedIn

This hop is easy and well-supported. Every Squarespace blog exposes an RSS feed at `https://yoursite.com/blog?format=rss` (append `?format=rss` after the page slug).

Point Buffer or Make at that feed, and new posts fan out to the LinkedIn company page automatically.

**On LinkedIn API approval, the key point:** you almost certainly do not need it. Building direct against LinkedIn means the Community Management API, which requires a registered legal entity, a verified company page, and a two-tier app review that takes weeks and is frequently rejected. Using Buffer, Hootsuite, or Make instead means you post through *their* already-approved app. You just OAuth your page in. OIP would likely qualify for direct access eventually, but at this volume it buys you nothing.

---

## 2. Where the AI step goes

Put Claude **between the filter and the review queue**, and give it three jobs in one call:

1. **Score relevance** 1 to 5 against a short OIP/Telia Towers context blurb. Anything below 4 never reaches a human.
2. **Draft the blog post**: 120 to 180 words, house voice, source linked and attributed, ending on an OIP point of view rather than a summary.
3. **Draft the LinkedIn variant**: shorter, no clickbait, with the blog permalink.

The relevance score is the part that actually saves you time. Raw Alerts volume is high and low-signal; a scoring gate is what turns 40 items a day into 3 worth reading.

One prompt-level rule worth hard-coding: **never reproduce more than a short quoted phrase from the source article.** See §5.

---

## 3. Two architectures

### Option A — Keep Squarespace, automate around it *(recommended)*

```
Google Alerts RSS  ─┐
Google News RSS    ─┴─→ Make.com ──→ dedupe ──→ Claude (score + draft)
                                                      │
                                          score >= 4  ▼
                                        Google Sheet review queue
                                          + morning digest email
                                                      │
                                          HUMAN (~10 min/day)
                                                      ▼
                                          Squarespace blog post
                                                      │
                                          blog?format=rss
                                                      ▼
                                     Buffer ──→ LinkedIn company page
                                                      │
                                                      └→ Slack/email nudge
                                                         to personal profiles
```

**Pros:** brand stays on one domain, all SEO equity on the main site, human gate is built in, cheap, standable this week.
**Cons:** the daily 10 minutes never goes away.

### Option B — Headless blog on a subdomain

Replace the Squarespace blog with Ghost (from ~$9/mo) or WordPress, both of which have real content APIs. Make writes the post as a **draft**, a human hits publish from their phone, and the rest is identical.

**Pros:** true end-to-end automation, publish becomes a one-tap approval rather than a copy-paste job, and you get a proper content API for anything you build later.
**Cons:** a second property to design and maintain, SEO split across two domains, and a migration of any existing posts. Realistically a 2 to 4 week project, not a this-week project.

**Recommendation:** Start with A. Revisit B only if you are publishing more than about 4 posts a week, at which point the manual paste starts to genuinely hurt. Below that cadence, B is not worth the fragmentation.

---

## 4. Cost and setup

### Monthly cost, Option A

| Item | Cost |
|---|---|
| Make.com Core (10,000 ops) | $9 |
| Anthropic API (Claude, ~50 drafts/mo) | $2 to $5 |
| Buffer (1 to 3 channels) | $0 (free tier) to $18 |
| Google Alerts / News RSS | $0 |
| Squarespace | already paid |
| **Total** | **~$12 to $32/mo** |

**On Make vs Zapier:** use Make. Zapier bills per *action step*, so a 6-step workflow burns 6 tasks per run and Zapier Professional starts around $20/mo for 750 tasks. Make bills per operation on a far cheaper curve ($9 for 10,000) and handles multi-step branching better. For this workload Make is roughly 3 to 4x cheaper. Zapier only wins if you need an integration Make lacks, which is not the case here.

### Setup order

1. **Build the keyword set** (60 min). Cluster into: transactions, competitors, partners, Telia/Telia Towers brand, technology, geography. Write the Google News RSS query strings.
2. **Create the feeds** (30 min). Google Alerts with delivery = RSS, plus the News RSS URLs. Drop all URLs in a sheet.
3. **Make scenario 1 "Radar"** (2 to 3 hrs). Watch RSS to Data Store dedupe by URL to Claude scoring/drafting call to filter `score >= 4` to append row to Google Sheet to daily digest email. Schedule 1x/day, 7am.
4. **Write the Claude prompt** (60 to 90 min). This is the piece that determines whether the output is usable. Include an OIP positioning blurb, 2 to 3 example posts in house voice, the relevance rubric, and the quoting rule. Budget a week of tuning on live output.
5. **Squarespace RSS check** (10 min). Confirm `yoursite.com/blog?format=rss` resolves and carries what you expect.
6. **Buffer to LinkedIn** (30 min). Connect the company page, add the Squarespace feed, set the queue. **Turn on Buffer's approval/draft mode for the first two weeks** rather than direct publish.
7. **Personal profile lane** (30 min). Make posts the drafted variant to a Slack channel or email so individuals can post in their own words.
8. **Review after 2 weeks.** Tune the relevance threshold, then decide whether to drop the Buffer approval step.

Total build: roughly one focused day, plus prompt tuning.

---

## 5. Compliance and brand-safety risks

These are real and worth reading before you switch anything to full auto.

**Copyright.** LinkedIn scans company pages more aggressively than personal accounts and removes flagged content without warning. Sharing news with a link and attribution plus your own commentary is fine. Reproducing article text is not. Hard rules for the prompt: no more than a short quoted phrase, always attribute and link the source, never lift the publisher's images.

**LinkedIn ToS.** LinkedIn's terms are aimed at automation that *mimics human action*. Scheduling curated content through an approved partner (Buffer, Hootsuite, Make) is explicitly fine. Fully unattended posting with no human in the loop sits closer to the line, and posting identical text simultaneously across several personal profiles sits on the wrong side of it. This is the main reason to keep personal profiles as **draft-and-nudge, not auto-post.** It also happens to perform better; identical cross-posted text is transparent to readers and gets suppressed.

**The commercial risk nobody flags.** You are an infrastructure investor. Auto-publishing commentary on live transactions can read as market signalling, imply a position you do not hold, or comment on a counterparty mid-negotiation. An AI drafting from a headline has no idea which names are sensitive. **Mitigation: maintain a blocklist of entities that never auto-draft**, covering active targets, counterparties, and anything under NDA, and enforce it as a hard filter in Make *before* the Claude step. This is the single most important control in the design.

**Accuracy.** AI drafting from a headline and snippet will occasionally invent detail. The human gate catches this, which is another argument against removing it.

**Corrections.** Agree in advance who can pull a LinkedIn post and how fast. A wrong post about a named counterparty needs a response measured in minutes.

---

## 6. Minimum viable version, standable this week

Strip out Make entirely for v0. You can have the value in about two hours:

1. **Google Alerts, delivery = email**, into a dedicated Gmail label or folder. (15 min)
2. **Add 4 to 6 Google News RSS queries** to any reader. (20 min)
3. **Once a day, paste the interesting headlines into Claude** with a saved prompt containing your positioning blurb and voice examples. Get back a blog draft and a LinkedIn variant. (10 min/day)
4. **Paste into Squarespace, publish.** (3 min)
5. **Buffer free tier**, connect the LinkedIn company page, add `yoursite.com/blog?format=rss`. This one step is genuinely automatic from day one. (20 min)

**Cost: $0.** That gets hops 1 and 3 working immediately and proves whether the content is any good before you spend a day building Make scenarios.

The honest sequencing argument: the automation is the easy part. Whether AI-drafted commentary on tower and data center news actually sounds like OIP is the open question, and the MVP answers it for free. Build the Make layer in week 2, once the prompt is earning its keep.

---

## Summary of what can and cannot be automated

| Fully automatic | Human required |
|---|---|
| Monitoring keywords | Publishing to Squarespace (no API) |
| Filtering and deduping | Approving draft accuracy and tone |
| Relevance scoring | Sensitive-entity judgement calls |
| Drafting blog + LinkedIn copy | Personal profile posts (by choice) |
| Squarespace to LinkedIn company page | |
| Scheduling and queueing | |

---

## Sources

- [Squarespace Developer Platform](https://developers.squarespace.com/) — commerce APIs only, no content API
- [Squarespace: Using RSS feeds](https://support.squarespace.com/hc/en-us/articles/215761717-Using-RSS-feeds) — `?format=rss`
- [Squarespace Forum: programmatic blog publishing](https://forum.squarespace.com/topic/290491-how-to-programatically-publish-new-posts-on-square-space-blog-any-restful-api-for-blog-creation/)
- [Squarespace Forum: API for creating/updating blog posts](https://forum.squarespace.com/topic/247971-do-squarespace-provide-and-api-for-creating-and-updating-blog-post/)
- [Adding form integrations with Zapier (Squarespace)](https://support.squarespace.com/hc/en-us/articles/360000641507-Adding-form-integrations-with-Zapier) — form trigger only
- [LinkedIn Community Management API](https://developer.linkedin.com/product-catalog/marketing/community-management-api)
- [Community Management API overview (Microsoft Learn)](https://learn.microsoft.com/en-us/linkedin/marketing/community-management/community-management-overview?view=li-lms-2026-04)
- [LinkedIn User Agreement](https://www.linkedin.com/legal/user-agreement)
- [The Legal Guide to LinkedIn Company Pages](https://aeroleads.com/blog/legal-linkedin-company-pages-whats-allowed-whats-not/)
- [Google News RSS feed URLs](https://www.wprssaggregator.com/google-news-rss-feed/)
- [Make: LinkedIn and RSS integration](https://www.make.com/en/integrations/linkedin/rss)
- [Zapier Pricing 2026](https://www.nocode.mba/articles/zapier-pricing-2026)
- [Auto-posting Squarespace blogs to social media](https://createwithsquarespace.com/how-to-automatically-post-your-squarespace-blogs-to-social-media/)
