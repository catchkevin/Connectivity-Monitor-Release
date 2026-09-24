# ConnectivityOverlay

## Project Overview

ConnectivityOverlay is a lightweight Windows desktop connectivity-monitoring utility designed to remain available on the desktop while continuously showing the health of the workstation's local and upstream network path.

The application is intentionally different from a full network-monitoring platform. It does not require a browser dashboard, a PowerShell window, a command prompt, or a large monitoring console to remain open. Instead, it provides a small always-on-top Windows overlay, a notification-area icon, an optional compact view, optional graphing, optional logging, and a richer set of network/Wi-Fi troubleshooting details when those details are needed.

ConnectivityOverlay is currently implemented in C# with .NET 8, WPF, and limited Windows Forms interoperability for the notification-area icon. The application targets Windows x64 and is configured to publish as a self-contained single-file executable.

Current application version in the project file:

```text
2.3.0
```

Current target framework:

```text
net8.0-windows
```

Current deployment target:

```text
win-x64
```

---

# 1. What ConnectivityOverlay Is Intended to Answer

ConnectivityOverlay is designed to help a user answer common connectivity questions quickly:

- Is my local default gateway reachable?
- Is latency to the local gateway normal?
- Is the problem only inside the local LAN/Wi-Fi path?
- Is the first publicly routable hop reachable?
- Is a known external DNS endpoint reachable?
- Are one or more custom destinations reachable?
- Is packet loss occurring?
- Did the active network adapter change?
- Did my local IPv4 address change?
- Did my public/egress IPv4 address change?
- Did I connect or disconnect a VPN?
- Did my Wi-Fi SSID change?
- Did my Wi-Fi BSSID/access point change?
- Did the workstation roam between wireless access points?
- What Wi-Fi signal quality is Windows reporting?
- Did the Wi-Fi signal move into a different quality/color band?
- When did the current access-point association first appear to this running application?
- Can I see recent Wi-Fi activity without opening Event Viewer?
- Can I graph only one or two targets while the other targets continue to be pinged?
- Can I retain up to 24 hours of monitoring history without drawing every sample directly on screen?
- Can I log network transitions and ping measurements for later troubleshooting?
- Where exactly along the path does a specific target stop responding?
- What is my current upload/download throughput, without opening a browser?

The application is built around the idea that a connectivity problem is easier to isolate when the troubleshooting chain is visible in one place:

```text
Active adapter / local IP
        ↓
Wi-Fi SSID / BSSID / AP / signal
        ↓
Egress/public IP
        ↓
Default gateway
        ↓
1st public hop
        ↓
DNS / custom targets
        ↓
Graph history / activity / logs
```

---

# 2. Primary Design Goals

## 2.1 Always-available status

The main overlay is designed to stay near the Windows taskbar and remain visible without consuming a large amount of desktop space.

## 2.2 Small by default, detailed when needed

The normal view can expose detailed information, but most features are optional. Compact mode reduces the application even further.

## 2.3 Local-versus-upstream isolation

GW, 1st Pub Hop, DNS, and custom targets create a simple hierarchy for determining how far traffic can travel.

## 2.4 Dynamic network awareness

The application does not assume that the gateway, adapter, IP address, VPN path, Wi-Fi association, or egress IP discovered at startup will stay the same.

## 2.5 Historical context

Ping samples, packet-loss calculations, Wi-Fi activity, optional log files, and graphs provide context that a single ping command cannot.

## 2.6 Safe supplemental telemetry

Wi-Fi telemetry, egress-IP lookup, and WLAN event collection are supplemental. A failure to read those sources must not stop the core ping monitor.

## 2.7 Simple deployment

The intended release artifact is a self-contained single Windows executable. End users should not need a separate .NET runtime or development tools.

---

# 3. Core Monitored Targets

ConnectivityOverlay supports up to fifteen ping targets: 3 auto-detected (GW, 1st Pub Hop, DNS),
up to 7 read from an external CSV file, and up to 5 fully custom targets entered directly in
Settings.

The intended display order is:

1. GW
2. 1st Pub Hop
3. DNS
4. CsvTarget1-7 (from the configured CSV file, in file order)
5. Custom1-5

## 3.1 GW

`GW` is the active IPv4 default gateway.

Important behavior:

- GW is always enabled.
- GW is always first.
- GW cannot be disabled.
- The application periodically reevaluates the active route.
- The gateway can therefore change while the application is running.
- The associated default-route adapter is also tracked.
- A gateway change triggers related network-path refresh behavior.

The gateway is the most useful first check when determining whether a problem is local.

## 3.2 1st Pub Hop

`1st Pub Hop` is the first publicly routable IPv4 hop discovered by traceroute-style TTL-limited ICMP probing.

Important behavior:

- Enabled by default.
- Can be disabled.
- Automatically discovered.
- Rediscovered periodically.
- Rediscovered after route/gateway changes.
- Default probe destination is `8.8.8.8`.
- Maximum discovery depth is 15 hops.
- Per-hop discovery timeout is 700 ms.
- Private, loopback, link-local, CGNAT, multicast, and reserved addresses are excluded from public-hop selection.

The first public hop is useful for distinguishing a local-router problem from a problem that exists farther upstream.

A missing 1st Pub Hop does not automatically mean the Internet is unavailable. Some routers and providers suppress the ICMP responses required for traceroute-style discovery.

## 3.3 DNS

DNS is auto-detected from the active network adapter's configured DNS server, the same way `GW` and `1st Pub Hop` are auto-detected — it is not a fixed, user-entered address. The resolved server's hostname is reverse-looked-up automatically and shown alongside the IP.

DNS is enabled by default and can be disabled, but its address is not manually editable.

Important clarification: the current DNS target test is an ICMP ping to the resolved endpoint. It is not a DNS-query transaction test.

## 3.4 CSV-Driven Targets (CsvTarget1-7)

Up to 7 additional targets can be defined in an external CSV file rather than typed into
Settings directly. The file path is set in Settings > Target Settings (default location
`%APPDATA%\ConnectivityOverlay\default_targets.csv`); the file itself is a header row followed
by up to 7 rows of `Name,Type,Value`, for example:

```text
googleDns,ip,8.8.8.8
googleDns,url,dns.google.com
```

Settings shows a read-only display grid of whatever the CSV currently contains, with a per-row
enable/disable checkbox — the Name/Type/Value themselves are edited in the CSV file, not in the
app. Like every other target, each resolves the IP-or-URL side it wasn't given automatically via
DNS lookup and caches it.

## 3.5 Custom Targets

Five custom targets (Custom1-5) are available directly in Settings.

Each target has:

- Enable/disable control.
- User-defined display name.
- A mode switch (`IP` / `URL`) plus one value box — the other side is resolved automatically via DNS lookup and cached, the same mechanism used by the CSV-driven targets above.
- Independent graph color.
- Independent graph visibility checkbox.

Custom and CSV-driven targets are useful for:

- Corporate gateways.
- VPN endpoints.
- Internal servers.
- Firewalls.
- Cloud endpoints.
- Branch-office routers.
- SaaS dependencies.
- Any frequently used host that responds to ICMP.

## 3.6 Traceroute and Ping Output

Every target row (including GW, 1st Pub Hop, and DNS) has two small buttons: `T` opens a live
traceroute to that target, and `P` opens a live view of that target's last 5-30 (configurable in
Settings > Target Settings, default 30) continuous ping results — both in a separate small window
positioned near the main overlay (Traceroute windows to the left, Ping Output windows below
those). Traceroute hops render as they resolve rather than after the whole route finishes; Ping
Output has no run/refresh action since it's just reading the ping engine that's already running
continuously in the background. Up to 5 Traceroute windows and 5 Ping Output windows can be open
at once; opening one more than the cap closes the oldest of that same kind. Traceroute windows
have Copy All, Write to Disk (never the main application log), Refresh (disabled while a trace is
running), and Cancel (distinct from the window's own close button). Ping Output windows have just
Copy All and Write to Disk, since there's no action to run or cancel.

---

# 4. Ping Timing and History

Default ping interval:

```text
1000 ms / 1 second
```

The configured ping interval applies to the active ping targets.

The application preserves actual ping samples for up to:

```text
24 hours
```

The graph duration can be selected independently from the monitoring interval.

Quick graph-duration presets include:

```text
1 minute
5 minutes
10 minutes
15 minutes
30 minutes
1 hour
4 hours
8 hours
12 hours
24 hours
```

A custom duration can also be entered using Hours and Minutes.

The packet-loss percentage uses the selected graph-duration window. That means changing the graph-duration setting also changes the rolling period used for packet-loss calculations.

The application does not silently convert a selected 1-minute graph to another duration. The selected duration remains the selected duration.

---

# 5. Latency Health Model

The Primary latency profile defaults are:

| Range | Status | Color |
|---|---|---|
| 0–30 ms | Good | Green |
| 31–60 ms | Fair | Yellow |
| 61–100 ms | Elevated | Pink |
| 101–175 ms | High | Orange |
| 176–250 ms | Severe | Brown/Peru |
| 251 ms and above | Critical | Purple |
| Timeout | Unreachable / Failure | Red |

These thresholds are configurable.

The five configurable maximum values are:

- Good max.
- Fair max.
- Elevated max.
- High max.
- Severe max.

Critical automatically means values above the Severe maximum.

Threshold validation requires values to be:

- Greater than zero.
- Strictly increasing.
- Severe no greater than 60,000 ms.

The effective ping timeout extends above the configured Severe threshold so that the Critical range can still be measured instead of being converted immediately into a timeout.

---

# 6. Advanced Latency Profiles

ConnectivityOverlay supports optional Advanced Latency Settings, **enabled by default**.

When Advanced mode is disabled:

- GW, 1st Pub Hop, and DNS all use Primary, exactly like every other target.
- The single Primary legend is shown once, above the header row.

When Advanced mode is enabled (the default):

- GW, 1st Pub Hop, and DNS can each use their own independent thresholds. Out of the box these are tuned tighter than Primary (Good/Fair/Elevated/High/Severe of 8/10/12/15/20 ms for GW, 10/15/20/25/30 ms for 1st Pub Hop, 25/30/35/40/45 ms for DNS), since those targets are expected to respond faster than a general external destination.
- Every CSV and Custom target continues to use Primary.
- GW/1st Pub Hop/DNS each get their own inline legend directly above their own row whenever their thresholds actually differ from Primary; the shared Primary legend moves down to sit above the custom-target section instead of the header row, since Primary is what governs that section. A full copy of the currently-in-effect legend (Primary plus any Advanced overrides) is always available as a static reference on the Legends tab in Settings.

The first time Advanced mode is enabled on a settings file that has never used it, the current Primary thresholds are copied into the Advanced profiles, so turning the feature on does not change colors unexpectedly. A fresh install starts with Advanced mode already on and pre-seeded with the tuned values above, not a copy of Primary.

Controls include:

- Copy Primary to Advanced.
- Restore Advanced Defaults.
- Restore Primary Defaults.

"Restore Defaults" in both cases now resets to the same tuned baseline described above and in `docs/WORKING_LOG.md`, not the original conservative factory numbers from earlier releases.

---

# 7. Normal Display

The normal display is intentionally configurable.

The base target rows can contain:

```text
Destination | URL | IP ADDRESS | LATENCY
```

The URL column shows each target's hostname when it has one (blank for GW/1st Pub Hop, which are IP-only auto-detected targets). A small `T` (Traceroute) button sits at the right edge of every row — see §3.6.

Optional columns:

- Field headers.
- Packet loss.
- Adapter.
- Adapter Description.

Example:

```text
Destination | URL             | IP ADDRESS  | LATENCY | PACKET LOSS    | ADAPTER | ADAPTER DESCRIPTION
GW          |                 | 192.168.1.1 | 6 ms    | Pk_Loss 00.00% | Wi-Fi   | Intel(R) Wi-Fi 7 BE200
1st Pub Hop |                 | 203.0.113.1 | 15 ms   | Pk_Loss 00.00% | Wi-Fi   | Intel(R) Wi-Fi 7 BE200
DNS         | dns.example.com | 8.8.8.8     | 25 ms   | Pk_Loss 00.00% | Wi-Fi   | Intel(R) Wi-Fi 7 BE200
```

The table itself is built from a real WPF `Grid` with `SharedSizeGroup` columns rather than manually padded text, so columns stay pixel-aligned regardless of content length or font rendering. The target-area width is measured dynamically so the main window can remain as narrow as practical for the visible content, while still growing wide enough for the longest visible line — the window grows leftward rather than off the right edge of the screen, since it's typically docked near the right side.

---

# 8. Network and IP Display

Current network/Wi-Fi display defaults are:

```text
Show Network Adapter(s) = ON
Show Bluetooth Info     = OFF
Show My Egress IP       = ON
Show My IP              = ON
Show Wi-Fi Info (SSID, AP) = ON
Show Wi-Fi Signal       = ON
Show Wi-Fi Logs (Adapter Activity) = ON
```

These settings affect screen visibility only.

If application logging is enabled, important network and Wi-Fi changes can still be logged even when the corresponding display option is hidden.

## 8.1 Network adapters

When enabled, the application can display active IPv4-capable adapters that appear relevant to current connectivity.

The application favors:

- Ethernet.
- Gigabit Ethernet.
- Wireless 802.11.
- PPP.
- Tunnel interfaces.
- Interfaces whose name or description appears VPN/TAP/TUN-related.

Loopback interfaces and interfaces without usable IPv4 addresses are excluded.

The active default-route adapter is marked as such.

## 8.2 My IP

`My IP` represents the local IPv4 address Windows would use for the current Internet route.

The application uses Windows route selection rather than simply choosing the first adapter.

## 8.3 Egress IP

`Egress IP` is the public IPv4 address observed through an external HTTPS lookup.

Current lookup providers are attempted in order:

```text
https://api.ipify.org
https://checkip.amazonaws.com
```

The HTTP timeout is five seconds.

Egress-IP lookup is supplemental. Failure to reach an external IP service does not stop ping monitoring.

When the observed egress IP changes, the UI can show the current value and change time, and the change can be logged.

---

# 9. Wi-Fi Telemetry

Wi-Fi state is collected using the Windows Native Wi-Fi API through `wlanapi.dll`.

The application retrieves:

- Connected Wi-Fi interface.
- SSID.
- BSSID.
- Windows signal-quality percentage.
- Wi-Fi interface description.

BSSID is not shown as its own field on screen - it's used internally to resolve the AP Name
(below) and to detect AP roaming - but it's still fully retrieved and available in Diagnostics.
It takes the form of a MAC address, for example:

```text
2E:B9:BE:F3:CC:9A
```

## 9.1 AP friendly name

Windows typically provides the BSSID but does not provide a user-friendly mesh/AP label such as `Office AP` or `Upstairs AP`.

ConnectivityOverlay therefore supports user-defined BSSID-to-name aliases.

Alias syntax:

```text
BSSID=Friendly Name
```

Example:

```text
8C:3B:AD:12:34:56=Office AP
8C:3B:AD:12:34:78=Upstairs AP
```

In addition to this per-user alias box, Settings has an "AP Information .csv
File Location" field pointing at an optional shared CSV lookup - useful for
a team/company-wide AP name list maintained by one person and referenced by
everyone (a local path, network share, or a synced cloud folder such as
OneDrive/SharePoint/Box/Dropbox all work, since those all mirror to a local
folder). The CSV has a header row followed by one row per AP:

```text
AP_Name,Mac_Address
Office AP,8C:3B:AD:12:34:56
```

A blank starter file is included alongside the application at
`template_csv/ap_list.csv`. A BSSID is looked up in the alias box first;
the CSV file is only checked as a fallback, so a personal alias always
overrides the shared file for the same BSSID.

If no alias or CSV entry is found, AP Name is shown as:

```text
Unknown
```

Note: on some enterprise/controller-based Wi-Fi networks, the BSSID Windows
reports as "currently connected" is a per-client virtualized value assigned
for fast roaming and never appears in a passive scan of broadcast BSSIDs.
On such networks, alias/CSV entries keyed by a scanned BSSID will not match
your active connection's reported BSSID, and AP Name may show "Unknown" for
your own connection even with a correct alias/CSV entry in place - this is
a network behavior, not an application defect.

## 9.2 Wi-Fi connected/AP timestamp

The application tracks when the current BSSID association was first observed during the running session.

A BSSID change is treated as an AP roam and resets the observed-current-AP timestamp.

This is an application-observed connection time, not a claim that Windows exposes a perfect historical AP-association timestamp for every scenario.

---

# 10. Wi-Fi Signal and Colors

Windows Native Wi-Fi reports signal quality as a value from 0 through 100.

ConnectivityOverlay also displays an RSSI-style dBm value using the standard Windows quality approximation:

```text
RSSI ≈ (SignalQuality / 2) - 100
```

This means:

- Signal Quality (%) is the Windows-reported value.
- Displayed RSSI dBm is an approximation derived from that value.

Wi-Fi signal-status ranges:

| RSSI | Status | Text Color |
|---|---|---|
| -30 to -49 dBm | Strong | White |
| -50 to -59 dBm | Excellent | White |
| -60 to -66 dBm | Good | Green |
| -67 to -69 dBm | Good - Voice/Video | Green |
| -70 to -79 dBm | Fair | Orange |
| -80 to -89 dBm | Weak | Pink |
| -90 dBm or worse | Very Poor / Unusable | Red |

The user-facing shorthand color bands are:

```text
0–59 absolute dBm magnitude  = White
60–69                       = Green
70–79                       = Orange
80–89                       = Pink
90+                         = Red
```

Because real RSSI values are negative, the actual code evaluates ranges such as `-60`, `-70`, and `-90`.

Signal-band change events are logged only when the signal crosses a defined band boundary. Normal small fluctuations inside the same band do not create a new signal-band-change event.

Examples:

```text
-58 → -61 : logs White → Green
-61 → -64 : no band-change log
-78 → -82 : logs Orange → Pink
```

The actual colors used to paint the Wi-Fi Signal text/swatches on screen (a 5-tier simplification
of the table above: Strong/Good/Fair/Weak/Poor) are user-editable in Settings > Adapter Settings
> RSSI Range Color, rather than fixed. A live copy of whatever is currently configured is always
shown as a static reference on the Legends tab in Settings, alongside the ping/latency legend.

---

# 11. Wi-Fi Activity Section

The Wi-Fi Activity section provides a compact recent-history view inside the application.

It can show:

- Connected.
- Disconnected.
- Connection failure.
- SSID changes.
- AP/BSSID roaming.
- Signal-band changes.
- Meaningful Windows WLAN AutoConfig activity.

The current display uses aligned auto-sized columns so the section remains compact while preserving readable column boundaries.

Columns:

```text
DATE / TIME | STATUS | DETAILS | AP NAME | MAC ADDRESS | SIGNAL | SIGNAL STATUS
```

Example:

```text
09/13/2026 06:48:02 PM | Signal    | White → Green (-60 dBm) | Office AP | 2E:B9:BE:F3:CC:9A | -60 dBm / 81% | Good (Green)
09/13/2026 06:46:07 PM | Connected | SpectrumSetup-CC96      | Unknown   | 2E:B9:BE:F3:CC:9A | -63 dBm / 75% | Good (Green)
```

The activity list is newest-first.

Settings:

```text
Minimum lines default = 5
Maximum lines default = 20
```

The main UI includes `Show More`.

When `Show More` is unchecked, the minimum-line count is used.

When checked, the activity panel can show up to the configured maximum.

The application keeps a larger in-memory activity buffer than the visible list so display min/max can change without losing the most recent session events.

---

# 12. Windows WLAN AutoConfig Events

The application also reads selected events from:

```text
Microsoft-Windows-WLAN-AutoConfig/Operational
```

Current normalized event IDs:

```text
8001 = Connected
8002 = Connection Failed
8003 = Disconnected
```

The current implementation calls `wevtutil.exe` to retrieve recent meaningful entries and normalizes those entries before adding them to the activity stream.

Live Native Wi-Fi state remains the primary source for current SSID/BSSID/AP roaming state.

Event-log collection is supplemental. If the WLAN event log is unavailable, the rest of monitoring continues.

---

# 13. Compact Mode

Compact mode is designed to provide a very small always-on-top view.

Every visible line is rendered as a bordered, right-justified label/value row — like a small table — rather than loosely stacked text, so everything lines up regardless of content length.

When the corresponding display settings are enabled, compact content is ordered as:

```text
Adapter (bare, e.g. "Wi-Fi")
----------------
My IP:      <ip>
Egress:     <ip>
SSID:       <ssid>
Signal:     <percent>%   <status, e.g. "(Good)">
----------------
Gateway IP: <ip>          <latency>
1st Pub Hop: <ip>         <latency>
DNS:        <hostname or ip>  <ip>  <latency>
CsvTarget1-7: <name>      <ip or url>  <latency>
----------------
Custom1-5 (same 3-column layout, using each target's own name/URL)
----------------
  [Copy]  [^^]
```

The `^^` button expands the application back to normal mode. The `Copy` button next to it copies every currently-visible Compact line, in display order, to the clipboard as plain text.

Important behavior:

- Compact honors the same visibility preferences as normal mode.
- Every target line shows its URL (or IP/name, for auto-detected targets with no URL) plus current latency; nothing wraps, and the window sizes itself to whatever the widest visible line requires.
- Wi-Fi signal shows Windows' signal-quality percentage plus a status word (e.g. "Good"); RSSI in dBm was removed from Compact specifically (it remains available in the normal view).
- Enabled ping-target lines remain color-coded according to their current latency-health category.
- Wi-Fi Activity history and full legends are intentionally excluded from Compact because they are reference/history sections rather than compact status values.
- Compact/normal state is saved and restored.

If no Wi-Fi connection is active, Wi-Fi-specific lines naturally disappear.

---

# 14. Graphing

Graphing is optional and disabled by default.

There are two graph surfaces:

1. Embedded graph in the normal main window.
2. Pop Graph / expanded graph window.

Both use the same stored ping history. Opening the expanded graph does not create a second ping process.

## 14.1 Per-target graph visibility

Every target has a graph visibility checkbox.

This allows the user to display only the series needed for troubleshooting.

Example:

```text
[x] GW
[ ] 1st Pub Hop
[ ] DNS
[ ] Custom1
```

Unchecking a graph checkbox does not:

- Stop pinging.
- Stop logging.
- Stop packet-loss tracking.
- Disable the target.

It only controls whether the target's graph line is drawn.

This is particularly useful when a single line needs to be inspected without visual overlap from other targets.

## 14.2 Rendering and downsampling

The application retains actual ping samples for up to 24 hours.

The graph renderer downsamples for display according to available width and preserves important points such as peaks and timeouts.

This avoids drawing hundreds of thousands of individual points on screen while retaining the actual samples for calculations and logging.

## 14.3 Graph threshold lines

Graph threshold lines reflect the active latency profiles.

If Advanced Latency Settings are active, GW, 1st Pub Hop, and DNS can therefore contribute different threshold profiles.

Identical threshold lines are combined rather than needlessly duplicated.

---

# 15. Speed Test

ConnectivityOverlay can run periodic download/upload speed tests using Cloudflare's public speed-test endpoints.

- Enabled by default, on a 60-minute interval.
- Runs once automatically on launch (when enabled), instead of waiting a full interval for the first result — the recurring schedule then continues on the configured interval from that point.
- Results and a short rolling history are shown on the main screen and included in that section's Copy All / Write to Disk output.
- A **Run Once** button runs a test immediately, regardless of the enabled/disabled setting, without turning the recurring setting on — usable any time, including between scheduled runs.
- The interval can be changed at any time; changing it while a test is already scheduled reliably takes effect immediately rather than waiting for a restart.

---

# 16. Notification-Area / Tray Behavior

ConnectivityOverlay creates a Windows notification-area icon.

The tray icon reflects the worst current latency-health category among enabled targets.

The application compares health severity rather than simply choosing the numerically highest latency because Advanced Latency Settings can give different targets different threshold profiles.

Tray notifications can be used for packet-loss and recovery events.

Windows itself controls whether an application icon appears directly on the taskbar notification area or inside the overflow menu. The application cannot reliably force its permanent taskbar placement.

In addition to the tray icon, the application also shows a regular taskbar icon while it's open
(in any view mode); this is separate from the tray icon above and behaves like a normal window's
taskbar entry. No custom icon image exists yet, so it currently uses a default one.

---

# 17. Window Controls

The normal toolbar includes:

- Settings
- Info
- Compact
- Extra Compact
- Pop Graph, when graphing is enabled
- Minimize to Tray
- Exit

## Settings

Opens application configuration.

## Info

Opens the detailed tabbed information/reference window.

Current Info tabs:

```text
Overview
Refresh Rates
Wi-Fi Signal
Wi-Fi Activity
Adapters
Logging
Graph
Compact
Troubleshooting
```

The selected Info tab uses a blue background with white bold text so it remains readable on the dark UI.

## Compact

Switches to the compact status view.

## Extra Compact

Switches to an even smaller third view (adapter name, My IP, Egress IP, SSID, Signal %, Signal
dBm, Gateway IP only), reachable from the main toolbar and from within Compact itself.

## Pop Graph

Opens the larger graph window.

## Minimize to Tray

Hides the main window but leaves monitoring active.

## Exit

Terminates ConnectivityOverlay.

## Always On Top

By default all three views (Main UI, Compact, Extra Compact) stay always-on-top. Each can
independently be allowed to drop behind other windows instead, via a checkbox per view in
Settings > App Settings.

---

# 18. Info Window

The Info window is intended as built-in operational documentation.

Topics include:

## Overview

- Purpose.
- Data sources.
- Supplemental-versus-core behavior.

## Refresh Rates

Typical collection rates include:

```text
Ping targets        Current configured interval; default 1 second
Adapter/IP state    Approximately 2 seconds
Wi-Fi live state    Approximately 2 seconds
WLAN events         Incremental / event-log based
Egress IP           Approximately 60 seconds
Graph rendering     Existing live graph behavior
```

## Wi-Fi Signal

- RSSI explanation.
- Signal-quality explanation.
- dBm approximation.
- Status ranges.
- Text colors.
- Band-change logging behavior.

## Wi-Fi Activity

- Column descriptions.
- Event sources.
- Show More behavior.
- Min/max visible line counts.

## Adapters

- Local IP.
- Adapter display.
- Default-route concepts.
- VPN considerations.

## Logging

- Log formats.
- UTC timestamps.
- Session IDs.
- Event logging behavior.

## Graph

- Duration.
- Retention.
- Per-target graph toggles.
- Downsampling.
- Threshold lines.

## Compact

- Compact ordering.
- Shared visibility settings.
- Signal display behavior.

## Troubleshooting

- Supplemental source failures.
- ICMP limitations.
- VPN/route behavior.
- Public-hop discovery limitations.

---

# 19. Appearance

Main-window appearance is configurable.

Current persisted appearance settings include:

```text
Background color
Normal/expanded opacity
Compact opacity
```

Default background:

```text
#101010
```

Default normal opacity:

```text
100%
```

Default compact opacity:

```text
100%
```

The main normal and compact view use the same base background color but independent opacity percentages.

Two additional colors are configurable, both in Settings > App Settings, each with a hex entry,
a preview swatch, and a Default button to clear it back to transparent:

- **App Header Color** — background for every window/flyout title bar row: the main toolbar,
  every flyout header, the Settings flyout's own header, and the Traceroute/Ping Output popup
  windows. Never covers the red X/Exit buttons themselves.
- **App Sub-Header Color** — background for the main screen's in-page section headers: IP
  Information, Adapter Info for..., Adapter Activity, Speed Test.

Both default to transparent (no color), so upgrading users see no visual change until a color is
actually set.

---

# 20. Logging

Logging is disabled by default, and always starts disabled on every launch regardless of what was last saved — a forgotten "on" setting from a previous session can't silently keep filling up disk.

## 20.1 Copy All / Write to Log File / Write to Disk

Every flyout (Diagnostics, Adapter Activity, Adapter Info, SSID/APs) and the main screen expose the same three export actions:

- **Copy All** — copies the section's content to the clipboard.
- **Write to Log File** — writes a structured entry to the configured application log; shows a warning if logging is currently disabled rather than doing nothing silently.
- **Write to Disk** — writes a one-off, timestamped snapshot file to the configured log folder, independent of whether logging is enabled. This is deliberate: a one-time export shouldn't require turning on continuous logging (which could otherwise silently fill up disk unnoticed). If the log folder doesn't exist, Write to Disk creates it; if it can't, it reports "Update Logging Path in Settings and try again" instead of failing silently.

Write to Disk file names are prefixed with the machine name, e.g. `MYPC_write_to_disk_diagnostics_20260921_143022.txt`, so files collected from multiple machines don't collide.

Default log directory:

```text
%USERPROFILE%\Documents\logs\connectivity_monitor_local
```

Supported formats:

```text
csv
txt
json
all
```

Each application run receives a unique `LogSessionId`.

Timestamps in log files are UTC ISO-8601 with milliseconds.

Example:

```text
2026-09-14T01:48:02.123Z
```

Daily file naming:

```text
connectivity_monitor_local_yyyyMMdd.csv
connectivity_monitor_local_yyyyMMdd.txt
connectivity_monitor_local_yyyyMMdd.json
```

Dedicated debug files, when configured, use:

```text
connectivity_monitor_local_yyyyMMdd_debug.*
```

Logging errors are intentionally swallowed so a file-system/logging problem cannot stop network monitoring.

Current stable field order:

```text
Timestamp
LogSessionId
AppName
Level
Status
EventType
Source
TargetIP
LatencyMs
PacketLossPercent
Adapter
AdapterType
LocalIP
Gateway
PublicHop
EgressIP
SSID
BSSID
APName
RSSIDbm
SignalPercent
SignalBand
OldValue
NewValue
Computer
User
Message
ErrorMessage
```

Important connectivity events can include:

```text
ADAPTER_CONNECTED
ADAPTER_DISCONNECTED
ADAPTER_IP_CHANGED
LOCAL_IP_CHANGED
DEFAULT_ROUTE_CHANGED
GATEWAY_CHANGED
EGRESS_IP_DISCOVERED
EGRESS_IP_CHANGED
EGRESS_IP_UNAVAILABLE
EGRESS_IP_RESTORED
WIFI_CONNECTED
WIFI_DISCONNECTED
WIFI_SSID_CHANGED
WIFI_BSSID_CHANGED
WIFI_AP_ROAM
WIFI_SIGNAL_BAND_CHANGED
VPN_ADAPTER_CONNECTED
VPN_ADAPTER_DISCONNECTED
VPN_IP_CHANGED
```

The exact event emitted depends on the observed transition and current implementation path.

---

# 21. Settings Persistence

Settings are stored as JSON under the current user's roaming application-data folder:

```text
%APPDATA%\ConnectivityOverlay\settings.json
```

If settings cannot be loaded, the application falls back to default settings.

If settings cannot be saved, monitoring continues.

This is deliberate fault isolation.

---

# 22. Auto Start

The application supports start-with-Windows behavior through the current-user Windows Registry Run key.

This does not require an installer.

The application also supports:

```text
Start hidden (tray only)
```

The normal default is visible startup.

---

# 23. Current Technology Stack

```text
Language:             C#
Framework:            .NET 8
Target Framework:     net8.0-windows
UI:                   WPF
Tray:                 Windows Forms NotifyIcon
Wi-Fi API:            Windows Native Wi-Fi / wlanapi.dll
Ping:                 System.Net.NetworkInformation.Ping
Route discovery:      .NET NetworkInterface + UDP route selection
Settings:             JSON
Logging:              CSV / TXT / JSON
Runtime target:       win-x64
Deployment:           Self-contained single-file executable
```

---

# 24. Project Structure

Current major project structure:

```text
ConnectivityOverlay/
├── App.xaml
├── App.xaml.cs
├── ConnectivityOverlay.csproj
├── Models/
│   ├── AppSettings.cs
│   ├── DeviceDriverInfo.cs
│   ├── DiagnosticState.cs
│   ├── NetworkAdapterInfo.cs
│   ├── PingSample.cs
│   ├── SpeedTestResult.cs
│   ├── TracerouteHop.cs
│   ├── WifiActivityEvent.cs
│   ├── WifiConnectionInfo.cs
│   ├── WifiQueryResult.cs
│   └── WifiScanResult.cs
├── Services/
│   ├── AppInfo.cs
│   ├── BootstrapDiagnosticLogger.cs
│   ├── DeviceInfoService.cs / IDeviceInfoService.cs
│   ├── DiagnosticService.cs
│   ├── DnsLookupService.cs / IDnsLookupService.cs
│   ├── EgressIpService.cs
│   ├── GraphRenderer.cs
│   ├── LatencyHealthService.cs
│   ├── LoggingService.cs
│   ├── NetworkAdapterService.cs
│   ├── NetworkPathService.cs / INetworkPathService.cs
│   ├── PingEngine.cs
│   ├── SettingsService.cs / ISettingsService.cs
│   ├── SnapService.cs
│   ├── SpeedTestService.cs / ISpeedTestService.cs
│   ├── StartupService.cs / IStartupService.cs
│   ├── TracerouteService.cs / ITracerouteService.cs
│   ├── WifiEventLogService.cs
│   └── WifiInfoService.cs / IWifiInfoService.cs
├── Tray/
│   └── TrayManager.cs
└── Views/
    ├── ExpandedGraphWindow.xaml(.cs)
    ├── InfoWindow.xaml(.cs)
    ├── MainWindow.xaml(.cs)
    ├── SettingsFlyout.xaml(.cs)
    └── TracerouteWindow.xaml(.cs)

ConnectivityOverlay.Tests/
└── (xUnit test project - pure-logic tests only; see §26)

template_csv/
└── ap_list.csv (blank starter file for the "AP Information .csv File Location" setting - see §9.1)
```

Several previously-`static`, hard-to-mock network/Wi-Fi services (`NetworkPathService`, `WifiInfoService`, `SettingsService`, `StartupService`, `DeviceInfoService`, `SpeedTestService`, `DnsLookupService`, `TracerouteService`) are now interface-based instance classes specifically so they can be unit tested and/or swapped, per `docs/DEVELOPER_GUIDE.md`.

---

# 25. Build and Publish

Development build:

```powershell
dotnet clean
dotnet build
```

Run:

```powershell
dotnet run
```

Run the automated tests:

```powershell
dotnet test
```
(from `ConnectivityOverlay.Tests/`)

Publish:

```powershell
dotnet publish -c Release
```

Expected published executable location:

```text
bin\Release\net8.0-windows\win-x64\publish\ConnectivityOverlay.exe
```

Because the project is self-contained and single-file, the published EXE is intended to run on a compatible Windows x64 machine without requiring the user to install the .NET runtime separately.

For a full distributable release package (zip + SHA256 checksum), see `scripts/Build-Release.ps1` and `docs/HOW_TO_CREATE_RELEASE_PACKAGE.md`. `README.md` and the relevant `docs/RELEASE_NOTES_vX.Y.Z.md` should be updated *before* running the release build, since both are bundled into the release zip.

---

# 26. Important Operational Limitations

ConnectivityOverlay is a troubleshooting aid, not a replacement for enterprise monitoring.

Important limitations:

- ICMP can be blocked or deprioritized.
- A ping failure does not necessarily prove the target service is unavailable.
- `DNS` currently tests ICMP reachability, not DNS resolution.
- 1st Pub Hop discovery depends on TTL-expired ICMP behavior and can fail on otherwise functional networks.
- Public egress IP requires external HTTPS access.
- Wi-Fi data depends on Windows Native Wi-Fi support and an active Wi-Fi association.
- Friendly AP names require user aliases because Windows normally exposes BSSID rather than the AP's human-readable mesh label.
- Displayed RSSI is derived from Windows signal quality and should be treated as an estimate.
- WLAN Event Log availability can vary by Windows configuration.
- VPNs can change default-route behavior and the meaning of gateway/local/egress addresses.
- Tray-icon placement is ultimately controlled by Windows.
- The program currently targets Windows only.

---

# 27. Recommended Troubleshooting Interpretation

A useful sequence is:

1. Check Wi-Fi signal if using Wi-Fi.
2. Confirm SSID/BSSID/AP if roaming is suspected.
3. Check My IP and adapter selection.
4. Check GW.
5. Check 1st Pub Hop.
6. Check DNS.
7. Check custom targets.
8. Review packet loss.
9. Review the graph.
10. Run a Traceroute against the specific target that looks unhealthy.
11. Review Wi-Fi Activity.
12. Review log files for exact transition timing.

This sequence helps isolate whether the problem is:

- Wireless signal.
- AP roaming.
- Local addressing.
- VPN/adapter routing.
- Gateway reachability.
- ISP/upstream path.
- Destination-specific connectivity.

---

# 28. Repository and Distribution Notes

A public source repository inherently exposes source code.

If source must remain private but binaries should be public, use:

- A private source repository for development.
- A separate public distribution repository or release location for executable ZIPs and documentation.

Do not commit build output directories such as `bin` and `obj` unless there is a specific reason.

A normal source-control workflow should commit source and documentation after the application successfully builds and runs.

---

# 29. Summary

ConnectivityOverlay combines:

- Continuous ICMP monitoring.
- Dynamic gateway discovery.
- First-public-hop discovery.
- DNS (auto-detected) and up to 12 custom/CSV-driven ping targets, with URL or IP entry and automatic DNS resolution.
- Per-target Traceroute and Ping Output (live recent-results view), each with up to 5 concurrent windows.
- Configurable latency-health profiles.
- Advanced per-target latency thresholds, with independent legends and a static Legends tab in Settings.
- Rolling packet-loss calculations.
- 24-hour history retention.
- Embedded and expanded graphing.
- Per-target graph visibility toggles.
- Network adapter and local-IP telemetry.
- Public egress-IP tracking.
- Periodic internet speed testing (Speed Test), with an on-demand Run Once option.
- Native Windows Wi-Fi SSID/BSSID telemetry.
- AP aliasing.
- Wi-Fi signal percentage and dBm display.
- Wi-Fi signal-band coloring.
- Wi-Fi Activity history.
- WLAN AutoConfig event integration.
- Redesigned, bordered Compact mode with one-click Copy, plus an even smaller Extra Compact mode.
- Notification-area operation, plus a regular taskbar icon while open.
- Per-view Always On Top control (Main UI/Compact/Extra Compact independently).
- Configurable header and sub-header background colors.
- Optional multi-format logging, with unified Copy All / Write to Log File / Write to Disk export tooling.
- Built-in tabbed Info/reference documentation.
- Self-contained Windows deployment.

The intended result is a small desktop utility that can stay out of the way during normal work but expose useful troubleshooting detail immediately when connectivity begins to behave unexpectedly.


# 30. Version History

## v2.3.0 — CSV-Driven Targets, Ping Output, Settings Redesign, and a Large UI Polish Pass

Version 2.3.0 supersedes the never-tagged v2.2.0 draft below and includes everything built since
v2.1.0 was last actually released. Highlights: the 4 fixed-identity default targets (VPN-Cisco,
VPN-Global Protect, Microsoft MyApps, XiFin SSO) were removed and replaced with CSV-driven
targets (CsvTarget1-7, read from an external file) and two more Custom targets (Custom4/5, for
5 total); a new "P" (Ping Output) button next to every "T" (Traceroute) button shows a
live-updating view of a target's last 5-30 ping results, and the concurrent-window cap for both
was raised from 3 to 5; Settings was redesigned into a tabbed, left-docked panel with a new
"Legends" tab (a static, always-current reference combining the ping/latency legend and the
Wi-Fi Signal/RSSI legend) and new App Header Color / App Sub-Header Color settings; the RSSI
color palette became user-editable; per-view Always On Top control and a taskbar icon were
added; the main screen was reordered (My IP Info promoted above Adapter Info, which is now
collapsible along with Speed Test), the Wi-Fi Signal row moved and now shows SSID first, the
standalone BSSID row was removed, "Wi-Fi Speed" was renamed to "Link Rate" and switched to MB/s
(fixing a genuine off-by-100 driver bug on some Wi-Fi 6E/7 hardware that had been showing
implausible multi-gigabyte rates), and every close ("X") button now uses the same red as Exit.
See `docs/RELEASE_NOTES_v2.3.0.md`.

## v2.2.0 — Traceroute, Speed Test, URL Targets, and Compact Redesign

Version 2.2.0 adds a per-target Traceroute tool, periodic and on-demand internet Speed Testing, four new default ping targets with URL-based entry and automatic DNS resolution, DNS auto-detection (replacing the fixed `8.8.8.8` default), a redesigned bordered Compact view, unified Copy All / Write to Log File / Write to Disk export tooling across every flyout, a fix for Advanced per-target legends never refreshing, and a personalized tuned-baseline set of default settings. See `docs/RELEASE_NOTES_v2.2.0.md`.

## v2.1.0 — Diagnostics, Resilience, Permission Awareness, and Enterprise Compatibility

Version 2.1.0 adds the Diagnostics flyout, Native Wi-Fi error/result visibility, Location Services awareness, detailed diagnostic logging fields, startup/emergency diagnostics, settings corruption recovery, Restore All Defaults, WLAN event-log health reporting, recovery events, and standard-user/enterprise troubleshooting improvements. See `docs/RELEASE_NOTES_v2.1.0.md`.

## v2.0.0 — Network, Wi-Fi, Graphing, Logging, and UX Expansion

Version 2.0.0 is a major feature and architecture expansion of ConnectivityOverlay.

Major additions include:

- Active network-adapter awareness.
- Local IPv4 display.
- Public/egress IPv4 monitoring and change tracking.
- Native Windows Wi-Fi telemetry.
- SSID and BSSID display.
- Friendly AP alias mapping.
- AP-roam detection.
- Wi-Fi signal percentage and RSSI-style dBm display.
- Wi-Fi signal-health colors and status labels.
- Wi-Fi Activity history.
- WLAN AutoConfig event integration.
- Per-target graph visibility controls.
- Improved graph retention and rendering behavior.
- Enhanced Compact mode with network and Wi-Fi information.
- Expanded structured logging for adapter/IP/Wi-Fi changes.
- Built-in tabbed Info/reference window.
- Improved main-window auto-sizing.
- Auto-sized Wi-Fi Activity columns.
- Additional reliability and WPF/WinForms interoperability fixes.

For the complete release history, implementation summary, upgrade notes, and v1-to-v2 comparison, see:

```text
RELEASE_NOTES_v2.0.0.md
```

## v1.0.0 — Initial ConnectivityOverlay Release

Version 1.0.0 established the original ConnectivityOverlay concept and core monitoring experience.

The first major release included:

- Default-gateway monitoring.
- 1st public-hop discovery and monitoring.
- DNS endpoint monitoring.
- Three custom ping targets.
- Latency-health coloring.
- Packet-loss calculation.
- Rolling graph history.
- Notification-area operation.
- Compact overlay mode.
- Settings persistence.
- Auto-start support.
- Basic structured logging.
- Self-contained Windows x64 publishing.

For the retrospective release summary of the original application, see:

```text
RELEASE_NOTES_v1.0.0.md
```

## v2.1.0 Diagnostics and Enterprise Resilience

Version 2.1.0 adds permission/privacy-aware diagnostics without changing the core v2.0.0 monitoring design. A new **Diag** flyout shows the current application, security, logging, network and Wi-Fi health state. Windows Native Wi-Fi errors are retained with native result codes; protected Wi-Fi access denied by Windows is shown as **RESTRICTED** rather than being silently interpreted as no activity. Location Services state is reported when it can be determined, and the application distinguishes Disabled, Unavailable, Restricted and Failed states.

The existing structured TXT/CSV/JSON/ALL logs now include diagnostic context such as application version, elevation/security context, Location Services state, Wi-Fi access state, native API/result, possible cause, impact and recommended action. A separate bootstrap diagnostic log under `%LOCALAPPDATA%\ConnectivityOverlay\diagnostics` captures startup/settings/logging failures that may occur before normal logging is available.

Settings now include **Restore All Defaults**, and invalid settings are preserved when possible before safe defaults are loaded. See `docs/RELEASE_NOTES_v2.1.0.md` and `docs/TEST_PLAN_v2.1.0.md` for details.
