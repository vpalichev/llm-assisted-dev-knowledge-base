# Selecting a Stealth Toolchain: Decision Framework

Before reaching for a plugin or fork, the question reduces to: _what is the detection budget of the target site, and what fidelity is required to stay under it?_ Tooling selection is a function of adversary sophistication, not a default choice.

## 1. Tiering the Target

**Tier 0 — No active detection.** Static sites, internal tools, sites with no commercial incentive to block automation. Vanilla Playwright or Puppeteer suffices. No stealth layer required.

**Tier 1 — Basic flag checks.** Sites that probe `navigator.webdriver`, the automation infobar, or the `cdc_` ChromeDriver variables. Disabling automation switches and clearing the WebDriver flag is sufficient. Achievable with launch arguments alone.

**Tier 2 — Fingerprint-aware sites.** E-commerce, ticketing, sneaker sites, mid-tier scraping targets. Probe canvas, WebGL, plugin arrays, permissions coherence, and iframe realms. Requires a stealth plugin or patched fork.

**Tier 3 — Commercial bot management.** Cloudflare Bot Management, DataDome, PerimeterX/HUMAN, Akamai Bot Manager, Kasada, Imperva. Use behavioral telemetry, TLS fingerprints, and proprietary obfuscated client-side challenges. Stealth plugins are insufficient; success requires a special browser build, residential proxies, and often cookie-recycling infrastructure.

**Tier 4 — Anti-fraud and adversarial.** Banking, government identity portals, ad-fraud-targeted properties. Detection is multi-layered, server-correlated, and frequently updated. Automation is generally infeasible without insider access or out-of-band session establishment.

## 2. Stealth Plugins (Tier 1–2)

### 2.1 `puppeteer-extra-plugin-stealth`

A modular collection of evasions distributed as Puppeteer-extra plugins. Each module addresses one detection vector: `chrome.runtime`, `iframe.contentWindow`, `media.codecs`, `navigator.languages`, `navigator.permissions`, `navigator.plugins`, `navigator.webdriver`, `navigator.vendor`, `webgl.vendor`, `window.outerdimensions`. Modules can be enabled or disabled individually.

Plugin updates lag novel detections by weeks to months. The maintained-but-unofficial fork landscape means the version on npm may not include the latest evasions; community forks under names such as `puppeteer-real-browser` and `rebrowser-puppeteer` carry patches that have not been upstreamed.

### 2.2 `playwright-extra` Bridge

`playwright-extra` is a wrapper enabling `puppeteer-extra` plugins on Playwright. Functional but introduces a maintenance dependency on two ecosystems. Coverage gaps exist where plugins assume Puppeteer-specific internals.

### 2.3 `undetected-chromedriver`

Python-only. Patches the ChromeDriver binary at runtime to remove `cdc_` JavaScript variable injection, suppresses the automation infobar, and modifies several runtime properties. Adequate for Tier 1 and lower-end Tier 2 targets. Detected by Cloudflare Bot Management, DataDome, and similar in default configuration.

### 2.4 `nodriver`

Successor to `undetected-chromedriver` from the same author. Bypasses Selenium and the WebDriver protocol entirely, communicating directly via CDP from Python. Eliminates the WebDriver-specific detection surface (no `navigator.webdriver`, no Selenium-injected variables). Async API. Currently the strongest pure-Python stealth option for Tier 2 and lower Tier 3 targets.

## 3. Patched Forks and Special Builds (Tier 2–3)

### 3.1 `patchright`

A drop-in Playwright replacement (`pip install patchright` or `npm install patchright`) maintained as a fork addressing Playwright-specific tells: the `Runtime.Enable` CDP leak (a side effect where Playwright's call to `Runtime.enable` triggers detectable console behavior), the `__playwright__binding__` global, and several other runtime artifacts. API-compatible with Playwright; substituting `from patchright.sync_api import sync_playwright` for the Playwright import is typically sufficient.

### 3.2 `rebrowser-patches`

A set of Chromium patches and a corresponding Puppeteer/Playwright patch addressing the `Runtime.Enable` leak at the source. Distributed both as runtime monkey-patches and as a custom Chromium build. Higher fidelity than `patchright` for Tier 3 targets at the cost of build complexity.

### 3.3 Custom Chromium Builds

Compiling Chromium from source with anti-detection patches applied. Removes detection vectors that runtime patching cannot reach: source-level constants, V8 internals, compile-time feature flags. Build time on Windows is substantial (8–20 hours depending on hardware) and requires `depot_tools`, Visual Studio 2022 with the C++ workload, the Windows 10 SDK, and ~100 GB of disk. Generally not worth the investment unless operating against Tier 3 targets at scale.

### 3.4 Commercial Antidetect Browsers

Purpose-built browsers marketed at multi-account management, scraping, and ad verification. Each browser instance presents a configurable, persistent fingerprint: canvas noise, WebGL parameters, fonts, audio context, screen geometry, timezone, language, and TLS profile. Profiles are stored server-side and can be shared across team members.

Notable products: **Multilogin**, **GoLogin**, **AdsPower**, **Kameleo**, **Dolphin Anty**, **Incogniton**. Pricing tiers from ~$30/month (consumer) to several hundred per month (team/API access). Most expose a local automation API (typically a CDP endpoint) consumable from Playwright or Puppeteer.

The technical advantage over open-source stealth is twofold: (1) fingerprints are sourced from real-browser telemetry pools, ensuring coherent values rather than randomized-and-incoherent ones; (2) maintainers update against new detections as a paid service. The trade-off is operational dependence on a vendor and trust in their handling of session data.

### 3.5 `Camoufox`

Open-source Firefox fork with anti-detection patches at the C++ level. Targets the Firefox detection surface, which is undersaturated relative to Chrome — many bot-detection vendors prioritize Chromium signatures, leaving Firefox-based automation comparatively under-fingerprinted. Distributed as prebuilt binaries for Windows, macOS, and Linux. Python API via `camoufox` package.

### 3.6 Brave, Mullvad Browser, and Hardened Forks

Privacy-focused forks (Brave, LibreWolf, Mullvad Browser) randomize or minimize fingerprintable values for anti-tracking purposes. Coincidentally useful for automation in some contexts, but not designed for it; lack programmatic fingerprint control and sometimes fail bot checks specifically because their hardening is anomalous relative to baseline browser populations.

## 4. Components Beyond the Browser

A stealth browser alone is insufficient against Tier 3 targets. The full stack typically comprises:

**Residential or mobile proxies.** IP reputation is independently scored. A perfect browser fingerprint emerging from a known datacenter ASN is flagged regardless. Bright Data, Oxylabs, Smartproxy, IPRoyal are the major commercial providers.

**TLS-correct HTTP fallback.** When mixing browser-rendered flows with API calls (e.g., authenticated XHR after login), the API calls must originate from the same TLS profile as the browser. `curl_cffi` (Python) and `tls-client` (Go) provide named browser profiles; the alternative is routing API calls through the browser via `page.evaluate(() => fetch(...))`, which is slower but fingerprint-coherent by construction.

**Session warming.** Fresh sessions on commercial bot-managed sites are scored more aggressively than aged sessions with browsing history. Warming entails simulating human navigation (homepage → category → product) before the target action, with realistic timing.

**Cookie persistence.** `cf_clearance` and equivalent challenge-clearance cookies are reusable until expiry; harvesting them once via a real interactive browser and reusing across automated sessions reduces challenge frequency.

## 5. Decision Heuristic

The economically rational sequence:

1. Run vanilla Playwright against the target. If it works, stop.
2. If blocked, add `patchright` (Playwright) or `nodriver` (Python). Reassess.
3. If still blocked, route through residential proxies. Reassess.
4. If still blocked and the target is Tier 3, evaluate commercial antidetect browsers or `Camoufox`. The cost of subscription is typically lower than the engineering hours required to maintain a custom Chromium build.
5. Custom Chromium builds are justified only at scale (hundreds of concurrent sessions) where per-session licensing of commercial products becomes uneconomical.

## 6. Windows-Specific Notes

`patchright`, `nodriver`, and `Camoufox` install cleanly on Windows via `pip` with no additional toolchain. `puppeteer-extra-plugin-stealth` installs via `npm` identically to other Node packages.

Custom Chromium builds on Windows require Visual Studio 2022 (Community edition is sufficient) with the _Desktop development with C++_ workload, the _Windows 10 SDK (10.0.20348 or later)_, and `depot_tools` added to `PATH`. Set `DEPOT_TOOLS_WIN_TOOLCHAIN=0` as an environment variable to use the locally installed Visual Studio rather than Google's internal toolchain. Build invocation:

```
gn gen out\Stealth --args="is_debug=false is_component_build=false symbol_level=0"
autoninja -C out\Stealth chrome
```

Commercial antidetect browsers ship native Windows installers (`.exe` or `.msi`) and expose their local automation API on `127.0.0.1:<port>`; the connection URL is typically displayed in the product UI after launching a profile. Connection from Playwright on Windows:

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.connect_over_cdp("http://127.0.0.1:54321")
    context = browser.contexts[0]
    page = context.new_page()
    page.goto("https://target.example.com")
```

Profile data directories for antidetect browsers default to `%APPDATA%\<VendorName>\` or `%LOCALAPPDATA%\<VendorName>\`; back these up before major version upgrades, as profile schema migrations have historically caused fingerprint drift.