# timesync

**English** | [中文](README.zh-CN.md)

Provision an X5-class board for one of two mutually exclusive PTP roles, and switch
between them later. One self-contained shell script; no dependency on the VIO stack
or on any other part of this workspace.

The two roles are exclusive by construction, not by convention: both `phc2sys` and
`chrony` are able to steer `CLOCK_REALTIME`, and two control loops on one clock fight
each other. Each role therefore hands `CLOCK_REALTIME` to exactly one owner.

| | `slave` | `master` |
|---|---|---|
| `ptp4l` | slave (`slaveOnly 1`), never takes over | master / grandmaster (`slaveOnly 0`) |
| `phc2sys` | `-s eth0 -c CLOCK_REALTIME` — PHC **→** system clock | `-s CLOCK_REALTIME -c eth0` — system clock **→** PHC |
| `chrony` | stopped and disabled | enabled and started |
| owns `CLOCK_REALTIME` | `phc2sys` | `chrony` |

## The two roles

`slave` — follow a PTP master on the network:

```
PTP master on the network
        │  PTP over UDPv4, domain 0
        ▼
   ptp4l ── steers ──▶ PHC /dev/ptp0
   (slaveOnly 1)              │
                              │  phc2sys -s eth0 -c CLOCK_REALTIME
                              ▼
                        CLOCK_REALTIME            chrony: stopped + disabled
```

`ptp4l` never touches `CLOCK_REALTIME`; it only steers the NIC's hardware clock. It is
`phc2sys` that copies that clock into the system clock, which is what makes `date`
correct.

`master` — serve this board's time to the network, disciplined from GNSS:

```
GNSS receiver
        │  chrony refclock (e.g. SHM / PPS)
        ▼
     chrony ── disciplines ──▶ CLOCK_REALTIME
                                     │
                                     │  phc2sys -s CLOCK_REALTIME -c eth0
                                     ▼
                                PHC /dev/ptp0
                                     │
                                     ▼
                    ptp4l (slaveOnly 0) ── advertises ──▶ network slaves
```

`chrony` disciplines `CLOCK_REALTIME` from the GNSS refclock and never touches the
PHC. It is `phc2sys` that copies the system clock out to the PHC, which is what gives
`ptp4l` something worth advertising.

Without `ptp4l-grandmaster.conf` in the `-f` list, `master` is a **skeleton**: the
defaults (`clockClass 248`, `clockAccuracy 0xFE`) tell every slave "not a reference",
so they will not lock onto it. That file is only added when the install is given a
GNSS recipe plus `--advertise-gnss-quality`, and `verify` refuses to pass a master
that is missing either.

## Quick start

On the board, as root:

```sh
# Follow a PTP master on this network, and survive a reboot:
./time_synchronization_installer.sh install slave

# Later, switch this board to serve GNSS time instead:
timesync switch master --refclock-file ./my-gps.recipe --gps-device /dev/pps0 \
                       --advertise-gnss-quality

# Ship gate for a production line:
timesync verify
```

`slave` and `master` are the spelling everywhere a role is expected. `1` and `2` are
still accepted as shorthands and are normalised to the words before any message is
built. A bare `timesync 1` is refused rather than treated as a switch, so a typo
cannot change the machine.

## Commands

| command | what it does |
|---|---|
| `install slave\|master` | Provision this machine for the role and start it. Converges: writes every config, unit and the switcher, then restarts only what changed. Safe to re-run — a second run over an already-correct machine changes nothing. |
| `switch slave\|master` | Change the role of an **already provisioned** machine. Refuses if `install` has not run yet. |
| `slave` \| `master` | Shorthands for `switch slave` / `switch master`. |
| `status` | Configured role, live state of every piece, recent `ptp4l` output, current system clock. Changes nothing. |
| `verify [slave\|master]` | The production-line gate. Checks the role file, the arguments the units will pass to each daemon, that both units are active **and** enabled, the live argv of each process, hardware timestamping, and chrony's state. Changes nothing. |
| `uninstall` | Stop and remove everything this tool installed, and re-enable the NTP units it had disabled. The `linuxptp` package itself is left in place. |
| `wait-iface IFACE [seconds]` | Wait until `IFACE` is administratively up. This is the `ExecStartPre` of both generated units, not something an operator runs — see [below](#why-both-units-run-timesync-wait-iface). |

`switch` is a role change, not a re-provision: everything that is not the role carries
over — the interface, the domain and transport, and, for a board provisioned as a GNSS
master, the recipe file and the GNSS time quality. That is what lets `switch master`
bring the grandmaster config back without the flags being repeated. A plain `install`
is authoritative instead: giving it no recipe removes any recipe installed earlier, so
what is on disk always matches the role just installed.

## Options

| option | meaning |
|---|---|
| `--iface IFACE` | Interface PTP runs on. Default: the one recorded in `/etc/default/ptp4l`, else the only interface that can hardware-timestamp. |
| `--domain N` | `domainNumber`, 0–127 (default 0). |
| `--transport T` | `UDPv4` \| `UDPv6` \| `L2` (default `UDPv4`). |
| `--refclock-file F` | `master`: a chrony snippet describing this machine's GNSS receiver (its `refclock` line, and a `pps` line if it has one). Installed as `/etc/chrony/conf.d/20-ptp-refclock.conf`. |
| `--gps-device DEV` | `master`, required with `--refclock-file`: the device the receiver is on (`/dev/ttyS0`, `/dev/pps0`, …). Its existence is checked **and** `F` must mention `DEV`, which catches a recipe pasted onto a differently wired unit. |
| `--advertise-gnss-quality` | `master`, required with a GNSS recipe: have `ptp4l` announce `clockClass 6` / accuracy `0x21` / `timeSource GNSS`. |
| `--no-apt` | Never call `apt`; fail the preflight instead if a package is missing. |
| `--no-verify` | Skip the post-install verify. Exit 0 then means only "the apply steps ran", not "the unit is fit to ship". |
| `--force` | Override the checks that are judgement calls rather than facts: a leftover `ptp4l` instance on another interface, a `--gps-device` that does not exist yet, or a `master` install with no GNSS recipe at all. |
| `--root DIR` | Stage the generated tree under `DIR` instead of `/`. `apt`, `systemctl` and every hardware probe are skipped, so the tree can be inspected on a laptop. `install` and `uninstall` only. |
| `--dry-run` | Print every action, change nothing. |

## Exit codes

| code | meaning |
|---|---|
| 0 | success |
| 1 | usage or argument error |
| 2 | machine preflight refused — not root, no interface that can hardware-timestamp, a conflicting PTP instance, an unreadable or mismatched GNSS recipe, missing packages under `--no-apt`, not provisioned yet |
| 3 | a unit failed to restart after the role was applied |
| 4 | `verify`: a check failed |
| 5 | `verify`: `master` with no GNSS reference configured |
| 130 | interrupted |

`verify` exit 0 / 4 / 5 are the three answers a production line needs: fit to ship,
not fit, and honest about being a GNSS-less skeleton.

## What it installs

| path | role |
|---|---|
| `/etc/linuxptp/ptp4l-common.conf` | both — transport, domain, logging |
| `/etc/linuxptp/ptp4l-slave.conf` | both (written always, selected by the `-f` list) |
| `/etc/linuxptp/ptp4l-master.conf` | both (written always, selected by the `-f` list) |
| `/etc/linuxptp/ptp4l-grandmaster.conf` | `master` + `--advertise-gnss-quality` only |
| `/etc/chrony/conf.d/20-ptp-refclock.conf` | `master` with a recipe only |
| `/etc/systemd/system/ptp4l@.service` | both — shadows the packaged unit |
| `/etc/systemd/system/phc2sys@.service` | both — shadows the packaged unit |
| `/etc/default/ptp4l` | both — **the role lives here** |
| `/usr/local/sbin/timesync` | both — the tool itself, as the short command |
| `/var/lib/timesync/state` | both — what was installed, by which version |
| `/var/lib/timesync/disabled-ntp-units` | the ledger of clock daemons turned off, so `uninstall` can put them back |

Both role configs are always written and the `-f` list in `/etc/default/ptp4l` selects
exactly one. A switch therefore never depends on a removal having happened first, and
switching back is guaranteed to find the other file.

Both unit files are role-agnostic — they carry no PTP knowledge beyond
`EnvironmentFile=/etc/default/ptp4l` and the daemon's own arguments:

```
ExecStartPre=/usr/local/sbin/timesync wait-iface %I
ExecStart=/usr/sbin/ptp4l $PTP4L_ARGS -i %I
```

```
ExecStartPre=/usr/local/sbin/timesync wait-iface %I
ExecStart=/usr/sbin/phc2sys -w $PHC2SYS_ARGS
```

So switching rewrites one small file and restarts the daemons: no unit edit, no
enable/disable churn. `$PTP4L_ARGS` is written unbraced on purpose — systemd splits an
unbraced variable into separate arguments, which is how both `-f` flags get through.

## Why both units run `timesync wait-iface`

A NIC's PHC can only be opened once the driver has opened the MAC. On a cold boot
systemd reaches the daemons before that has happened, and both of them die:

```
Sep 16 16:51:44.991509 ubuntu systemd[1]: Started Precision Time Protocol (PTP) service for eth0.
Sep 16 16:51:44.999954 ubuntu ptp4l[1252]: [12.584] selected /dev/ptp0 as PTP clock
Sep 16 16:51:45.010267 ubuntu ptp4l[1252]: [12.594] Failed to open /dev/ptp0: Operation not permitted
Sep 16 16:51:45.010624 ubuntu ptp4l[1252]: failed to create a clock
Sep 16 16:51:45.015977 ubuntu kernel: hobot_gmac 35010000.horizon_tsn eth0: eth_ptp_get_time, device has not been brought up
Sep 16 16:51:45.052360 ubuntu phc2sys[1276]: [12.636] cannot open /dev/ptp0 for eth0: Operation not permitted
Sep 16 16:51:45.338152 ubuntu systemd[1]: ptp4l@eth0.service: Main process exited, code=exited, status=255/EXCEPTION
Sep 16 16:51:45.338765 ubuntu systemd[1]: ptp4l@eth0.service: Failed with result 'exit-code'.
Sep 16 16:51:50.590869 ubuntu systemd[1]: ptp4l@eth0.service: Scheduled restart job, restart counter is at 1.
Sep 16 16:51:50.707231 ubuntu systemd[1]: Started Precision Time Protocol (PTP) service for eth0.
Sep 16 16:51:50.710645 ubuntu ptp4l[3027]: [18.294] selected /dev/ptp0 as PTP clock
Sep 16 16:51:50.712000 ubuntu ptp4l[3027]: [18.296] port 1: INITIALIZING to LISTENING on INIT_COMPLETE
```

Note what this log hides: `Restart=on-failure` masked the whole thing. The unit looked
healthy, and nothing in `systemctl status` would have suggested otherwise, while on
every boot both daemons had been dying and coming back ~6 s later.

`-w` does **not** protect `phc2sys` from this. `phc2sys` resolves `-s eth0` to
`/dev/ptp0` while parsing its options, before any waiting happens, which is why it died
independently of `ptp4l` at almost the same instant.

`After=network-online.target` gave zero protection here: the NetworkManager
wait-online helper is not enabled on this image, so that target is reached at t=0.
`After=sys-subsystem-net-devices-eth0.device` goes active at netdev registration,
which on this board is ~3.5 s — still long before the MAC is up.

`wait-iface` therefore gates on `IFF_UP` (bit 0 of `/sys/class/net/<iface>/flags`,
which is set when `ndo_open` returns, and is set even with no cable plugged in) and
not on `carrier`, so an unplugged board still comes up armed instead of wedging.

## Real device logs

All of the following was captured on a D-Robotics X5 board, Ubuntu 22.04.5 aarch64,
systemd 249.11, `eth0`, no PTP master present on the LAN.

### A fresh install, staged on the device

`--root` writes the whole tree somewhere harmless and reports what a first install
would do — used here so the live board is not disturbed:

```
$ timesync install slave --iface eth0 --root /tmp/freshroot
timesync: --root /tmp/freshroot: staging only; apt, systemctl and hardware checks are skipped
timesync: installing role slave on eth0
timesync: wrote     /etc/linuxptp/ptp4l-common.conf
timesync: wrote     /etc/linuxptp/ptp4l-slave.conf
timesync: wrote     /etc/linuxptp/ptp4l-master.conf
timesync: wrote     /etc/systemd/system/ptp4l@.service
timesync: wrote     /etc/systemd/system/phc2sys@.service
timesync: absent    /etc/linuxptp/ptp4l-grandmaster.conf
timesync: absent    /etc/chrony/conf.d/20-ptp-refclock.conf
timesync: wrote     /usr/local/sbin/timesync
timesync: wrote     /etc/default/ptp4l

timesync: staged under /tmp/freshroot; nothing on this machine was changed
```

Two things are worth reading off that list: both role configs are written even for
`slave`, and the two `absent` lines are the GNSS-only files being deliberately *not*
present in this role.

### Re-running `install` on an already-correct board

Idempotent — everything reports `unchanged`, nothing restarts, and the verify at the
end still runs:

```
$ timesync install slave
          hardware timestamping: yes; PTP hardware clock: /dev/ptp0
timesync: installing role slave on eth0
timesync: unchanged /etc/linuxptp/ptp4l-common.conf
timesync: unchanged /etc/linuxptp/ptp4l-slave.conf
timesync: unchanged /etc/linuxptp/ptp4l-master.conf
timesync: unchanged /etc/systemd/system/ptp4l@.service
timesync: unchanged /etc/systemd/system/phc2sys@.service
timesync: absent    /etc/linuxptp/ptp4l-grandmaster.conf
timesync: absent    /etc/chrony/conf.d/20-ptp-refclock.conf
timesync: wrote     /usr/local/sbin/timesync
timesync: unchanged /etc/default/ptp4l
timesync: already in role slave on eth0; nothing needed restarting

timesync: role set to slave on eth0 (chrony disabled)
          ptp4l@eth0.service and phc2sys@eth0.service are enabled, so this survives a reboot
          run "timesync status" for the live state
```

### `verify` — the production-line gate

```
$ timesync verify slave
role slave on eth0
  [ ok ] role file arguments are the canonical ones for role slave
  [ ok ] common config: transport UDPv4, domain 0
  [ ok ] eth0 hardware-timestamps both directions
  [ ok ] ptp4l@eth0.service active
  [ ok ] ptp4l@eth0.service enabled (survives a reboot)
  [ ok ] ptp4l@eth0.service running as expected (pid 11903)
  [ ok ] ptp4l@eth0.service has not been restarted
  [ ok ] phc2sys@eth0.service active
  [ ok ] phc2sys@eth0.service enabled (survives a reboot)
  [ ok ] phc2sys@eth0.service running as expected (pid 11909)
  [ ok ] phc2sys@eth0.service has not been restarted
  [ ok ] no other PTP instance enabled
  [ ok ] no other clock daemon active or enabled (phc2sys owns CLOCK_REALTIME)
  [ ok ] ptp4l bound the NIC clock: [1708.461] selected /dev/ptp0 as PTP clock

verify: OK -- this unit is fit to ship in role slave
```

The last check reads the journal **of the current boot** (`journalctl -b`) and takes the
newest match, so a `verify` cannot pass by citing a line from a boot that ended hours
ago.

### `status`

```
$ timesync status
Configuration (/etc/default/ptp4l)
  role             slave
  interface        eth0
  ptp4l args       -f /etc/linuxptp/ptp4l-common.conf -f /etc/linuxptp/ptp4l-slave.conf
  phc2sys args     -s eth0 -c CLOCK_REALTIME
  provisioned      by timesync 1.0.0 on 2026-09-16T17:50:11+08:00

Live state
  ptp4l@eth0.service active/enabled
  phc2sys@eth0.service active/enabled
  chrony.service   inactive/disabled

Recent ptp4l output.  A "master offset" line roughly once a second
means it is locked to a master; silence means none has been heard:
  [2487.793] selected local clock 6a6ab8.fffe.db7ec4 as best master
  [2495.638] selected local clock 6a6ab8.fffe.db7ec4 as best master
  [2503.221] selected local clock 6a6ab8.fffe.db7ec4 as best master
  [2510.854] selected local clock 6a6ab8.fffe.db7ec4 as best master
  [2518.718] selected local clock 6a6ab8.fffe.db7ec4 as best master
  [2525.484] selected local clock 6a6ab8.fffe.db7ec4 as best master

System clock now: 2026-09-16T18:44:28+08:00
```

`inactive/disabled` for `chrony.service` is the correct state for `slave`, not a
problem: `phc2sys` owns `CLOCK_REALTIME` in this role.

`selected local clock ... as best master`, repeating, is also expected on a board with
**no PTP master on the network**. `slaveOnly 1` means this board announces that it
would not mind being master, but it will not take the role — so the port stays in
`LISTENING` and no `master offset` line ever appears. On a network with a real master
those lines are replaced by a `master offset` line once a second.

### `switch master` — the plan, without changing the machine

```
$ timesync switch master --force --dry-run
          interface from the existing /etc/default/ptp4l: eth0
          hardware timestamping: yes; PTP hardware clock: /dev/ptp0
timesync: WARNING: eth0 MAC 6a:6a:b8:db:7e:c4 is locally administered, so it is regenerated
          on every boot and the slaves will re-lock once per reboot.  Time sync still runs;
          fix it at provisioning by pinning the MAC if that re-lock matters
timesync: unchanged /etc/linuxptp/ptp4l-common.conf
timesync: unchanged /etc/linuxptp/ptp4l-slave.conf
timesync: unchanged /etc/linuxptp/ptp4l-master.conf
timesync: unchanged /etc/systemd/system/ptp4l@.service
timesync: unchanged /etc/systemd/system/phc2sys@.service
timesync: absent    /etc/linuxptp/ptp4l-grandmaster.conf
timesync: unchanged /usr/local/sbin/timesync
timesync: would enable chrony.service
timesync: would start chrony.service
timesync: would write /etc/default/ptp4l
timesync: would restart ptp4l@eth0.service phc2sys@eth0.service

timesync: role set to master on eth0 (chrony enabled)
          ptp4l@eth0.service and phc2sys@eth0.service are enabled, so this survives a reboot
          run "timesync status" for the live state
timesync: would write /var/lib/timesync/state

timesync: dry run: nothing was changed, so there is nothing to verify
```

`--force` is there because this board has no GNSS recipe; without it the tool refuses
to install a skeleton. Read the two `would` lines together: the role change to `master`
is exactly *enable chrony* + *flip the arguments in `/etc/default/ptp4l`*, nothing more.

### Boot, before and after the gate

Before, with `Restart=on-failure` hiding it — the unit reports `Started`, and both
daemons die immediately after:

```
Sep 16 16:51:44.991509 ubuntu systemd[1]: Started Precision Time Protocol (PTP) service for eth0.
Sep 16 16:51:44.999954 ubuntu ptp4l[1252]: [12.584] selected /dev/ptp0 as PTP clock
Sep 16 16:51:45.010267 ubuntu ptp4l[1252]: [12.594] Failed to open /dev/ptp0: Operation not permitted
Sep 16 16:51:45.010624 ubuntu ptp4l[1252]: failed to create a clock
Sep 16 16:51:45.338152 ubuntu systemd[1]: ptp4l@eth0.service: Main process exited, code=exited, status=255/EXCEPTION
Sep 16 16:51:45.338765 ubuntu systemd[1]: ptp4l@eth0.service: Failed with result 'exit-code'.
Sep 16 16:51:50.590869 ubuntu systemd[1]: ptp4l@eth0.service: Scheduled restart job, restart counter is at 1.
Sep 16 16:51:50.707231 ubuntu systemd[1]: Started Precision Time Protocol (PTP) service for eth0.
Sep 16 16:51:50.710645 ubuntu ptp4l[3027]: [18.294] selected /dev/ptp0 as PTP clock
Sep 16 16:51:50.712000 ubuntu ptp4l[3027]: [18.296] port 1: INITIALIZING to LISTENING on INIT_COMPLETE
```

After, on the current boot. The gap between `Starting` and `Started` is `wait-iface`
holding the unit back until the MAC is up:

```
Sep 16 18:02:29.114172 ubuntu systemd[1]: Starting Precision Time Protocol (PTP) service for eth0...
Sep 16 18:02:33.350292 ubuntu systemd[1]: Started Precision Time Protocol (PTP) service for eth0.
Sep 16 18:02:33.364375 ubuntu ptp4l[2686]: [16.994] selected /dev/ptp0 as PTP clock
Sep 16 18:02:33.365490 ubuntu ptp4l[2686]: [16.995] port 1: INITIALIZING to LISTENING on INIT_COMPLETE
```

4.2 s of waiting instead of an EPERM, a failed unit and a restart counter. `ptp4l` goes
straight to `LISTENING` on the first attempt, and `NRestarts` stays at 0:

```
$ systemctl show ptp4l@eth0.service phc2sys@eth0.service -p NRestarts
NRestarts=0
NRestarts=0
```

`phc2sys`, meanwhile, sits dormant by design — its `-w` holds it back until `ptp4l`
reports a synchronized state, which will not happen until a master exists:

```
Sep 16 18:02:33.393268 ubuntu systemd[1]: Started Synchronize system clock or PTP hardware clock (PHC).
Sep 16 18:02:34.402287 ubuntu phc2sys[2713]: [18.032] Waiting for ptp4l...
Sep 16 18:02:35.403621 ubuntu phc2sys[2713]: [19.033] Waiting for ptp4l...
```

That is why this role is safe to leave running on a board whose master is not there
yet: while it waits, it touches no clock.

## Notes and limitations

- **A locally administered MAC is reported, never gated.** On this board the MAC is
  regenerated on every boot, so the `grandmasterIdentity` changes each reboot and the
  slaves re-lock once per reboot. Time synchronisation still runs, so this is not a
  reason to refuse a deployment — but if that re-lock matters, pin the MAC at
  provisioning (`cloned-mac-address`, or a fixed address written at boot).
- **The slave role's lock is not proven by these logs.** There is no PTP master on the
  LAN this was developed against, so the logs show a healthy, `LISTENING`, waiting
  slave — not a locked one. Locking has to be confirmed against a real master.
- **`master` is a skeleton until GNSS is wired.** Without a recipe and
  `--advertise-gnss-quality`, `verify` returns 5 by design rather than pretending.
- **The unit files shadow the packaged ones.** A file in `/etc/systemd/system/` shadows
  the one in `/lib/systemd/system/` entirely, which is required here (the packaged
  units hardcode the packaged config path and have no `Restart=`). The trade-off is
  that a future `linuxptp` package update to those units will not arrive; the files are
  short, so diff them after an upgrade if that matters.
- **`CLOCK_REALTIME` has exactly one owner per role**, and that is the invariant to
  preserve when editing anything here.
