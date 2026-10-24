# Puppeteer Web Scraping: How to Keep Your Headless Browser Unblocked, Plus Which ScraperAPI Plan Actually Fits Your Crawl

Puppeteer is one of the best tools for scraping JavaScript-heavy pages — it drives a real Chrome instance, waits for dynamic content, and lets you pull data out of the rendered DOM. The part most tutorials gloss over is what happens next: your scraper works perfectly on day one, then the target site flags your IP, serves a CAPTCHA, and the whole thing falls apart. That's the problem a scraping API like ScraperAPI exists to solve, and it plugs into Puppeteer through a proxy port in about three lines of code.

This guide covers both sides: how to wire ScraperAPI into a Puppeteer project correctly (including the integration mistake the official docs warn about), what each request actually costs in credits, and how to pick a plan without overpaying.

> **Quick picks**
> - Just testing? The 7-day trial gives you 5,000 API credits with no credit card required. Start there before paying anything.
> - Solo dev or small team scraping standard e-commerce/SERP targets? **Hobby at $49/month** (100,000 credits, 20 concurrent threads) is the realistic starting point.
> - 👉 [Start the 7-day free trial and see current plan pricing](https://www.scraperapi.com/pricing?fp_ref=coupons)

## Why plain Puppeteer gets you blocked

Puppeteer itself is free and excellent at rendering JavaScript. The cost isn't the tool — it's the traffic pattern. A headless browser hitting the same site hundreds of times from one IP looks nothing like a human, and modern bot management (Cloudflare, Datadome, PerimeterX, Akamai) is built specifically to catch it.

You have two ways to deal with this:

1. **Build it yourself**: a rotating proxy pool, retry logic, CAPTCHA handling, session management. Doable, but you're now maintaining infrastructure instead of collecting data.
2. **Route Puppeteer's traffic through a scraping API** like ScraperAPI, which handles proxy rotation (a pool of 40M+ IPs across residential, datacenter, and mobile networks in 50+ countries), ban detection, CAPTCHAs, and retries for you.

ScraperAPI claims roughly a 98% average success rate with a 1–3% expected failure rate, and failed requests aren't billed. For anyone scraping a few tens of thousands of pages a month, option 2 usually beats maintaining your own proxy stack.

## The right way to integrate ScraperAPI with Puppeteer

Use the proxy port. That's the short answer, and it comes from ScraperAPI's own integration guide.

Here's the mistake they explicitly warn against: pointing Puppeteer at the API endpoint directly. The guide walks through two problems with that approach. First, when Puppeteer loads a page, it requests all the sub-resources — CSS, scripts, images — using relative URLs. Request `quotes.toscrape.com` through the API endpoint and the relative path `/static/main.css` gets sent to `api.scraperapi.com/static/main.css`, which errors out. As the guide puts it, "Puppeteer will only get access to the first HTML response (which kinda defeats the purpose of using a headless browser)." Second, clicking relative links for navigation breaks the same way.

The proxy port treats ScraperAPI like any ordinary HTTP proxy, so Puppeteer keeps doing everything it does natively:

1. **Set the proxy at launch:**

javascript
const browser = await puppeteer.launch({
  headless: true,
  args: ['--proxy-server=http://proxy-server.scraperapi.com:8001'],
  ignoreHTTPSErrors: true
});


2. **Authenticate the page:**

javascript
const page = await browser.newPage();
await page.authenticate({
  username: 'scraperapi',
  password: 'YOUR_API_KEY'
});


3. **Scrape normally.** Every request now exits through ScraperAPI's rotating pool, and the target site never sees your real IP.

Full runnable examples live in ScraperAPI's public GitHub repo (`scraperapi-code-examples`), including a NodeJS Puppeteer scraper that pulls `quotes.toscrape.com` through the proxy port.

One caveat worth knowing before you build on this: the docs note that conflicts occasionally occur between Puppeteer and the proxy port — usually showing up as certain assets failing to load. Their fix is contacting support, which works, but it's not the kind of thing you want to debug at 2 a.m. on a production job. On Hobby and Startup plans, support is email/ticket only; live chat and phone support come with higher tiers.

If you've never used Puppeteer at all, the pattern is the same either way: `npm install puppeteer` (expect a 170–280MB download since it ships a compatible Chrome), launch a browser, `page.goto()` your URL with `waitUntil: 'networkidle0'` so dynamic content finishes loading, then `page.waitForSelector()` and `page.evaluate()` to extract the data — often with Cheerio for the parsing side.

## What a page actually costs: the credit math nobody spells out

Here's the part most reviews skip, and it's the difference between a $49 bill and a $200 surprise. ScraperAPI prices everything in API credits, and the headline "100,000 credits" on a plan is not the same as "100,000 pages."

The base rates, from the official pricing page: a standard page costs **1 credit**. But the moment you touch real-world targets, the numbers move:

| Request type | Credits per request |
| --- | --- |
| Standard page | 1 |
| Amazon product/search page | 5 |
| Google or Bing (incl. subdomains) | 25 |
| LinkedIn | 30 |
| Site behind Cloudflare/Datadome/PerimeterX (when bypassed) | +10 |
| JS rendering (`render=true`) | 10 |
| Premium proxy + JS rendering | 25 |
| Ultra premium proxy + JS rendering | 75 |

Two tools keep this under control, and both are free to use: the **Domain Cost Estimator** in your dashboard shows the exact cost for any URL before you burn credits, and the `max_cost` parameter caps what any single scrape request can spend.

To make it concrete: one third-party reviewer (who's used the service since 2023 for price monitoring) worked through the math on a Startup plan — a million credits at $149/month looks like $0.000149 per credit on paper. Scrape Amazon search results with JS rendering enabled and those million credits cover maybe 40,000–100,000 actual pages. Their advice is worth repeating: budget against your real per-page cost, not the credit count on the pricing page.

The good news for typical Puppeteer jobs: if your targets aren't behind heavy bot protection, the default pool clears most requests at 1 credit each. Check your target with the estimator before committing to a plan size.

## ScraperAPI plans: the full comparison

Every plan includes the full feature set — JS rendering, premium proxies, CAPTCHA handling, JSON auto-parsing, custom headers, automatic retries, unlimited bandwidth. What changes between tiers is credits, concurrency, and geotargeting scope. All paid plans also come with a 7-day no-questions-asked refund, so the trial-to-paid transition is low risk.

| Plan | Monthly price | Annual (per month) | Credits | Concurrency | Geotargeting |
| --- | --- | --- | --- | --- | --- |
| Free | $0 | — | 1,000/mo | 5 connections | — |
| [Hobby](https://www.scraperapi.com/pricing?fp_ref=coupons) | $49 | $44.10 | 100,000 | 20 threads | US & EU only |
| [Startup](https://www.scraperapi.com/pricing?fp_ref=coupons) | $149 | $134.10 | 1,000,000 | 50 threads | US & EU only |
| [Business](https://www.scraperapi.com/pricing?fp_ref=coupons) | $299 | $269.10 | 3,000,000 | 100 threads | Country-level |
| [Scaling](https://www.scraperapi.com/pricing?fp_ref=coupons) | $475 | $427.50 | 5,000,000 | 200 threads | Country-level |
| [Professional](https://www.scraperapi.com/pricing?fp_ref=coupons) | $975 | $877.50 | 10,500,000 | 300 threads | Country-level |
| [Advanced](https://www.scraperapi.com/pricing?fp_ref=coupons) | $1,975 | $1,777.50 | 21,500,000 | 500 threads | Country-level |
| Enterprise | Custom | Custom | 22,000,000+ | 500+ | Country-level |

A couple of things the table can't show:

- **Hobby and Startup are locked to US & EU geotargeting.** If your crawl needs `country_code` targeting beyond those regions, that feature starts at Business ($299/month). This is the single most common reason to jump two tiers.
- **Overages don't silently bill you.** Exceed your concurrency limit and the API returns a 429 telling you to throttle back — the request is refused, not queued, and there's no extra charge.
- **Scaling, Professional, Advanced, and Enterprise include Pay-As-You-Go**, so hitting your credit cap mid-cycle lets you continue at a fixed per-credit rate up to a monthly spending cap you set, without changing plans. On monthly Hobby/Startup/Business plans you can opt into Auto-Upgrade instead, which moves you to the next tier (usually at a better price per credit).
- Docs also list a Student plan in the billing structure; it doesn't appear on the main pricing grid, so ask support if that's relevant to you.
- 👉 [Check the current pricing page and trial terms](https://www.scraperapi.com/pricing?fp_ref=coupons)

## Where the free plan fits

The free tier is genuinely usable for a Puppeteer proof of concept: 1,000 credits per month with 5 concurrent connections, plus 5,000 free requests during your first 7 days for larger-scale testing. At 1 credit per standard page, that's a thousand pages a month at zero cost — enough to validate your selectors and finish the integration before spending anything.

What it won't do: anything involving Amazon (5 credits/page means ~200 pages), Google (25), or JS rendering at any volume. It's a development sandbox, not a production tier.

## The trade-offs you accept with a scraping API

Honest version, based on what's verifiable:

**Session length is short.** Session support (pinning requests to the same IP via `session_number`) is designed for single-page crawls, with sessions lasting around an hour per third-party testing. If your job needs long-lived sticky sessions — account-level scraping, ad verification — this isn't the right tool for that part.

**You can't pick individual IPs.** You control parameters (`country_code`, `premium`, `ultra_premium`, `session_number`), not specific IPs or ASNs. There's no subnet-level control panel.

**The `render=true` shortcut doesn't replace Puppeteer.** Adding `render=true` renders the page in Chromium and returns HTML — useful when you just need markup. But the API can't click buttons, fill forms, or simulate user behavior the way Puppeteer can. That's exactly why the proxy port integration matters: you keep Puppeteer's full interactivity and only borrow ScraperAPI's IP pool. Note that by default the API endpoint doesn't accept Puppeteer/Selenium requests at all (Enterprise users can request an exception) — the proxy port is the standard route.

**JS rendering has its own throttle**: 3 render requests per second, independent of your plan's thread count. Enterprise can raise it.

None of these kill the tool for its target use case — price monitoring, SERP tracking, lead-gen against standard e-commerce and search sites at 20K–200K pages/month. They just define where it stops fitting.

## FAQ

**Do unused credits roll over?**
No. Credits reset when your subscription renews; whatever you didn't use is gone. You can set automatic renewal at 95% or 100% usage in the dashboard, or renew manually.

**Can I cancel anytime?**
Yes — cancel from the dashboard or via support, with no cancellation fee.

**What if it doesn't work out?**
Official policy: "We offer a 7-day no questions asked refund policy." Contact support and they process the refund. Combined with the no-credit-card trial, you can evaluate the service without meaningful financial risk.

**What happens when I run out of credits mid-month?**
Depends on your plan: Auto-Upgrade (opt-in, Hobby through Professional), Pay-As-You-Go (Scaling and up), or a custom plan from support. Nothing just shuts off silently.

**How fast is a request?**
Average latency is 4–12 seconds depending on the site, with a 60-second maximum per request. Fine for background crawls; not designed for real-time use.

## Which plan should you actually pick

The decision rule, based on the verified numbers:

- **Still writing your scraper** → Free plan or the 7-day trial. Don't pay until the selectors work.
- **Solo dev, standard sites, ~10K–80K pages/month, no exotic geo needs** → Hobby at $49/month, or $44.10/month on annual billing.
- **~100K–600K pages/month, or need real credit headroom** → Startup at $149/month. It's 10x Hobby's credits for 3x the price.
- **Need country-level geotargeting outside US/EU, or meaningful concurrency** → Business is the entry point at $299/month. Don't buy Hobby hoping geo will work — it's capped at US & EU by design.
- **Heavy premium/render workloads on protected targets** → run the Domain Cost Estimator on your actual target URLs first, size the plan against credits-per-page × volume, and pick Scaling or above so Pay-As-You-Go absorbs the overage months.

Before you commit to any paid tier, spend the trial credits on your real target sites and multiply out the true per-page cost — that one calculation tells you more than any comparison chart. When you're ready, 👉 [compare all plans and activate the free trial on ScraperAPI's pricing page](https://www.scraperapi.com/pricing?fp_ref=coupons).
