# timesync

[English](README.md) | **中文**

给 X5 类板子配置两种 PTP 角色中的一种，并在两者之间切换。

两种角色互斥：`phc2sys` 和 `chrony` 都会调 `CLOCK_REALTIME`，两个控制环同时调一个钟会互
相打架。

| | `slave` | `master` |
|---|---|---|
| `ptp4l` | 从（`slaveOnly 1`），不会去抢主 | 主 / grandmaster（`slaveOnly 0`） |
| `phc2sys` | `-s eth0 -c CLOCK_REALTIME`，PHC 搬到系统钟 | `-s CLOCK_REALTIME -c eth0`，系统钟搬到 PHC |
| `chrony` | 停止并禁用 | 启用并启动 |
| `CLOCK_REALTIME` 属主 | `phc2sys`（software 模式下是 `ptp4l`） | `chrony` |

`phc2sys` 只在 hardware 打时间戳的部署里用到。

## 快速上手

在板子上以 root 运行：

```sh
# 跟随网络上的 PTP master：
sudo ./time_synchronization_installer.sh slave

# 同上，但时戳取网卡自己的时钟，而不是内核（只在网卡真的在硅片里打时戳时用）：
sudo ./time_synchronization_installer.sh slave --time-stamping hardware

# 改成对外提供 GNSS 时间。master 需要一份配方：描述这台接收机的 chrony 片段。
# 从仓库里的模板抄一份，按自己的硬件改：
cp gnss/refclock.example.conf gnss/refclock.conf
$EDITOR gnss/refclock.conf
sudo timesync master --refclock-file gnss/refclock.conf --gps-device /dev/pps0 \
                     --advertise-gnss-quality

# 产线出货门禁：0 = 可以出货，4 = 不可以，5 = 没有 GNSS 参考源：
sudo timesync verify
```

角色名就是命令：首次运行把机器配置好，之后的运行只改角色，别的一律不动。`1` 和 `2` 代替
`slave`、`master` 照样能用，直接当命令敲也行。首次运行之后工具会装成
`/usr/local/sbin/timesync`。

默认的时间戳模式是 `software`，任何网口都能用，精度是 NTP 级。`hardware` 是亚微秒级，但
要求网卡真的在打时戳——选它之前先看 [NOTES.zh-CN.md](NOTES.zh-CN.md)。

## GNSS 配方

只有 `master` 角色需要。它就是一段 chrony 配置片段，指名接收机在哪儿：

```conf
refclock SHM 0 refid GPS  precision 1e-1 offset 0.0 delay 0.2
refclock PPS /dev/pps0 refid PPS  lock GPS prefer
```

仓库里的 `gnss/refclock.example.conf` 就是这个文件加注释：抄一份，留下与你的接收机相符的
行，删掉其余的。具体取值看接收机的文档，或者找一台同款接收机已经在跑的样机，抄它的
`/etc/chrony/conf.d/`——后者更好，因为那是在你的驱动栈上验证过能用的。

上批量之前先在一台样机上验证：

```sh
chronyc sources -v      # 参考源出现，reachability 是 377
chronyc tracking        # Reference ID 是那个参考源，Offset 在 µs 级
```

然后把评审过的文件提交进部署仓库，每台板子都用 `--refclock-file` 传它，这样全队机器用的
是同一份文件。

## 命令

| 命令 | 作用 |
|---|---|
| `slave` \| `master` | 设置角色。机器没配置过就配置好，配置过就只改角色。 |
| `status` | 当前角色、时间戳模式、每一件的实时状态、最近的 `ptp4l` 输出、当前系统时间。什么都不改。 |
| `verify [slave\|master]` | 出货门禁，见下。什么都不改。 |
| `uninstall` | 停掉并删掉本工具装的一切，并把被它关掉的 NTP unit 恢复回去。`linuxptp` 包本身不动。 |
| `wait-iface IFACE [seconds]` | 等 `IFACE` 起来。是生成的 unit 用的，不是给运维敲的。 |

`verify` 检查：角色配置文件、unit 传给每个守护进程的参数、该模式需要的每个 unit 是否
active **且** enabled（不需要的是否确实不在）、每个进程实际运行的 argv、`ptp4l` 到底打开
了哪个钟、以及 `chrony` 的状态。没起来的 unit 判失败；当前是起来的、但早先重启过的 unit
只报告不判失败，这样开机时碰到一次瞬时重启的板子不会被拦在出货之外。

改角色时其他一切照旧：网口、domain、transport、时间戳模式、GNSS 配方、GNSS 质量设置。所
以对配置成 GNSS master 的板子敲一次 `timesync master`，不用再重复那些参数，完整的
grandmaster 配置就回来了。

## 参数

| 参数 | 含义 |
|---|---|
| `--iface IFACE` | PTP 跑在哪个网口。默认：`/etc/default/ptp4l` 里记的那个；否则是唯一能 hardware 打时戳的网口（software 模式下没有这种网口时，取物理网口）。 |
| `--domain N` | `domainNumber`，0–127（默认 0）。 |
| `--transport T` | `UDPv4` \| `UDPv6` \| `L2`（默认 `UDPv4`）。 |
| `--time-stamping MODE` | `software` \| `hardware` \| `legacy`（默认 `software`）。`ptp4l` 在哪个钟里打时戳。 |
| `--refclock-file F` | `master`：一段描述这台机器的 GNSS 接收机的 chrony 片段。见 [GNSS 配方](#gnss-配方)。原样装成 `/etc/chrony/conf.d/20-ptp-refclock.conf`。 |
| `--gps-device DEV` | `master`，配 `--refclock-file` 时必给：接收机所在的设备（`/dev/ttyS0`、`/dev/pps0` …）。它必须存在，**并且** `F` 里要提到 `DEV`，这能拦住把配方贴到接法不同的机器上。 |
| `--advertise-gnss-quality` | `master`，配 GNSS 配方时必给：让 `ptp4l` 宣告 `clockClass 6`、accuracy `0x21`、`timeSource GNSS`。不给的话没有从钟会锁它。 |
| `--no-apt` | 不调 `apt`；缺包时预检直接失败。 |
| `--no-verify` | 跳过应用之后的 verify。此时退出码 0 只表示「步骤跑过了」。 |
| `--force` | 越过那些属于判断而非事实的检查：另一个网口上残留的 `ptp4l` 实例、还不存在的 `--gps-device`、以及完全没有 GNSS 配方的 `master` 运行。 |
| `--root DIR` | 把生成的树摆到 `DIR` 而不是 `/`。`apt`、`systemctl` 和所有硬件探测都跳过，所以可以在笔记本上直接看这棵树。只对角色命令和 `uninstall` 有效。 |
| `--dry-run` | 打印每一步动作，什么都不改。 |

## 退出码

| 码 | 含义 |
|---|---|
| 0 | 成功 |
| 1 | 用法或参数错误 |
| 2 | 机器预检拒绝：不是 root、没有可用的网口、有冲突的 PTP 实例、GNSS 配方读不了或不匹配、`--no-apt` 下缺包、还没配置过 |
| 3 | 应用角色后有 unit 起不来 |
| 4 | `verify`：有检查项没过 |
| 5 | `verify`：`master` 但完全没有 GNSS 参考源 |
| 130 | 被中断 |

`verify` 的 0、4、5 就是产线需要的三个答案：可以出货、不可以出货、以及老实承认这台没有
GNSS 参考源。

## 装了哪些文件

| 路径 | 何时 |
|---|---|
| `/etc/linuxptp/ptp4l-slave.conf` | 总是；两个角色的文件都保持最新 |
| `/etc/linuxptp/ptp4l-master.conf` | 总是（配置了 GNSS 质量时含那几行） |
| `/etc/chrony/conf.d/20-ptp-refclock.conf` | 只有带配方的 `master` |
| `/etc/systemd/system/ptp4l@.service` | 总是；覆盖系统包里的那个 |
| `/etc/systemd/system/phc2sys@.service` | 总是；覆盖系统包里的那个，但只在 hardware 模式下启用 |
| `/etc/default/ptp4l` | 总是；**角色和模式都在这里** |
| `/usr/local/sbin/timesync` | 总是；工具本身，以短命令形式 |
| `/var/lib/timesync/state` | 总是；这台机器配了什么、哪个版本配的 |
| `/var/lib/timesync/disabled-ntp-units` | 被关掉的时钟守护进程清单，`uninstall` 靠它恢复 |

两个角色的配置总是都写，`/etc/default/ptp4l` 里的 `-f` 只选其中一个，所以改角色就是重写
一个小文件再重启守护进程。unit 文件里除了 `EnvironmentFile=/etc/default/ptp4l` 和守护进程
自己的参数，不含任何 PTP 知识：

```
ExecStartPre=/usr/local/sbin/timesync wait-iface %I
ExecStart=/usr/sbin/ptp4l $PTP4L_ARGS -i %I

ExecStartPre=/usr/local/sbin/timesync wait-iface %I
ExecStart=/usr/sbin/phc2sys -w $PHC2SYS_ARGS
```

`$PTP4L_ARGS` 故意不带花括号：systemd 会把不带花括号的变量拆成多个参数，写成
`${PTP4L_ARGS}` 会变成一个参数，`ptp4l` 直接拒绝。

## 排障

```sh
sudo timesync status                      # 什么角色、什么模式、什么在跑
sudo timesync verify                      # 为什么不能出货
journalctl -u ptp4l@eth0.service -n 50    # 守护进程说了什么
systemctl --failed                        # 有没有留下 failed 的
```

- **`verify` 退出 5**——`master` 但没有 GNSS 参考源。接上接收机，然后传
  `--refclock-file`；如果暂时就是有意留个空壳，用 `--force` 也能配上。
- **`verify` 退出 4**——看 `[FAIL]` 行，它们指名了出问题的 unit 或文件。
- **`phc2sys@eth0.service` 显示 `failed`**——它是在等 master 的时候被停掉的。切到
  `software` 模式就会清掉这条记录。
- **`ptp4l` 的日志里没有 `master offset` 行**——这块板子是从钟，而网络上没有 master
  应答。`slaveOnly 1` 下这是预期行为：它会等，而不是去抢主。

## 更多

`NOTES.zh-CN.md` 是长版本：每种时间戳模式在这块板子上到底做什么、一次真实部署的设备日志、
以及还有什么是没被证明的。
