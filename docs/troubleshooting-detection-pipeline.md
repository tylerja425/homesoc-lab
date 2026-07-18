# Troubleshooting: Getting the Detection Pipeline Working End to End

This is a writeup of a real debugging session: after building the full
lab (Security Onion, Kali attacker VM, Metasploitable target VM, all on
the isolated `vmbr1` network — see `lab-network-topology.md`), the first
attempt to actually generate and detect traffic turned up **zero
results**, despite every individual piece appearing to work. Getting from
"nothing shows up" to a genuine, correctly triaged Suricata alert took
four separate, stacked root causes. Documented here in full because the
diagnostic process is arguably more valuable than the fix itself —
each layer looked like "the" answer before turning out to be one piece
of a bigger picture, which is a realistic shape for this kind of network
troubleshooting.

## The starting point

With Kali (`10.10.10.10`) and Metasploitable (`10.10.10.20`) both up and
able to ping each other successfully, an `nmap -sV 10.10.10.20` scan was
run from Kali. Checking Security Onion's Hunt page for any trace of that
traffic (`10.10.10.20`) returned **zero results**. Ping worked. The scan
completed and returned real service data. But Security Onion had
recorded nothing at all.

## Layer 1: search technique (ruled out)

First hypothesis: maybe the search query itself was wrong. A bare IP
address is valid Onion Query Language (it searches across all fields),
which was confirmed — a similar bare-IP search briefly returned results,
but they turned out to be Security Onion's *own* web dashboard API logs
(`event.dataset: soc.server`), i.e. the browser's own traffic to the SOC
interface while refreshing the page, not network capture data at all.

**Lesson:** a query returning *something* doesn't mean it's returning
the *right* something. Switched to a more specific query
(`source.ip:10.10.10.10 OR destination.ip:10.10.10.20`) to filter out
dashboard noise for future searches, and confirmed this narrower query
also returned zero real results — so the search technique wasn't the
issue, but it was still worth ruling out explicitly before moving on.

## Layer 2: the monitor interface was never actually active

Checked network state directly on Security Onion (`ip a`). The monitor
NIC (`ens19`) was `UP` at the link level, but missing the `PROMISC` flag
it should have in a working setup. More tellingly, `bond0` — the bonded
interface Security Onion's capture processes actually read from — showed
`NO-CARRIER` and `state DOWN`.

`nmcli con show` explained why: a connection profile named
`bond0-slave-ens19` existed (the intended binding of `ens19` into the
bond) but had never been activated — its `DEVICE` column showed `--`.

Attempting to activate it manually failed:
```
$ sudo nmcli con up bond0-slave-ens19
Error: Connection activation failed: Unknown error
```

This exact error text had appeared once before, during the original
Security Onion install, and was dismissed at the time as a transient
false alarm (the install's actual work — 1667 Salt states — had
completed successfully, and `so-status` showed everything healthy). In
hindsight, that dismissal was premature: this looks like the same
underlying problem, just not caught at the time because nothing was
actually being tested against it yet.

Getting the real reason required checking `journalctl -u NetworkManager`
directly (the connection-specific `journalctl -xe NM_CONNECTION=...`
filter suggested in the error hint returned nothing useful — a broader
`--since "5 minutes ago"` filter was what actually surfaced the detail):

```
mtu: failure to set MTU. Did you configure the MTU correctly?
... mtu greater than device maximum
```

Security Onion's own configuration specifies MTU 9000 (jumbo frames) for
the monitor interface — this is Security Onion's documented Proxmox
guidance, not a mistake made in this setup. But the Proxmox bridge
(`vmbr1`) and Security Onion's virtual NIC were both still at Proxmox's
default of 1500. The bond-slave attachment was failing at the MTU-setting
step every time, silently, for the entire life of the install up to this
point.

**Fix:** set `vmbr1`'s MTU to 9000 in Proxmox, left Security Onion's NIC
set to "inherit from bridge" (avoids the two settings drifting apart
later). This required a **full VM restart** to take effect — a live
Proxmox-side MTU change doesn't propagate to an already-running guest,
the same lesson learned earlier with the QEMU guest agent needing a
restart to see a newly-exposed virtio device. After restart, the bond
activated cleanly and `ens19` showed the expected `PROMISC` flag.

## Layer 3: the bridge doesn't mirror unicast traffic by default

With the bond genuinely active, the same ping/nmap test was rerun. This
time, watching Security Onion's capture directly with `tcpdump -i bond0`,
exactly one packet showed up: an ARP broadcast. No ICMP traffic at all
— despite the ping succeeding perfectly (0% loss) from Kali's side.

This turned out to be the real structural gap, and it's a fact about how
Linux bridges work, not a misconfiguration: a standard Linux bridge
forwards unicast traffic directly between the two ports actually
involved in a conversation, based on MAC address learning. It does
**not** flood unicast traffic to every port the way an old-style hub
would. Being in promiscuous mode only affects what a given port sees
*when traffic actually arrives there* — a two-way conversation between
two other ports on the same bridge simply never gets forwarded to a
third, uninvolved port unless something explicitly copies it there.

This is worth being explicit about because it's easy to assume
"promiscuous mode + same bridge" is sufficient for sniffing — it isn't.
Real SPAN/port-mirror configuration is a deliberate, additional step on
top of a switch (physical or virtual), not a side effect of one port
being promiscuous.

**Fix:** explicit Linux `tc` (traffic control) mirroring rules, copying
both Kali's and Metasploitable's tap-interface traffic to Security
Onion's tap interface. Full detail on the exact commands and the
resulting `tc mirred` setup is in `lab-network-topology.md`. Retesting
immediately after applying the rules showed the complete ICMP
conversation in `tcpdump` output — confirmed working.

## Layer 4: confirming the whole pipeline for real

With mirroring in place, re-ran `nmap -sV 10.10.10.20` from Kali and
searched Security Onion's Hunt page again. This time, real data: dozens
of Zeek connection log entries (`zeek.conn`, `zeek.weird`) across every
port the scan probed, and a genuine Suricata alert —
**"ET SCAN Suspicious inbound to PostgreSQL port 5432"** (category:
Potentially Bad Traffic) — correctly triggered by the scan hitting
Metasploitable's exposed PostgreSQL service.

The full chain — isolated network, traffic generation, mirroring, Zeek/
Suricata analysis, visibility in the SOC dashboard — was confirmed
working end to end at this point.

## A fifth thing found afterward: persistence

Once the pipeline worked, one more gap surfaced: the `tc` mirroring
rules applied by hand don't survive a VM restart, since Proxmox destroys
and recreates tap interfaces on every stop/start. This was fixed with a
Proxmox hookscript that reapplies the rules automatically — documented
in full in `lab-network-topology.md` — and verified against both
individual VM restarts and a full Proxmox host reboot. That verification
turned up one more real edge case: a VM that stays running *through* a
host reboot (rather than itself stopping and starting) never triggers
the hookscript, since the `post-start` event it depends on never fires.

## Overall lesson

None of these four (five, counting persistence) issues were visible from
"is the service running?" checks alone — `so-status` reported everything
healthy the entire time. Getting to the real root causes required
checking each layer of the stack directly and skeptically: the search
query's actual results, not just whether it ran; the interface's actual
link state, not just whether the VM booted; the actual packet capture,
not just whether a ping succeeded; and the actual state of mirroring
rules after a real restart, not just after the fix was first applied.
Each fix genuinely was necessary and correct — but each one, on its own,
would have looked like "the" fix without the escalating verification
that this was in fact a five-layer problem.
