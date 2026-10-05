# playwright proxy: Route Chromium, Firefox and WebKit Through Real Exit IPs Without 407s or Silent Fallthroughs

You added a proxy to your Playwright script, launched Chromium, and the page still came back from your own IP. No exception, no error, just a request that never went where you told it to go. That is the standard failure mode, and it happens because Playwright's proxy option is a plain object with four fields, and three of them are easy to get wrong.

Below is what actually matters when wiring a proxy into Playwright — the credential trap, the SOCKS5 exception, where rotation really belongs, and how to pick a proxy product that doesn't burn your budget on a scraper that runs for ten minutes. The proxy provider referenced throughout is DataImpulse, whose gateway endpoints are used in the code samples.

## How Playwright takes a proxy (and where it takes it)

The `proxy` option exists in two places, and the difference decides your architecture:

- **At launch** — every browser context in that browser uses the same proxy.
- **At context creation** — each context gets its own proxy, so concurrent contexts can exit from different IPs.

js
const { chromium } = require('playwright');

// Global: one proxy for the whole browser
const browser = await chromium.launch({
  proxy: {
    server: 'http://gw.dataimpulse.com:823',
    username: process.env.PROXY_USER,
    password: process.env.PROXY_PASS,
  },
});

// Per context: different exit IP per isolation unit
const context = await browser.newContext({
  proxy: {
    server: 'http://gw.dataimpulse.com:823',
    username: process.env.PROXY_USER,
    password: process.env.PROXY_PASS,
  },
});


A context is the boundary. Pages created inside one context share its cookies, its storage, and its IP. If you open ten pages from a single context expecting ten different addresses, you get one. Rotation in Playwright means creating new contexts, not new pages.

## The credential trap that produces a 407

Unlike `requests` or `httpx`, Playwright does not read credentials out of the proxy URL. This fails silently in Chromium:

js
// Wrong: username and password are dropped
await browser.newContext({
  proxy: { server: 'http://user:pass@gw.dataimpulse.com:823' },
});


The connection goes out unauthenticated, the gateway answers `407 Proxy Authentication Required`, and depending on how your script handles page errors you may only see it much later as a timeout or an empty DOM. Playwright wants the parts split into separate fields:

js
// Right
await browser.newContext({
  proxy: {
    server: 'http://gw.dataimpulse.com:823',
    username: 'your_username',
    password: 'your_password',
  },
});


One extra detail that catches people: for the per-context form, Playwright requires the `server` value to be a complete `scheme://host:port` string. Leaving off `http://` is a common cause of a launch error that points nowhere useful.

## SOCKS5 with authentication is the exception, not a config mistake

Playwright's documentation lists SOCKS in the `server` field description and lists username/password under HTTP proxy authentication. Those are two different lists. SOCKS5 servers that require credentials are not covered.

In practice that means this raises `net::ERR_PROXY_AUTH_UNSUPPORTED` in Chromium:

js
// Chromium: authenticated SOCKS5 is not supported
await chromium.launch({
  proxy: {
    server: 'socks5://gw.dataimpulse.com:824',
    username: process.env.PROXY_USER,
    password: process.env.PROXY_PASS,
  },
});


The relevant upstream feature request has been open since November 2021, so this is a known gap rather than a version you forgot to upgrade. Three routes work around it:

1. **Use the HTTP(S) endpoint instead.** Most providers, DataImpulse included, publish both an HTTP and a SOCKS5 port for the same pool. The HTTP one is the path Chromium actually supports with credentials.
2. **Run a local relay.** A small local proxy listens unauthenticated on `127.0.0.1` and authenticates upstream for you. Playwright points at `socks5://127.0.0.1:<port>` with no credentials, which is a supported configuration. Costs you one supervised process.
3. **Authenticate by IP instead of password.** If your provider supports allowlisting the runner's IP, no credentials are needed at all. Serverless environments make this awkward, since the runner's IP changes, but a fixed VPS or CI runner with a static address works fine.

## Rotation belongs to contexts, and stickiness belongs to a token

Residential gateways usually rotate the exit IP on every connection. That suits jobs where each request is independent — price checks, SERP lookups, product pages. It is wrong for anything with continuity: a login, a multi-step checkout, a paginated crawl that counts sessions.

Playwright gives you both modes without much code, because the session identity rides on the credential string rather than the browser:

js
const { chromium } = require('playwright');

const browser = await chromium.launch();
const targets = ['https://example.com/a', 'https://example.com/b', 'https://example.com/c'];

for (const url of targets) {
  // Each context opens a new connection, so the rotating gateway hands out a new IP
  const context = await browser.newContext({
    proxy: {
      server: 'http://gw.dataimpulse.com:823',
      username: process.env.PROXY_USER,
      password: process.env.PROXY_PASS,
    },
    locale: 'en-US',
    timezoneId: 'America/New_York',
    geolocation: { latitude: 40.7128, longitude: -74.0060 },
    permissions: [],
  });
  const page = await context.newPage();
  await page.goto(url, { waitUntil: 'domcontentloaded' });
  await context.close();
}


For stickiness, the provider's sticky port range or a session token in the credentials pins one IP to one connection for a fixed window. DataImpulse documents sticky sessions from 1 to 120 minutes, defaulting to around 30 when you don't specify, with sticky connections sitting in the 10000–20000 port range. Reuse the same session token and the same browser context and you stay on one address; change it and you're somebody else.

## Make the browser fingerprint agree with the exit IP

A residential IP that renders as a US address while the browser reports `en-GB` locale and an `Asia/Tokyo` timezone is not a convincing user. Playwright lets you set all of it per context:

js
const context = await browser.newContext({
  proxy: { server: 'http://gw.dataimpulse.com:823', username, password },
  locale: 'en-US',
  timezoneId: 'America/New_York',
  geolocation: { latitude: 40.7128, longitude: -74.0060 },
  userAgent: 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36',
  viewport: { width: 1920, height: 1080 },
});


Matching that block to the proxy's geography is the difference between a request that looks like a person in New Jersey and one that looks like a datacenter with a confused clock.

The cleanest way to keep geography aligned is to let the proxy decide it. DataImpulse includes country-level targeting in the base price, so the country travels in the credential string. City, state, ZIP, and ASN selection is a **paid add-on that consumes traffic at double the standard rate** on residential plans — worth knowing before you decide every context needs ZIP-level precision.

## The DataImpulse setup that matters for Playwright

DataImpulse sells residential, mobile, datacenter, and premium residential IPs on a pay-as-you-go balance, with traffic that doesn't expire and no monthly subscription. For automation work, the pieces you actually touch are:

- **Gateway host:** `gw.dataimpulse.com`
- **Rotating HTTP/HTTPS:** port `823`
- **Rotating SOCKS5:** port `824`
- **Sticky sessions:** ports `10000`–`20000`, 1–120 minutes
- **Authentication:** username/password shown in the dashboard, or IP allowlisting
- **Targeting:** country included; state/city/ZIP/ASN billed at 2× on standard residential
- **Pool:** 90M+ ethically sourced residential IPs across 195 countries

Credentials are copied from the dashboard rather than assembled from memory — the exact targeting and session syntax is part of the string the panel issues you, and improvising it is how you end up with a session token that quietly does nothing.

👉 [Check DataImpulse's current pay-as-you-go rates and endpoint details](https://bit.ly/dataimPulse)

If your automation runs through an MCP browser agent rather than a script you wrote, the same gateway works: Playwright MCP accepts a `--proxy-server` flag plus `PROXY_USERNAME` and `PROXY_PASSWORD` environment variables, which routes every page the agent visits through the proxy without touching the agent's code.

## Verify the exit IP before you trust the run

Test with an IP echo endpoint, not with a page load. A page that renders correctly can still have gone out from your own address — a silent fallthrough looks exactly like success until you check the number.

js
const page = await context.newPage();
await page.goto('https://httpbin.org/ip');
console.log(await page.innerText('body')); // compare against your own IP


Throw a DNS check in while you're there. A proxy that leaks DNS resolves hostnames from your real network, which defeats a chunk of the point.

## Which proxy type actually fits the job

Residential is not the default answer for everything, and paying mobile rates for a job that runs fine on datacenter IPs is wasted budget.

| Job | Proxy type | Why |
| --- | --- | --- |
| Playwright tests against internal or unprotected staging | None | Adding a proxy to your own app tests usually just adds failure modes |
| High-volume scraping of sites without bot protection | Datacenter | Fastest and cheapest; server IP reputation is irrelevant where nothing checks it |
| E-commerce, SERP, social, geo-restricted content | Residential | Exit IPs come from consumer ISPs, so ASN reputation doesn't sink the request |
| Mobile-first platforms and the toughest anti-bot stacks | Mobile | Carrier IPs are shared behind NAT and hardest to block |
| Latency-sensitive flows that need high-trust residential exits | Premium residential | Smaller, faster sub-pool with all targeting included |

## Full DataImpulse pricing, all four proxy products

Pay-as-you-go across the board; purchased traffic doesn't expire and there's no subscription. Prices and tiers below were cross-checked against the provider's published pricing pages and third-party price summaries — confirm the number at checkout, since proxy pricing changes.

| Proxy product | Plan / tier | Traffic | Price | Rate per GB | Purchase |
| --- | --- | --- | --- | --- | --- |
| Residential | Intro pack | 5 GB | $5 | $1.00 | [Start with the 5 GB test pack](https://bit.ly/dataimPulse) |
| Residential | Pay-as-you-go | Any amount | Scales linearly | $1.00 | [Buy residential traffic](https://bit.ly/dataimPulse) |
| Residential | Volume | 1 TB | $800 | $0.80 | [See the 1 TB residential tier](https://bit.ly/dataimPulse) |
| Residential | Volume | 5 TB | — | $0.70 | [Ask for the 5 TB residential rate](https://bit.ly/dataimPulse) |
| Datacenter | Basic | 10 GB | $5 | $0.50 | [Buy the 10 GB datacenter pack](https://bit.ly/dataimPulse) |
| Datacenter | Standard | 100 GB | $50 | $0.50 | [Buy 100 GB of datacenter traffic](https://bit.ly/dataimPulse) |
| Datacenter | Volume | 1 TB | $450 | $0.45 | [See the 1 TB datacenter tier](https://bit.ly/dataimPulse) |
| Datacenter | Custom | 5 TB+ | From $2,250 | Custom | [Request datacenter volume pricing](https://bit.ly/dataimPulse) |
| Mobile | Starter | 2.5 GB | $5 | $2.00 | [Buy the 2.5 GB mobile pack](https://bit.ly/dataimPulse) |
| Mobile | Standard | 25 GB | $50 | $2.00 | [Buy 25 GB of mobile traffic](https://bit.ly/dataimPulse) |
| Mobile | Volume | 1 TB | $1,600 | $1.60 | [See the 1 TB mobile tier](https://bit.ly/dataimPulse) |
| Mobile | Custom | 5 TB+ | From $8,000 | Custom | [Request mobile volume pricing](https://bit.ly/dataimPulse) |
| Premium residential | Starter | 1 GB | $5 | $5.00 | [Buy the 1 GB premium residential pack](https://bit.ly/dataimPulse) |
| Premium residential | Standard | 10 GB | $50 | $5.00 | [Buy 10 GB of premium residential](https://bit.ly/dataimPulse) |
| Premium residential | Custom | 5 TB+ | From $20,000 | Custom | [Request premium residential pricing](https://bit.ly/dataimPulse) |

Two commercial notes worth factoring in before you commit: new users get a refund window on their first purchase (reported as seven days), and premium residential is the one tier where city/ZIP/ASN targeting doesn't carry the 2× surcharge that applies on standard residential.

## What a Playwright run actually costs

Browser automation is heavier than plain HTTP fetching, because a browser pulls HTML, CSS, JS bundles, images, and fonts — not just the document. A rendered page commonly moves a few hundred kilobytes to a couple of megabytes; a `requests` call for the same URL might be 40 KB.

At DataImpulse's $1/GB residential rate, the arithmetic works out like this: 10,000 page loads at roughly 300 KB each is about 3 GB, so around $3. At the datacenter rate of $0.50/GB, the same run is about $1.50. Add blocking images and fonts with `route()` if you're only after text — it cuts bandwidth more than any pricing negotiation will.

Two habits keep the bill from drifting:

- Block heavy resources you don't need (`*.png`, `*.woff2`, analytics hosts) via route interception.
- Don't route downloads, video, or bulk file transfers through residential traffic. A 200 MB export doesn't care whether the IP looks like a home connection.

## Errors you'll hit, and what each one means

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `net::ERR_PROXY_CONNECTION_FAILED` | Wrong host/port, dead endpoint, or network blocking the port | Confirm port 823 for HTTP vs 824 for SOCKS5, and that the runner can reach the gateway |
| `407 Proxy Authentication Required` | Credentials in the URL instead of separate fields, or the session/targeting suffix malformed | Split into `server`, `username`, `password` and copy the credential string from the dashboard |
| `net::ERR_PROXY_AUTH_UNSUPPORTED` | Authenticated SOCKS5 in Chromium | Use the HTTP(S) endpoint, a local relay, or IP allowlisting |
| Page loads, but from your own IP | Proxy never applied — set on the wrong object, or a context created before it | Set `proxy` at launch or `newContext`; verify with an IP echo endpoint |
| Cloudflare or bot wall regardless of proxy | Datacenter ASN and TLS fingerprint scored before your JS runs | Switch to residential or mobile exits |

## If you're scaling past one browser

Concurrency is where proxy choices stop being a detail. A rotating gateway handles parallel contexts fine, but per-context isolation only holds if each context authenticates independently — which means the credential string, not the browser instance, is your unit of identity. Keep a pool of credentials in environment variables or a secret store rather than hardcoding them, cap concurrency per domain to something polite, and close contexts when you're done so the connection and its exit IP are released.

👉 [Compare all four DataImpulse proxy types on one pay-as-you-go balance](https://bit.ly/dataimPulse)

## Questions that come up while wiring this up

**Does Playwright need a proxy for scraping?** Only when the target scores IP reputation. If a plain fetch works, adding a proxy adds a failure mode for no gain. Reach for residential exits when you see 403s, 503s, or a bot-check page instead of content.

**Can I use one proxy for parallel contexts?** Yes, but they'll share the exit IP unless the gateway rotates per connection. If the target counts requests per IP, that concurrency is invisible to it.

**Do I need a GPU-heavy stealth stack on top of a residential proxy?** Not automatically. A residential exit plus consistent locale, timezone, and viewport covers a lot of ground. Stealth plugins help when the target fingerprints the browser itself rather than just the network.

**How do I keep proxy costs predictable?** Buy pay-as-you-go traffic that doesn't expire, monitor cost per successful request rather than cost per gigabyte, and start with the $5 entry pack to measure your own success rate on your own targets before committing to a volume tier.

That last point is the one worth taking seriously. The per-GB price on the sticker tells you what bandwidth costs; the number that decides your budget is what a successful extraction costs after retries, blocked requests, and wasted bandwidth on pages that rendered nothing. Run the $5 pack against your real target list, count the successes, and do that math before you buy a terabyte.

👉 [Get the DataImpulse proxys starting at $1 per GB with never-expiring traffic](https://bit.ly/dataimPulse)
