# Can Camoufox Reliably Scrape PowerPoint Online Co-Authoring Presence? (June 2026)

## 1. Executive verdict

**Partially viable for observation, but Camoufox is the wrong tool and adds risk rather than removing it — the single biggest blocker is that this is the user's OWN authenticated enterprise account behind Entra ID, where the threat model is identity/behavior risk scoring, not third-party fingerprint anti-bot.** Reading the live editing-collaborator list from PowerPoint for the web is technically achievable for any authenticated co-editor because the data flows to the browser over the Real-Time Channel (RTC, a SignalR-over-WebSocket service) and is rendered as presence avatars ("Coauth Gallery") in the editor DOM — and the Microsoft Graph API does NOT expose this live list (confirmed against current Graph docs). However, Camoufox's stealth value (C++-level fingerprint spoofing to beat Cloudflare/DataDome) is irrelevant against Microsoft 365, which uses Entra ID Protection risk scoring, conditional access, and Intune device-compliance checks — not Cloudflare/DataDome. Worse, Camoufox is net-negative here: it presents an unfamiliar Firefox fingerprint, a Kazakhstan ASN, and a non-device-registered browser — exactly the "unfamiliar sign-in properties" pattern Entra flags. The next-best architecture is a Tampermonkey userscript injected into the user's real signed-in session, or plain Playwright on stock Edge/Chromium with a persistent authenticated profile.

## 2. Camoufox current-state assessment

- **Latest tagged release on daijro/camoufox:** `v135.0.1-beta.24` (tag dated 15 March 2026 on the releases page) is still marked "Latest" and fetches Firefox 135. Newer experimental pre-releases exist: `v146-hardware` (14 March 2026, "has not been tested yet… don't use unless you know what you are doing") and `FF146-BETA` (7 January 2026, macOS-only — "Linux builds but fails to launch pages; Windows is still being worked on"). The Camoufox site lists `v146.0.1-beta.25` (January 2026) as the experimental current build; the Python launcher package is `camoufox` 0.4.11.
- **Project status:** The project went through a roughly one-year maintenance gap and resumed in late 2025/early 2026. Active browser development moved to `github.com/CloverLabsAI/camoufox` and `github.com/VulpineOS/VulpineOS`; daijro's repo is now the checkpoint mirror/source of truth. Camoufox's own README/site states it "has gone down in performance due to the base Firefox version and newly discovered fingerprint inconsistencies."
- **Detection signatures (last 6 months):** The Web Scraping Club (Pierluigi Vinciguerra, 4 June 2026) reports practitioners finding Camoufox "does not pass the harder targets the way it used to," attributing this partly to its open-source code letting anti-bot vendors build exact counter-signals. They confirmed a **WebRTC IP leak under proxy on the official Firefox 146 build** — the real WAN IP leaks via the `srflx` ICE candidate even behind a working proxy — which a community fork (JWriter20) fixes via extra `media.peerconnection` prefs. Canvas anti-fingerprinting noise is **OFF by default** in current builds (CloverLabs "Disable Canvas Noise" commit), and the stock noise algorithm perturbs ~50% of pixels including flat fills, a tell that known-pixel checks (DataDome, Castle) catch. An independent 2026 benchmark (Ian L. Paterson, 31 Cloudflare targets) found nodriver outperformed Camoufox/Patchright, and Camoufox uniquely failed dev.to on a "Firefox TLS quirk the CDN flags." Camoufox.com itself warns some WAFs test for SpiderMonkey engine behavior "which is impossible to spoof."
- **Architecture relevant here:** Camoufox patches Playwright's Juggler protocol so the page agent's JS is sandboxed and invisible to the page; it can run headless patched to look headed, with a Python virtual-display fallback. None of these strengths address Microsoft's identity-layer defenses.

**Alternatives (maintenance / base / strength vs. Camoufox for M365 sign-in):**

- **patchright** (Kaliiiiiiiiii-Vinyzu) — Maintained. Patches Playwright CDP leaks (Runtime.enable etc., ~22 patches / ~5,856 lines); runs system Chrome via `channel=chrome` (Chrome 148 in the 2026 benchmark). **Strength for M365:** a Chromium/Edge-like fingerprint matches what enterprise tenants expect far better than Firefox. Best Chromium stealth option _if_ stealth is wanted at all.
- **nodriver** — Maintained; successor to undetected-chromedriver. Drives Chrome directly over CDP with no Playwright shim — won the 2026 benchmark with zero blocked Cloudflare targets by avoiding automation-protocol fingerprinting entirely. Async-only; fewer tutorials.
- **rebrowser-patches** — Maintained; patches Puppeteer/Playwright (Runtime.enable fix). The 2026 benchmark found it performs essentially identically to vanilla (ships Chromium 136, behind current). Weak incremental value.
- **undetected-playwright** — Largely superseded by patchright/rebrowser; less active. Not recommended for this case.
- **Bright Data Scraping Browser** — Commercial, hosted, residential-proxy-backed (treat marketing skeptically). Counterproductive for signing into the user's own enterprise tenant: introduces foreign egress IPs that worsen Entra risk.
- **Browserless** — Commercial hosted Chrome/Playwright infra. Same objection: datacenter egress is bad for enterprise sign-in.
- **Kameleo / Multilogin** — Commercial anti-detect "browser-profile" managers built for multi-accounting (marketing-heavy; verify independently). Not needed for one legitimate account; their core value (many distinct identities) is the opposite of what this task requires.

**Public reports of stealth browsers vs. login.microsoftonline.com:** Essentially none specific to Camoufox + M365. The reproducible reports are mundane Firefox login-flow issues (blocking third-party cookies breaks login.microsoftonline.com; clean-profile/cookie advice). There is no evidence Camoufox is known to break M365 login — and no evidence it helps.

## 3. PowerPoint Online collaboration data surface

**Microsoft Graph does NOT expose the live editor list (verified).** Graph's `/presence` API returns Teams availability/activity (Available, DoNotDisturb, Presenting…), not document co-authors. Graph also exposes no document lock-state/active-editor property: Microsoft's own support staff confirm "Microsoft Graph does not expose any API that returns whether a document is [open/locked/being co-authored]," and the only signal is an HTTP 423 Locked on write. So the live co-editor list must come from the web client itself.

**Transport (well documented):** Office for the web's real-time collaboration runs over the **Real-Time Channel (RTC)**. Per author Gilad Oren on the Microsoft .NET Blog (25 Jan 2024, "Office's RTC migration to modern .NET"): _"Real-Time Channel (RTC) is Microsoft Office Online's websocket service that powers the real-time collaboration experiences… It serves hundreds of millions of document sessions per day… It is mainly built around a SignalR service."_ The **web client uses the JavaScript SignalR client over WebSocket** (with SignalR's long-poll fallback); desktop apps use a C++ SignalR client. RTC code is shared across Word, Excel, and PowerPoint Online. A second service, the **Office Collaboration Service (OCS)**, holds merged document state. Per Microsoft Learn ("Supporting services," CSPP Plus, 2022), RTC carries "information about presence in the document by other users, such as **cursor location, display of users in the Coauth Gallery**, and so on."

**Endpoints/hostnames:** The editor iframe is served from `*.officeapps.live.com` (PowerPoint editor frame `/p/PowerPointFrame.aspx`, analogous to Word's `word-edit.officeapps.live.com/we/wordeditorframe.aspx`). The literal RTC WebSocket URL is resolved **dynamically per file via WOPI discovery** — the CheckFileInfo CSPP Plus properties `RealTimeChannelEndpointUrl` (the `rtc` WOPI action) and `OfficeCollaborationServiceEndpointUrl` (the `collab` action) carry the runtime endpoints; WOPI discovery defines permission-gated actions `rtc`, `collab`, and `documentchat`. Microsoft is migrating Office web from `*.officeapps.live.com` to the unified `cloud.microsoft` domain. **The exact `wss://` path (e.g., any `/pods/` or `/ocs/` segment) is NOT in public primary sources** — a fresh DevTools/HAR capture of a live session is required to pin both the path and the SignalR presence message schema (hub name, payload fields).

**Where identity appears in the client:** Avatars render in the top-right "Coauth Gallery" using Microsoft Fluent UI `Persona`/`PersonaCoin`/`PersonaPresence` components. Identity-bearing props: `text`/`primaryText` (display name), `secondaryText`, `imageUrl` (profile photo), `imageInitials`, `presence` (enum), `presenceTitle` (hover tooltip), and `showUnknownPersonaCoin` (the "?" coin for anonymous/guest). Legacy Office UI Fabric class names were `ms-Persona-*` (e.g., `ms-Persona-primaryText`, `ms-Persona-presence`), but **current Fluent builds use mergeStyles-generated hashed/obfuscated class names**, so stable literal selectors are NOT guaranteed. The exact live PowerPoint editor selectors and JS globals are **not documented publicly (INFERRED** from the Fluent component model and Microsoft's "Coauth Gallery" naming).

**Fields exposed per collaborator:** Display name, initials, profile-photo URL, and **editing-vs-viewing status** are surfaced (documented by feature; Microsoft Support: thumbnails appear top-right of the ribbon, with an indicator showing where someone is working). Anonymous/guest is indicated via the unknown-persona coin. **Email and Azure AD object ID (oid) are NOT confirmed exposed** in the presence UI; corroborating evidence — Word's JavaScript API uses per-session ephemeral coauthor GUIDs that "differ across sessions and coauthors" — suggests Office deliberately avoids exposing stable directory IDs in co-authoring surfaces. Treat any claim that oid/email is readable from presence data as **unverified**.

**Availability and cadence:** Presence is visible to **any authenticated co-editor** (not owner/admin only) — that is the purpose of co-authoring presence. It is **real-time push** over RTC (join/leave + cursor events). This is distinct from the WOPI autosave channel: per Microsoft Learn ("Co-author using Microsoft 365 for the web"), _"During a single-user editing session, PowerPoint for the web only calls PutFile every three minutes. During an active co-authoring session, that frequency is increased to every 60 seconds,"_ and PowerPoint re-verifies permissions by _"calling CheckFileInfo at least every five minutes."_ Those WOPI cadences are server-side and not the presence channel.

**Critical architectural constraint:** The editor is a **cross-origin iframe** (`*.officeapps.live.com`) embedded in the host page (microsoft365.com / SharePoint / OneDrive). Same-origin policy means a script on the _host_ page CANNOT read into the iframe DOM. Reading presence requires either (a) Playwright frame access (Playwright can address child frames across origins) or (b) a userscript **matched to the `officeapps.live.com` origin** so it runs inside the editor frame. A host-page-only extension cannot see it.

**Existing tools:** No browser extension or userscript that scrapes Office-for-web presence was found — likely because of the cross-origin constraint.

## 4. Authentication and account risk analysis

**Microsoft does not use Cloudflare/DataDome here.** The relevant defenses are Entra ID Protection (risk-based sign-in), conditional access, and Intune device-compliance — which inverts Camoufox's value proposition.

**Sign-in flow constraints:**

- **MFA** is effectively unavoidable in enterprise tenants and cannot be fully automated cryptographically (and the task forbids attempting to). The standard community pattern is **interactive first-run sign-in (human completes MFA) once, then persist and reuse the authenticated session** (Playwright storage-state / persistent profile). TOTP can be scripted in test tenants, but FIDO2/Windows Hello/passkey and Authenticator push approvals require the human. (Note a 2026 Windows wrinkle: on builds ≥ 26100.6725, WebAuthn with `userVerification: "preferred"` can unexpectedly prompt to create a FIDO2 PIN — another reason to keep a human in the auth loop.)
- **Conditional access / device compliance:** Many enterprises require "device marked compliant" or "hybrid Entra joined" for Office 365. Entra identifies the device via a client certificate provisioned at device registration; the browser must present it, and **the device check fails in private mode or with cookies disabled**. A non-registered Camoufox/Firefox instance on a personal Windows box will **FAIL a compliant-device policy outright** — a hard block independent of any stealth. "Require approved client app" / "require app protection policy" similarly block browser automation.
- **Entra ID Protection risk scoring:** The "Unfamiliar sign-in properties" detection "triggers… when a sign-in occurs with properties that are unfamiliar to the user. These properties can include **IP, ASN, location, device, browser, and tenant IP subnet**." Newly created users sit in a dynamic "learning mode" with a **minimum five-day** duration (and re-enter it after long inactivity). A sign-in from **AS35104 (Jusan Mobile JSC, Kazakhstan/Almaty — RIPE-registered 2005-06-03, ~169 IPv4 prefixes; registry lists it as an ISP/carrier rather than a distinct mobile ASN)** using an unfamiliar **Firefox** build on an unregistered device is a textbook unfamiliar/anomalous sign-in. Microsoft enhanced these detections in August 2025. If the tenant uses location-based conditional access (country allow-list), a Kazakhstan IP may be blocked outright — "block access always overrides grant access."

**Persistent session viability:** Interactive first-run + persistent profile is far more viable than fully automated login and is the standard approach. Session lifetime depends on the tenant's token-lifetime / sign-in-frequency conditional-access settings (commonly ~1-hour access tokens with silent refresh; refresh/session can last days–weeks unless "sign-in frequency: every time" or risk-based reauth is enforced). Plan for periodic interactive re-auth.

**Account/tenant risk:** Browser-automation patterns can trip tenant-admin alerts — impossible travel, anonymous/unfamiliar IP, atypical/anomalous token characteristics, unusual user agent. Microsoft Threat Intelligence may itself remediate (block user, revoke tokens) in high-confidence cases. Using a residential/mobile proxy that differs from the user's normal egress **increases** risk for an enterprise account; a static, allow-listed corporate egress IP is what enterprise sign-in expects.

**Legal/ToS:** Enterprise Microsoft 365 is governed by the **Microsoft Product Terms / Online Services Terms** and the organization's agreement — NOT the consumer Microsoft Services Agreement, which explicitly excludes "Microsoft 365 for enterprise, education, or government." So the consumer MSA's anti-scraping/AI clauses (effective 30 Sept 2025) don't directly govern, but the tenant's acceptable-use rules and the Product Terms do. On US case law, hiQ v. LinkedIn and Van Buren v. United States narrowed the CFAA to a "gates-up-or-down" test concerning data the user is **not** authorized to access; accessing data you ARE authorized to see as a legitimate co-editor of your own/your tenant's file is on much safer footing than public scraping. But note hiQ's **final outcome (7 Dec 2022 stipulated consent judgment, N.D. Cal. 17-cv-03301-EMC): a $500,000 judgment for breach of LinkedIn's user agreement plus stipulated CFAA liability "based on hiQ's direct access to password-protected pages… using fake accounts."** The clean distinction protecting this use case: you are a legitimate authenticated participant viewing data the application already shows you — no fake accounts, no circumvention. Still, automating one's own corporate account may violate internal IT policy; confirm with the tenant owner.

**Geo note (Kazakhstan):** Beyond Entra risk scoring, KZ has historically had episodes of national TLS-interception (root-CA) requirements — verify the egress path does not present an injected root CA, which would itself be a sign-in anomaly. (Flagged as a known regional consideration; not confirmed active in 2026.)

## 5. Concrete architecture recommendation

**Is the goal achievable with Camoufox today?** The _observation_ goal (read the live co-editor list) is achievable with browser automation in principle, but **Camoufox specifically is the wrong tool and likely net-negative** because: (a) its stealth targets Cloudflare/DataDome, which M365 doesn't use; (b) Firefox is an unfamiliar browser for most enterprise users, raising Entra risk; (c) it cannot present an Intune device certificate, so it fails compliant-device conditional access; (d) current builds have a known WebRTC IP leak under proxy and degraded performance; (e) it is beta/experimental and recently emerged from a maintenance gap.

**What specifically breaks it:** Compliant/hybrid-joined-device conditional access → hard block. Country/location allow-listing → the KZ IP is blocked. Even with neither, the unfamiliar Firefox + KZ ASN + unregistered-device combination maximizes Entra "unfamiliar sign-in properties" risk and may force step-up auth or trigger admin alerts.

**Recommended architecture (priority order):**

1. **Best — Tampermonkey/Greasemonkey userscript in the user's REAL signed-in browser.** The user is a legitimate co-editor signing in normally (Edge/Chrome, real registered device, normal IP, passing MFA and conditional access for real), so there is **no automation/sign-in risk surface at all**. Scope the userscript to the editor origin (`@match https://*.officeapps.live.com/*`, plus forthcoming `cloud.microsoft`) so it runs INSIDE the editor iframe, where it can read the Coauth Gallery Persona DOM and/or hook the SignalR/WebSocket presence frames. This sidesteps every Entra/conditional-access/device problem AND the cross-origin constraint. Lowest-risk, highest-reliability way to prove viability with 1 file / 1 watcher.
    
2. **Next — Plain Playwright on stock Edge/Chromium (or Firefox) with a persistent authenticated profile**, no stealth patches, since this is the user's own account. Use `launch_persistent_context` with a fixed user-data dir; sign in interactively once (human completes MFA); reuse the profile across runs. Use Playwright's cross-origin frame API to address the `officeapps.live.com` editor frame and read presence, or attach a `page.on("websocket")` handler to capture RTC frames. Prefer the Edge/Chromium channel to match enterprise expectations. Works only if the tenant does NOT require a compliant/registered device.
    
3. **Hybrid — human signs in, automation attaches to the existing session.** Launch the real Edge/Chrome with `--remote-debugging-port` after the human authenticates, then attach Playwright/CDP to the live tab to read the iframe. Combines real-session legitimacy with programmatic extraction; good middle ground if a persistent Playwright profile alone trips conditional access.
    
4. **Avoid** — Bright Data / Browserless / Kameleo / Multilogin (foreign egress + multi-identity tooling worsen enterprise-account risk) and fully automated headless login against login.microsoftonline.com (MFA + risk scoring make it brittle and alert-prone). **Camoufox itself belongs in this "avoid" bucket for this specific use case.**
    
5. **Fallback — manual observation.** If conditional access is strict (compliant device required) and userscript injection is disallowed by tenant policy, the only compliant path may be a human watching the Coauth Gallery, or obtaining the data via a supported admin channel.
    

**Extraction strategy (options 1–3):**

- **DOM path:** Locate the Coauth Gallery container in the editor iframe and read Fluent Persona nodes. Because class names are hashed, anchor on stable structures — the avatar-group region near the Share button, `aria-label`/`title`/tooltip text carrying display names, and `img` `src` for photo URLs — rather than `ms-Persona-*` literals. Verify selectors live in DevTools; they are undocumented and version-dependent.
- **Network path (more robust):** Capture the RTC WebSocket frames (SignalR messages) and parse presence/join/leave/cursor payloads — survives DOM redesigns better. You must first capture a live session to learn the schema (hub name, field names); not publicly documented.
- **Cadence:** Presence is push-based — subscribe/observe rather than poll. If polling the DOM, 2–5 s is ample and human-plausible; avoid sub-second polling.
- **Failure detection:** Watch for redirect to login.microsoftonline.com (session expiry/step-up), WebSocket disconnect/handshake failure (presence channel down), or the editor going read-only. On expiry, surface an interactive re-auth prompt to the human rather than auto-retrying.

**Illustrative launch config (Windows, Python, option 2 — stock Edge/Chromium persistent profile):**

```python
from playwright.sync_api import sync_playwright

USER_DATA = r"C:\m365observer\profile"   # persistent; survives runs

with sync_playwright() as p:
    ctx = p.chromium.launch_persistent_context(
        USER_DATA,
        channel="msedge",          # match enterprise-expected browser
        headless=False,            # headed; device/CA checks dislike headless
        args=["--start-maximized"],
        # NO stealth patches, NO foreign proxy: use the user's normal egress
    )
    page = ctx.pages[0] if ctx.pages else ctx.new_page()
    page.goto("https://www.microsoft365.com/")   # human completes MFA on first run
    # ... open the PowerPoint file ...

    # capture RTC presence frames:
    page.on("websocket", lambda ws: ws.on("framereceived",
            lambda f: handle_rtc_frame(f)))      # parse SignalR presence payloads

    # or read the Coauth Gallery inside the editor iframe:
    for fr in page.frames:
        if "officeapps.live.com" in fr.url or "cloud.microsoft" in fr.url:
            avatars = fr.locator("[role='img'][aria-label]")  # verify live in DevTools
            # extract aria-label (name), img src (photo), viewing/editing state
```

(For option 1, the equivalent is a `@match https://*.officeapps.live.com/*` userscript using a `MutationObserver` on the Coauth Gallery plus a `WebSocket.prototype` hook to read SignalR frames.)

## 6. Risks and unknowns

- **Exact RTC WebSocket URL/path and SignalR presence message schema** — NOT in public sources; requires a live HAR/DevTools capture. Hub name and presence field names unknown until captured.
- **Exact live editor DOM selectors / JS globals** — undocumented; current Fluent builds use hashed class names. Must be reverse-engineered live and will drift across releases.
- **Whether oid/email is exposed client-side** — unverified; evidence (ephemeral per-session coauthor GUIDs in Word's API) suggests Office may expose only display name + photo + initials + viewing/editing status + guest flag, not stable directory identifiers.
- **The specific tenant's conditional-access posture** — decisive and unknown. A compliant-device requirement or country allow-list flips the verdict from "viable via plain automation" to "userscript-in-real-session only" or "manual."
- **Session lifetime** — depends on the tenant's token/sign-in-frequency settings; not determinable without testing in the actual tenant.
- **Kazakhstan TLS-interception** — historically reported regionally; not confirmed active in 2026; flagged as a sign-in-anomaly risk to verify on the egress path.
- **Safe scale:** For feasibility, **1 file / 1 watcher on the user's own account is low-risk** via options 1–3. Keep to a single authenticated session per account, reuse one persistent profile, and do not parallelize sign-ins. Scaling to many files or many concurrent watcher sessions per account raises token-velocity and impossible-travel/anomalous-token risk; concurrency beyond a handful of watched files per account is not advisable without tenant-admin coordination. The thresholds that should change the plan: if the tenant enables compliant-device CA → drop to userscript-in-real-session (option 1); if Entra flags repeated step-up/risk detections on the watcher → stop automated sign-ins and switch to the hybrid/attach model (option 3) or manual observation.

---

### Bottom line

The Graph API genuinely cannot give you the live co-editor list, and that data IS reachable as a legitimate authenticated co-editor — over the RTC SignalR/WebSocket channel and the Coauth Gallery DOM inside the `officeapps.live.com` iframe. But Camoufox solves a problem Microsoft 365 doesn't pose (Cloudflare/DataDome fingerprinting) while creating the problems it does pose (unfamiliar browser/ASN/device against Entra risk scoring and conditional access). Prove viability with a userscript in your real, normally-authenticated session — or plain Playwright on a persistent Edge profile — not with Camoufox.