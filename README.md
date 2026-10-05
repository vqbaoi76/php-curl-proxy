# php curl proxy: Working cURL Code, Country Targeting, and How to Stop Getting Blocked Mid-Scrape

Most people searching this already have half the answer. They know the request should go through a proxy, they know it's a `curl_setopt` somewhere, and they're stuck on the part nobody documents properly: which option takes the credentials, why the same code worked locally and returns 403 on the server, and where the country parameter is supposed to go.

So let's skip the theory. Here's a PHP script that exits through someone else's IP, how to bend that IP to a specific country, and what to do when it starts failing halfway through a run.

## The three options that do the actual work

PHP's cURL binding is thin. Anything libcurl can do with a proxy, `curl_setopt` exposes. For 90% of use cases you need these:

- `CURLOPT_PROXY` — `host:port`, or a full URL with scheme
- `CURLOPT_PROXYUSERPWD` — `login:password`
- `CURLOPT_PROXYAUTH` — usually `CURLAUTH_BASIC`
- `CURLOPT_PROXYTYPE` — only when you switch away from HTTP

php
<?php
$ch = curl_init('https://httpbin.org/ip');

curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);

curl_setopt($ch, CURLOPT_PROXY, 'gw.dataimpulse.com:823');
curl_setopt($ch, CURLOPT_PROXYUSERPWD, 'your_login:your_password');
curl_setopt($ch, CURLOPT_PROXYAUTH, CURLAUTH_BASIC);

curl_setopt($ch, CURLOPT_CONNECTTIMEOUT, 10);
curl_setopt($ch, CURLOPT_TIMEOUT, 30);
curl_setopt($ch, CURLOPT_ENCODING, '');  // accept gzip/br — see the cost note below

$response = curl_exec($ch);

if ($response === false) {
    echo 'curl error ' . curl_errno($ch) . ': ' . curl_error($ch);
} else {
    echo $response;
}
curl_close($ch);


The endpoint above is DataImpulse's rotating residential gateway, which their own Guzzle tutorial uses with port 823. If you keep `CURLOPT_PROXY` as plain `host:port`, libcurl ignores `CURLOPT_PROXYPORT`, so don't set both and expect the second one to win.

For HTTPS targets, libcurl issues the `CONNECT` tunnel itself. You don't need `CURLOPT_HTTPPROXYTUNNEL` — that option is for forcing a tunnel on plain HTTP, and turning it on is one of the more popular ways to break a working setup.

## Credentials: three formats, one of which bites

Putting everything in `CURLOPT_PROXY` works:

php
curl_setopt($ch, CURLOPT_PROXY, 'http://login:password@gw.dataimpulse.com:823');


It also fails in an annoying, non-obvious way when the password contains characters like `@`, `:`, `/`, `+`, or `?`. The username gets percent-encoded by some HTTP clients and not others, and Basic auth expects standard base64, not the URL-safe variant. If your credentials end up mangled, the proxy closes the socket and you get a TLS error that has nothing to do with TLS. Keep host and port in `CURLOPT_PROXY`, keep `login:password` in `CURLOPT_PROXYUSERPWD`, and if you must build a URL, run both halves through `rawurlencode()` first.

The third option is the one worth knowing about: IP whitelisting. Once your server's egress IP is whitelisted in the provider dashboard, the credentials disappear from the request entirely.

php
$client = new \GuzzleHttp\Client();
$response = $client->request('GET', 'http://ip-api.com/', [
    'proxy' => 'http://gw.dataimpulse.com:823',
]);


That's cleaner — no secrets in the repo, no `.env` leak risk. It also breaks the moment your production egress IP isn't static, which for anything on a container platform is most of the time. Credentials in code are uglier and more portable.

## Rotating or sticky: the port number is the switch

This is the part that's genuinely provider-specific, and it's where reading the docs saves an afternoon. DataImpulse routes by port:

| What you want | Port | Notes |
| --- | --- | --- |
| Rotating, HTTP/HTTPS | 823 | New IP per request |
| Rotating, SOCKS5 | 824 | Same pool, different protocol |
| Sticky, HTTP/HTTPS or SOCKS5 | 10000–20000 | IP bound to that port for a set interval |

Sticky rotation runs from 1 to 120 minutes; if you don't specify an interval, it defaults to 30. The port is your session handle:

php
curl_setopt($ch, CURLOPT_PROXY, 'gw.dataimpulse.com:10000');
curl_setopt($ch, CURLOPT_PROXYUSERPWD, 'your_login:your_password');


Same port on the next request means the same exit IP, which is what you want for login flows, paginated crawls, and anything with a cart. Different port means a different IP. The concurrency ceiling is effectively how many sticky ports you're willing to juggle, so don't spawn 500 workers on port 10000 and wonder why they all share one address.

## Country targeting without opening the dashboard

Targeting lives in the username, after a `__` delimiter, as `key.value,value` pairs:

bash
curl -x "http://login__cr.de:password@gw.dataimpulse.com:823" https://api.ipify.org/


Two countries in one pool:

bash
curl -x "http://login__cr.de,au:password@gw.dataimpulse.com:823" https://api.ipify.org/


Multiple parameter sets are separated by a semicolon. DataImpulse documents `__cr` for country selection and `__noasn` for excluding specific ASNs, with country-level targeting included in the base rate and state, city, ZIP, and ASN precision billed as add-ons. That's a reasonable split: most PHP scraping jobs need a country, not a postcode, and paying for postcodes you never use is how a $1/GB pool becomes a $4/GB pool.

One gotcha that costs people real time: if you configure parameters through the dashboard instead of the username, you have to hit **Save Configuration**. The selected parameters are not applied until you do.

Test every change with a request to `https://api.ipify.org` or `http://ip-api.com/json`. Logging the exit IP per request is the difference between debugging a proxy problem in ten minutes and debugging it in a day.

## SOCKS5, and the one extra option it needs

php
curl_setopt($ch, CURLOPT_PROXY, 'gw.dataimpulse.com:824');
curl_setopt($ch, CURLOPT_PROXYTYPE, CURLPROXY_SOCKS5);
curl_setopt($ch, CURLOPT_PROXYUSERPWD, 'your_login:your_password');


If you want DNS resolved by the proxy rather than by your server — which matters when the target's DNS is geo-split or your host's resolver leaks — use `CURLPROXY_SOCKS5_HOSTNAME` instead. HTTP proxies can't do this, and it's the main practical reason to pick SOCKS5.

## When it breaks: read the number, don't guess

| Symptom | Usually means | Fix |
| --- | --- | --- |
| `curl error 7`, `CURLE_COULDNT_CONNECT` | Wrong host/port, or your host blocks outbound 823 | Test with the CLI: `curl -x "http://login:pass@gw.dataimpulse.com:823" https://api.ipify.org` |
| `407 Proxy Authentication Required` | Missing or malformed credentials | Move creds to `CURLOPT_PROXYUSERPWD`, set `CURLAUTH_BASIC` |
| `curl error 56` / `35` on HTTPS | Socket closed during handshake — often credential encoding | Stop embedding creds in the URL; check for `+` `/` in the password |
| HTTP `403` or `429` from a non-proxy host | The proxy worked; the target doesn't like you | Rotate, reduce request rate, fix headers — not a proxy config issue |
| Requests hang for 30+ seconds | No timeouts set | `CURLOPT_CONNECTTIMEOUT` at 10, `CURLOPT_TIMEOUT` at 30 |
| Works via CLI, fails in PHP | Different PHP process environment | libcurl reads lowercase `http_proxy` / `https_proxy` env vars when `CURLOPT_PROXY` isn't set — an inherited env var can silently override your intent |

That last row deserves a sentence. If a deployment injects `http_proxy` into the environment, PHP's libcurl will honor it for requests where you didn't explicitly set `CURLOPT_PROXY`, and you'll see traffic leaving through an address you never configured. Set `CURLOPT_NOPROXY` for internal hostnames you want to reach directly.

And whatever you do, don't carry `CURLOPT_SSL_VERIFYPEER => false` into production. It disables certificate validation for the whole request, proxy or not. Every Stack Overflow answer that "fixed" a proxy issue by adding it also removed your TLS guarantees.

## The same thing in Guzzle, Laravel, and the lazy shortcut

Guzzle takes a proxy string or an array with separate `http` and `https` entries:

php
$client = new \GuzzleHttp\Client();

$response = $client->request('GET', 'https://example.com/data', [
    'proxy'   => 'http://your_login:your_password@gw.dataimpulse.com:823',
    'timeout' => 30,
    'headers' => ['User-Agent' => 'Mozilla/5.0 (compatible; MyBot/1.0)'],
]);


Laravel's HTTP client passes options straight through, so it's one line:

php
$data = \Illuminate\Support\Facades\Http::withOptions([
        'proxy' => 'http://your_login:your_password@gw.dataimpulse.com:823',
    ])
    ->retry(3, 1000)
    ->get($url)
    ->body();


If you're using `Http::pool()`, the proxy option has to be applied per request inside the pool closure — setting it on one request doesn't carry to the others, which is a common reason pool jobs return your own IP for half the responses.

`file_get_contents()` can do it too, via a stream context with `proxy` set to `tcp://host:port` and `request_fulluri => true`. It works. It also throws HTTP status codes away by default, gives you no useful error information without extra header parsing, and can't do SOCKS5. Fine for a cron job hitting one URL, a bad foundation for scraper code.

## Level two: keep bandwidth (and cost) down

At per-GB billing, your bill is the bytes that pass through the proxy, not the requests. Two settings change that number more than anything else:

php
curl_setopt($ch, CURLOPT_ENCODING, '');      // ask for gzip/br — HTML shrinks 60-80%
curl_setopt($ch, CURLOPT_FOLLOWLOCATION, false); // stop chasing redirects you didn't ask for
curl_setopt($ch, CURLOPT_RANGE, '0-262143'); // if you only need the <head> for price data


Rough arithmetic: a 200 KB HTML page is about 5,000 requests per gigabyte at $1/GB. Compressed, that's several times more pages for the same money, which matters more than haggling over a ten-cent rate difference between providers. Retries also double-bill you, so a 403 that you retry four times is a 5× cost multiplier on that request — fix the request, not the retry counter.

For parallel work, `curl_multi_exec` with a rotating port is the standard pattern. Cap concurrency at something your target will tolerate and add jitter between batches; 200 simultaneous requests from a rotating residential pool is not stealthy, it's a pattern.

## Which proxy type your PHP job actually needs

This decision costs more than the provider choice does.

- **Datacenter** — fast, cheap, and blocked by anything that inspects IP reputation. Right for high-volume pulls from sites that don't fight back. DataImpulse lists these at $0.50/GB.
- **Residential** — real consumer IPs. The default for SERP scraping, price monitoring, and anything behind Cloudflare. $1/GB, country targeting included.
- **Mobile** — 4G/5G carrier IPs, where NAT means thousands of users share one address, so blocking it is expensive for the target. $2/GB. Use it when residential gets 403s on a specific domain, not as a default.
- **Premium residential** — a higher-speed pool with all targeting options included and a dedicated account manager, listed at $5/GB. Justifiable when a job has a deadline and you're burning engineer hours on timeouts.

An honest limit from DataImpulse's own FAQ: it's a rotating residential, mobile, and datacenter network. If you need static ISP proxies, a fully managed scraping API, or access to banking and government sites, it's the wrong tool and you should look elsewhere.

## All current DataImpulse plans

Everything below is pay-as-you-go, billed per GB, with traffic that doesn't expire and no subscription requirement — that last part is the reason it fits irregular scraping workloads, since a half-used monthly plan costs more per usable GB than a balance you draw down whenever.

| Plan | Price | Billing | Best for | Purchase |
| --- | --- | --- | --- | --- |
| Datacenter | $0.50/GB | Pay-as-you-go, per GB | High-volume pulls from lenient targets | [ Check the DataCenter proxy rate](https://bit.ly/dataimPulse) |
| Residential | $1/GB | Pay-as-you-go, per GB | Defended targets: SERPs, e-commerce, social | [ Start with 5 GB of residential traffic](https://bit.ly/dataimPulse) |
| Mobile | $2/GB | Pay-as-you-go, per GB | Hardest targets, app and mobile-web data | [ See the Mobile proxy plans](https://bit.ly/dataimPulse) |
| Premium Residential | $5/GB | Pay-as-you-go, per GB | Deadline work, high-speed pool, all targeting included | [ Compare Premium Residential options](https://bit.ly/dataimPulse) |

The published entry point is a $5 / 5 GB package, and volume tiers drop the rate as you commit: $0.8/GB at 1 TB and $0.7/GB at 5 TB, according to the provider's own comparison page. Country targeting is included across plans; state, city, ZIP, and ASN targeting are paid add-ons.

For a PHP script in development, the $5 package is the sensible starting point. You'll learn your real cost per successful request — which is the number that matters, not the headline rate — long before you need to think about bulk tiers.

Supporting details worth knowing before you commit: 90M+ IPs across 195 countries, HTTP/HTTPS and SOCKS5, rotating and sticky sessions, API access, IP authorization as an alternative to credentials, 24/7 human support. The provider publishes a 99.51% success rate and a 4.8/5 G2 score, and says it has served 500,000+ customers over the last four years. Treat those as vendor-published figures, not independent verification.

## Questions that come up constantly

**Does PHP need a special extension for proxies?**
No. Proxy support is in libcurl. You need the `curl` extension compiled in, and `CURLPROXY_SOCKS5` is available by default. If `curl_setopt` is undefined, you have an installation problem, not a proxy problem.

**Why does the same script return different IPs on consecutive runs?**
Because rotating ports assign a new exit IP per request. That's the feature. If you need consistency, move to a sticky port in the 10000–20000 range and keep using it.

**Can I put the country in a query string instead of the username?**
Not the way these gateways work — targeting is parsed from the username after `__`. Adding `&country=de` to your target URL does nothing except make the target site see a weird query string.

**How do I know the proxy is actually being used?**
Request `https://api.ipify.org` and compare against your server's IP. If they match, your request went direct — check for an inherited `http_proxy` env var or a typo'd `CURLOPT_PROXY`.

**Is a proxy enough to avoid blocks?**
No. IP reputation is one signal among many, and a residential IP with a headless-browser fingerprint and 40 requests per second still gets blocked. Rotate, throttle, send sane headers, respect `robots.txt`, and don't hammer one host.

## The ten-minute version

Set `CURLOPT_PROXY` to `gw.dataimpulse.com:823`, put credentials in `CURLOPT_PROXYUSERPWD`, set a connect timeout and a total timeout, add `CURLOPT_ENCODING`, hit `api.ipify.org`, and confirm the IP changed. Then add `__cr.xx` to the username for the country you need, decide between a rotating port and a sticky one based on whether your code holds a session, and log the exit IP plus the HTTP status on every request so the first 403 is a five-second diagnosis.

That's the whole job. The proxy is the easy part — the requests you send through it are what get you blocked.
