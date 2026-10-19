<p align="center"><a href="https://apify.com/DevilScrapes/mobile-bg-bulgaria-cars"><img src="assets/banner.svg" alt="Mobile.bg Bulgaria Car Scraper — JavaScript example" width="100%"></a></p>

# Mobile.bg Bulgaria Scraper for JavaScript and Node.js: car ads as JSON for your backend

<p align="center">
<a href="https://apify.com/DevilScrapes/mobile-bg-bulgaria-cars"><img alt="Run on Apify" src="https://img.shields.io/badge/Run%20on-Apify-DC2626?logo=apify&logoColor=white"></a>
<img alt="JavaScript" src="https://img.shields.io/badge/Node.js-20%2B-339933?logo=nodedotjs&logoColor=white">
<a href="LICENSE"><img alt="MIT license" src="https://img.shields.io/badge/license-MIT-0A0606"></a>
<a href="https://github.com/devilscrapes/mobile-bg-bulgaria-scraper-javascript/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/devilscrapes/mobile-bg-bulgaria-scraper-javascript/actions/workflows/ci.yml/badge.svg"></a>
</p>

Use this repo to call the **[Mobile.bg Bulgaria Car Scraper](https://apify.com/DevilScrapes/mobile-bg-bulgaria-cars)** from Node.js through the official `apify-client` package, in a single ES module. You pass a mobile.bg results address, or skip it, and get back plain objects for Bulgarian car ads: a euro price and a lev price as integers, kilometres, year, power, fuel, gearbox and HTTPS photo links ready for an `<img>` tag. The scraping happens on Apify, so your API route, queue consumer or scheduled function only starts a run and reads the rows when it finishes. Store them, diff them, or post them somewhere useful; the devil is in the dataset, not in your request handler.

> **Actor:** [Mobile.bg Bulgaria Car Scraper](https://apify.com/DevilScrapes/mobile-bg-bulgaria-cars) · **Publisher:** [Devil Scrapes](https://apify.com/DevilScrapes) · **Also in Python:** [mobile-bg-bulgaria-scraper-python](https://github.com/devilscrapes/mobile-bg-bulgaria-scraper-python)

## 🔥 What you get

- **Aim it with a link, or don't**: paste a results page you already narrowed on mobile.bg into `searchUrl`, or leave it out and the Actor reads the general cars-and-SUVs feed.
- **Two currencies, both integers**: `price` holds euros and `price_bgn` holds the lev figure when the ad shows one, so arithmetic starts immediately.
- **The spec sheet in columns**: year, `mileage_km`, `engine_power_hp`, `engine_size_cc`, fuel, gearbox, body style, colour and the first-registration month each get their own field.
- **Cyrillic that reads correctly**: the site serves the windows-1251 encoding, and the Actor decodes it properly, so `Дизелов` arrives as `Дизелов` and not as mojibake.
- **Optional detail visit**: with `enrichDetails` on, each ad page is opened as well to add the town, a private-or-dealer flag and the seller's own text, and to fill spec gaps the result card left.
- **One row per ad**: an ad that shows up on two result pages is written once, keyed by `listing_id`.
- **Refusals are its problem**: when the site rejects a request or answers with a server error, the Actor waits, opens a new connection and tries again, up to five times per page.

## 🎯 Use cases

### Feed a car-search front end from your own API

Run the Actor from a worker, upsert rows by `listing_id` into Postgres or MongoDB and serve them from your endpoint. Your UI filters by price, year or fuel without ever sending traffic to mobile.bg.

### Post new ads and price cuts to a webhook

After each scheduled run, compare the fresh `listing_id` set with the stored one and send new cars, or cars whose euro price fell, to Slack, Discord or any HTTP endpoint. A dealer or buyer watching one model sees changes within the schedule interval.

### Show photo cards in a dealer tool

`photo_urls` arrives as absolute HTTPS links taken from the result card, so a React or Vue component can render a small gallery next to the spec line without rewriting URLs.

### Normalise Bulgarian labels at the edge

Fuel, gearbox and body type come in Bulgarian. A tiny lookup object in your service turns `Автоматична` into `automatic` once, and every client downstream gets English values.

## ⚡ Quick start

You need an Apify account and an API token from [Console → Settings → Integrations](https://console.apify.com/settings/integrations). Apify's free plan includes monthly usage credit; this Actor's event prices are listed under Pricing below.

```bash
git clone https://github.com/devilscrapes/mobile-bg-bulgaria-scraper-javascript.git
cd mobile-bg-bulgaria-scraper-javascript
npm install
export APIFY_TOKEN=your_token_here
node index.js                       # uses input.json
node index.js my-input.json --out data --max-charge 2
```

The script starts the [Mobile.bg Bulgaria Car Scraper](https://apify.com/DevilScrapes/mobile-bg-bulgaria-cars), waits for it to finish, prints a link to the run in Apify Console and saves every result to `results.json` and `results.csv`. It exits with code 1 and the run's status message if the run fails.

## 🧩 The core call

If you only want the integration, these lines are all of it. The awaited `call()` returns once the Actor has stopped, and `listItems()` resolves to that run's rows as an array.

```javascript
import { ApifyClient } from 'apify-client';

const client = new ApifyClient({ token: 'YOUR_APIFY_TOKEN' });
const run = await client.actor('DevilScrapes/mobile-bg-bulgaria-cars').call({"maxResults": 10, "enrichDetails": true, "proxyConfiguration": {"useApifyProxy": true, "apifyProxyGroups": ["RESIDENTIAL"]}});
const { items } = await client.dataset(run.defaultDatasetId).listItems();
console.log(items);
```

## 📥 Input

`index.js` loads `input.json` and passes it along verbatim. This one leaves `searchUrl` out, so the default feed is read; it stops at 10 cars and keeps the per-ad detail visit, which costs about 22 cents in total. Do not drop the `RESIDENTIAL` group from `proxyConfiguration`, since the Actor will refuse to start without it. In production, build `searchUrl` from a filtered mobile.bg address in your handler and lift `maxResults` as far as 1,000.

```json
{
  "maxResults": 10,
  "enrichDetails": true,
  "proxyConfiguration": {
    "useApifyProxy": true,
    "apifyProxyGroups": [
      "RESIDENTIAL"
    ]
  }
}
```

| Field | Type | Required | Default | Allowed values |
|---|---|---|---|---|
| `searchUrl` | string | no | — | — |
| `maxResults` | integer | no | `50` | — |
| `enrichDetails` | boolean | no | `true` | — |
| `proxyConfiguration` | object | no | `{"useApifyProxy": true, "apifyProxyGroups": ["RESIDENTIAL"]}` | — |

## 📤 Output

One object per ad. Your code can rely on `listing_id`, `listing_url`, `title` and `scraped_at`; type every other property as nullable. `price` and `price_bgn` are integers, never formatted strings. `location`, `seller_type` and the full `description` need `enrichDetails`; `region` and `seller_name` are present but empty in the current version.

| Field | Type | Always present |
|---|---|---|
| `listing_id` | string | yes |
| `listing_url` | string | yes |
| `title` | string | yes |
| `make` | string or null | no |
| `model` | string or null | no |
| `year` | integer or null | no |
| `price` | integer or null | no |
| `currency` | string or null | no |
| `price_bgn` | integer or null | no |
| `mileage_km` | integer or null | no |
| `fuel_type` | string or null | no |
| `transmission` | string or null | no |
| `engine_power_hp` | integer or null | no |
| `engine_size_cc` | integer or null | no |
| `body_type` | string or null | no |
| `color` | string or null | no |
| `first_registration` | string or null | no |
| `location` | string or null | no |
| `region` | string or null | no |
| `seller_type` | string or null | no |
| `seller_name` | string or null | no |
| `photo_urls` | array | no |
| `description` | string or null | no |
| `posted_date` | string or null | no |
| `scraped_at` | string | yes |

## 💰 Pricing

| Event | Price (USD) |
|---|---|
| Actor start | $0.2 |
| result-row | $0.002 |

Prices as configured on 2026-10-16. The [Actor page](https://apify.com/DevilScrapes/mobile-bg-bulgaria-cars) always shows the current price. Runs are billed to the Apify account that owns the token. Use `--max-charge` to cap what a single run can cost.

## 🛠️ Working with the data in JavaScript

This script keeps a `previous.json` between runs and reports ads that are new or got cheaper, then writes them to `changes.json` for a webhook sender to pick up. It needs nothing beyond Node.js built-ins.

```javascript
import { readFile, writeFile } from 'node:fs/promises';
import { existsSync } from 'node:fs';

const current = JSON.parse(await readFile('results.json', 'utf8'));
const previous = existsSync('previous.json') ? JSON.parse(await readFile('previous.json', 'utf8')) : [];
const before = new Map(previous.map((car) => [car.listing_id, car]));

const changes = [];
for (const car of current) {
    const old = before.get(car.listing_id);
    if (!old) {
        changes.push({ kind: 'new', title: car.title, eur: car.price, km: car.mileage_km, url: car.listing_url });
    } else if (car.price !== null && old.price !== null && car.price < old.price) {
        const drop = old.price - car.price;
        changes.push({ kind: 'cheaper', title: car.title, eur: car.price, drop, url: car.listing_url });
    }
}

console.table(changes);
await writeFile('changes.json', JSON.stringify(changes, null, 2));
await writeFile('previous.json', JSON.stringify(current));
```

## ❓ FAQ

### Can I run this from AWS Lambda or a Vercel function?

Yes, if the platform lets an invocation wait for the Actor. A detail-enriched run adds one page visit per car, so with short timeouts start the run with `.start()`, return at once and read the dataset later from an Apify webhook.

### What happens if mobile.bg stops answering halfway through?

If the first page cannot be loaded after retries, the run fails so your job notices. If a later page fails, the run ends successfully with the cars it already has, so check the row count before you treat a short run as the whole market.

### Does it return the seller's phone number?

No. Phone numbers sit behind the site's contact widget and are not collected. You get the ad link, so a person can make contact through mobile.bg itself.

### How do I keep test runs cheap?

Set `enrichDetails` to false when you only need card fields, keep `maxResults` low, and pass `node index.js --max-charge 0.5`. Apify then stops charging the run at fifty cents.

## 🧯 Troubleshooting

- **`Set APIFY_TOKEN first`**: export the token in the same shell, or put it in a local `.env` file your shell loads. Never commit it.
- **Run finished with `FAILED` or `ABORTED`**: open the printed Console link; the run log says why. Input validation errors name the field.
- **Fewer rows than expected**: a `--max-charge` cap also limits how many results can be charged. Raise the cap or lower the limits in your input.
- **Something looks wrong in the data**: open an issue on the [Actor page](https://apify.com/DevilScrapes/mobile-bg-bulgaria-cars) so the maintainers see it.

## 🔗 More from Devil Scrapes

- [Mobile.bg Bulgaria Car Scraper on Apify Store](https://apify.com/DevilScrapes/mobile-bg-bulgaria-cars)
- [Python version of this example](https://github.com/devilscrapes/mobile-bg-bulgaria-scraper-python)
- [All Devil Scrapes Actors](https://apify.com/DevilScrapes)
- [Apify web scraping cookbook](https://github.com/devilscrapes/apify-web-scraping-cookbook)

Related Actors:

- [AutoScout24 Germany Car Scraper](https://apify.com/DevilScrapes/autoscout24-germany-cars) — German used-car ads, handy as a Western price benchmark.
- [Njuškalo Car Scraper (Croatia)](https://apify.com/DevilScrapes/njuskalo-croatia-cars) — car classifieds from Croatia, another Balkan market.
- [Blocket Car Scraper (Sweden)](https://apify.com/DevilScrapes/blocket-sweden-cars) — Swedish car ads with SEK prices for a Nordic comparison.

---

MIT licensed example code. The Actor itself runs on [Apify](https://apify.com/DevilScrapes/mobile-bg-bulgaria-cars) and is maintained by [Devil Scrapes](https://apify.com/DevilScrapes). Not affiliated with Mobile.bg.
