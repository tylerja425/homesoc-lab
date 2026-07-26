# Home SOC Lab

A hands on home lab for learning SOC analyst and blue team skills, built while studying for CompTIA CySA+ (CS0-004). This repo documents the build process, study notes, and day to day progress, all in one place.

## Who's working on this

Tyler Jackson (IT Support Specialist, currently Security+/Network+/A+ certified) and my brother, studying together toward CySA+.

## Goal

Build practical, documented experience with SOC tooling (SIEM, log analysis, detection writing) to support a move into a SOC analyst or similar cybersecurity role, remote or local, alongside earning CySA+.

## Lab environment

- Hypervisor: Proxmox VE
- SIEM / NSM platform: Security Onion (Standalone)
- Attacker VM: Kali Linux
- Target VM: Metasploitable2
- Vulnerability scanner: Greenbone Community Edition (OpenVAS)
- Isolated lab network (`10.10.10.0/24`), separate from the home network, with traffic mirrored to Security Onion for full visibility

See `docs/` for full setup details.

## Repo structure

- `docs/` — Finished writeups of what's been built: setup guides, architecture, configuration decisions.
- `notes/` — Study notes organized by CySA+ exam domain.
- `journal/` — Dated, running log of work sessions: what was done, what broke, what was learned.

## Status

Security Onion (Standalone) is installed on Proxmox and verified working. An isolated lab network (`10.10.10.0/24`) hosts a Kali attacker VM and a Metasploitable2 target VM, with traffic mirrored to Security Onion using Linux `tc` rules and a Proxmox hookscript that survives both VM restarts and full host reboots. The full detection pipeline has been verified end to end: a real `nmap` scan generated genuine Zeek connection logs and a correctly triaged Suricata alert. See `docs/security-onion-install.md`, `docs/lab-network-topology.md`, and `docs/troubleshooting-detection-pipeline.md` for the full build and debugging writeups.

A vulnerability scanner (Greenbone Community Edition / OpenVAS) has since been added, scanning Metasploitable and returning real findings (66 findings, 118 CVEs on the first pass). One finding, a known vsftpd backdoor, was manually exploited via Metasploit and confirmed detected in Security Onion, completing a full scan to exploit to detection loop. See `journal/` for the day by day account.

SSH password authentication has been disabled across all lab hosts; key based auth only.

Next up: continuing CySA+-aligned detection exercises, and extending the shared lab so a second person can run their own attacker/target VMs against the same shared Security Onion instance.

## Disclaimer

This is a personal learning lab. Nothing here reflects production security practices or represents any employer's systems or data.
