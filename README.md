# ScraperAPI Proxy Mode: How to Configure It, Where the Parameters Go, and Why Your Headless Browser Needs It (Plans Compared)

ScraperAPI proxy mode is a proxy front end bolted onto the same scraping API. You point your client at `proxy-server.scraperapi.com` on port `8001`, authenticate with the username `scraperapi` and your API key as the password, and the service handles proxy rotation, CAPTCHA solving and retries behind that connection. ScraperAPI's own documentation says it delivers the same features and performance as the API endpoint.

The smallest working request looks like this:

bash
curl --proxy 'http://scraperapi:API_KEY@proxy-server.scraperapi.com:8001' \
  -k \
  'https://www.example.com'


The `-k` flag matters more than it looks. Drop it and the request fails on certificate verification, which is the single most common reason a first proxy-mode attempt returns nothing.

Below: the four connection fields, the dot syntax that replaces query parameters, the SSL trap and its fix, the specific reason Puppeteer breaks on the API endpoint, a troubleshooting table, and what each request costs in credits.

> **Quick decision.** If your code already has a proxy slot (`requests` `proxies=`, an Axios `proxy` object, a `--proxy-server` browser flag), swap in the ScraperAPI URL and you're done. If you drive a headless browser, proxy mode is the only route that works properly. If your script fires one GET and parses the HTML, stay on the API endpoint; it's less setup.
>
> 👉 [Start the free trial and get your API key](https://www.scraperapi.com/?fp_ref=coupons)

## Plans and what a request costs

Proxy-mode requests are billed on the same credit system as endpoint requests. A standard page is 1 credit. Amazon costs 5, Google and Bing (including subdomains) cost 25, LinkedIn costs 30, and domains sitting behind Cloudflare, DataDome or PerimeterX add 10 credits when the protection has to be bypassed. Rendering JavaScript or forcing residential proxies pushes the number higher still.

That matters for proxy mode because the parameters that make hard sites work are the same ones that multiply the cost of each request.

| Plan | Price | Monthly credits and limits | Get started |
| --- | --- | --- | --- |
| Free trial | $0 for 7 days | 5,000 credits, one-off, no card required | [Start the trial](https://www.scraperapi.com/?fp_ref=coupons) |
| Free | $0 | 1,000 credits, up to 5 concurrent connections | [Open a free account](https://www.scraperapi.com/?fp_ref=coupons) |
| Hobby | $49/mo, or $44.10/mo billed yearly | 100,000 credits | [See Hobby pricing](https://www.scraperapi.com/?fp_ref=coupons) |
| Startup | Per tier on the pricing page | Higher allowance than Hobby | [Compare Startup](https://www.scraperapi.com/?fp_ref=coupons) |
| Business | Per tier on the pricing page | Higher allowance than Startup | [Compare Business](https://www.scraperapi.com/?fp_ref=coupons) |
| Scaling | Per tier on the pricing page | Adds pay-as-you-go overage | [Compare Scaling](https://www.scraperapi.com/?fp_ref=coupons) |
| Professional | Per tier on the pricing page | Adds pay-as-you-go overage | [Compare Professional](https://www.scraperapi.com/?fp_ref=coupons) |
| Advanced | Per tier on the pricing page | Adds pay-as-you-go overage | [Compare Advanced](https://www.scraperapi.com/?fp_ref=coupons) |
| Enterprise | Custom quote | Custom volume, minimum rates on some request types | [Talk to sales](https://www.scraperapi.com/?fp_ref=coupons) |

ScraperAPI publishes a separate credit allowance, concurrency limit and price for every paid tier from Startup upward, and those numbers get revised often enough that printing them here would date the article within weeks. The live pricing table is the only figure worth trusting at checkout. Two rules hold across all tiers: credits reset at renewal instead of rolling over, and annual plans hand you the year's credits up front.

Run your actual target URLs through the Domain Cost Estimator in the dashboard before picking a tier. A scraper hitting Google 20,000 times a month burns 500,000 credits, which puts a "100,000 credit" Hobby plan out of reach on the first afternoon.

## The four connection fields

Every proxy-mode configuration, whatever the language, comes down to these four values:

| Field | Value |
| --- | --- |
| Proxy host | `proxy-server.scraperapi.com` |
| Port | `8001` |
| Username | `scraperapi` |
| Password | your API key |

Node with Axios:

js
import axios from 'axios';

axios.get('https://www.example.com', {
  proxy: {
    host: 'proxy-server.scraperapi.com',
    port: 8001,
    auth: {
      username: 'scraperapi',
      password: 'API_KEY'
    },
    protocol: 'http'
  }
});


Python with `requests`:

python
import requests

proxies = {
    "http": "http://scraperapi:API_KEY@proxy-server.scraperapi.com:8001",
    "https": "http://scraperapi:API_KEY@proxy-server.scraperapi.com:8001",
}

response = requests.get("https://www.example.com", proxies=proxies, verify=False)


The proxy URL itself stays on plain `http` even when the target is HTTPS. ScraperAPI terminates TLS on its side and re-issues the request, which is exactly why your client can't chain the certificate back to a public root and why verification has to be relaxed.

## Advanced parameters go in the username, not the URL

This is where most integrations go wrong. With the API endpoint, ScraperAPI's options live in the query string alongside `api_key` and `url`. With proxy mode, the query string belongs to the target site, so ScraperAPI's parameters have to move into the `username` field, appended with dots:


http://scraperapi.render=true.country_code=us:API_KEY@proxy-server.scraperapi.com:8001


The general shape is `<protocol>://<username>[.param=value.param=value]:<password>@<host>:<port>`. Any number of parameters can be chained before the colon.

The four you'll reach for most:

- `render=true` runs the page through a headless browser, which is what you need for JavaScript-heavy sites that return an empty shell otherwise.
- `country_code=us` sends the request from a specific country. Note the field name: ScraperAPI's current documentation uses `country_code`, while older tutorials floating around use `country=us`. If geotargeting appears to be ignored, check which spelling you're using.
- `premium=true` restricts the request to residential proxies rather than the default mix.
- `keep_headers=true` forwards your own User Agent, Referer and Cookies to the target.

That last one deserves its own example, because without it your carefully crafted headers are silently replaced by ScraperAPI's defaults:

bash
curl --request GET \
  --header "X-MyHeader: 123" \
  --proxy "http://scraperapi.keep_headers=true:API_KEY@proxy-server.scraperapi.com:8001" \
  --insecure \
  "http://example.com"


Anything you can pass as an API parameter can be passed this way, so a job that needs rendering and US geolocation in the same request becomes `scraperapi.render=true.country_code=us`.

## SSL verification has to be off, or the proxy CA has to be trusted

ScraperAPI's documentation is explicit about this: your code must be configured not to verify SSL certificates, otherwise requests can't be routed properly. In practice that means `-k` / `--insecure` in cURL, `verify=False` in `requests`, and `ignoreHTTPSErrors: true` in Puppeteer.

Disabling verification is fine for throwaway scraping runs. It is not fine if the same code path also handles logins or anything sensitive, because you've turned off the check that would catch a man-in-the-middle. The alternative is to install ScraperAPI's proxy CA certificate, which the documentation makes available for download, into your machine's trusted root store:

- **Windows 10/11:** open `mmc`, add the Certificates snap-in for the computer account, then import the certificate under Trusted Root Certification Authorities.
- **macOS:** drag the file into Keychain Access under the System keychain, open the "ScraperAPI CA" entry, and set it to Always Trust.
- **Linux:** install the `.pem` file into your system CA directory and update the stored authority bundle.

Once the certificate is trusted, drop the `-k` flag and leave verification on.

## When the API endpoint is the better call

For a plain HTTP request there's no reason to bother with proxy mode. Send the URL to the endpoint with your key, get HTML back, parse it.

Browser automation flips that recommendation. ScraperAPI's Puppeteer guide labels the endpoint approach "not recommended" and explains why with a concrete failure. If you load `quotes.toscrape.com` through `api.scraperapi.com`, the browser gets the first HTML document, then hits the page's relative assets. The stylesheet reference `/static/main.css` normally resolves to `http://quotes.toscrape.com/static/main.css`. Through the endpoint it resolves to `http://api.scraperapi.com/static/main.css`, which isn't a real resource, and the request errors out. You end up with unstyled HTML and none of the JavaScript-rendered content that made you reach for a headless browser in the first place.

Clicking links has the same problem. Relative hrefs resolve against the endpoint host, so any attempt to navigate or mimic a real user's session breaks.

Proxy mode fixes this because the browser sees a normal network route rather than a rewritten URL. Relative paths resolve against the target domain, assets load, and navigation works.

## Wiring Puppeteer to the proxy port

Two steps. First, launch the browser with the proxy server flag and HTTPS errors ignored:

js
const browser = await puppeteer.launch({
  ignoreHTTPSErrors: true,
  args: ['--proxy-server=http://proxy-server.scraperapi.com:8001']
});


Second, authenticate on the page after it's created:

js
const page = await browser.newPage();
await page.authenticate({
  username: 'scraperapi',
  password: 'API_KEY'
});


From that point on, every request the browser makes goes out through ScraperAPI's pool, so the target site sees the proxy IP rather than yours. The authentication call belongs on each new page you open, not just the first one.

ScraperAPI does flag one caveat in its own guide: resource loading conflicts occasionally occur with Puppeteer and the proxy port, and the recommended fix is to contact support with the specifics. If a page renders in your local browser but comes back missing assets through the proxy, that's the case to escalate rather than debug blind.

## Troubleshooting the usual failures

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Certificate verification error, request never completes | SSL verification left on | Add `-k`, `verify=False`, or `ignoreHTTPSErrors: true` |
| Page returns HTML but no CSS, JS or rendered content | API endpoint used with a headless browser | Switch to the proxy port |
| Custom User Agent or Referer ignored | `keep_headers` missing | Append `keep_headers=true` to the username |
| Geotargeting appears to do nothing | Wrong parameter name | Use `country_code`, not `country` |
| Authentication failure on the proxy | Key in the username slot, or username misspelled | Username must be `scraperapi`; the key is the password |
| 429 responses under load | Concurrency limit for your plan | Reduce parallel requests or move up a tier |

## FAQ

**Does proxy mode cost more than the API endpoint?**
ScraperAPI describes proxy mode as the same product with the same features and performance, so the credit system is the one that applies either way. What drives cost is the target domain and the parameters you attach: a `render=true` request costs more than a plain page whether you send it through the endpoint or the proxy port. Confirm per-URL costs with the dashboard estimator before you commit to a plan size.

**Can I use SOCKS with proxy mode?**
ScraperAPI's documentation mentions other proxy protocols such as SOCKS, but the proxy port guide doesn't publish a SOCKS port or a configuration example. Ask support before building around it.

**Do I need to rewrite my scraper?**
Only the connection settings change. Existing proxy configuration keeps its shape; the host, port and credentials get swapped for ScraperAPI's. Everything downstream, including your parsing logic, stays as it is.

**What happens when I run out of credits?**
Scraping pauses until the next renewal by default. Some tiers can auto-upgrade to the next plan, and the higher tiers add pay-as-you-go overage at a fixed per-credit rate up to a monthly cap you set. Requests that exceed your concurrency limit get turned away with a 429 rather than queued and charged.

## Which setup fits your code

Match the integration to what your scraper actually does:

- A single GET request per URL, no browser: API endpoint.
- Existing proxy pool or proxy config in `requests`, Axios or cURL: proxy mode, four fields, done.
- Puppeteer or any browser automation: proxy port, no alternative, plus `page.authenticate` on every page.
- Requests that keep failing on protected domains: proxy mode with `render=true` and, where needed, `premium=true`, then check the credit multiplier before scaling up.

Before you buy, three things are worth ten minutes: run your top target domains through the Domain Cost Estimator, compare the monthly and annual price on the tier you're eyeing, and note that the advertised refund window applies to a first purchase rather than to renewals later on. The 7-day trial with 5,000 credits and no card is enough to validate a proxy-mode integration against your real targets before any money moves.

👉 [Get your API key and test proxy mode against your own targets](https://www.scraperapi.com/?fp_ref=coupons)
