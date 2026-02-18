# Diagnosing slow Windows boot with WPR, WPA, and xbootmgr

**The fastest path to finding why a domain-joined Windows laptop takes forever to reach a usable desktop is a boot trace captured with WPR (or xbootmgr), analyzed in WPA, and cross-referenced with Group Policy operational logs.** Most enterprise boot delays fall into three buckets: synchronous Group Policy processing waiting on a slow network, security/DLP agents consuming I/O during service initialization, or blocking logon scripts and printer/drive mappings timing out against unreachable servers. A single boot trace viewed in WPA's Boot Phases and Services graphs will usually pinpoint the exact offender within minutes. This guide walks through every step — capture, analysis, diagnosis, and fix — for IT admins on Windows 10/11 Enterprise.

---

## 1. Capturing a boot trace with WPR

WPR ships built-in at `C:\Windows\System32\wpr.exe` on Windows 10 and 11 — no ADK installation required. It uses the Windows autologger mechanism to begin tracing at the earliest stage of the next boot. The workflow is three commands from an elevated prompt.

**Step 1 — Configure the autologger.** Run from an elevated command prompt:

```
wpr -addboot GeneralProfile -addboot CPU -addboot DiskIO -filemode -recordtempto C:\temp
```

The `-addboot` flag writes autologger registry entries so tracing starts automatically on the next boot. **`-filemode` is required** for boot traces (writes to disk rather than a circular memory buffer). The `-recordtempto` parameter sets a scratch directory for interim trace files. You can stack multiple profiles — `GeneralProfile` provides first-level triage data, `CPU` adds sampled and context-switch CPU data, and `DiskIO` captures storage activity. Other useful profiles include `Network`, `Registry`, and `Minifilter` (the last one is valuable for spotting antivirus filter driver overhead).

**Step 2 — Reboot the machine.** A normal restart is sufficient:

```
shutdown /r /t 0
```

Tracing begins automatically before the Windows kernel finishes initialization. It continues through the entire boot, logon, and post-boot stabilization period.

**Step 3 — Save the trace.** After logging in and reaching the desktop, wait roughly two minutes for post-boot activity to settle, then open another elevated prompt:

```
wpr -stopboot C:\temp\boottrace.etl
```

This stops tracing, removes the autologger registry entries, merges the kernel and user-mode buffers, and writes the final `.etl` file. If you need to cancel without saving, run `wpr -cancelboot`. The autologger entries persist across reboots until explicitly stopped or cancelled, so always clean up.

WPR also supports an automated on/off scenario mode that handles the reboot and post-boot collection in one command: `wpr -start GeneralProfile -start CPU -onoffscenario Boot -onoffresultspath C:\wpr -numiterations 1`. This triggers an immediate reboot and auto-saves the trace, but the autologger method above gives more control and is generally preferred for enterprise troubleshooting.

**Tip:** Configure auto-logon temporarily when capturing traces so credential entry time does not skew the Winlogon phase measurement. Boot trace `.etl` files can reach **1–3 GB** — ensure adequate disk space on the target drive.

---

## 2. Capturing with xbootmgr and when to use it instead

Xbootmgr is the older boot-trace tool from the Windows Performance Toolkit, installed via the Windows ADK (select "Windows Performance Toolkit" during setup). It still works on Windows 10 and 11 and continues to ship in current ADK releases, though Microsoft's documentation files it under "Previous Versions."

**Basic xbootmgr command:**

```
mkdir C:\Trace && cd C:\Trace
xbootmgr -trace boot -traceflags base+latency+dispatcher -stackwalk profile+cswitch+readythread -notraceflagsinfilename -postbootdelay 120 -resultPath C:\Trace
```

Unlike WPR, xbootmgr **reboots the machine immediately** when you execute the command — be ready. After reboot and logon, a countdown timer runs for the duration specified by `-postbootdelay` (default 120 seconds). When it expires, xbootmgr merges interim trace files into a single `.etl` in the result path. To abort a pending trace, run `xbootmgr -remove`.

The `-prepSystem` flag runs multiple automated reboot cycles (typically six) to optimize the Prefetch/ReadyBoot cache and defragment boot files. This was designed for HDDs in the Vista/Win7 era and **is not useful on SSDs**. Some users have reported BCD corruption with `-prepSystem` on Windows 10 — avoid it on modern hardware.

**When to choose xbootmgr over WPR.** Use WPR for day-to-day boot troubleshooting — it requires no installation, uses intuitive profile names, and is Microsoft's actively maintained tool. Use xbootmgr when a Microsoft support engineer specifically requests an xbootmgr trace, when you need explicit control over kernel trace flags and stackwalk filters, or when you want the fully automated reboot-and-capture workflow. Per Jeff Stokes (Microsoft-affiliated performance analyst), xbootmgr historically included certain boot-specific ETW providers by default that WPR's basic profiles may omit, which can matter when traces are reviewed by Microsoft CSS. Both tools produce `.etl` files that WPA opens identically.

---

## 3. Analyzing the boot trace in WPA

Open WPA (installed with the ADK or available standalone) and load the `.etl` via File → Open. **The critical step most admins miss**: you must apply a boot analysis profile to see boot-specific graphs. Go to **Profiles → Apply → Browse Catalog** and select **`FullBoot.Boot.wpaProfile`**. Without this profile, WPA will not display the Boot Phases timeline or Regions of Interest graphs that make boot analysis practical. After the profile loads, switch to the **Timeline** or **Deep Analysis** tab.

Configure symbols before deep analysis: go to Trace → Configure Symbol Paths and set `srv*C:\Symbols*https://msdl.microsoft.com/download/symbols`. Then Trace → Load Symbols. First-time symbol loading is slow but enables function-level call stack visibility.

### The five graphs that matter most

**Boot Phases** (under System Activity) displays a horizontal bar for each sequential boot phase with its duration. The phases are:

- **Pre Session Init** — UEFI/BIOS POST through OS loader and kernel initialization. Long times here point to firmware, driver, or ELAM issues.
- **Session Init** — Session Manager (SMSS) loads kernel subsystems and boot-start drivers. Video driver problems and security minifilter drivers (antivirus, DLP) loading show up here.
- **Winlogon Init** — The phase where **most enterprise delays live**. Service Control Manager starts auto-start services, machine Group Policy is applied, startup scripts run, the logon screen appears, user authenticates, user Group Policy is applied, logon scripts run, and the user profile loads. On a clean SSD image this takes under 15 seconds; on enterprise machines it routinely exceeds 60.
- **Explorer Init** — Desktop shell creation. Ends when the desktop is visible.
- **Post Boot** — Desktop is visible but startup apps and background services continue loading. The phase ends after the system accumulates **10 seconds of combined CPU and disk idle above 80%**.

Click any phase bar to highlight the corresponding time range across all other graphs, or right-click and "Zoom to Selection" to focus every view on that window.

**Regions of Interest** (System Activity) provides a hierarchical, expandable view within each boot phase. Expanding Boot-Winlogon-Phase reveals sub-regions for machine GP processing, user GP processing, profile load, and network wait. Hovering over any region bar shows its duration in a tooltip.

**Services** (System Activity) shows each auto-start service as a colored bar from start request to running state. **Sort by duration descending** to find slow-starting services. Because Windows starts auto-start services in dependency groups, a single slow service can block an entire group and cascade delays to everything downstream. In documented cases, the UevAgentService taking 5+ seconds held up a queue of services that immediately started once it finished.

**Process Lifetimes** (System Activity → Processes → Lifetime By Process) shows when each process starts and ends during boot. After zooming to the Winlogon phase, look for security agent processes (`CSFalconService.exe`, `ccSvcHst.exe`, `edpa.exe`, `MsMpEng.exe`) and their durations. Transient processes like `cmd.exe`, `powershell.exe`, or `cscript.exe` appearing during the GPClient window indicate logon scripts executing.

**Disk Usage** (Storage → Utilization by Disk) reveals whether disk I/O is the constraint. **100% disk utilization sustained during a boot phase is a clear bottleneck.** Sort by IO Time or Service Time per process to find the heaviest consumers. The **Mini-Filter Delays** sub-graph shows time added by each filesystem filter driver — this is where antivirus and DLP minifilter overhead becomes quantifiable.

### Identifying the worst offender

The workflow is consistent: identify the longest boot phase in Boot Phases → zoom into it → examine Services, CPU Usage (Sampled), and Disk Usage within that window → sort each by duration or weight descending → the top entries are your bottlenecks. For Winlogon-phase delays specifically, expand the Regions of Interest to separate GP processing from service startup from profile load, then cross-reference with Generic Events filtered to `Microsoft-Windows-GroupPolicy` to see per-CSE processing times.

---

## 4. Common enterprise culprits that inflate time-to-desktop

**Synchronous Group Policy processing** is the single most frequent cause of slow boot on domain-joined machines. When the GPO "Always wait for the network at computer startup and logon" is enabled — or when Folder Redirection, Roaming Profiles, or GP Software Installation are in use — Windows forces synchronous foreground processing. The machine waits for the full network stack (DHCP → DNS → NLA → DC discovery) before enumerating and applying every GPO. Each Client-Side Extension processes sequentially: Registry, Security, Scripts, Printers, Drive Maps, Folder Redirection, Software Installation. WMI filters on GPOs are especially expensive — every filter requires a WMI query evaluation. Loopback processing in Merge mode effectively doubles the evaluation scope.

**Network initialization delays** compound GP problems. Slow DHCP servers, 802.1X authentication on switch ports, Spanning Tree Protocol convergence delays (30–50 seconds on some enterprise switches), and NLA misidentifying the network as Public instead of DomainAuthenticated all push out the point at which GP processing can begin. If the machine cannot reach a domain controller, synchronous GP processing blocks until the **Startup Policy Processing Wait Time** expires (default **120 seconds**).

**Security agents and DLP products** affect boot at two levels. Kernel-mode drivers (CrowdStrike's `csagent.sys`, Symantec's minifilter drivers, Microsoft Defender's `WdFilter.sys` and `Wdboot.sys` ELAM driver) load during Session Init and evaluate or scan every subsequent driver and file operation. User-mode services (`CSFalconService.exe`, `ccSvcHst.exe`, `edpa.exe`, `dgagent.exe`) start during Winlogon Init and compete for CPU and disk I/O with Group Policy and other services. CrowdStrike's ELAM driver evaluates all boot-start drivers, adding per-driver overhead. Real-time AV scanning during boot creates an I/O storm as hundreds of executables and DLLs load simultaneously.

**Logon scripts** remain a persistent drag. PowerShell scripts incur a **~4.5-second cold-start penalty** for .NET library loading before a single line executes — compared to ~0.3 seconds for a batch file. Scripts accessing unreachable network shares hang until timeout (default **600 seconds** for VBScript). Synchronous logon script processing blocks desktop appearance entirely.

**Mapped drives and printers** to unreachable servers each add roughly **20 seconds of timeout**. Multiple broken UNC paths compound linearly. Certificate revocation list (CRL) checks against unreachable distribution points add **15–30 seconds per check**. VPN clients establishing machine tunnels, SCCM/MECM client policy evaluation, and BitLocker PCR validation on configuration changes all contribute additional seconds.

---

## 5. ETW providers and events that pinpoint the delay type

Distinguishing whether a boot delay is network-related, GP-related, or security-agent-related requires filtering WPA's Generic Events graph to specific ETW providers.

**For Group Policy delays**, filter to `Microsoft-Windows-GroupPolicy` (GUID `{AEA1B4FA-97D1-45F2-A64C-4D69FFFD92C9}`). **Event ID 4016** marks the start of each CSE's processing and includes the CSE name and applicable GPO list. **Event ID 5016** marks completion and contains `CSEElaspedTimeInMilliSeconds` — pair these to measure each CSE's duration. **Event ID 8000** reports total computer GP processing time at boot; **Event ID 8001** reports total user GP processing time at logon. Event IDs **5312** and **5317** list applied and filtered GPOs respectively. In Event Viewer's System log, **Event ID 6006** from Winlogon ("The winlogon notification subscriber GPClient took X seconds to handle the notification event CreateSession") is a direct indicator that GP caused the delay.

Key CSE GUIDs to watch in 4016/5016 events: `{42B5FAAE-6536-11D2-AE5A-0000F87571E3}` (Scripts), `{5794DAFD-BE60-433F-88A2-1A31939AC01F}` (Drive Maps), `{BC75B1ED-5833-4858-9BB8-CBF0B166DF9D}` (Printers), `{A2E30F80-D7DE-11D2-BBDE-00C04F86AE3B}` (Folder Redirection), `{827D319E-6EAC-11D2-A4EA-00C04F79F83A}` (Security).

**For network delays**, filter to `Microsoft-Windows-DHCP-Client` (`{15A7A4F8-0072-4EAB-ABAD-F98A4D666AEB}`) — look for large gaps between boot start and DHCP lease acquisition events like `NLANotified` (opcode 53) and `PerfTrackGatewayReachable` (opcode 60). Filter to `Microsoft-Windows-NetworkProfile` (`{1B1A3F95-2F19-4AD3-B049-7B18984B0243}`) for NLA — watch for `NetworkCategoryChanged` events showing delayed transition to DomainAuthenticated. `Microsoft-Windows-DNS-Client` (`{1C95126E-7EEA-49A9-A3FE-A378B03DDB4D}`) reveals DC discovery delays. `Microsoft-Windows-NCSI` (`{314DE49F-CE63-4779-BAD7-2D4391DA2643}`) shows when connectivity is confirmed.

**For security/DLP agent delays**, the approach is process-centric rather than provider-centric. In the Services graph sorted by duration, look for: `CSFalconService` (CrowdStrike), `ccSvcHst` and `Smc` (Symantec SEP), `EdpaService` (Symantec DLP), `FPDLPService` (Forcepoint), `DGService` (Digital Guardian), `SentinelAgent` and `SentinelServiceHost` (SentinelOne), `MsMpEng` (Defender). In Disk Usage, filter by process to see I/O attributed to these agents. The Mini-Filter Delays graph directly quantifies overhead from filter drivers like `WdFilter.sys`, `csagent.sys`, and vendor-specific minifilters. In Session Init, check `Microsoft-Windows-Kernel-PnP` events for boot-start driver load times — ELAM and security minifilter drivers appear here.

**For service startup delays**, `Microsoft-Windows-Services` (`{0063715B-EEDA-4007-9429-AD526F62696E}`) provides SCM events for every service start, stop, and failure. The WPA Services graph derived from this provider is the most efficient view. For Winlogon/shell delays, filter to `Microsoft-Windows-Winlogon` (`{DBE9B383-7CF3-4331-91CC-A3CB16A3B538}`) and watch for the `Display Welcome Screen` and `Request Credential` task events. `Microsoft-Windows-Shell-Core` (`{30336ED4-E327-447C-9DE0-51B652C86108}`) marks Explorer Init boundaries.

---

## 6. Fixing the bottleneck once you find it

### Eliminating synchronous Group Policy waits

The single highest-impact change for most enterprises is disabling synchronous foreground GP processing. Set the GPO at **Computer Configuration → Administrative Templates → System → Logon → "Always wait for the network at computer startup and logon"** to **Disabled**. This writes `SyncForegroundPolicy = 0` (REG_DWORD) at `HKLM\SOFTWARE\Policies\Microsoft\Windows NT\CurrentVersion\Winlogon`. With this disabled, Windows uses cached GP data and applies updates asynchronously in the background — the user reaches the desktop without waiting for network or DC contact.

Disable this unless your environment absolutely requires first-logon application of Folder Redirection, GP Software Installation, or Roaming Profiles, which inherently force synchronous processing. Even with those features, consider whether the tradeoff is worthwhile — per Microsoft's Mark Renoden, "it's preferable to avoid synchronous Group Policy processing" and organizations should migrate away from the technologies that trigger it.

Reduce the startup policy wait time via **Computer Configuration → Administrative Templates → System → Group Policy → "Startup policy processing wait time"** — the default is **120 seconds**. Lowering this reduces the maximum wait if the network is slow but risks GP Software Installation failing on slow links.

Run `gpresult /h report.html` on affected machines. The HTML report's Component Status section shows **processing time per CSE**, immediately revealing which extensions are slow. In Event Viewer under `Microsoft-Windows-GroupPolicy/Operational`, check Event IDs 8000/8001 for total processing time and 4016/5016 pairs for per-CSE timing. Consolidate many small GPOs into fewer comprehensive ones, disable unused configuration halves (Computer or User) on each GPO, eliminate WMI filters wherever possible (use Item-Level Targeting in GP Preferences instead), and remove cross-domain GPO links.

### Tuning network wait and desktop switch timing

Set `DelayedDesktopSwitchTimeout` to **0** at `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System` (REG_DWORD). This controls how long Windows displays the "Preparing Windows" screen before switching to the desktop. The default can impose up to 30 seconds of wait. Setting it to 0 makes the desktop appear as soon as possible. Deploy via GP Preferences Registry item.

For logon scripts, set **"Run logon scripts synchronously"** to **Disabled** (User Configuration → Administrative Templates → System → Scripts). Configure the **"Configure Logon Script Delay"** GPO (Computer Configuration → Administrative Templates → System → Group Policy) to defer script execution until after the desktop appears.

### Reducing security agent boot impact

Change non-critical security service components from Automatic to **Automatic (Delayed Start)**: `sc.exe config "ServiceName" start= delayed-auto`. This defers their startup until after all Automatic services complete. The global delayed-start timer defaults to 120 seconds and can be adjusted at `HKLM\SYSTEM\CurrentControlSet\Control\AutoStartDelay` (REG_DWORD, value in seconds).

For kernel-mode boot-start drivers (ELAM, minifilters), changes require vendor coordination — these cannot simply be delayed without breaking the security product. Work with vendor TAMs armed with your WPA trace data showing the specific impact. Exclude boot-critical paths from DLP real-time scanning: `C:\Windows\System32\`, `C:\Windows\Prefetch\`, pagefile, and hibernation files. For CrowdStrike, review the sensor update adoption cadence — some sensor versions have caused documented boot degradation.

### Fast Startup considerations for enterprise

Fast Startup (hybrid shutdown) resumes the kernel from hibernation rather than performing a full boot. **Computer startup Group Policies are not processed** on Fast Startup resume, nor are startup scripts executed. This creates security policy drift. The setting lives at `HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Power\HiberbootEnabled` (1 = on, 0 = off). **Disable Fast Startup on domain-joined machines** where GP compliance matters by setting this to 0 via GP Preferences or by running `powercfg /h off`. If boot speed is critical and GP drift is acceptable, keep it enabled but schedule weekly full restarts (via SCCM maintenance windows or scheduled tasks) to force GP processing.

### Essential diagnostic checks outside WPA

Run these commands on any machine with slow boot before or alongside trace analysis:

```
gpresult /h C:\temp\gpreport.html          — GP processing time per CSE
eventvwr.msc → Applications and Services Logs → Microsoft → Windows →
    Diagnostics-Performance → Operational  — Event ID 100 (total boot duration),
    101 (slow app), 102 (slow driver), 103 (slow service)
eventvwr.msc → Microsoft → Windows → GroupPolicy → Operational  
    — Event IDs 8000/8001 (total GP time), 4016/5016 (per-CSE timing)
bcdedit /enum                              — Boot configuration review
systeminfo | find "Boot Time"              — Last boot timestamp
```

Event ID **100** in the Diagnostics-Performance log contains `BootDuration`, `MainPathBootTime`, and `BootPostBootTime` in milliseconds — a quick baseline without WPA. Event IDs 101–110 identify specific degradation sources (applications, drivers, services, machine/user policy).

### Additional quick wins

Disable the first sign-in animation ("Hi, we're getting things ready") via `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\EnableFirstLogonAnimation = 0` (REG_DWORD). Eliminate the startup app delay on SSD machines by creating `HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\Serialize\StartupDelayInMSec = 0` (REG_DWORD). Replace logon scripts with GP Preferences for drive mappings, registry settings, and shortcuts — a batch script takes ~0.3 seconds to start while PowerShell takes ~4.5 seconds just to load .NET. Redirect root certificate auto-update to an internal distribution point via `certutil -syncWithWU \\server\share` if machines lack reliable internet at boot. Configure SCCM maintenance windows to prevent client policy evaluation and mandatory deployments from overlapping with business-hours boot times. Enable verbose boot status messages for visual troubleshooting via **Computer Configuration → Policies → Administrative Templates → System → "Display highly detailed status messages"** = Enabled.

---

## Conclusion

The diagnostic workflow is straightforward once internalized: capture with `wpr -addboot`, reboot, save with `wpr -stopboot`, open in WPA with the `FullBoot.Boot.wpaProfile` applied, and identify the longest phase. On enterprise machines, **Winlogon Init is almost always the culprit**, inflated by synchronous GP processing, slow security agent services, or network initialization delays. The three highest-impact fixes — disabling synchronous GP processing, setting `DelayedDesktopSwitchTimeout` to 0, and moving non-critical security services to Delayed Auto-Start — typically cut 30–60 seconds from boot on machines that previously exceeded two minutes. The ETW providers and event IDs documented above let you distinguish definitively between GP, network, and security-agent causes rather than guessing. Every registry path and GPO setting in this guide can be deployed fleet-wide through Group Policy Preferences or your configuration management platform, making remediation as scalable as the diagnosis.