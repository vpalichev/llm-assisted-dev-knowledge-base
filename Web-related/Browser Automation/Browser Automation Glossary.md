# Protocols and Drivers

**Chrome DevTools Protocol (CDP)** — A bidirectional WebSocket-based protocol exposed by Chromium-family browsers for instrumentation, debugging, and remote control. Backbone of Puppeteer and (historically) Playwright. Domains include `Page`, `Network`, `DOM`, `Runtime`, `Fetch`, `Target`.

**WebDriver Classic (W3C WebDriver)** — The HTTP/JSON-based standard underlying Selenium 4. Synchronous request-response model; one command per HTTP round trip. Limited event subscription model.

**WebDriver BiDi** — The successor standard combining WebDriver's stability with CDP-style bidirectional events over WebSocket. Supported in Firefox, Chrome, and increasingly the default transport in Playwright and Selenium 4.

**Marionette** — Firefox's internal automation protocol, wrapped by `geckodriver` for WebDriver compliance.

**RemoteDebuggingPort** — The TCP port (commonly 9222) Chromium exposes when launched with `--remote-debugging-port=<n>` to accept CDP connections.

## Execution Topology

**Browser Context** — An isolated session within a single browser process — its own cookies, storage, cache, and permissions. Cheaper than launching a new browser; analogous to an incognito profile. Native concept in Playwright (`BrowserContext`); approximated in Puppeteer via `createIncognitoBrowserContext`.

**Target** — In CDP terminology, any attachable entity: page, iframe, service worker, shared worker, or background page.

**Page / Tab** — A top-level browsing context. Distinct from a `Frame`, which is any nested browsing context including the main frame.

**Frame Tree** — The hierarchy of main frame plus child iframes, each with independent JavaScript realms and document state.

**Realm / Execution Context** — A distinct JavaScript global environment. Each frame, worker, and isolated extension content script has its own.

## Locators and Selection

**Selector** — A string identifying DOM nodes. Common engines: CSS, XPath, text-based, ARIA role, test-id.

**Locator** — A Playwright/modern abstraction representing a _query_, not a snapshot. Re-resolved on each action, which avoids stale-element exceptions characteristic of WebDriver's `WebElement`.

**StaleElementReferenceException** — Selenium error raised when a previously-resolved element is no longer attached to the DOM. Locators (above) eliminate this class of failure.

**Test ID (`data-testid`)** — Convention of attaching automation-only attributes to elements to decouple selectors from styling and copy changes.

**Accessibility Tree / a11y Tree** — The browser-computed semantic tree exposed to assistive technology. Increasingly used as a selection surface (`getByRole`) and as the primary perception layer for AI browser agents.

## Action Semantics

**Actionability Checks** — Pre-action assertions a framework runs before clicking or typing: element is attached, visible, stable (not animating), receives events (not occluded), and enabled. Playwright's defining feature versus raw CDP.

**Auto-Waiting** — Automatic polling until actionability conditions are met or a timeout elapses. Replaces explicit `sleep` calls.

**Trusted Events** — Input events dispatched by the browser's input pipeline, carrying `isTrusted: true`. Distinguished from `dispatchEvent`-synthesized events, which many anti-bot systems reject.

**Input Pipeline Injection** — Sending events through CDP's `Input.dispatchMouseEvent` / `Input.dispatchKeyEvent` so they appear trusted, versus DOM-level synthesis.

**Hydration** — In SSR frameworks (Next.js, Nuxt), the phase where client-side JavaScript binds handlers to server-rendered markup. Clicks issued before hydration completes silently no-op — a common source of flake.

## Failure Modes

**Flake / Flakiness** — Non-deterministic test failure caused by timing, network variability, or race conditions. Combated via auto-waiting, retry-ability, and deterministic test data.

**Retry-ability** — Property of an assertion or action that re-evaluates on a polling loop until success or timeout. Playwright's `expect()` web assertions are retry-able; classical `assert` calls are not.

**Race Condition** — Test logic depending on the order of asynchronous events (network, animations, hydration). Symptom of insufficient waiting primitives.

**Quarantine** — CI practice of isolating known-flaky tests so they do not block the pipeline while remaining tracked.

## Headless Modes

**Headless** — Browser running without a visible window. Chrome supports `--headless=new` (full-fidelity, shares code path with headed) and the legacy `--headless=old` (separate, deprecated implementation with detectable differences).

**Headed** — Browser running with full UI. Slower; required for some debugging and certain media APIs.

**Xvfb** — Linux virtual framebuffer for running headed browsers without a display server. Not relevant on Windows; the Windows equivalent is simply running headed on a session-attached desktop, or using headless mode.

## Anti-Automation Surfaces

**Bot Detection / Fingerprinting** — Server- or client-side classification of automated traffic. Signals include `navigator.webdriver`, missing/abnormal plugins, WebGL renderer strings, font enumeration, canvas hashes, TLS fingerprint (JA3/JA4), and behavioral patterns.

**`navigator.webdriver`** — A boolean property set to `true` when a WebDriver session is active. Trivially detectable; the most basic bot signal.

**Stealth Plugin / Patching** — Modifications (e.g., `puppeteer-extra-plugin-stealth`, `playwright-stealth`) that overwrite fingerprintable properties to mask automation. An ongoing arms race; never a complete solution.

**CDP Detection** — Anti-bot heuristics that detect CDP attachment by side effects such as `Runtime.enable` triggering specific console behaviors or memory layout changes.

**TLS Fingerprint (JA3 / JA4)** — Hash of the ClientHello packet. Automation tools using non-browser HTTP stacks (`requests`, `httpx`) produce different fingerprints than real browsers, regardless of headers sent.

**Residential Proxy** — Egress IP belonging to a real consumer ISP, sold via proxy networks. Distinguished from datacenter IPs, which are easily blocklisted.

**CAPTCHA Solver** — External service (2Captcha, CapSolver, Anti-Captcha) accepting a challenge and returning a token, typically via human labor or ML.

## Network Layer

**Request Interception** — Pausing in-flight network requests to inspect, modify, mock, or abort them. CDP `Fetch.requestPaused`; Playwright `page.route()`.

**HAR (HTTP Archive)** — JSON format capturing network traffic. Used for replay-based testing and offline reproducibility.

**Service Worker** — Background script intercepting network requests for the page's origin. Can interfere with automation by serving cached responses; often disabled in test environments.

## Recording, Replay, Tracing

**Codegen** — Tool that records user interactions and emits framework code (`playwright codegen`, Selenium IDE).

**Trace Viewer** — Playwright's post-mortem inspector: per-action DOM snapshots, network log, console, and screenshots in a navigable timeline.

**rrweb** — Open-source library for recording DOM mutations and replaying full browser sessions. Foundation of products like LogRocket and OpenReplay.

## AI / Agentic Browsing

**Browser Agent** — An LLM-driven system that controls a browser to accomplish goals expressed in natural language. Examples: Claude for Chrome, Anthropic's Computer Use, Browser Use, OpenAI Operator.

**Perception Layer** — How an agent observes the page. Three dominant approaches: (1) raw screenshots with vision models; (2) accessibility tree serialization; (3) DOM-derived "set-of-marks" overlays numbering interactive elements.

**Set-of-Marks (SoM)** — Visual prompting technique in which interactive elements are labeled with numeric overlays so a vision model can reference them by index.

**Action Space** — The discrete set of operations an agent can emit: `click(id)`, `type(id, text)`, `scroll`, `navigate(url)`, `wait`, etc.

**Prompt Injection (in agentic contexts)** — Adversarial content embedded in a webpage that attempts to subvert the agent's instructions via the perception channel. A primary safety concern unique to autonomous browsing.

## Windows-Specific Practical Notes

Browser binaries on Windows live under `%LOCALAPPDATA%\Google\Chrome\Application\chrome.exe` and `C:\Program Files\Mozilla Firefox\firefox.exe`. Driver binaries (`chromedriver.exe`, `geckodriver.exe`, `msedgedriver.exe`) are PE executables, not ELF. Process trees launched by automation frameworks are visible via `Get-Process` and orphan-cleaned with `Stop-Process -Name chrome -Force` rather than `pkill`. Path separators in selectors and config files should use forward slashes or escaped backslashes (`\\`) to avoid escape-sequence issues in JSON/JS string literals.