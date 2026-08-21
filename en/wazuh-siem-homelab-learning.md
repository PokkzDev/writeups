# Wazuh SIEM: Learning Security Monitoring on My Homelab

- **Date:** 2026-08-21
- **Environment:** Proxmox node with an LXC container (Debian 13, 4 vCPU / 6 GB RAM / 50 GB disk), Wazuh 4.14.7 all-in-one (indexer + manager + dashboard), Tailscale overlay network.
- **Status:** Working. Two agents enrolled (Windows 11 and macOS on Apple Silicon), both reporting remotely; vulnerability detection live.

## Goal

Learn what a SIEM actually does by deploying one on my own hardware and pointing it at my own machines. Not a course demo: a real, running stack that watches my laptops and teaches me how alerts, logs, and vulnerability feeds behave in practice.

## Why a SIEM (and why Wazuh)

A SIEM centralizes logs from many machines and turns them into questions: who failed to log in, what changed on disk, which software is outdated and exposed. Wazuh is a solid open-source choice for learning because the whole stack (indexer + manager + dashboard) installs on modest hardware, and the agent supports Windows, macOS, and Linux.

The architecture in one line: agents collect, the manager analyzes with rules, the indexer stores, the dashboard visualizes.

## What I did

1. **Deployed Wazuh in an LXC container on my Proxmox node.** The all-in-one install is one installer script, with one real gotcha: the OpenSearch indexer needs `vm.max_map_count`, a kernel parameter that in an unprivileged container must be set on the host, not inside the guest.
2. **Enrolled my two daily machines as agents:** a Windows 11 laptop (MSI package) and a macOS laptop on Apple Silicon (pkg, plus a manual Full Disk Access grant so the agent can read unified logs).
3. **Made the agents reachable from anywhere with Tailscale.** The manager joins the overlay network and both agents report through it, so monitoring keeps working when the machines are off my LAN.
4. **Enabled vulnerability detection** in the manager config (`<vulnerability-detection>` block) and let it download the CVE feeds and run a first scan.

## What I observed

- Both agents report as **Active** and their events appear in the dashboard in real time: logins (including failed ones, which the rule engine flags after repeated attempts), process launches, file changes.
- The **vulnerability scan** was the most instructive part. The first scan after the CVE feed update left no doubt about my patch hygiene:
  - the macOS agent reported a large set of OS-level CVEs, most of them already addressed in newer macOS releases;
  - the Windows agent flagged a handful of app-level CVEs in older installed software.
- Nothing exotic: no zero-days, just the usual "your systems are behind on updates" story that every SIEM will tell you. Which is exactly the point of running one.

## Lessons

- **The CVE feeds ARE the detector.** Vulnerability detection is useless until the feeds are downloaded; the scan only became meaningful after a feed update. "Update the feeds" is the first thing to check when a scan looks empty.
- **Findings are not alerts.** Vulnerability results land in a dedicated index and the dashboard's Vulnerability Detection tab; they do not necessarily generate alert events. "No alerts" does not mean "not scanning".
- **A SIEM is a mirror of your update habits.** The fastest way to make the numbers go down is patching, and the SIEM verifies it afterwards. Also a good argument for keeping backups and an update schedule.
- **Kernel parameters belong to the host.** Any mmap-hungry service in an LXC (OpenSearch, Elasticsearch...) needs `vm.max_map_count` set on the Proxmox host.
- **Right-sizing is real.** The official "minimum" specs assume dozens of agents; a homelab with two laptops runs the full stack comfortably on 4 vCPU / 6 GB RAM.

## Next steps

- Tune rules so important findings generate alerts instead of living only in the index.
- Add more log sources (my router, the Proxmox host itself).
- Explore the API to build simple scripts on top of the data.

## Verification

```bash
# on the manager (Debian LXC):
systemctl is-active wazuh-indexer wazuh-manager wazuh-dashboard   # all active
/var/ossec/bin/agent_control -l                                   # agents listed as Active
# dashboard: Agents tab -> both machines Active
#            Vulnerability Detection tab -> per-agent CVE inventory
```
