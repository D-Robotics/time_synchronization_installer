# timesync

[English](README.md) | **中文**

把一个 X5 板子部署成两种 PTP 角色中的一种，并且以后可以方便地互相切换。一个自包含的
shell 脚本，不依赖 VIO 那一套，也不依赖本工作区里的任何其他东西。

两个角色是**结构上**互斥的，不是靠约定：`phc2sys` 和 `chrony` 都能操纵 `CLOCK_REALTIME`，
两个控制环调同一个钟会互相顶。所以每个角色都把 `CLOCK_REALTIME` 交给唯一一个主人。

| | `slave` | `master` |
|---|---|---|
| `ptp4l` | 从时钟（`slaveOnly 1`），永不接管 | 主时钟 / grandmaster（`slaveOnly 0`） |
| `phc2sys` | `-s eth0 -c CLOCK_REALTIME` —— 硬件钟 **→** 系统钟 | `-s CLOCK_REALTIME -c eth0` —— 系统钟 **→** 硬件钟 |
| `chrony` | 停止并禁用 | 启用并启动 |
| `CLOCK_REALTIME` 的主人 | `phc2sys` | `chrony` |

## 两个角色

`slave` —— 跟随网络上的 PTP 主时钟：

```
网络上的 PTP 主时钟
        │  基于 UDPv4 的 PTP，domain 0
        ▼
   ptp4l ── 调整 ──▶ PHC /dev/ptp0
   (slaveOnly 1)            │
                            │  phc2sys -s eth0 -c CLOCK_REALTIME
                            ▼
                      CLOCK_REALTIME          chrony：停止 + 禁用
```

`ptp4l` 从不碰 `CLOCK_REALTIME`，它只调网卡的硬件钟。是 `phc2sys` 把那个钟拷进系统钟，
`date` 变准靠的是这一步。

`master` —— 把本机时间（来自 GNSS）分发给网络：

```
GNSS 接收机
        │  chrony refclock（例如 SHM / PPS）
        ▼
     chrony ── 调整 ──▶ CLOCK_REALTIME
                              │
                              │  phc2sys -s CLOCK_REALTIME -c eth0
                              ▼
                        PHC /dev/ptp0
                              │
                              ▼
             ptp4l (slaveOnly 0) ── 广播 ──▶ 网络上的从时钟
```

`chrony` 用 GNSS refclock 调 `CLOCK_REALTIME`，全程不碰 PHC。是 `phc2sys` 把系统钟拷到
PHC，`ptp4l` 才有值得广播的东西。

如果 `-f` 列表里没有 `ptp4l-grandmaster.conf`，`master` 就是一个**空壳**：默认值
（`clockClass 248`、`clockAccuracy 0xFE`）等于告诉所有从时钟"我不是时间源"，它们不会
锁上来。那个文件只有在给了 GNSS recipe 加 `--advertise-gnss-quality` 时才会写入；两者
缺一，`verify` 都拒绝放行。

## 快速开始

在板子上以 root 执行：

```sh
# 跟随本网络的 PTP 主时钟，并且重启后自动恢复：
./time_synchronization_installer.sh install slave

# 以后要改成对外分发 GNSS 时间：
timesync switch master --refclock-file ./my-gps.recipe --gps-device /dev/pps0 \
                       --advertise-gnss-quality

# 产线出厂闸：
timesync verify
```

凡是需要角色的地方，写法都是 `slave` / `master`。`1` 和 `2` 仍然接受，并且在生成任何
消息之前就被规范化成单词。裸的 `timesync 1` 会被拒绝，而不是当成一次切换，所以打错字
不会改变机器状态。

## 命令

| 命令 | 作用 |
|---|---|
| `install slave\|master` | 把本机部署成该角色并启动。收敛式的：写全部配置、unit 和切换器，然后只重启真正变了的东西。可以反复执行——在已经正确的机器上再跑一次什么都不会变。 |
| `switch slave\|master` | 切换一台**已经部署过**的机器的角色。没跑过 `install` 会拒绝。 |
| `slave` \| `master` | `switch slave` / `switch master` 的简写。 |
| `status` | 配置的角色、各部分实时状态、最近的 `ptp4l` 输出、当前系统时间。不改动任何东西。 |
| `verify [slave\|master]` | 产线出厂闸。检查角色文件、两个 unit 将要传给各守护进程的参数、两个 unit 是否既 active **又** enabled、每个进程的实时 argv、硬件时间戳能力、chrony 的状态。不改动任何东西。 |
| `uninstall` | 停止并删除本工具装的一切，并把它之前禁用的 NTP unit 恢复。`linuxptp` 包本身保留。 |
| `wait-iface IFACE [seconds]` | 等 `IFACE` 在管理上 up。这是两个生成的 unit 的 `ExecStartPre`，不是给人手动跑的——见[下文](#为什么两个-unit-都跑-timesync-wait-iface)。 |

`switch` 是换角色，不是重新部署：非角色的东西全部继承——接口、domain 和 transport，
以及对一台已经部署成 GNSS master 的板子来说还包括 recipe 文件和 GNSS 时间质量。所以
`switch master` 不必重新打一遍参数就能把 grandmaster 配置带回来。而普通的 `install` 是
权威口径：不带 recipe 就会删掉之前装过的 recipe，盘上的内容始终与刚装的角色一致。

## 选项

| 选项 | 含义 |
|---|---|
| `--iface IFACE` | PTP 使用的接口。默认取 `/etc/default/ptp4l` 里记录的，否则取机器上唯一能做硬件时间戳的那个。 |
| `--domain N` | `domainNumber`，0–127（默认 0）。 |
| `--transport T` | `UDPv4` \| `UDPv6` \| `L2`（默认 `UDPv4`）。 |
| `--refclock-file F` | `master`：描述本机 GNSS 接收机的 chrony 片段（它的 `refclock` 行，如果有 pps 就再加一行）。安装为 `/etc/chrony/conf.d/20-ptp-refclock.conf`。 |
| `--gps-device DEV` | `master`，与 `--refclock-file` 同时给：接收机所在的设备（`/dev/ttyS0`、`/dev/pps0` …）。会检查它是否存在，**并且**要求 `F` 里提到 `DEV`——这能抓出"把 recipe 贴到了接线不同的机器上"。 |
| `--advertise-gnss-quality` | `master`，与 GNSS recipe 同时给：让 `ptp4l` 广播 `clockClass 6` / accuracy `0x21` / `timeSource GNSS`。 |
| `--no-apt` | 绝不调用 `apt`；缺包时直接让 preflight 失败。 |
| `--no-verify` | 跳过安装后的 verify。此时退出码 0 只代表"应用步骤跑了"，不代表"可以出厂"。 |
| `--force` | 覆盖那些属于判断题而非事实的检查：另一个接口上残留的 `ptp4l` 实例、暂时还不存在的 `--gps-device`、以及完全没有 GNSS recipe 的 `master` 安装。 |
| `--root DIR` | 把生成的目录树落到 `DIR` 而不是 `/`。`apt`、`systemctl` 和所有硬件探测都跳过，所以可以在笔记本上查看这棵树。只对 `install` 和 `uninstall` 有效。 |
| `--dry-run` | 打印每一步动作，什么都不改。 |

## 退出码

| 码 | 含义 |
|---|---|
| 0 | 成功 |
| 1 | 用法或参数错误 |
| 2 | 机器 preflight 拒绝——不是 root、没有能做硬件时间戳的接口、有冲突的 PTP 实例、GNSS recipe 不可读或与设备不匹配、`--no-apt` 下缺包、还没部署过 |
| 3 | 应用角色后，有 unit 重启失败 |
| 4 | `verify`：有检查项失败 |
| 5 | `verify`：`master` 但没配置 GNSS 参考 |
| 130 | 被中断 |

`verify` 的 0 / 4 / 5 正是产线需要的三个答案：可以出厂、不可以、以及诚实地承认"这是个
没有 GNSS 的空壳"。

## 装了哪些文件

| 路径 | 适用角色 |
|---|---|
| `/etc/linuxptp/ptp4l-common.conf` | 两者——transport、domain、日志 |
| `/etc/linuxptp/ptp4l-slave.conf` | 两者（总是写入，由 `-f` 列表选用） |
| `/etc/linuxptp/ptp4l-master.conf` | 两者（总是写入，由 `-f` 列表选用） |
| `/etc/linuxptp/ptp4l-grandmaster.conf` | 仅 `master` + `--advertise-gnss-quality` |
| `/etc/chrony/conf.d/20-ptp-refclock.conf` | 仅带 recipe 的 `master` |
| `/etc/systemd/system/ptp4l@.service` | 两者——遮蔽打包版本 |
| `/etc/systemd/system/phc2sys@.service` | 两者——遮蔽打包版本 |
| `/etc/default/ptp4l` | 两者——**角色就住在这里** |
| `/usr/local/sbin/timesync` | 两者——工具本身，短命令名 |
| `/var/lib/timesync/state` | 两者——装了什么、哪个版本装的 |
| `/var/lib/timesync/disabled-ntp-units` | 被关掉的时钟守护进程账本，供 `uninstall` 还原 |

两份角色 conf 都总是写入，由 `/etc/default/ptp4l` 里的 `-f` 列表选用其中一份。所以切换
永远不依赖"先删掉某个文件"这个前提，切回来必定能找到另一份。

两个 unit 都不含角色知识，除了 `EnvironmentFile=/etc/default/ptp4l` 和守护进程自己的
参数以外什么都没有：

```
ExecStartPre=/usr/local/sbin/timesync wait-iface %I
ExecStart=/usr/sbin/ptp4l $PTP4L_ARGS -i %I
```

```
ExecStartPre=/usr/local/sbin/timesync wait-iface %I
ExecStart=/usr/sbin/phc2sys -w $PHC2SYS_ARGS
```

所以切换 = 重写一个小文件 + 重启守护进程：不改 unit、不做 enable/disable 抖动。
`$PTP4L_ARGS` 故意不加花括号——systemd 会把不带花括号的变量按空白拆成多个参数，两个
`-f` 就是这样传进去的。

## 为什么两个 unit 都跑 `timesync wait-iface`

网卡的 PHC 只有在驱动把 MAC 打开之后才能被打开。冷启动时 systemd 在这之前就启动了
守护进程，于是两个都死了：

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

注意这段日志**掩盖**了什么：`Restart=on-failure` 把整件事盖住了。unit 看起来是健康的，
`systemctl status` 也看不出任何问题，而实际上每次开机两个守护进程都在死、6 秒后再回来。

`-w` **保护不了** `phc2sys`：`phc2sys` 在解析自己选项的时候就把 `-s eth0` 解析成了
`/dev/ptp0`，那发生在任何等待之前——所以它在几乎同一时刻独立于 `ptp4l` 死掉了。

`After=network-online.target` 在这个镜像上等于零保护：NetworkManager 的 wait-online
helper 没有启用，那个 target 在 t=0 就到了。`After=sys-subsystem-net-devices-eth0.device`
在 netdev 注册时就 active，在这块板子上约 3.5 秒——仍然远早于 MAC 起来。

所以 `wait-iface` 用 `IFF_UP` 作为闸门（`/sys/class/net/<iface>/flags` 的第 0 位，
`ndo_open` 返回即置位，**没插网线也是置位的**），而不是用 `carrier`——这样拔了网线的板子
照样能起好，而不会被卡死。

## 实机日志

以下全部来自一块 D-Robotics X5 板子，Ubuntu 22.04.5 aarch64，systemd 249.11，`eth0`，
局域网上没有 PTP 主时钟。

### 一次全新安装（在板子上用 `--root` 试跑）

`--root` 会把整棵树写到无害的地方，并报告全新安装会做什么——这里用它，是为了不打扰正在
运行的那块板子：

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

这份清单里有两处值得注意：即使是 `slave`，**两份角色 conf 都会被写入**；而两个
`absent` 行说明只属于 GNSS 的那两个文件在这个角色下是**故意不存在**的。

### 在已经正确的板子上重跑 `install`

幂等——全部报告 `unchanged`，什么都不重启，末尾的 verify 照常执行：

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

### `verify` —— 产线出厂闸

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

最后一项读的是**当前这一次开机**的 journal（`journalctl -b`）并取最新一条，所以
`verify` 不可能拿几小时前那次开机的日志来蒙混过关。

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

`chrony.service` 显示 `inactive/disabled` 是 `slave` 角色**正确**的状态，不是出问题：
这个角色里 `CLOCK_REALTIME` 的主人是 `phc2sys`。

`selected local clock ... as best master` 反复出现，在一台**网络上没有 PTP 主时钟**的
板子上同样是预期行为。`slaveOnly 1` 意味着本板会宣告"我不介意当主时钟"，但不会真的接管
——所以端口停在 `LISTENING`，永远不会出现 `master offset` 行。在有真实主时钟的网络上，
这些行会被每秒一条的 `master offset` 取代。

### `switch master` —— 只看计划，不改机器

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

`--force` 是因为这块板子没有 GNSS recipe，不给它的话工具会拒绝安装一个空壳。把两行
`would` 连起来读就是重点：切换到 `master` 本质上就是**启用 chrony** + **翻转
`/etc/default/ptp4l` 里的参数**，没有别的。

### 开机：加闸门之前 / 之后

之前，`Restart=on-failure` 把它盖住了——unit 报 `Started`，然后两个守护进程立刻死掉：

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

之后，当前这次开机。`Starting` 与 `Started` 之间的间隔，就是 `wait-iface` 把 unit 按住在
等 MAC 起来：

```
Sep 16 18:02:29.114172 ubuntu systemd[1]: Starting Precision Time Protocol (PTP) service for eth0...
Sep 16 18:02:33.350292 ubuntu systemd[1]: Started Precision Time Protocol (PTP) service for eth0.
Sep 16 18:02:33.364375 ubuntu ptp4l[2686]: [16.994] selected /dev/ptp0 as PTP clock
Sep 16 18:02:33.365490 ubuntu ptp4l[2686]: [16.995] port 1: INITIALIZING to LISTENING on INIT_COMPLETE
```

4.2 秒的等待，换来的是没有 EPERM、没有 failed unit、没有 restart counter。`ptp4l`
第一次尝试就直奔 `LISTENING`，`NRestarts` 保持 0：

```
$ systemctl show ptp4l@eth0.service phc2sys@eth0.service -p NRestarts
NRestarts=0
NRestarts=0
```

而 `phc2sys` 按设计处于休眠状态——它的 `-w` 按住它，直到 `ptp4l` 报告进入同步状态，
在主时钟出现之前这不会发生：

```
Sep 16 18:02:33.393268 ubuntu systemd[1]: Started Synchronize system clock or PTP hardware clock (PHC).
Sep 16 18:02:34.402287 ubuntu phc2sys[2713]: [18.032] Waiting for ptp4l...
Sep 16 18:02:35.403621 ubuntu phc2sys[2713]: [19.033] Waiting for ptp4l...
```

这就是为什么在还没有主时钟的板子上让这个角色跑着是安全的：等待期间它不碰任何时钟。

## 注意事项与已知限制

- **本地管理的 MAC 只报告、不拦。** 这块板子的 MAC 每次重启都会重新生成，所以
  `grandmasterIdentity` 每次开机都变，从时钟每重启一次要重新锁一次。时间同步照常运行，
  所以这不构成拒绝部署的理由——但如果这个重新锁定有影响，就在产线阶段固定 MAC
  （`cloned-mac-address`，或在启动时写死一个地址）。
- **从时钟是否真的锁上了，这些日志证明不了。** 开发时局域网上没有 PTP 主时钟，所以日志
  展示的是一个健康、`LISTENING`、在等待的从时钟，而不是一个已锁定的。锁定时必须对着真实
  主时钟确认。
- **GNSS 没接之前，`master` 就是空壳。** 没有 recipe 和 `--advertise-gnss-quality` 时，
  `verify` 按设计返回 5，而不是假装通过。
- **两个 unit 遮蔽了打包版本。** `/etc/systemd/system/` 下的文件会整体遮蔽
  `/lib/systemd/system/` 下的同名文件，这里是必须的（打包版硬编码了打包的配置路径，而且
  没有 `Restart=`）。代价是将来 `linuxptp` 包更新这两个 unit 时不会生效；这两个文件很短，
  升级后如果在意，直接 diff 一下。
- **每个角色里 `CLOCK_REALTIME` 有且只有一个主人**，改动这里任何东西时，这是要保持的
  不变量。
