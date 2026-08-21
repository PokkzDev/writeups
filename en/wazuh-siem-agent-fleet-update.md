# Wazuh SIEM Fleet Update: 2 Tailnet Agents + Vulnerability Detection Live (CT 300)

- **Date:** 2026-08-21
- **Source:** Live SSH session to CT 300 (SSH alias `wazuh`, tailnet `100.109.208.61`): `agent_control`, `ossec.conf`, `ossec.log`, alert archive, scanner RocksDB forensics, `tailscale status`. Every fact verified on the box.
- **Status:** Working. 2 agents **Active**, both over Tailscale; CVE feeds updated today (12:53-12:55 CLT) with re-scan completed; **macbookairm2 now reports vulnerabilities** in the dashboard (gp77-13vg too).

## Goal

Document the SIEM's current fleet state: which agents are enrolled, how they reach the manager (Tailscale), and the state of vulnerability detection after today's feed update. In short: "the Mac now sees vulnerabilities".

## Background

- Wazuh 4.14.7 (rc1) all-in-one on CT 300 since 2026-08-16 (see the deployment writeup). At that point GP77 was enrolled and Active; the MacBook Air was still pending.
- Since 4.9/4.10 the vulnerability scanner is a new C++ module (`wazuh-modulesd:vulnerability-scanner`) configured in `ossec.conf` under `<vulnerability-detection>`. The old `<vulnerability-detector>` block / `vulnerability-detector.yml` no longer exists. Grep for the right tag.
- **Key mechanic:** findings come from the CVE feeds. No feeds = no findings. The full feed only landed today, which is why the Mac "started seeing vulnerabilities" now rather than at install.
- `/root/wazuh-passwords.txt` is gone from the box. All evidence for this doc was gathered without API credentials. Recovery path if ever needed: `bash /root/wazuh-install.sh -p`.

## Fleet snapshot (verified 2026-08-21 ~14:40 CLT)

| ID | Agent | OS (agent_control) | Agent ver | Tailscale device | Tailnet IP | Path | Last keep-alive |
|---|---|---|---|---|---|---|---|
| 001 | `gp77-13vg` | Microsoft Windows 11 Pro | v4.14.7 | `pokkzseclaptop` | 100.122.146.80 | **direct** | ~14:40 |
| 002 | `macbookairm2` | Darwin 27.0.0 arm64 (macOS 27; `PokkzSec-MacBook-Air-M2.local`) | v4.14.7 | `pokkzsec-macbook-air-m2` | 100.126.178.88 | **relay "sao"** | ~14:25 |
| 000 | `wazuh` (the manager itself) | Debian 13 | v4.14.7 | - | - | local | - |

Both agents share the same config hash (`ab73af41...`): identical pushed agent configuration.

## Tailscale transport: evidence

- `tailscale status` on CT 300: both agent devices **active**; GP77 direct, Mac via DERP relay `sao` (idle, so no direct path right now; it returns when the Mac is on the same network as another node).
- Live TCP at capture: `ESTAB 100.109.208.61:1514 <- 100.122.146.80:61532` (GP77 session).
- Alert-archive IP attribution (Aug 16-21): **4,237 events from `100.122.146.80`** (gp77), **350 from `100.126.178.88`** (macbook), **0 from LAN `192.168.1.x`**. All agent traffic rides the tailnet; the LAN path is unused by agents.
- Interfaces on the box: `eth0 192.168.1.250/24` (LAN), `tailscale0 100.109.208.61/32`.

## Vulnerability detection: the update

### Config (ossec.conf, enabled since install)

```xml
<vulnerability-detection>
  <enabled>yes</enabled>
  <index-status>yes</index-status>
  <feed-update-interval>60m</feed-update-interval>
</vulnerability-detection>
```

### What happened today (ossec.log, local CLT)

```
12:53:21 wazuh-modulesd:vulnerability-scanner: INFO: Initiating update feed process.
12:55:43 wazuh-modulesd:vulnerability-scanner: INFO: Triggered a re-scan after content update.
12:55:43 wazuh-modulesd:vulnerability-scanner: INFO: Feed update process completed.
```

- Feed store (`/var/ossec/queue/vd/feed/`, RocksDB): **~12 GB** after the update (was a stub before): the full CVE database.
- Inventory DB (`/var/ossec/queue/vd/inventory/`) rewritten at 12:55; the scanner's event DB was still chewing the **Mac's app inventory** (PIPAgent.app etc.) at 14:29. The first full scan was still settling in after the feed update.
- Findings land in the `wazuh-vulnerabilities-*` index, which feeds the **Vulnerability Detection tab** in the dashboard. That is exactly what "the Mac sees vulnerabilities" means.

### What the scanner has on file (RocksDB inventory forensics)

- **Agent 002 (macbookairm2):** OS-level CVE set of **~120** (CVE-2026-28911 ... CVE-2026-64783 range, consistent with a current macOS 27 security-bulletin batch), plus **17 app-level CVEs across 4 apps**, e.g. `CVE-2026-8643, CVE-2026-6357, CVE-2025-8869, CVE-2026-1703, CVE-2026-3219, CVE-2023-5752` (one app); `CVE-2026-59890, CVE-2025-47273, CVE-2024-6345, CVE-2022-40897` (another); `CVE-2026-3029`; `CVE-2026-64783, CVE-2026-64719, CVE-2026-64728, CVE-2026-64730, CVE-2026-64757, CVE-2026-43804` (another).
- **Agent 001 (gp77-13vg):** 8 app-level CVEs (2015-2021 era): `CVE-2021-30481, CVE-2020-15530, CVE-2019-17180, CVE-2019-15316, CVE-2019-15315, CVE-2019-14743, CVE-2015-7985, CVE-2015-4016`.
- Alert archive (Aug 16-20): the 54 `vulnerab` string hits are **all SCA/CIS benchmark alerts** (Debian 13 / Win11 / macOS Tahoe policies), not vulnerability alerts. So **no vuln alerts ever fired before the feed landed** (expected: nothing to compare against).
- Exact per-agent totals: the **dashboard is authoritative** (Vulnerability Detection tab). The box-side inventory is the raw material, and counts change as OS/apps get patched.

## Results

- **Fleet:** 2/2 agents Active, both over Tailscale (direct + relay), same config hash.
- **Vulnerability detection:** feeds current (updated today, 60-min cadence), re-scan triggered, both agents' inventories populated. The Mac now shows its CVE set in the dashboard.
- **Server health:** up 4d21h, RAM 2.2/6.0 GiB, disk 19/50 GiB (41%), of which the CVE feeds are ~12 GB.

## Caveats / What still applies

- **Feeds = disk + bandwidth:** 12 GB, re-downloaded every 60 min. Watch `/` and the PVE NIC.
- **Findings are not alert events by default** (`index-status: yes` only). They surface in the Vulnerability Detection tab; generating alert noise for new CVEs needs rule tailoring on top.
- **The Mac rides relay "sao" while idle** (low bandwidth, slow inventory sync); the direct path resumes when it is co-located with another node.
- **No `/root/wazuh-passwords.txt`:** `admin` recovery is `bash /root/wazuh-install.sh -p` (resets and prints). Store the new password somewhere safe.
- Dashboard totals may differ slightly from the lists above (incremental scans; solved/patched CVEs drop off).

## Lessons

- The new scanner config lives in `ossec.conf` `<vulnerability-detection>`; the old `vulnerability-detector.yml` is gone.
- Vulnerability results live in an OpenSearch **index**, not the alert stream. "No alerts" does not mean "not scanning".
- RocksDB forensics: `strings` on `queue/vd/inventory/*.log` dumps the per-agent CVE keys: a no-credentials way to prove what the scanner knows.
- No feeds = no findings. **The feed update today is the actual lever** that lit up the Mac.

## Verification (reproducible)

```bash
ssh wazuh
systemctl is-active wazuh-indexer wazuh-manager wazuh-dashboard   # active x3
/var/ossec/bin/agent_control -l                                   # 001, 002 Active
/var/ossec/bin/agent_control -i 002                               # Darwin 27.0.0 arm64
tailscale status                                                  # both agents active
grep -a vulnerability-scanner /var/ossec/logs/ossec.log | tail    # feed update + re-scan
du -sh /var/ossec/queue/vd/feed                                   # ~12G
# dashboard -> Vulnerability Detection -> macbookairm2 -> CVE list
```
