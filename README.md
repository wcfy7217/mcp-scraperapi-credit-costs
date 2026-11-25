# MCP Server Web Scraping: Hosted vs Self-Hosted, What a Scrape Really Costs in Credits, and Which ScraperAPI Plan Covers Your Volume (Full Plan Table)

Most MCP scraping servers fall into two groups, and the split matters more than any feature list. One group is open-source servers you run on your own machine for nothing. The other is metered servers where the bill depends on switches you set per request. ScraperAPI is in the second group, and it publishes the per-request prices instead of leaving them inside a vague "credit" fog: a standard page costs 1 credit, `render=true` costs 10, `premium=true` costs 10, and premium plus rendering is 25. On the Startup plan, 1,000,000 credits for $149/month, that is either a million plain pages or 40,000 pages rendered through residential IPs.

What follows is how the ScraperAPI MCP server plugs into Claude, the full plan table, the credit arithmetic that decides whether it is expensive for your workload, and the cases where a free self-hosted server is the smarter pick.

> If your agent mostly reads documentation, articles and server-rendered pages, the smallest ScraperAPI tier that covers your monthly page count is enough, because a plain scrape stays at 1 credit as long as you leave `render`, `premium`, `screenshot` and `ultra_premium` off. Volume starts at 1,000,000 credits for $149/month, or $134.10/month on annual billing. 👉 [Compare the current ScraperAPI plans and prices](https://www.scraperapi.com/pricing?fp_ref=coupons)

## Every ScraperAPI plan currently on the pricing page

| Plan | API credits / month | Concurrent threads | Geotargeting | Price and purchase link |
| --- | --- | --- | --- | --- |
| Free | 1,000 credits on signup | 5 | — | $0 ·  [Create the free account](https://www.scraperapi.com/?fp_ref=coupons) |
| Hobby | 100,000 | 20 | US & EU | [Current Hobby price](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Startup | 1,000,000 | 50 | US & EU | $149/mo, or $134.10/mo billed annually ·  [Get Startup](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Business | 3,000,000 | 100 | Country-level | [Current Business price](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Scaling | 5,000,000 | 200 | Country-level | [Current Scaling price](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Professional | 10,500,000 | 300 | Country-level | [Current Professional price](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Advanced | 21,500,000 | 500 | Country-level | [Current Advanced price](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Enterprise | 22,000,000+ | 500+ | Country-level | Custom pricing ·  [Talk to sales](https://www.scraperapi.com/pricing?fp_ref=coupons) |

Startup is the tier whose list price is spelled out on the page, so it is the one you can do arithmetic on. Annual billing there costs $1,609.20 a year against $1,788 at the monthly rate, a $178.80 difference. Credits top out at 22,000,000+ on Enterprise, which is roughly 22 times the Startup allowance, and custom volume discounts kick in at that end.

Two differences inside the table decide more upgrades than the credit counts do. Geotargeting is US & EU on Hobby and Startup, then country-level from Business upward, so a German or Japanese proxy target forces the jump regardless of page volume. Analytics history follows the same line, 30 days on the two small tiers and unlimited above them. Concurrency climbs from 20 threads on Hobby to 500 on Advanced, which only binds when you fan a crawler across many URLs at once. A single Claude or Cursor session calling `scrape` in a loop rarely touches 20.

## What the ScraperAPI MCP server actually is

It is a local MCP server, installed with Python or Docker, that passes the parameters from your prompt to ScraperAPI's scraping API and drops the response back into the conversation. The proxy pool, the browser and the anti-bot handling stay on ScraperAPI's side. What sits on your machine is a config file and a runtime.

Setup is the standard Claude Desktop path: Settings, Developer, Edit Config, paste the JSON block from the docs into `claude_desktop_config.json`, quit and reopen. The server exposes one tool, `scrape`, and the documented trigger is the word "scrape" at the start of a prompt. That tool takes `url` (required), plus `render`, `country_code` as an ISO-2 code, `premium`, `ultra_premium`, `device_type` set to `mobile` or `desktop`, and `autoparse`. Auto parsing defaults to false and only matters when you want CSV or JSON back from the domains on ScraperAPI's supported list.

The docs' own prompt template is worth reading as a cost map. It tells the model to retry with a `country_code` after a 500 or a geo-block, then escalate to `premium=true`, then to `ultra_premium=true`, and to re-run with `render=true` when the fields come back missing or incomplete. Each rung of that ladder has a price attached in the next section, and `premium` and `ultra_premium` cannot be combined on one request.

One structural point if you are comparing this against hosted MCP servers: ScraperAPI's MCP server is local, so your API key lives in a JSON file on each machine. Every teammate and every dev box needs its own copy of the config. There is no shared remote endpoint to hand around.

## The credit table is the real price list

"1 credit per page" is true for plain static HTML and stops being true the moment you turn something on.

| Request type | Credits per request |
| --- | --- |
| Standard page | 1 |
| E-commerce (Amazon, Walmart, eBay) | 5 |
| SERP (Google, Bing, subdomains included) | 25 |
| Social (LinkedIn) | 30 |
| Bot-protection bypass (Cloudflare, Turnstile, DataDome, PerimeterX/HUMAN, Akamai) | +10 |
| JS rendering (`render=true`) | 10 |
| Premium proxies (`premium=true`) | 10 |
| Screenshot (`screenshot=true`) | 10 |
| Premium proxies + JS rendering on one request | 25 |
| Ultra premium (`ultra_premium=true`, paid plans only) | 30 |
| Ultra premium + JS rendering | 75 |
| Geotargeting, sessions, device type, output format, auto parsing | Free |

Rendering is the biggest lever you control. A documentation page costs 1 credit; the same page with `render=true` costs 10. Premium proxies add another 10 on their own, and a screenshot another 10. Notice that premium plus rendering is listed at 25 rather than 20, so resist the urge to add the rows together. The Domain Cost Estimator in the dashboard gives the real number for any URL, every response carries its own cost in the `sa-credit-cost` header, and `max_cost` caps a single request before it goes out. Those three tools are how you avoid discovering a 25-credit surprise after a 3,000-page crawl.

The bypass line is the one people miss. When ScraperAPI has to get past Cloudflare, DataDome, PerimeterX or Akamai, that adds 10 credits, and it happens because the target site is protected, not because you asked for it. A site you budgeted at 1 credit can quietly bill at 11.

Everything else in that table is free or nearly so. Rotating the country, holding a session, switching to a mobile user agent, changing the output format and turning on auto parsing cost nothing extra. If a scrape returns the wrong currency or a US-only price, reach for `country_code` first. It is the cheapest fix on the list.

Run the arithmetic on Startup's 1,000,000 credits and the spread becomes concrete: 1,000,000 standard pages, 100,000 rendered pages, 40,000 pages with premium proxies and rendering, or 13,333 requests at the ultra premium plus rendering rate of 75.

## Self-hosted, hosted, or metered: three shapes of the same job

|  | webscrape_mcp (open source) | HasData Web Scraping MCP | ScraperAPI MCP |
| --- | --- | --- | --- |
| Where it runs | Your machine, Python 3.10+ | Hosted remote endpoint | Local server, hosted scraping API |
| Cost | Free, MIT | 1 credit plain fetch, 10 with rendering on (its default); residential 5 and 15 | 1 credit standard, 10 with `render=true` |
| Entry price | $0 | $59/mo for 200,000 credits, 1,000 free credits monthly | Free tier 1,000 credits, Startup $149/mo for 1,000,000 |
| Browser | You run `playwright install chromium` | Handled server-side | Handled server-side |
| Batch limits | 20 URLs per call, crawl to 100 pages / depth 5, 25,000-character response cap | Single URL per call | Crawl and batch handled through the API, not the MCP tool |
| Structured extraction | None | CSS rules and LLM-schema extraction | Auto parsing and structured data endpoints on supported domains |

The free server is genuinely capable: six tools covering single scrape, batch of up to 20 URLs, recursive crawl, link extraction, Playwright rendering and screenshots, all MIT licensed. Its costs are operational. You install Chromium, you keep the process alive, you handle the site that blocks your IP, and every response gets cut at 25,000 characters. It also has no AI extraction and no web search, which its own comparison table against Firecrawl admits.

HasData's hosted server inverts the trade. Nothing to host, one tool, proxies and rendering as parameters, and a default that matters: rendering is on unless you turn it off, so an unconfigured call costs 10 credits rather than 1. Its paid plans start at $59/month for 200,000 credits, and the free tier gives 1,000 credits a month with a single concurrent request. It also bills 404s and 403s as successful calls, so check the status before you parse.

Firecrawl's MCP server needs an API key and covers search, scrape, interact, parse, crawl, map, Agent and Monitor. Bright Data's bundles web search, page scraping, structured extraction and browser automation. Both are fuller platforms than a single scrape tool, and both are priced accordingly.

## Where budgets leak in practice

The bypass surcharge is automatic and invisible until the invoice: protected sites cost 10 extra credits whether or not you chose it.

Rendering gets left on. A prompt template that says "if data is missing, set render=true" turns into a permanent `render=true` in a saved config, and every static page after that bills at 10×.

Combos are not sums. Premium plus rendering is 25, and ultra premium plus rendering is 75, which is why the estimator beats mental arithmetic.

Hitting your concurrency ceiling gets you a 429, and the request is turned away rather than queued. There is no overage charge, but there is also no retry: an unattended agent crawling a list will hit the wall and lose the tail of the list unless you build backoff.

When credits run out, scraping pauses until they renew by default. Auto-upgrade, which moves you to the next plan automatically, is opt-in and available on Hobby, Startup, Business, Scaling and Professional. Pay-as-you-go at a fixed per-credit rate with a monthly cap is available on Scaling, Professional, Advanced and Enterprise. Annual plans can also renew early for a fresh batch of credits. If a mid-month stall would hurt, one of those two switches is the fix. 👉 [See which plans include auto-upgrade and pay-as-you-go](https://www.scraperapi.com/pricing?fp_ref=coupons)

## Picking a tier by page count, not by tier name

Start from credits per month, not pages. Divide your page count by the credit cost of the request type you actually send.

Static pages, docs, articles, server-rendered HTML at 1 credit each: 100,000 pages a month fits Hobby's 100,000 credits with nothing to spare, and 1,000,000 fits Startup.

Rendered pages at 10 credits each: 20,000 rendered pages consume 200,000 credits, so Hobby covers it and Startup leaves room for retries. At 80,000 rendered pages you need 800,000 credits, which is Startup territory.

Premium plus rendering at 25 credits: 40,000 pages exhausts the whole Startup allowance.

Country-level geotargeting: if you need proxies outside the US and EU, Hobby and Startup will not do it, and Business is the first tier that will.

Concurrency: 20 threads on Hobby is plenty for one assistant session. A crawler running 50 parallel fetches needs Startup, and a data team running scheduled jobs across sources is looking at Business or above.

👉 [Check what your monthly credit count costs on each ScraperAPI tier](https://www.scraperapi.com/pricing?fp_ref=coupons)

## Who should not buy a metered MCP server

If your agent reads a handful of static pages a week, the free open-source server or a plain `fetch` in your own code is the correct answer, and paying for credits is waste.

If you already run a browser farm and a proxy pool, you are buying convenience rather than capability, and the math only works if your ops time costs more than the subscription.

If you need typed fields from a specific site rather than a page of HTML, check whether that domain is on ScraperAPI's auto-parsing and structured data list before you buy. When it is, you get parsed fields; when it is not, you get a page and the parsing is your problem, same as with any general-purpose scraping MCP server.

## Two things to settle before you pay

Will your targets need rendering? If yes, budget 10 to 25 credits per request, not 1, and pick the tier from that number. A JS-heavy site can turn a plan that looked generous into one that runs dry in a week.

Do your targets sit behind bot protection? Add 10 credits per bypassed request. A list of Cloudflare-fronted sites is effectively 11 credits a page at minimum.

You can test both answers before committing: the free plan gives 1,000 credits with 5 concurrent connections, and the pricing page advertises a 7-day trial with 5,000 API credits and no credit card required. Run your real URLs through the Domain Cost Estimator during the trial, read the `sa-credit-cost` header on each response, and you will know your actual monthly bill before the first invoice. 👉 [Start the 7-day, 5,000-credit trial](https://www.scraperapi.com/?fp_ref=coupons)
