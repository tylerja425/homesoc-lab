# AI Context — Home SOC Lab

This public repository is the authoritative public-safe record of Tyler's home SOC lab: the build, detection exercises and portfolio writeups. It is not a complete record of the server, network or services behind the lab.

## Reading order

1. `README.md` for scope and current status.
2. Only the relevant file in `docs/`.
3. `journal/` for dated build history and troubleshooting evidence.

Don't load the whole repository by default.

## Source authority

- `docs/` is authoritative for the setup it describes; `README.md` is the overview.
- `journal/` is dated evidence, not automatically current truth.
- Tyler's private Home Server context may hold newer or fuller information. Absence here doesn't mean a system, service, decision or issue doesn't exist. When the two conflict, show the conflict rather than choosing silently.

## Shared lab

Tyler's brother uses the lab too. They share Security Onion, the Metasploitable2 target and the Greenbone scanner, and each runs his own other VMs. This repository documents Tyler's build and the shared parts; his brother's own VMs and notes stay in his own records.

## Changes

Reading or discussing this repository doesn't authorize a change. Change files only when Tyler asks: show the exact files and what changes in each, make one commit after he approves, then read it back.

## Public boundary

Everything committed here is public and durable. Never add:

- Passwords, tokens, private keys, recovery material or credential contents
- SSH key values or secret-bearing commands
- Public-facing access details that create unnecessary exposure
- Private personal, household, employer, customer, student or staff information
- A complete private server or network inventory merely for AI convenience
- Raw AI transcripts or private context notes

Keep the documentation technically credible while minimizing operational exposure. Private context links to these public files rather than copying them, and nothing private is copied here.
