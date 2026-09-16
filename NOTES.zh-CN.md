# timesync：详细说明与板端日志

第一次部署看 [README.zh-CN.md](README.zh-CN.md)。这里放长版本：每种模式为什么是这个行为、在真机上量到了什么、以及哪些还没有被证明。

---

## 时间戳模式

`--time-stamping` 决定 `ptp4l` 从哪里取包的时戳，也就决定了它最终去调哪个钟。默认
`software`，这个模式在哪都能用。

| 模式 | `ptp4l` 在哪里打时戳 | 它调哪个钟 | 第二个守护进程 | 精度 |
|---|---|---|---|---|
| `software`（默认） | 内核，`CLOCK_REALTIME` | `CLOCK_REALTIME` | 无 | 几十微秒，NTP 级 |
| `hardware` | 网卡自己的时钟（`/dev/ptp0`） | PHC | `phc2sys` | 亚微秒 |
| `legacy` | 内核，`CLOCK_REALTIME` | `CLOCK_REALTIME` | 无 | 已废弃 |

`software` 对网口没有要求。内核给包打时戳，`ptp4l` 直接调 `CLOCK_REALTIME`，整条链里
没有 PTP 硬件时钟，`phc2sys` 也就没东西可搬。

`hardware` 要求网卡真的在硅片里打时戳。`ethtool -T` 报出 `hardware-transmit`、
`hardware-receive` 并不说明这件事：驱动会声明这个能力，而硅片返回的时戳可能是零或者
是错的。唯一的确认办法是接上真实的 master 跑起来看 offset。

`legacy` 是老的取时戳接口，时钟关系跟 `software` 一样，但要求驱动支持。这块板子的驱动
直接拒绝：

```
[4675.287] interface 'eth0' does not support requested timestamping mode
failed to create a clock
```

模式走命令行参数（`-S`、`-H`、`-L`），不写在配置文件里。参数不会被悄悄丢掉（配置项
会），而且它直接出现在 `ps` 里，守护进程实际拿到的是什么一望便知。`verify` 从三个方向核
对模式：记录下来的值、unit 实际传的参数、以及运行中的 `ptp4l` 打开着哪些 `/dev/ptp*`。

模式记在 `/var/lib/timesync/state` 里，改角色时会带过去，所以 `timesync slave` 不会把它
重置掉。

## 两个角色

`slave`，hardware 打时戳：

```
网络上的 PTP master
        │  PTP over UDPv4, domain 0
        ▼
   ptp4l ── 调 ──▶ PHC /dev/ptp0
   (slaveOnly 1)          │
                          │  phc2sys -s eth0 -c CLOCK_REALTIME
                          ▼
                    CLOCK_REALTIME            chrony: 停止 + 禁用
```

这里 `ptp4l` 不碰 `CLOCK_REALTIME`，它调的是网卡时钟。`phc2sys` 把那个钟搬进系统钟，
`date` 才对得上。

`slave`，software 打时戳（默认）：`ptp4l` 在 `CLOCK_REALTIME` 里取时戳，也由它自己调这
个钟，`phc2sys` 被禁用，图只剩上面那一半。

`master`，hardware 打时戳：

```
GNSS 接收机
        │  chrony refclock（例如 SHM / PPS）
        ▼
     chrony ── 调 ──▶ CLOCK_REALTIME
                           │
                           │  phc2sys -s CLOCK_REALTIME -c eth0
                           ▼
                      PHC /dev/ptp0
                           │
                           ▼
          ptp4l (slaveOnly 0) ── 对外宣告 ──▶ 网络上的从钟
```

`chrony` 用 GNSS 参考源调 `CLOCK_REALTIME`，不碰 PHC。`phc2sys` 把系统钟搬到 PHC，
`ptp4l` 才有值得对外宣告的时间。

没有 GNSS 质量行的 `master` 是个空壳。默认值（`clockClass 248`、`clockAccuracy 0xFE`）
的意思是"我不是参考源"，从钟不会锁它。只有给定了 GNSS 配方加 `--advertise-gnss-quality`
才会加上那几行。`verify` 只在完全没有 GNSS 参考源时返回 5；有配方但漏了
`--advertise-gnss-quality` 属于普通的检查不过，返回 4。

## 为什么一个角色一个配置文件，而不是公共文件加角色文件

`ptp4l` 合并多个 `-f` 文件的方式是**丢掉前面的**。给 `-f a -f b`，只写在 `a` 里的配置项
根本不起作用。这一条是在板子上量出来的，不是从 man 手册上抄的：第一个文件里写
`domainNumber 5`，用 `pmc -u -b 0 'GET DEFAULT_DATA_SET'` 从运行中的守护进程读回来是
`domainNumber 0`。同一个文件内部是会累加的，包括跨两个 `[global]` 段。

本工具更早的版本传的是 `-f ptp4l-common.conf -f ptp4l-<role>.conf`，于是整个公共文件被
忽略，`--domain` 和 `--transport` 静悄悄地什么都不做。当时的部署看起来是对的，只是因为
那几个值恰好等于 `ptp4l` 的默认值。现在每个角色一个自包含文件，并且角色命令会删掉旧布
局留下的那两个文件，避免磁盘上有没人读的配置。

## 为什么 unit 要跑 `timesync wait-iface`

网卡的 PHC 只有在驱动把 MAC 打开之后才能打开。冷启动时 systemd 会在那之前就启动守护
进程，两个都会死：

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

`Restart=on-failure` 把这事盖住了。unit 报 `Started`，`systemctl status` 里看不出任何异
常，而每次开机两个守护进程都死一次、约 6 秒后再回来。

`-w` 保不住 `phc2sys`。它在解析参数的时候就把 `-s eth0` 解析成 `/dev/ptp0`，那是在任何
等待之前，所以它是独立于 `ptp4l` 在同一瞬间死的。

`After=network-online.target` 也没有用：这个镜像里 NetworkManager 的 wait-online 辅助
没启用，那个 target 在 t=0 就到了。`After=sys-subsystem-net-devices-eth0.device` 在
netdev 注册时就 active，这块板子上大约是 3.5 秒，仍然远早于 MAC 起来。

这个门禁在所有模式下都留着，不只是 hardware。software 模式没有 PHC 要开，但 `ptp4l` 仍
然要往网口绑 PTP socket，起早了同样失败。`wait-iface` 等的是 `IFF_UP`
（`/sys/class/net/<iface>/flags` 的第 0 位，`ndo_open` 返回时置位，没插网线也是置位的），
不是 `carrier`，所以没接线的板子照样能起来，不会卡住。

## 实机 log

以下全部来自 D-Robotics X5 板子，Ubuntu 22.04.5 aarch64，systemd 249.11，linuxptp
3.1.1，网口 `eth0`，局域网上没有 PTP master。

### 全新机器首次配置，software 打时戳

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

最后一项是关键那项。它读运行中的 `ptp4l` 的 `/proc/<pid>/fd`，报的是守护进程真正打开的
钟，而不是参数要求的钟。

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

这个角色、这个模式下 `chrony.service` 和 `phc2sys@eth0.service` 是
`inactive/disabled` 是对的，不是故障：这里 `CLOCK_REALTIME` 的属主是 `ptp4l`。

进到 `LISTENING` 就停在那里，在局域网上没有 PTP master 时同样是正常的。`slaveOnly 1`
意味着这块板子不会去当 master，所以永远不会有 `master offset` 行。网络上有真 master
时，这些行会被每秒一条的 `master offset` 取代。

### 在 1.0.0 装过的板子上改成 `master`

1.0.0 不记录时间戳模式，所以模式从 unit 实际运行的参数里取回来。除了角色，别的什么都不会
被顺带改掉：

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

把那些 `would` 行当成一组读：换角色就是*启用 chrony*、*重写 `/etc/default/ptp4l`*、
*重启*，外加清理旧布局留下的那个文件。

### 改回 `slave` 并加 `--time-stamping hardware`

把模式改回去会重新启用 `phc2sys`，两个守护进程都起来：

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

属主那一行注意一下：hardware 模式下它写 `phc2sys`，software 模式下写 `ptp4l`。两种模式
断的是同一条不变量，只是把属主的名字填进去。

### 这块板子上 `--time-stamping legacy` 会大声失败

`verify` 返回 4，journal 说明原因，所以一个 legacy 部署不可能悄悄出线：

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

网口能力检查是通过的，因为这块网卡声明了 hardware 时戳能力。只有守护进程拒绝这个模式，
这也是 `verify` 要查运行中的进程而不只是查声明能力的原因。切回 `hardware` 一条命令就恢复。

### 开机，加门禁前后

加之前，`Restart=on-failure` 把它盖住了：

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

加之后，当前这次开机。`Starting` 和 `Started` 之间的间隔就是 `wait-iface` 把 unit 压住等
MAC 的时间：

```
Sep 16 18:02:29.114172 ubuntu systemd[1]: Starting Precision Time Protocol (PTP) service for eth0...
Sep 16 18:02:33.350292 ubuntu systemd[1]: Started Precision Time Protocol (PTP) service for eth0.
Sep 16 18:02:33.364375 ubuntu ptp4l[2686]: [16.994] selected /dev/ptp0 as PTP clock
Sep 16 18:02:33.365490 ubuntu ptp4l[2686]: [16.995] port 1: INITIALIZING to LISTENING on INIT_COMPLETE
```

等了 4.2 秒，换来的是一次 EPERM、一个失败 unit 和一个重启计数的消失。`ptp4l` 第一次尝试
就进 `LISTENING`，`NRestarts` 保持 0：

```
$ systemctl show ptp4l@eth0.service phc2sys@eth0.service -p NRestarts
NRestarts=0
NRestarts=0
```

`phc2sys` 则按设计待着。它的 `-w` 压住它，直到 `ptp4l` 报出同步状态；没有 master 就不会
发生：

```
Sep 16 18:02:33.393268 ubuntu systemd[1]: Started Synchronize system clock or PTP hardware clock (PHC).
Sep 16 18:02:34.402287 ubuntu phc2sys[2713]: [18.032] Waiting for ptp4l...
Sep 16 18:02:35.403621 ubuntu phc2sys[2713]: [19.033] Waiting for ptp4l...
```

## 说明与限制

- **这块板子的 hardware 时戳并没有被证明。** 驱动声明了 hardware 收发时戳能力，
  `ptp4l -H` 也绑上了 `/dev/ptp0` 并一直持有，`verify` 是通过的。但 MAC 是不是真的插入了
  时戳，在局域网上没有 PTP master 的情况下无法确认；SoC 上"驱动声明了能力、硅片返回零时
  戳"是已知的失效方式。要确认只能接真 master 跑起来读 offset。默认值取 `software` 也有
  这个原因。
- **从钟的锁定状态不是这些 log 证明的。** 没有接 PTP master，所以 log 展示的是一个健康
  的、在等待的从钟，不是一个已锁定的从钟。
- **本机管理 MAC 只报告，不作门禁。** 这块板子的 MAC 每次开机重新生成，所以
  `grandmasterIdentity` 每次开机都变，从钟每次开机会重锁一次。时间同步照常跑，所以这不构
  成拒绝部署的理由；如果在意重锁，就在配置时把 MAC 固定下来。
- **`master` 在 GNSS 接好之前是空壳。** 不给配方和 `--advertise-gnss-quality`，
  `verify` 就按设计返回 5。
- **unit 文件覆盖了系统包里的那份。** `/etc/systemd/system/` 下的文件会整体覆盖
  `/lib/systemd/system/` 下的那份，这里是必需的：包里的 unit 把配置路径写死了，而且没有
  `Restart=`。代价是以后 `linuxptp` 包更新那两个 unit 时不会生效；文件很短，如果在意的
  话，升级后 diff 一下。
- **每个角色、每种模式下 `CLOCK_REALTIME` 只有一个属主。** 改这里任何东西时，这条不变量
  要守住。
