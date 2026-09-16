# timesync

**English** | [中文](README.zh-CN.md)

A single shell script that provisions an X5-class board for one of two PTP roles and
switches between them. It does not depend on the VIO stack or on anything else in this
workspace.

The two roles are mutually exclusive. `phc2sys` and `chrony` can both steer
`CLOCK_REALTIME`, and two control loops on one clock fight each other, so every role
gives `CLOCK_REALTIME` to exactly one owner.

| | `slave` | `master` |
|---|---|---|
| `ptp4l` | slave (`slaveOnly 1`), never takes over | master / grandmaster (`slaveOnly 0`) |
| `phc2sys` | `-s eth0 -c CLOCK_REALTIME`, PHC to system clock | `-s CLOCK_REALTIME -c eth0`, system clock to PHC |
| `chrony` | stopped and disabled | enabled and started |
| owns `CLOCK_REALTIME` | `phc2sys` (`ptp4l` in software mode) | `chrony` |

`phc2sys` is part of a hardware-stamping deployment only. See the next section.

## Time stamping

`--time-stamping` picks which clock `ptp4l` reads its packet timestamps from, and that
decides which clock it ends up disciplining. Default is `software`, which is the mode
that works everywhere.

| mode | `ptp4l` stamps in | it disciplines | second daemon | accuracy |
|---|---|---|---|---|
| `software` (default) | kernel, `CLOCK_REALTIME` | `CLOCK_REALTIME` | none | tens of µs, NTP-grade |
| `hardware` | the NIC's own clock (`/dev/ptp0`) | the PHC | `phc2sys` | sub-µs |
| `legacy` | kernel, `CLOCK_REALTIME` | `CLOCK_REALTIME` | none | deprecated |

`software` uses whatever interface you point it at. The kernel timestamps the packets
and `ptp4l` steers `CLOCK_REALTIME` directly, so there is no PTP hardware clock in the
loop and nothing for `phc2sys` to copy.

`hardware` needs a NIC that really stamps in silicon. `ethtool -T` advertising
`hardware-transmit` and `hardware-receive` does not establish that, because the driver
advertises the capability and the silicon then returns timestamps that are zero or
wrong. The only way to know is to run it against a master and read the offset.

`legacy` is the old kernel timestamping interface. Its clocks work like `software`, but
the driver has to support it. This board's driver refuses it outright:

```
[4675.287] interface 'eth0' does not support requested timestamping mode
failed to create a clock
```

The mode is chosen with a command-line flag (`-S`, `-H`, `-L`) rather than a line in the
config file. A flag cannot be silently discarded the way a config key can, and it is
what you see in `ps` when a clock is not being corrected. `verify` checks the mode from
three directions: the recorded value, the arguments the unit passes, and the `/dev/ptp*`
descriptors the running `ptp4l` holds open.

The mode is recorded in `/var/lib/timesync/state` and carried across a role change, so
`timesync slave` does not reset it.

## The two roles

`slave`, hardware stamping:

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

`ptp4l` does not touch `CLOCK_REALTIME` here; it steers the NIC clock. `phc2sys` copies
that clock into the system clock, which is what makes `date` correct.

`slave`, software stamping (the default): `ptp4l` takes its timestamps in
`CLOCK_REALTIME` and steers that clock itself. `phc2sys` is disabled, and the diagram
collapses to the top half.

`master`, hardware stamping:

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

`chrony` disciplines `CLOCK_REALTIME` from the GNSS refclock and leaves the PHC alone.
`phc2sys` copies the system clock out to the PHC, which gives `ptp4l` something to
advertise.

A `master` without the GNSS quality lines is a skeleton. The defaults
(`clockClass 248`, `clockAccuracy 0xFE`) mean "not a reference", so no slave will lock
onto it. Those lines are added only when the command is given a GNSS recipe plus
`--advertise-gnss-quality`, and `verify` returns 5 for a master missing either.

## Quick start

On the board, as root:

```sh
# Follow a PTP master on this network, software stamping (the default).
# On a machine that is not provisioned yet this installs and starts everything;
# on one that already runs this tool it changes the role:
./time_synchronization_installer.sh slave

# The same, but taking timestamps from the NIC clock, on a board that really
# hardware-stamps:
./time_synchronization_installer.sh slave --time-stamping hardware

# Later, this board serves GNSS time instead.  The role is the only thing that
# changes; the interface, the mode and the rest carry over.  The master role
# needs a GNSS recipe: a chrony snippet describing this receiver.  Start from
# the template in this repo and edit it for your hardware:
cp gnss/refclock.example.conf gnss/refclock.conf
$EDITOR gnss/refclock.conf
timesync master --refclock-file gnss/refclock.conf --gps-device /dev/pps0 \
                --advertise-gnss-quality

# Ship gate for a production line:
timesync verify
```

The role is the command, and that is the whole interface. `1` and `2` are still accepted
wherever a role is expected, including as the command itself, and are normalised to the
words before any message is built.

## The GNSS recipe file

`--refclock-file` takes a **chrony snippet**: the `refclock` lines that tell chrony
where this machine's GNSS receiver is. It is the one input the tool cannot write for
you, because it is a property of the receiver, not of the board — which device the
receiver presents (`/dev/pps0`, a serial port reached through gpsd's SHM, a PHC index)
and which parameters that model needs. Your file is installed verbatim as
`/etc/chrony/conf.d/20-ptp-refclock.conf`. The tool checks three things about it and
nothing else: that it is readable, that it holds a `refclock` line, and that it mentions
the `--gps-device` you passed. That last check is why a gpsd/SHM recipe — which names no
serial port — should note the port in a comment; the template shows where.

A receiver with a PPS output and gpsd-fed NMEA over SHM:

```conf
# gnss/refclock.conf -- chrony snippet for <receiver model> on /dev/pps0
refclock SHM 0 refid GPS  precision 1e-1 offset 0.0 delay 0.2
refclock PPS /dev/pps0 refid PPS  lock GPS prefer
```

That file is not something you have to invent from nothing: this repo ships a
commented copy of it as **`gnss/refclock.example.conf`**. Copy it, keep the lines that
match your receiver, delete the rest, and fix the device names. The comments in it
explain each line.

Where the real values come from:

1. **The receiver's documentation.** A GNSS module datasheet or manual normally
   carries a chrony or gpsd example for exactly that module; copy it, then correct the
   device names for this board's wiring.
2. **A bench unit where the receiver already works.** If chrony is already disciplining
   from it somewhere, that machine's `/etc/chrony/conf.d/` holds the recipe — take the
   file. This is the more reliable of the two, because it is known to work with your
   driver stack rather than with the vendor's.

Validate it on one bench unit before it reaches the fleet. Two questions: does chrony
see the reference, and does the system clock follow it?

```sh
chronyc sources -v      # the refclock appears with a reachability of 377
chronyc tracking        # "Reference ID" is the refclock; Offset settles to µs
```

Then let the tool gate it: `timesync verify` returns 5 while a master has no GNSS
reference, and stops doing so once chrony is locked to yours. Commit the reviewed recipe
to your deployment repo — that file, not the module datasheet, is what production ships,
so the whole fleet gets the same reviewed bytes.

## Commands

| command | what it does |
|---|---|
| `slave` \| `master` | **The role, and the whole interface.** On a machine that is not provisioned yet the command installs everything and starts it; on a machine already running this tool it changes the role and nothing else. Running the command for the role the machine is already in is a no-op. |
| `status` | Configured role, time-stamping mode, live state of every piece, recent `ptp4l` output, current system clock. Changes nothing. |
| `verify [slave\|master]` | The production-line gate. Checks the role file, the arguments the units pass to each daemon, every unit the mode needs active **and** enabled (and the ones it does not need absent), the live argv of each process, which clock `ptp4l` actually opened, and `chrony`'s state. Changes nothing. |
| `uninstall` | Stop and remove everything this tool installed, and re-enable the NTP units it had disabled. The `linuxptp` package itself is left in place. |
| `wait-iface IFACE [seconds]` | Wait until `IFACE` is administratively up. This is the `ExecStartPre` of the generated units, not something an operator runs. See [below](#why-the-units-run-timesync-wait-iface). |

The role command converges rather than appends: every config, unit and the switcher are
(re)written, and only what changed is restarted, so a second run over an already-correct
machine changes nothing.

Everything that is not the role carries over. The interface, the domain, the transport
and the time-stamping mode all do. For a board provisioned as a GNSS master, so do the
recipe file and the GNSS time quality, which is what lets `timesync master` bring the
grandmaster config back without repeating the flags. The recipe is never removed by a
role change — the wiring on the board does not change when its role does — so `uninstall`
is what clears it, or pass a new `--refclock-file` to replace it.

## Options

| option | meaning |
|---|---|
| `--iface IFACE` | Interface PTP runs on. Default: the one recorded in `/etc/default/ptp4l`, else the only interface that can hardware-timestamp (in software mode, with no such interface, the physical interface). |
| `--domain N` | `domainNumber`, 0–127 (default 0). |
| `--transport T` | `UDPv4` \| `UDPv6` \| `L2` (default `UDPv4`). |
| `--time-stamping MODE` | `software` \| `hardware` \| `legacy` (default `software`). Which clock `ptp4l` stamps in, as described [above](#time-stamping). |
| `--refclock-file F` | `master`: a chrony snippet describing this machine's GNSS receiver (its `refclock` lines, and a `pps` line if it has one). See [The GNSS recipe file](#the-gnss-recipe-file) for what it contains and where to get it. Installed verbatim as `/etc/chrony/conf.d/20-ptp-refclock.conf`. |
| `--gps-device DEV` | `master`, required with `--refclock-file`: the device the receiver is on (`/dev/ttyS0`, `/dev/pps0`, …). Its existence is checked **and** `F` must mention `DEV`, which catches a recipe pasted onto a differently wired unit. |
| `--advertise-gnss-quality` | `master`, required with a GNSS recipe: have `ptp4l` announce `clockClass 6`, accuracy `0x21`, `timeSource GNSS`. |
| `--no-apt` | Never call `apt`; fail the preflight instead if a package is missing. |
| `--no-verify` | Skip the post-install verify. Exit 0 then means only "the apply steps ran", not "the unit is fit to ship". |
| `--force` | Override the checks that are judgement calls rather than facts: a leftover `ptp4l` instance on another interface, a `--gps-device` that does not exist yet, or a `master` run with no GNSS recipe at all. |
| `--root DIR` | Stage the generated tree under `DIR` instead of `/`. `apt`, `systemctl` and every hardware probe are skipped, so the tree can be inspected on a laptop. The role command and `uninstall` only. |
| `--dry-run` | Print every action, change nothing. |

## Exit codes

| code | meaning |
|---|---|
| 0 | success |
| 1 | usage or argument error |
| 2 | machine preflight refused: not root, no usable interface, a conflicting PTP instance, an unreadable or mismatched GNSS recipe, missing packages under `--no-apt`, not provisioned yet |
| 3 | a unit failed to restart after the role was applied |
| 4 | `verify`: a check failed |
| 5 | `verify`: `master` with no GNSS reference configured |
| 130 | interrupted |

`verify` exit 0, 4 and 5 are the three answers a production line needs: fit to ship, not
fit, and honest about being a GNSS-less skeleton.

## What it installs

| path | when |
|---|---|
| `/etc/linuxptp/ptp4l-slave.conf` | always, both roles' files are kept up to date |
| `/etc/linuxptp/ptp4l-master.conf` | always (includes the GNSS quality lines when configured) |
| `/etc/chrony/conf.d/20-ptp-refclock.conf` | `master` with a recipe only |
| `/etc/systemd/system/ptp4l@.service` | always, shadows the packaged unit |
| `/etc/systemd/system/phc2sys@.service` | always, shadows the packaged unit; enabled only in hardware mode |
| `/etc/default/ptp4l` | always; **the role and the mode live here** |
| `/usr/local/sbin/timesync` | always; the tool itself, as the short command |
| `/var/lib/timesync/state` | always; what is provisioned here, and by which version |
| `/var/lib/timesync/disabled-ntp-units` | the ledger of clock daemons turned off, so `uninstall` can put them back |

Both role configs are always written, and the `-f` list in `/etc/default/ptp4l` selects
exactly one. A role change therefore never depends on a removal having happened first,
and changing back is guaranteed to find the other file.

The unit files carry no PTP knowledge beyond `EnvironmentFile=/etc/default/ptp4l` and
the daemon's own arguments:

```
ExecStartPre=/usr/local/sbin/timesync wait-iface %I
ExecStart=/usr/sbin/ptp4l $PTP4L_ARGS -i %I
```

```
ExecStartPre=/usr/local/sbin/timesync wait-iface %I
ExecStart=/usr/sbin/phc2sys -w $PHC2SYS_ARGS
```

So a role change rewrites one small file and restarts the daemons. No unit edit, no
enable/disable churn. `$PTP4L_ARGS` is written unbraced on purpose: systemd splits an
unbraced variable into separate arguments, and `${PTP4L_ARGS}` would arrive as one
argument that `ptp4l` rejects.

## Why one config file per role, not a shared file plus a role file

`ptp4l` merges multiple `-f` files by discarding the earlier ones. With `-f a -f b`, a
key that only `a` sets is not in effect at all. This was measured on the board rather
than taken from the man page: a `domainNumber 5` set in the first file read back as
`domainNumber 0` from the running daemon with `pmc -u -b 0 'GET DEFAULT_DATA_SET'`. Keys
inside a single file do accumulate, including across two `[global]` sections.

An earlier version of this tool passed `-f ptp4l-common.conf -f ptp4l-<role>.conf`, so
the whole common file was ignored and `--domain` and `--transport` silently did nothing.
The deployment looked correct only because those values happened to match `ptp4l`'s
defaults. Each role therefore now gets one self-contained file, and the role command
deletes the two files the old layout used so nothing on disk is read by nobody.

## Why the units run `timesync wait-iface`

A NIC's PHC can only be opened once the driver has opened the MAC. On a cold boot
systemd reaches the daemons before that has happened, and both die:

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

`Restart=on-failure` hid the failure. The unit reported `Started`, and nothing in
`systemctl status` suggested a problem, while on every boot both daemons died and came
back about 6 s later.

`-w` does not protect `phc2sys` from this. `phc2sys` resolves `-s eth0` to `/dev/ptp0`
while parsing its options, before any waiting happens, so it died independently of
`ptp4l` at almost the same instant.

`After=network-online.target` gave no protection either: the NetworkManager wait-online
helper is not enabled on this image, so that target is reached at t=0.
`After=sys-subsystem-net-devices-eth0.device` goes active at netdev registration, which
on this board is about 3.5 s, still well before the MAC is up.

The gate is kept in every mode, not just hardware. In software mode there is no PHC to
open, but `ptp4l` still binds a PTP socket to the interface, which fails the same way.
`wait-iface` gates on `IFF_UP` (bit 0 of `/sys/class/net/<iface>/flags`, set when
`ndo_open` returns, and set even with no cable plugged in) and not on `carrier`, so an
unplugged board still comes up armed instead of wedging.

## Real device logs

All of the following was captured on a D-Robotics X5 board running Ubuntu 22.04.5
aarch64, systemd 249.11, linuxptp 3.1.1, `eth0`, with no PTP master present on the LAN.

### Provisioning a fresh machine, software stamping

```
# timesync slave
          interface from the existing /etc/default/ptp4l: eth0
          time stamping: software (the NIC also advertises hardware timestamping; that is not proof it works)
timesync: installing role slave on eth0
timesync: wrote     /etc/linuxptp/ptp4l-slave.conf
timesync: wrote     /etc/linuxptp/ptp4l-master.conf
timesync: wrote     /etc/systemd/system/ptp4l@.service
timesync: wrote     /etc/systemd/system/phc2sys@.service
timesync: removed   /etc/linuxptp/ptp4l-common.conf
timesync: absent    /etc/linuxptp/ptp4l-grandmaster.conf
timesync: absent    /etc/chrony/conf.d/20-ptp-refclock.conf
timesync: wrote     /usr/local/sbin/timesync
timesync: wrote     /etc/default/ptp4l
timesync: disable phc2sys@eth0.service (time stamping is software: there is no second clock for it to copy)
timesync: restart ptp4l@eth0.service

timesync: role set to slave on eth0 (chrony disabled, time stamping software)
          ptp4l@eth0.service enabled, so this survives a reboot
          run "timesync status" for the live state

role slave on eth0, time stamping software
  [ ok ] role file arguments are the canonical ones for role slave
  [ ok ] role config: transport UDPv4, domain 0
  [ ok ] eth0: software time stamping in use (the NIC also advertises hardware timestamping, which is not the same as delivering it)
  [ ok ] ptp4l@eth0.service active
  [ ok ] ptp4l@eth0.service enabled (survives a reboot)
  [ ok ] ptp4l@eth0.service running as expected (pid 27322)
  [ ok ] ptp4l@eth0.service has not been restarted
  [ ok ] phc2sys@eth0.service is not active or enabled (not needed in software mode)
  [ ok ] no other PTP instance enabled
  [ ok ] no other clock daemon active or enabled (ptp4l owns CLOCK_REALTIME)
  [ ok ] ptp4l stamps in CLOCK_REALTIME, not in a NIC clock (pid 27322 holds no /dev/ptp* descriptor, as software mode requires)

verify: OK -- this unit is fit to ship in role slave
```

The last check is the one that matters. It reads `/proc/<pid>/fd` of the running
`ptp4l`, so it reports the clock the daemon actually opened rather than the clock the
arguments asked for.

### `status`

```
# timesync status
Configuration (/etc/default/ptp4l)
  role             slave
  interface        eth0
  time stamping    software (kernel, in CLOCK_REALTIME -- no NIC clock involved)
  ptp4l args       -f /etc/linuxptp/ptp4l-slave.conf -S
  phc2sys args     (none -- not used in software mode)
  provisioned      by timesync 1.1.0 on 2026-09-16T17:50:11+08:00

Live state
  ptp4l@eth0.service active/enabled
  phc2sys@eth0.service inactive/disabled
  chrony.service   inactive/disabled

Recent ptp4l output.  A "master offset" line roughly once a second
means it is locked to a master; silence means none has been heard:
  ptp4l@eth0.service: Deactivated successfully.
  Stopped Precision Time Protocol (PTP) service for eth0.
  Starting Precision Time Protocol (PTP) service for eth0...
  Started Precision Time Protocol (PTP) service for eth0.
  [4615.780] port 1: INITIALIZING to LISTENING on INIT_COMPLETE
  [4615.781] port 0: INITIALIZING to LISTENING on INIT_COMPLETE

System clock now: 2026-09-16T19:18:07+08:00
```

`inactive/disabled` for `chrony.service` and `phc2sys@eth0.service` is correct for this
role in this mode, not a problem: `ptp4l` owns `CLOCK_REALTIME` here.

Going to `LISTENING` and staying there is also expected with no PTP master on the
network. `slaveOnly 1` means the board will not take the master role, so no
`master offset` line ever appears. On a network with a real master those lines are
replaced by a `master offset` line once a second.

### Changing the role to `master`, on a board installed by version 1.0.0

Version 1.0.0 recorded no time-stamping mode, so the mode is recovered from the
arguments the units are actually running with. Nothing changes role by accident:

```
$ timesync master --force --dry-run
          keeping time stamping hardware from the existing install
          interface from the existing /etc/default/ptp4l: eth0
          time stamping: hardware; PTP hardware clock: /dev/ptp0
timesync: WARNING: eth0 MAC 6a:6a:b8:db:7e:c4 is locally administered, so it is regenerated on every boot and the slaves will re-lock once per reboot.  Time sync still runs; fix it at provisioning by pinning the MAC if that re-lock matters
timesync: would write /etc/linuxptp/ptp4l-slave.conf
timesync: would write /etc/linuxptp/ptp4l-master.conf
timesync: unchanged /etc/systemd/system/ptp4l@.service
timesync: unchanged /etc/systemd/system/phc2sys@.service
timesync: would remove /etc/linuxptp/ptp4l-common.conf
timesync: absent    /etc/linuxptp/ptp4l-grandmaster.conf
timesync: would write /usr/local/sbin/timesync
timesync: would run systemctl daemon-reload
timesync: would enable chrony.service
timesync: would start chrony.service
timesync: would write /etc/default/ptp4l
timesync: would restart ptp4l@eth0.service phc2sys@eth0.service

timesync: role set to master on eth0 (chrony enabled, time stamping hardware)
          ptp4l@eth0.service phc2sys@eth0.service enabled, so this survives a reboot
          run "timesync status" for the live state
timesync: would write /var/lib/timesync/state

timesync: dry run: nothing was changed, so there is nothing to verify
```

Read the `would` lines as a group. Changing the role is *enable chrony*, *rewrite
`/etc/default/ptp4l`*, *restart*, and one cleanup of the file the old layout left behind.

### Back to `slave` with `--time-stamping hardware`

Changing the mode back re-enables `phc2sys` and both daemons come up:

```
role slave on eth0, time stamping hardware
  [ ok ] role file arguments are the canonical ones for role slave
  [ ok ] role config: transport UDPv4, domain 0
  [ ok ] eth0 hardware-timestamps both directions
  [ ok ] ptp4l@eth0.service active
  [ ok ] ptp4l@eth0.service enabled (survives a reboot)
  [ ok ] ptp4l@eth0.service running as expected (pid 28530)
  [ ok ] ptp4l@eth0.service has not been restarted
  [ ok ] phc2sys@eth0.service active
  [ ok ] phc2sys@eth0.service enabled (survives a reboot)
  [ ok ] phc2sys@eth0.service running as expected (pid 28536)
  [ ok ] phc2sys@eth0.service has not been restarted
  [ ok ] no other PTP instance enabled
  [ ok ] no other clock daemon active or enabled (phc2sys owns CLOCK_REALTIME)
  [ ok ] ptp4l holds the NIC clock open (/proc/28530/fd -> /dev/ptp*)
  [ ok ] ptp4l bound the NIC clock: [4653.344] selected /dev/ptp0 as PTP clock

verify: OK -- this unit is fit to ship in role slave
```

Note the ownership line: in hardware mode it names `phc2sys`, in software mode `ptp4l`.
Both modes assert the same invariant with the name of whoever holds it.

### `--time-stamping legacy` on this board fails loudly

`verify` returns 4 and the journal says why, so a legacy deployment cannot ship quietly:

```
  [ ok ] eth0 hardware-timestamps both directions
  [FAIL] ptp4l@eth0.service is not active (activating)
  [ ok ] ptp4l@eth0.service enabled (survives a reboot)
  [FAIL] ptp4l@eth0.service has no main process (MainPID=0)
  [FAIL] ptp4l@eth0.service has restarted 1 time(s) -- it is failing; see journalctl -u ptp4l@eth0.service -n 50
  [ ok ] phc2sys@eth0.service is not active or enabled (not needed in legacy mode)
  [ ok ] no other PTP instance enabled
  [ ok ] no other clock daemon active or enabled (ptp4l owns CLOCK_REALTIME)
  [skip] ptp4l@eth0.service has no main process, so the clock it opened cannot be checked

verify: FAILED
```

The interface capability check passes, because this NIC advertises hardware timestamping.
Only the daemon refuses the mode, which is why `verify` checks the running process and
not just the advertised capabilities. Changing back to `hardware` recovers in one
command.

### Boot, before and after the gate

Before, with `Restart=on-failure` masking it:

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
straight to `LISTENING` on the first attempt and `NRestarts` stays at 0:

```
$ systemctl show ptp4l@eth0.service phc2sys@eth0.service -p NRestarts
NRestarts=0
NRestarts=0
```

`phc2sys` sits dormant by design. Its `-w` holds it back until `ptp4l` reports a
synchronized state, which will not happen until a master exists:

```
Sep 16 18:02:33.393268 ubuntu systemd[1]: Started Synchronize system clock or PTP hardware clock (PHC).
Sep 16 18:02:34.402287 ubuntu phc2sys[2713]: [18.032] Waiting for ptp4l...
Sep 16 18:02:35.403621 ubuntu phc2sys[2713]: [19.033] Waiting for ptp4l...
```

## Notes and limitations

- **Hardware stamping is not proven on this board.** The driver advertises hardware
  transmit and receive timestamping, `ptp4l -H` binds `/dev/ptp0` and holds it open, and
  `verify` passes. Whether the MAC really inserts timestamps cannot be established
  without a PTP master on the LAN, and a driver advertising the capability while the
  silicon returns zero timestamps is a known failure mode on SoCs. Development against
  a real master, reading the offset, is the check that settles it. The default is
  `software` partly for this reason.
- **The slave role's lock is not proven by these logs.** No PTP master was present, so
  the logs show a healthy, waiting slave, not a locked one.
- **A locally administered MAC is reported, never gated.** This board's MAC is
  regenerated on every boot, so `grandmasterIdentity` changes each reboot and slaves
  re-lock once per reboot. Time synchronisation still runs, so this is not a reason to
  refuse a deployment; pin the MAC at provisioning if the re-lock matters.
- **`master` is a skeleton until GNSS is wired.** Without a recipe and
  `--advertise-gnss-quality`, `verify` returns 5 by design.
- **The unit files shadow the packaged ones.** A file in `/etc/systemd/system/` shadows
  the one in `/lib/systemd/system/` entirely, which is required here: the packaged units
  hardcode the packaged config path and ship no `Restart=`. A future `linuxptp` package
  update to those units will not arrive; the files are short, so diff them after an
  upgrade if that matters.
- **`CLOCK_REALTIME` has exactly one owner per role and mode.** That is the invariant to
  preserve when editing anything here.
