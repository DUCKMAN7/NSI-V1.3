<p align="center">
  <img src="<img width="1280" height="640" alt="image (2)" src="https://github.com/user-attachments/assets/efb848b2-95cd-4f3a-90e8-658eb52b2082" />
" alt="NSI - Network Security Insight" width="720">
</p>

# NSI — Network Security Insight  `v1.3`

A defensive, non-intrusive **router & network auditing tool** for **Windows PowerShell** (C++17, Winsock2 + IP Helper API).
Designed by **Dev.Fadel & Dev.Basil**.

NSI renders everything inside the PowerShell window (no GUI): an animated intro (1.5 s NSI character rain → large pixel eye that blinks and looks right/left/centre → glitch title → credits),
then an interactive monochrome (white/grey/black) menu with orange accents, with a sidebar, option cards, a scrollable result pager and a status bar.

> **Use only on networks and devices you own or are explicitly authorised to assess.**
> NSI sends no exploit payloads, does no password guessing and no denial-of-service traffic. The only "active" probes are
> TCP connects, passive banner reads, `HEAD /`, a DNS `version.bind` query, ARP requests on your own subnet and **one read-only SNMPv1 GET** (community `public`).

## Build (PowerShell)

```powershell
cd path\to\nsi
.\build.ps1                 # auto-detects g++ (MinGW-w64) or cl.exe (MSVC)
.\build.ps1 -Compiler gcc   # force MinGW
.\build.ps1 -Compiler msvc  # force MSVC (loads vcvars64 automatically if needed)
```

Output: `nsi.exe` (links `ws2_32` and `iphlpapi`) with the NSI icon and version info embedded from `nsi.rc` / `assets\nsi.ico`
(needs `windres` for MinGW or `rc.exe` for MSVC; without them the build still succeeds, just without the icon). MSVC builds use `/utf-8`; g++ builds use UTF-8 source/exec charsets.
Use **Windows Terminal** or PowerShell 7 for best rendering (24-bit colour, Unicode box drawing). Window size ≥ 90×26.

## Run

```powershell
.\nsi.exe                                   # intro + interactive menu, live mode, target = default gateway
.\nsi.exe --demo --no-intro                 # menu with simulated vulnerable router (safe for presentations)
.\nsi.exe --target 192.168.1.1 --full-scan  # non-interactive full scan + report export
.\nsi.exe --demo --full-scan --no-intro     # fully offline, non-interactive demo
.\nsi.exe --help
```

| Flag | Meaning |
|---|---|
| `--target <IP>` | IPv4 address to audit (default: default gateway) |
| `--demo` | Simulated router scan data, CVEs, LAN hosts, posture and report |
| `--full-scan` | Run all modules, print summary, export reports, exit |
| `--ports <A-B>` | Port range to scan (default `1-65535`) |
| `--quick` | Scan only the 9 classic router ports |
| `--no-intro` | Skip the animation |
| `--help` | Usage |

Run `nsi.exe` from the project folder (or keep `data\cve_db.json` beside/under the exe) so the CVE database is found.
Reports are written to `reports\nsi_report.md` and `reports\nsi_report.json` relative to the current directory.

### Menu controls

`↑/↓` or `W/S` move · `→`/`Tab` enter the cards · `←`/`Esc` back · `Enter` run · `1–9` jump to a section · `Q` quit.
In the result pager: `↑/↓`, `PgUp/PgDn`, `Home/End` scroll; `Esc`/`Enter` return.

## Modules

| # | Module | Files | What it does |
|---|---|---|---|
| 1 | Port & Service Discovery | `scanner.*` | Multi-threaded (256 workers) TCP connect scan of **all ports 1-65535** (or `--quick` for the 9 classic router ports, or `--ports A-B`), banner grabbing, service/version identification, flags Telnet/FTP/plain-HTTP/SMB/RDP/VNC/TR-069/backdoor ports |
| 2 | Vulnerability & Misconfiguration | `vuln_checker.*`, `data/cve_db.json` | Regex-matches versions to router/service CVEs; checks HTTP→HTTPS redirect, alternate management ports, WAN exposure, SNMP `public` on UDP 161 |
| 3 | Host Discovery | `host_discovery.*` | ARP sweep (`SendARP`, capped to a /24) + `GetIpNetTable`; IP, MAC, interface (`GetAdaptersAddresses`), vendor via embedded OUI table |
| 4 | Posture Check | `posture_check.*` | Active adapter, IPv4, mask, gateway, DNS, DHCP; ICMP/ARP gateway reachability; DNS sanity (public name → private answer, NXDOMAIN hijack) |
| 5 | Report Generator | `reporter.*` | Aggregates findings, classifies `HIGH/MEDIUM/LOW/INFO`, remediation steps, console summary, Markdown + JSON export |

UI: `ui_intro.*` (intro, console setup), `ui_menu.*` (menu, pager, full-scan orchestration), entry point `main.cpp`.

## Changelog

* **1.3** — Intro is now resize-aware: maximising or resizing the window while the intro plays re-centres the rain, eye and title every frame
  (no more art stuck in a corner). Sharper multi-size EXE icon, `--version` flag, version shown in the menu and in reports.
* **1.2** — Full 1–65535 multi-threaded port scan, larger CVE database with vulnerability names, monochrome + orange theme, new eye animation.
* **1.0** — First release.

## Notes & limits

* **CVE matching is indicator-based.** Router vendors backport fixes without changing banners, and many routers hide versions.
  Treat every hit as "verify against the vendor advisory / NVD". Re-check CVSS values for your records. Add your own entries to `data/cve_db.json`
  (`match` is a case-insensitive ECMAScript regex tested against `"<version> | <banner>"`).
* HTTPS/TLS ports are detected as open but not inspected (no TLS handshake or certificate analysis).
* The embedded OUI table is a small subset; unknown prefixes show as *Unknown vendor*, locally-administered MACs as *Private/randomized MAC*.
* ARP discovery sees only the local subnet. Live scanning/posture/host features use Windows APIs; on other platforms only `--demo` works.
* Demo mode never touches the network.

## Project layout

```
nsi/
├─ build.ps1
├─ nsi.rc                  (EXE icon + version info)
├─ assets/                 (logo: nsi-banner.svg/png, nsi-icon.svg/png, nsi.ico)
├─ README.md
├─ data/cve_db.json
├─ reports/                (generated: nsi_report.md, nsi_report.json)
└─ src/
   ├─ main.cpp  common.hpp  json.hpp
   ├─ ui_intro.hpp/.cpp   ui_menu.hpp/.cpp
   ├─ scanner.hpp/.cpp    vuln_checker.hpp/.cpp
   ├─ host_discovery.hpp/.cpp   posture_check.hpp/.cpp
   └─ reporter.hpp/.cpp
```
