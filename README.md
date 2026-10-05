# selenium proxy authentication: why Chrome ignores `user:pass` and the four fixes that actually work

You passed a proxy with a username and password into ChromeOptions, started the driver, and now one of two things happens. Either every request dies with a 407, or the browser throws up a native login dialog that Selenium cannot touch — you can't find it with `find_element`, you can't dismiss it with JavaScript, and the script just sits there.

This is not a bug in your code. Chrome's `--proxy-server` flag has no field for credentials, so it dials the proxy anonymously, gets challenged, and hands the challenge to the user. Selenium is not the user. Below are the four setups that actually resolve it, including the one most people should use and never bother writing an extension for.

## Why `--proxy-server=http://user:pass@host:port` silently degrades

That argument looks reasonable and it is parsed as host-and-port only. Chrome drops the userinfo portion and connects without credentials. The same applies to Selenium's own `Proxy` class: `setSocksUsername()` and `setSocksPassword()` are accepted by the API and then ignored by the Chrome driver, because Chrome has never implemented SOCKS5 authentication (`crbug#40323993`). Java and Python bindings will happily let you write code that cannot work.

So the 407 is not the proxy's fault, and the popup is not something you can script around. Proxy auth happens in the network stack, before the page exists, which is exactly why DOM-level tools can't reach it.

There are only three places you can put credentials so that Chrome actually uses them:

1. Nowhere — authorize your own IP on the provider side instead.
2. Inside a small Chrome extension that answers Chrome's auth callback.
3. Inside a driver wrapper that generates that extension for you.

Everything else is a dead end.

## Option 1: Skip authentication entirely with IP whitelisting

This is the option people overlook because the tutorials are all extension-shaped. Most serious proxy providers, including DataImpulse, support two authentication modes: username/password and IP authorization. In IP mode you register the public IP of the machine or CI runner that will run Selenium, and the gateway stops asking for credentials.

For browser automation this removes an entire class of failure. No extension to build, no zip to write to a temp directory, no Manifest V3 permission to get wrong, no risk that a Chrome update breaks your auth extension mid-sprint. Your options collapse to one line:

python
opts.add_argument("--proxy-server=http://gw.dataimpulse.com:823")


Two caveats worth knowing before you commit. First, whitelisting is per IP, so a rotating pool of CI runners means re-registering addresses, and a laptop that changes networks will break. Second, if the runner sits behind a shared NAT, you may be authorizing an address you don't fully control. For a fixed workstation, a dedicated VM, or a static-egress CI job, it's the least fragile setup available.

If your Selenium jobs run from one or two predictable machines, start here: 👉 [set up DataImpulse IP-authenticated proxy access](https://bit.ly/dataimPulse) and skip the extension code entirely.

## Option 2: The auth extension, done correctly for Manifest V3

This is the standard workaround, and most blog snippets you'll find are still written for Manifest V2, which Chrome no longer loads. If you copy a 2019-era snippet, you get an extension that fails silently and the popup returns.

The working shape is: a `fixed_servers` proxy config set from the extension, plus a `webRequest.onAuthRequired` listener that returns credentials when the challenge is for the proxy itself.

`manifest.json`:

json
{
  "manifest_version": 3,
  "name": "DataImpulse proxy auth",
  "version": "1.0",
  "permissions": ["proxy", "webRequest", "webRequestAuthProvider"],
  "host_permissions": ["<all_urls>"],
  "background": { "service_worker": "background.js" }
}


`background.js`:

js
const HOST = "gw.dataimpulse.com", PORT = 10000;
const USER = "YOUR_LOGIN", PASS = "YOUR_PASSWORD";

chrome.proxy.settings.set({
  value: {
    mode: "fixed_servers",
    rules: {
      singleProxy: { scheme: "http", host: HOST, port: PORT },
      bypassList: ["localhost"]
    }
  },
  scope: "regular"
});

chrome.webRequest.onAuthRequired.addListener(
  (details, callback) => {
    if (details.isProxy === false) { callback({}); return; }
    callback({ authCredentials: { username: USER, password: PASS } });
  },
  { urls: ["<all_urls>"] },
  ["asyncBlocking"]
);


Then zip it and hand it to Chrome:

python
import zipfile
from selenium import webdriver

with zipfile.ZipFile("proxy_auth.zip", "w") as z:
    z.writestr("manifest.json", open("manifest.json").read())
    z.writestr("background.js", open("background.js").read())

opts = webdriver.ChromeOptions()
opts.add_argument("--headless=new")
opts.add_extension("proxy_auth.zip")

driver = webdriver.Chrome(options=opts)
driver.get("https://api.ipify.org")
print(driver.find_element("tag name", "body").text)
driver.quit()


Four things bite people here. The password is stored in plaintext inside the zip, so write it to a temp directory you delete afterward and never commit it. The `webRequestAuthProvider` permission is mandatory on MV3; omit it and the listener never fires. The old `--headless` does not load extensions at all, so if you need headless you must use `--headless=new`. And if the temp directory isn't writable, Chrome loads an empty or broken extension and you get the popup back with no useful error, which is a genuinely annoying hour to lose.

## Option 3: Let the driver wrapper build the extension

Several projects do the above for you, with varying levels of maintenance.

The `.NET` binding has this built in: Selenium 4's `NetworkAuthenticationHandler` attached via `driver.Manage().Network` handles proxy challenges in-process, no extension file involved. If you're on C#, that's the shortest path.

On Python, `selenium-wire` was the classic answer for years, but it is no longer maintained — treat it as legacy code you may have to migrate off. SeleniumBase generates a proxy-auth extension on the fly and has tracked Chrome's extension changes reasonably well; when Chrome 137 broke proxy auth extensions, SeleniumBase shipped a patch for it, and when Chrome 142 broke it again the maintainer's guidance was to activate CDP Mode before navigating to any site. There's also `selenium-proxy-auth` on PyPI, a small package that builds an MV3 auth extension on request and returns the path so you can clean it up yourself.

The trade-off is the usual one: less code to write, but when Chrome changes extension behaviour again you're waiting on someone else's release rather than editing your own `background.js`.

## Rotating or sticky? This matters more in Selenium than in a scraper

Here's where browser automation differs from an HTTP client, and where a lot of DataImpulse configs go wrong.

DataImpulse's gateway is `gw.dataimpulse.com`, port 823 for HTTP/HTTPS and 824 for SOCKS5. The rotating endpoint hands out a new exit IP with each request. Sticky connections are port-based: you pick a port between 10000 and 20000 and the IP stays bound to it for a set interval, configurable from 1 to 120 minutes with 30 minutes as the default.

For Selenium, rotating is usually the wrong default. A browser session isn't a single request; a login flow, a cart, or a multi-step form is a dozen or more requests that must appear to come from one person. Point a browser at the rotating port and you can watch the identity change underneath the session. Use a sticky port, or the `sessid` parameter in the username (`login__cr.us;sessid.123:password`), which pins you to the same IP for roughly 30 minutes.

Country targeting rides in the username too, in the form `login__cr.us:password`. That's worth knowing for a different reason: if you ever swap the credentials into a SOCKS5 endpoint, Chrome will still ignore them, so for authenticated Selenium traffic stick to the HTTP gateway on 823 and leave SOCKS5 to curl, requests, or Scrapy.

## Which proxy type to buy for a Selenium job

This is the part that quietly decides your bill. A browser loads images, fonts, CSS, analytics scripts and every ad on the page; an HTTP scraper fetches one document. The same page can cost five to twenty times more bandwidth through Chrome than through `requests`. If you buy traffic by the gigabyte, that distinction is your entire budget.

| Proxy type | Rate | Bulk rate (1 TB+) | Entry purchase | Billing | What it's actually for |
| --- | --- | --- | --- | --- | --- |
| Residential | $1/GB | $0.80/GB | $5 for 5 GB | Pay-as-you-go, traffic never expires | The default for Selenium against sites that check IP reputation |
| Datacenter | $0.50/GB | $0.45/GB | Pay-as-you-go top-up | Pay-as-you-go, traffic never expires | Fast, cheap, unprotected targets and internal test suites |
| Mobile (4G/5G) | $2/GB | $1.60/GB | Pay-as-you-go top-up | Pay-as-you-go, traffic never expires | Sites that block residential ranges, mobile-web and app flows |
| Premium Residential | $5/GB | Custom (5 TB+) | $5 for 1 GB, $50 for 10 GB | Pay-as-you-go, dedicated account manager | High-trust targets where a blocked request costs more than the traffic |

Buy links for each, all pointing at the same signup and dashboard:

- 👉 [Compare residential proxy plans](https://bit.ly/dataimPulse)
- 👉 [See datacenter proxy rates](https://bit.ly/dataimPulse)
- 👉 [Check mobile proxy pricing](https://bit.ly/dataimPulse)
- 👉 [Look at premium residential options](https://bit.ly/dataimPulse)

Coverage differs by product too: the provider lists 214 locations for residential, 191 for mobile, 123 for datacenter and 210 for premium residential, drawn from a pool of 90M+ IPs across 195 countries with a published 99.51% success rate. Country-level targeting is included in the base rate; state, city, ZIP and ASN targeting is billed at double the standard per-GB rate on residential plans. That surcharge is easy to trip by accident — a `__cr.us;city.newyork` suffix in the username quietly doubles what that traffic costs, and on a browser workload burning gigabytes it adds up fast. Test with country targeting, add city targeting only when a specific target genuinely requires it.

One more thing worth pricing before you scale: run your first thousand requests against your real target and divide your spend by the number of *successful* requests. At $1/GB the headline number is low, but a cheap IP that gets blocked on 60% of attempts is more expensive per usable result than a $5/GB pool that succeeds most of the time.

## The troubleshooting table

Most Selenium proxy-auth failures fall into one of these buckets. Match the symptom, apply the fix, move on.

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| 407 on every request | Credentials never reached Chrome | Use IP whitelisting, or load an auth extension |
| Native auth popup, Selenium can't dismiss it | Same as above | Same as above |
| Extension loaded but popup still appears | Missing `webRequestAuthProvider` permission (MV3), or non-writable temp dir | Fix permissions, check the zip actually contains both files |
| Popup returns after it used to work | Chrome update changed extension behaviour | Update SeleniumBase/chromedriver, or activate CDP Mode before navigation |
| Extension works in windowed mode, not headless | Using old `--headless` | Switch to `--headless=new` |
| Proxy ignored entirely after switching | `user_data_dir` is pinned to the proxy it was created with | Use a fresh profile directory per proxy |
| SOCKS5 credentials rejected | Chrome does not support SOCKS5 auth | Use the HTTP gateway instead |
| Exit IP changes mid-session | Rotating endpoint in a browser | Use a sticky port or `sessid` |
| Wrong country every time | Targeting not in the username | Add `__cr.xx` to the login |

## Quick answers

**Does Selenium support authenticated proxies?** Not through `--proxy-server` or the built-in `Proxy` class on Chrome. It works fine through IP authorization or an auth extension.

**Can I just put credentials in the URL?** No. Chrome strips them; you get the popup or a 407.

**Does proxy auth work in headless mode?** Yes, with `--headless=new`. The legacy headless mode doesn't load extensions, so the auth extension approach fails there.

**What about Selenium Grid?** The credentials need to reach the browser node, not the client. Either whitelist the Grid nodes' egress IPs, or bake the auth extension into the node image so it's present before the session starts.

**What if I only need to test the setup?** Buy the $5 / 5 GB entry package rather than committing to volume. Traffic doesn't expire, so nothing is wasted if the target turns out to need a different proxy type.

## What to actually do

If your Selenium runs from a fixed machine or a dedicated VM, whitelist that IP and stop reading about extensions. You'll spend ten minutes and never debug a `background.js` again.

If your runners move around, build the MV3 extension once, keep it in your repo minus the credentials, and inject the username and password at runtime from environment variables. Budget an hour for the first version and expect one Chrome release every year or two to break it — that's the cost of the approach, and it's why patched wrappers like SeleniumBase exist.

Whichever route you take, match the session behaviour to the workload. Browsers need sticky identity far more than API clients do, and per-GB billing punishes browser traffic harder. Get those two things right and the rest is ordinary Selenium.

Ready to wire it up? 👉 [grab DataImpulse credentials and start with 5 GB for $5](https://bit.ly/dataimPulse).
