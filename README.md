# timesync

**English** | [中文](README.zh-CN.md)

Provisions an X5-class board for one of two PTP roles, and switches between them. The
roles are mutually exclusive: `phc2sys` and `chrony` both steer `CLOCK_REALTIME`, and two
control loops on one clock fight each other.

| | `slave` | `master` |
|---|---|---|
| `ptp4l` | slave (`slaveOnly 1`), never takes over | master / grandmaster (`slaveOnly 0`) |
| `phc2sys` | `-s eth0 -c CLOCK_REALTIME`, PHC to system clock | `-s CLOCK_REALTIME -c eth0`, system clock to PHC |
| `chrony` | stopped and disabled | enabled and started |
| owns `CLOCK_REALTIME` | `phc2sys` (`ptp4l` in software mode) | `chrony` |

`phc2sys` belongs to a hardware-stamping deployment only.

## Quick start

On the board, as root:

```sh
# Follow a PTP master on this network:
sudo ./time_synchronization_installer.sh slave

# Same, but take the timestamps from the NIC's own clock instead of the kernel
# (only on a NIC that really stamps in hardware):
sudo ./time_synchronization_installer.sh slave --time-stamping hardware

# Serve GNSS time instead.  A master needs a recipe: the chrony snippet for this
# receiver.  Start from the template and edit it for your hardware:
cp gnss/refclock.example.conf gnss/refclock.conf
$EDITOR gnss/refclock.conf
sudo timesync master --refclock-file gnss/refclock.conf --gps-device /dev/pps0 \
                     --advertise-gnss-quality

# Ship gate for a production line: 0 = ship, 4 = do not ship, 5 = no GNSS reference:
sudo timesync verify
```

The role is the command: the first run provisions the machine, later runs change the role
and nothing else. `1` and `2` are accepted in place of `slave` and `master`, the command
included. After the first run the tool is installed as `/usr/local/sbin/timesync`.

The default time stamping is `software`, which works on any interface and gives NTP-grade
accuracy. `hardware` is sub-microsecond but needs a NIC that really stamps — see
[NOTES.md](NOTES.md) before choosing it.

## The GNSS recipe

Only the `master` role needs one. It is a chrony snippet naming the receiver's device:

```conf
refclock SHM 0 refid GPS  precision 1e-1 offset 0.0 delay 0.2
refclock PPS /dev/pps0 refid PPS  lock GPS prefer
```

`gnss/refclock.example.conf` in this repo is that file with comments: copy it, keep the
lines matching your receiver, delete the rest. The values to use come from the receiver's
documentation, or from `/etc/chrony/conf.d/` on a bench unit where the same receiver
already works — the better source, because it is known to work with your driver stack.

Check it on one bench unit before it reaches the fleet:

```sh
chronyc sources -v      # the refclock appears, reachability 377
chronyc tracking        # Reference ID is the refclock, Offset in µs
```

Then commit the reviewed file to your deployment repo and pass it with `--refclock-file`
on every board, so the fleet runs identical bytes.

## Commands

| command | what it does |
|---|---|
| `slave` \| `master` | Set the role. Provisions the machine if it is not provisioned, changes the role if it is. |
| `status` | Configured role, time-stamping mode, live state of every piece, recent `ptp4l` output, current system clock. Changes nothing. |
| `verify [slave\|master]` | The ship gate. See below. Changes nothing. |
| `uninstall` | Stop and remove everything this tool installed, and re-enable the NTP units it had disabled. The `linuxptp` package itself is left in place. |
| `wait-iface IFACE [seconds]` | Wait until `IFACE` is up. Used by the generated units, not by operators. |

`verify` checks: the role file, the arguments the units pass to each daemon, every unit
the mode needs active **and** enabled (and the ones it does not need absent), the live
argv of each process, which clock `ptp4l` actually opened, and `chrony`'s state. A unit
that is down fails; a unit that is up now but restarted earlier is only reported, so a
board that caught a transient boot-time restart is not blocked from shipping.

Changing the role keeps everything else: the interface, the domain, the transport, the
time-stamping mode, the recipe, and the GNSS quality setting. So `timesync master` on a
board that was provisioned as a GNSS master brings back the full config without repeating
the flags.

## Options

| option | meaning |
|---|---|
| `--iface IFACE` | Interface PTP runs on. Default: the one recorded in `/etc/default/ptp4l`, else the only interface that can hardware-timestamp (in software mode, with no such interface, the physical interface). |
| `--domain N` | `domainNumber`, 0–127 (default 0). |
| `--transport T` | `UDPv4` \| `UDPv6` \| `L2` (default `UDPv4`). |
| `--time-stamping MODE` | `software` \| `hardware` \| `legacy` (default `software`). Which clock `ptp4l` stamps in. |
| `--refclock-file F` | `master`: a chrony snippet describing this machine's GNSS receiver. See [The GNSS recipe](#the-gnss-recipe). Installed verbatim as `/etc/chrony/conf.d/20-ptp-refclock.conf`. |
| `--gps-device DEV` | `master`, required with `--refclock-file`: the device the receiver is on (`/dev/ttyS0`, `/dev/pps0`, …). It must exist, **and** `F` must mention `DEV`, which catches a recipe pasted onto a differently wired unit. |
| `--advertise-gnss-quality` | `master`, required with a GNSS recipe: have `ptp4l` announce `clockClass 6`, accuracy `0x21`, `timeSource GNSS`. Without it no slave will lock onto this master. |
| `--no-apt` | Never call `apt`; fail the preflight instead if a package is missing. |
| `--no-verify` | Skip the verify that follows the apply. Exit 0 then means only "the steps ran". |
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
| 5 | `verify`: `master` with no GNSS reference at all |
| 130 | interrupted |

## Files it writes

| path | when |
|---|---|
| `/etc/linuxptp/ptp4l-slave.conf` | always; both roles' files are kept up to date |
| `/etc/linuxptp/ptp4l-master.conf` | always (holds the GNSS quality lines when configured) |
| `/etc/chrony/conf.d/20-ptp-refclock.conf` | `master` with a recipe only |
| `/etc/systemd/system/ptp4l@.service` | always; shadows the packaged unit |
| `/etc/systemd/system/phc2sys@.service` | always; shadows the packaged unit, enabled only in hardware mode |
| `/etc/default/ptp4l` | always; **the role and the mode live here** |
| `/usr/local/sbin/timesync` | always; the tool itself, as the short command |
| `/var/lib/timesync/state` | always; what is provisioned here, and by which version |
| `/var/lib/timesync/disabled-ntp-units` | the ledger of clock daemons turned off, so `uninstall` can put them back |

The `-f` list in `/etc/default/ptp4l` selects one of the two role files, and the unit
files carry no PTP knowledge beyond `EnvironmentFile=/etc/default/ptp4l`:

```
ExecStartPre=/usr/local/sbin/timesync wait-iface %I
ExecStart=/usr/sbin/ptp4l $PTP4L_ARGS -i %I

ExecStartPre=/usr/local/sbin/timesync wait-iface %I
ExecStart=/usr/sbin/phc2sys -w $PHC2SYS_ARGS
```

So a role change rewrites one small file and restarts the daemons. `$PTP4L_ARGS` is
written unbraced on purpose: systemd splits an unbraced variable into separate arguments,
and `${PTP4L_ARGS}` would arrive as one argument that `ptp4l` rejects.

## Troubleshooting

```sh
sudo timesync status                      # what role, what mode, what is running
sudo timesync verify                      # why it would not ship
journalctl -u ptp4l@eth0.service -n 50    # what the daemon says
systemctl --failed                        # anything left in a failed state
```

- **`verify` exits 5** — this is a `master` with no GNSS reference. Wire the receiver, then
  pass `--refclock-file`; if the skeleton is intentional for now, `--force` provisions it.
- **`verify` exits 4** — read the `[FAIL]` lines: they name the unit or file at fault.
- **`phc2sys@eth0.service` shows as `failed`** — it was stopped while waiting for a master.
  Switching to `software` stamping clears the record.
- **No `master offset` lines in the ptp4l journal** — the board is a slave and no master is
  answering. With `slaveOnly 1` that is expected: the board waits rather than takes over.

## More

`NOTES.md` has the long version: what each time-stamping mode does on this board, the
device logs from a real deployment, and what is still unproven.
