# Mundo Pacífico CPE Liberation — TR-069 Kill + Manual Bridge

- **Date:** 2026-08-05
- **Source:** Luis — hands-on procedure on his own equipment (home line, Chillán, GPON fiber). Not AI-derived; verified live on the lab.
- **Status:** Working — ONT bridged, TR-069/CWMP disabled, own router does WAN/firewall/DHCP.

## Goal

Take full control of the ISP-provided router/ONT (Mundo Pacífico, CGNAT line):

1. Stop the ISP from pushing configs remotely (TR-069 / CWMP channel).
2. Bridge the device so my own router is the single source of truth: firewall, DHCP, DNS, Wi-Fi — no conflicting signals, no double NAT.
3. Keep the line working through the change — no support calls, no reprovisioning.

## Background

ISPs manage their CPE via TR-069 (CWMP). The router dials **out** to the ISP's ACS on **TCP 7547** and the ACS pushes configs: firmware, SSIDs, VLAN tagging, provisioning. The router ships with default factory credentials that get replaced on first provisioning.

Key insight: **the window between factory state and first ACS contact is the device's most useful moment** — it's a race condition against the provisioning system. The device is unmanaged, unprovisioned, and running default creds until the ACS wins the race.

## Procedure

1. **Factory reset the router OFFLINE** (no fiber/internet connected). It boots to factory defaults — default admin credentials still in place, nothing provisioned yet.
2. **Log in with the default factory password.**
3. **Reconnect the line and catch the initial config push** as the ACS starts provisioning.
4. **Disable the TR-069/CWMP channel** — wipe the ACS URL / inform settings so no further configs or firmware can be pushed.
5. **Bridge the ONT** — set WAN to bridge mode so my router terminates the WAN directly.
6. **Verify**: router pulls the WAN IP, firewall/DHCP/DNS live on my gear, ONT is now a dumb bridge.

## Results

- **WAN**: direct on my router — CGNAT address as expected (inbound still via Tailscale; this is the backbone anyway).
- **Network**: one source of truth. No double NAT, no ISP DNS in the path, no Wi-Fi overlap between boxes.
- **ISP management**: remote channel dead — no more config pushes, no remote resets, no surprise firmware updates.

## Caveats / What still applies

- **OMCI still exists**: on GPON the OLT manages the ONT at the PON layer via OMCI — that path is part of the fiber handshake and can't be disabled from the web UI. Killing TR-069 removes ~95% of remote management, not 100%.
- **Firmware is frozen**: no TR-069 = no auto-updates. Acceptable because the device is now a dumb bridge behind my own firewall.
- **Support calls may undo it**: a tech visit or reprovision can reset the ONT. There's no ACS to re-provision it afterward — rebuild would be manual.
- **Back up the config**: admin backup file + WAN details (VLAN ID, PPPoE creds if the plan uses them). Without TR-069, a second factory reset means a manual rebuild.
- **Watch after reboots**: confirm TR-069 stays disabled — some firmware re-arms it from volatile config. Tell-tale: ACS URL back in the CWMP page.

## Verification (quick check)

Compare router WAN IP vs external IP (ifconfig.me). Different → CGNAT (RFC 6598 100.64.0.0/10), Tailscale stays the inbound path. Identical → real public IP, inbound becomes possible.

## Lessons

- The provisioning window (factory → first ACS contact) is a real attack surface. Racing it is a legitimate CPE-liberation technique — derived independently, not looked up.
- ISP "management" is a trust boundary worth auditing on your own gear.
