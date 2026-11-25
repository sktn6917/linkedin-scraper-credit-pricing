# LinkedIn Scraper API: What 30 Credits per Request Buys, Which Plan Fits, and Which Profile Fields Are Still Live

ScraperAPI bills a LinkedIn page at **30 credits**, thirty times the cost of an ordinary page, and the three fields most buyers actually want have been coming back filled roughly **2% of the time** since 13 November 2025. Those two numbers decide most of the purchase. Below is the credit math, the plan ladder and its overflow rules, the current state of LinkedIn's public fields, and the operational details that determine whether a batch finishes or quietly loses rows.

## Quick pick

> **Run the trial, then buy Scaling if LinkedIn is a recurring target.** The 7-day trial includes 5,000 credits with no card, which at 30 credits per LinkedIn page is about 166 pages, or 125 if bot-protection bypass fires and adds 10 credits. For ongoing volume, Scaling is the lowest plan that gives you both auto-upgrade and pay-as-you-go, so a heavy month costs extra credits instead of forcing a plan change.

👉 [See current ScraperAPI plans, credits and concurrency](https://scraperapi.com/pricing?fp_ref=coupons)

## All plans, and the one setting that separates them

ScraperAPI lists seven plans: Hobby, Startup, Business, Scaling, Professional, Advanced and Enterprise. Credit allowances and dollar prices move with billing cycle and promotions, so confirm the figure on the pricing page at checkout. What doesn't move is how each plan behaves when your credits run out, and that is where the ladder actually splits.

| Plan | Auto-upgrade when credits run out | Pay-as-you-go overflow | Price and credit allowance |
| --- | --- | --- | --- |
| Hobby | Yes | No | [ Current Hobby pricing](https://scraperapi.com/pricing?fp_ref=coupons) |
| Startup | Yes | No | [ Current Startup pricing](https://scraperapi.com/pricing?fp_ref=coupons) |
| Business | Yes | No | [ Current Business pricing](https://scraperapi.com/pricing?fp_ref=coupons) |
| Scaling | Yes | Yes | [ Current Scaling pricing](https://scraperapi.com/pricing?fp_ref=coupons) |
| Professional | Yes | Yes | [ Current Professional pricing](https://scraperapi.com/pricing?fp_ref=coupons) |
| Advanced | No | Yes | [ Current Advanced pricing](https://scraperapi.com/pricing?fp_ref=coupons) |
| Enterprise | No | Yes, plus volume terms | [ Current Enterprise pricing](https://scraperapi.com/pricing?fp_ref=coupons) |

Auto-upgrade moves the account to the next plan mid-cycle, which keeps requests flowing but changes what you pay. Pay-as-you-go keeps you on the same plan at a fixed per-credit rate up to a monthly cap you set yourself. On the published allowance tiers, ScraperAPI goes from 5,000,000 credits with 200 concurrent threads to 21,500,000 credits with 500 threads, and custom volume above 22,000,000 credits. At 30 credits per LinkedIn page, 5,000,000 credits is roughly 166,600 pages a month.

There is also a free plan separate from the trial: 1,000 credits and a maximum of 5 concurrent connections, which is about 33 LinkedIn pages. Useful for a smoke test, not for a pipeline.

Concurrency limits behave differently from credit limits. Go over your thread count and you get a 429 telling you to come back within the plan's limit. No surcharge is applied, and the request is rejected rather than queued, so your code has to retry rather than wait.

## What LinkedIn still serves, and what it stopped serving

This is the part that changes budgets. Since 13 November 2025, LinkedIn has restricted public access to the **Position**, **Experience** and **Education** fields. Fill rates for those three fell from 50–64% to roughly 2%, and they now return sticky data from cache instead of a fresh scrape. Everything else still refreshes normally, with one exception: `current_company.industry` has been unsupported since 7 August 2025. The field-status tables published by vendors that track this, including Bright Data's LinkedIn documentation updated 21 October 2026, state the same figures.

Read that as a page-level change on LinkedIn's side, because that's what it is. If the public page no longer carries a reliable work-history block, then a fetcher reading that page doesn't get one either, whether it returns HTML, JSON or a CSV. Any vendor promising complete experience and education data at scale after November 2025 is worth testing on ten known profiles before you commit to volume. The fields that still refresh are the ones to build on: name, location, current company name, follower count, connections, headline and about text.

That reframes what you're buying. A LinkedIn data budget spent on career history is largely spent on cached values with a 2% fill rate. The same budget spent on identity, location, current employer and audience size buys live data.

## The credit math behind a LinkedIn request

ScraperAPI's credit system prices by target difficulty. A standard page costs 1 credit. Amazon costs 5. Google and Bing, including subdomains, cost 25. LinkedIn costs 30. Sites behind Cloudflare, Datadome or PerimeterX add 10 credits per request when ScraperAPI bypasses the protection, which puts a hard LinkedIn page at up to 40 credits. LinkedIn pages routinely sit behind that kind of protection, so budget the higher number until your own logs say otherwise.

Two controls keep this from running away. The Domain Cost Estimator in the dashboard quotes the exact cost for a specific URL before you send traffic at it, and the `max_cost` parameter caps what any single request is allowed to spend. Set both before a bulk run.

A worked example with the numbers that are actually published: 10,000 LinkedIn pages at 30 credits is 300,000 credits; at 40 credits it's 400,000. Whether that fits inside a plan depends on the allowance you buy, and the current per-plan figures are on the pricing page rather than in this article, because quoting a number that changes with promotions would be worse than pointing you at the source.

👉 [Check the credit allowance on each ScraperAPI plan](https://scraperapi.com/pricing?fp_ref=coupons)

## You get the page back, not the fields

ScraperAPI's documented LinkedIn workflow is a proxy layer. You send a LinkedIn URL through the API endpoint, it returns the page with rotating IPs and headers, and your code parses it. ScraperAPI's own LinkedIn tutorial walks through fetching public job-listing pages and pulling titles, companies, locations and links out of the HTML with BeautifulSoup.

That's a different product shape from vendors selling a field-level LinkedIn endpoint. Nothing is wrong with it, but it means the engineering cost sits on your side. Budget for the parser, for selector maintenance when LinkedIn changes markup, and for handling the guest endpoints that paginate in steps of 25 (their tutorial notes the last page sits at `start=975` and 1000 returns a 404). If your team wants named JSON keys with no parsing work, that's a different purchase.

## Batch runs, timeouts and rows that vanish

Synchronous scraping has a time limit, and the industry pattern for exceeding it is consistent: a request that can't finish in about a minute returns HTTP `202` with a `snapshot_id`, and you collect the result through an async endpoint or a webhook later. Bright Data documents exactly this shape, with one input recommended per synchronous call and unlimited URLs up to 1 GB of input per async job. Treat any long-running LinkedIn batch as async by default. A discovery job that searches for profiles by criteria will almost never finish inside a synchronous window.

Three implementation traps show up repeatedly in vendor SDK docs and cost people real data.

**Unit mismatches.** Bright Data's Node SDK takes `pollTimeout` in milliseconds, and copying the number straight from a Python example expires the poll before the first status check. The worked values there are `pollTimeout: 600_000` and `pollInterval: 5_000`. Their Python side has a second, separate limit: the call-level `timeout=180` doesn't override a client-level download timeout of 30 seconds.

**Silent row loss.** Without `includeErrors: true`, the API drops the failing row instead of telling you why. A batch of 500 URLs that returns 480 records looks like a coverage gap until you turn error output on.

**Truncated summaries.** The `about` field is described in the same schema as "a concise profile summary", with the caveat that LinkedIn sometimes displays only a truncated version ending in "…" and that is the version captured. Long-form profile text is not guaranteed to arrive complete.

Real error strings worth matching on: `The profile is hidden or private.` for URLs that aren't publicly visible, and a rate-limit message along the lines of "your system is sending too many of this type of request", which clears on its own after a wait.

## How the other two vendors price the same job

Worth knowing because the billing model changes the risk profile. No links here, just published terms.

|  | Bright Data | Apify |
| --- | --- | --- |
| Billing model | Per successful record | Actor runs, with a free trial and no card required |
| Free allowance | 5,000 credits per month on new accounts, no card | Free trial; all plans include Apify Proxy |
| Structured output | Five dataset IDs covering profiles, companies, jobs, posts and profile discovery | Named JSON fields per profile |
| Batching | Sync returns in up to 1 minute, then `202` + `snapshot_id`; async accepts unlimited URLs up to 1 GB of input | Actor runs via Python, JavaScript, CLI, cURL, OpenAPI or MCP |
| Export and delivery | NDJSON, JSON, JSONL, CSV, Parquet; webhook, S3, Snowflake, Azure, GCS | Excel, CSV, JSON, XML, HTML table, JSONL, RSS; Zapier, Make, Keboola, Google Drive |

Per-successful-record billing means failed or empty pages don't consume budget, which matters a lot when three profile fields are failing at a 98% rate. A record count of 10,000 then means 10,000 records.

Apify's sample output is the clearest illustration of the field problem. Bill Gates' record carries `followers: 38914463` and `companyName: "Gates Foundation"`, and also `experiences: []` and `educations: []`. The second example has `about: null`, `email: null`, `profilePic: null`, `skills: []`. The page carries no publication date and the image links inside it expire around late 2025, so the sample probably predates the November 2025 restriction. Either way, it shows that a schema can list forty fields while the response fills a fraction of them.

## Five things to confirm before you pay

1. **What one LinkedIn page costs you.** Run a handful of your real target URLs through the Domain Cost Estimator, not a homepage. The 30-credit base is published; the bypass surcharge depends on the page.
2. **Whether experience and education come back populated.** Test ten profiles whose work history you can verify by hand. If you get empty arrays, no plan tier fixes it.
3. **How overflow is handled on the plan you're buying.** Below Scaling, running out of credits triggers an upgrade instead of pay-as-you-go.
4. **Whether failed rows are visible.** Ask for error output in the response. Silent drops are the most expensive kind of bug.
5. **What happens on a 429.** ScraperAPI rejects the request rather than queueing it, so retry logic belongs in your client from day one.

## Which plan to buy

If LinkedIn profiles are a side task and you need identity, location, current company and follower counts, Hobby or Startup plus the free plan for testing covers it, with auto-upgrade protecting you from a mid-month stop. If LinkedIn is a standing data source and you can't afford to pause at the end of a billing cycle, Scaling is the entry point that gives you per-credit overflow without changing tiers. Teams moving more than a few hundred thousand pages a month should be talking to Enterprise, where volume terms apply.

Before you enter card details, confirm two things: the current price for the billing cycle you're choosing, since annual plans can also renew early for a fresh batch of credits, and the credit cost of your top three target URLs from the estimator. Those two figures plus the field test are the whole decision.

👉 [Compare ScraperAPI plans and start the 7-day, 5,000-credit trial](https://scraperapi.com/pricing?fp_ref=coupons)
