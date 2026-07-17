# Home SOC Lab

A hands on home lab for learning SOC analyst and blue team skills, built while studying for CompTIA CySA+ (CS0-004). This repo documents the build process, study notes, and day to day progress, all in one place.

## Who's working on this

Tyler Jackson (IT Support Specialist, currently Security+/Network+/A+ certified) and my brother, studying together toward CySA+.

## Goal

Build practical, documented experience with SOC tooling (SIEM, log analysis, detection writing) to support a move into a SOC analyst or similar cybersecurity role, remote or local, alongside earning CySA+.

## Lab environment

- Hypervisor: Proxmox VE
- SIEM / NSM platform: Security Onion
- Additional VMs: TBD as the network topology is built out

See `docs/` for full setup details as each piece comes online.

## Repo structure

- `docs/` — Finished writeups of what's been built: setup guides, architecture, configuration decisions.
- `notes/` — Study notes organized by CySA+ exam domain.
- `journal/` — Dated, running log of work sessions: what was done, what broke, what was learned.

## Status

Security Onion (Standalone) is installed on Proxmox and verified working: all core services running, web console reachable, SSH key authentication set up. See `docs/security-onion-install.md` for the full build writeup and `journal/` for the day-to-day log.

Next up: adding a second VM to generate test traffic and confirm the sniffing interface is capturing and analyzing it, then starting CySA+-aligned lab exercises.

## Disclaimer

This is a personal learning lab. Nothing here reflects production security practices or represents any employer's systems or data.