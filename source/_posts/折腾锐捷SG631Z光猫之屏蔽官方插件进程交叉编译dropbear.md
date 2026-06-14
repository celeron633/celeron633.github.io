---
title: 折腾锐捷SG631Z光猫之屏蔽官方插件进程交叉编译dropbear
date: 2026-06-14 13:18:10
tags:
    - OpenWRT
    - 光猫
    - dropbear
    - mtd-utils
categories:
    - 技术分享
    - 嵌入式Linux
---

# 折腾中国联通定制的锐捷 SG631Z 光猫：屏蔽官方插件进程 + 交叉编译 dropbear / mtd-utils

> 这台锐捷 SG631Z 是中国联通定制的光猫，Buildroot 衍生固件，kernel 4.19。厂商往里塞了 DPI 和应用框架两个插件，想把它们关掉。过程中顺带搞清楚了几个困扰我挺久的问题：插件为什么杀不死、文件为什么改了重启就没、mtd 分区为什么格式化完又自己长回来。最后是用 C 写了两个 daemon 桩替换掉官方二进制，另外交叉编译了 dropbear 和 mtd-utils。
>
> 设备一直是离线的（没接 ISP），全程靠 telnet 和网页后台那个 shell 操作。记下来备忘，也给有同款设备的人参考。

---

<!-- more -->

## 0. 环境与目标

- **设备**：锐捷 SG631Z ONT（光猫），SoC 是 ZTE ZX279127S（Cortex-A9，ARMv7，软浮点）
- **固件**：定制 Buildroot 系统，`Linux localhost 4.19.136 #2 SMP PREEMPT armv7l GNU/Linux`
- **架构**：armv7l（小端），glibc 2.26
- **现状**：设备完全离线（没有接入 ISP）
- **手头资源**：
  - 一份从光猫 rootfs 导出的 `/etc` 离线备份（在 Windows 上，用于分析配置文件）
  - telnet 进设备（参考：https://blog.moo.ac/posts/fiberopticmodem/）
  - 网页后台自带的一个 web shell（关键时刻救命用：`http://192.168.1.1/hidden_cmd.gch?pwd=aDm8H%25MdA`）

**目标**：让两个官方插件永久不再运行：

1. **CuInformLoader** —— DPI（深度包检测）插件加载器
2. **ufwmg** —— 跑在 LXC 容器里的应用框架插件（容器名 `ufw`，注意不是 Linux 那个防火墙 ufw）

收集到的几个账号密码：

|  KEY   | VALUE  |
|  ----  | ----  |
| 锐捷猫 telnet 用户名 | admin |
| 锐捷猫 telnet 密码 | chzhdpl@246 |
| 锐捷猫 su 密码 | aDm8H%MdA |
| 网页后台用户名 | CMCCAdmin |
| 网页后台密码 | aDm8H%MdA |

分区表（`cat /proc/mtd`），后面屏蔽插件和擦分区都要对着它看：

```
dev:    size   erasesize  name
mtd0: 10000000 00020000 "whole"
mtd1: 00200000 00020000 "boot"
mtd2: 00300000 00020000 "parameter tags"
mtd3: 00600000 00020000 "kernel0"
mtd4: 00600000 00020000 "kernel1"
mtd5: 00900000 00020000 "usercfg"
mtd6: 00100000 00020000 "others"
mtd7: 00300000 00020000 "wlan"
mtd8: 03600000 00020000 "rootfs0"
mtd9: 03600000 00020000 "rootfs1"
mtd10: 01000000 00020000 "framework0"
mtd11: 01000000 00020000 "framework1"
mtd12: 03e00000 00020000 "apps"
mtd13: 01600000 00020000 "plugin"
mtd14: 00200000 00020000 "GN25L95_datas"
```

---

## 1. 先搞清楚启动链

光猫的引导大致是这条链：

```
inittab
  └─ ::sysinit:/etc/init.d/rcS
         ├─ . /etc/default/S01userconfig
         │       └─ . /etc/default/S01userconfig_ctcc_smart   # 插件存储/overlay 都在这
         ├─ . /etc/init.d/cuinform-rcS                        # 释放并拷贝 CuInformLoader
         └─ pc&                                               # 厂商进程管理器/看门狗
```

`inittab` 本身只 respawn 一个 getty，**不负责拉插件**：

```
::sysinit:/etc/init.d/rcS
::respawn:/sbin/getty ttyAMA0 115200
```

真正不断把插件拉起来的是 **`pc`** —— 厂商的进程管理器兼看门狗，一堆守护进程的父进程，数据驱动（配置表里写了该拉哪些进程），**不是改几个字符串就能 patch 的**。

---

## 2. 为什么 mtd12/mtd13 用 busybox"格式化没用"

一开始的困惑：用光猫自带的工具把 `mtd12`、`mtd13` 格式化掉，重启之后内容又原样回来了，设备明明是离线的，哪来的"还原源"？

后来发现这其实不是"被还原"，而是压根没擦干净。光猫官方 busybox 里那套 NAND 工具（`nand_write` 之类）和它带的 ubi 工具能力被阉割过，或者有额外校验，执行完看着成功，实际没真正把内容擦掉。换成自己交叉编译的 mtd-utils，用里面的 `flash_erase`（nand erase）和 `ubiformat`，擦除/格式化就正常了。这也是第 9 节要自己编一套 mtd-utils 的直接原因。

---

## 3. overlay 的坑：为什么"改了 /bin 重启就没了"

`S01userconfig_ctcc_smart` 给 `/etc /usr /lib /bin` 都挂了 overlay，upperdir 在 tmpfs（RAM）里：

```sh
/bin/mount -t overlay overlay \
  -olowerdir=/bin,upperdir=/tmp/opt/cu/apps/bin,workdir=/tmp/opt/cu/apps/work/bin /bin
```

含义：

- **lowerdir=/bin** = 只读的 ubifs rootfs（底层）
- **upperdir=/tmp/opt/cu/apps/bin** = tmpfs（RAM，掉电即失）的可写层
- **workdir** = overlayfs 的**原子操作暂存区**（必须和 upperdir 同一个文件系统、必须空、内核独占）。copy-up / whiteout 都是"先在 workdir 里造好，再用 `rename()` 原子搬进 upperdir"，保证 upper 层永远不出现半成品

**直接往 `/bin/` 写，只会写进 tmpfs upper 层，重启蒸发。** 想持久必须绕过 overlay 写到底层 rootfs（下文的 bind-mount 技巧）。

### 顺手解决一个谜题：overlay 的 upperdir 里为什么会有一堆二进制，且 sha1 和底层一模一样？

`ls /tmp/opt/cu/apps/bin` 能看到 busybox、httpd、telnetd、dnsmasq…… 一大堆，sha1 和 rootfs `/bin` 里完全相同。

原因是 overlayfs 的 **copy-up 被"写操作这个动作本身"触发，跟值变没变无关**：

> lower 层只读。当一个**修改类系统调用**（`chmod` / `chown` / `setcap` 写 xattr / 以可写方式 `open` / `rename` …）打到还在 lower 的文件上时，内核会在执行前**先把整个文件原样拷到 upper 层**——因为没法在只读的 lower 上原地改。内核**不判断**新值是否等于旧值。

所以 `chown root:root` 作用在一个本来就是 `root:root` 的文件上，**chown 这个动作**就触发了 copy-up，结果 upper 里多了个内容/权限/mtime 全相同的副本。

那么是谁触发的？看 `rcS`：开机时对这些二进制批量 `setcap`（写 `security.capability` 这个 xattr，属于元数据写）：

```sh
/sbin/setcap 'cap_net_bind_service,...=eip' /bin/telnetd
/sbin/setcap '...' /bin/httpd
# ... 二十来个守护进程，逐一 setcap ...
chmod 4755 /bin/busybox          # busybox 是这句勾上来的
ln -s /etc/local/sbin/ipsec /bin/ipsec
```

upperdir 里那一坨文件，和 `rcS` 这些 `setcap`/`chmod`/`ln` 目标**一一对应**。这也解释了**为什么这个 /bin overlay 不能随便去掉**：

> telnetd 在 `passwd` 里是普通用户 `telnetd:x:106:105` 跑的，要监听 23 端口（<1024 特权端口）靠的就是 setcap 给二进制打的 `cap_net_bind_service` 文件能力。去掉 overlay → `/bin` 只读 → `setcap /bin/telnetd` 失败 → 没有文件能力 → telnetd 起不来 / 不 listen，启动还会因为一堆 RO 写失败卡很久。

（如果真想去掉这个 overlay：可以把这些 setcap/symlink/chmod **一次性"烤"进 rootfs**——file capability 是 xattr，ubifs 能持久保存——再把 `rcS` 里这些写 `/bin` 的行删掉。本文不展开。）

---

## 4. 定位目标进程

先得搞清楚这两个插件是哪个 PID、谁拉起来的。busybox 的 `ps` 比较弱（注意 **busybox ps 没有 `-w`**，长命令行会被截断），配合 `top` 和 `/proc` 一起看。

先 `ps | grep` 找到本体：

```
PID   USER     COMMAND
1115  root     pc
1967  root     ufwmg service 10 11 12
1978  root     CuInformLoader
1980  root     {ufwmg} [lxc monitor] /opt/cu/framework ufw
```

`ufwmg` 和 `CuInformLoader` 找到了，但光知道 PID 不够——直接 kill 掉它俩，过几秒又冒出来，说明有东西在盯着重拉。得找出它们的父进程是谁。busybox 的 `ps` 默认不显示 PPID，用 `top` 能看到，或者直接读 `/proc/<pid>/status` 里的 `PPid` 字段：

```sh
# 方法一：top 里能看到 PPID 列（按需开启），grep 出目标行
top -b -n1 | grep -E 'ufwmg|CuInform'

# 方法二（更直接）：读 /proc 拿 PPID
cat /proc/1967/status | grep PPid      # PPid: 1115
cat /proc/1978/status | grep PPid      # PPid: 1115
```

两个进程的 PPID 都是 **1115**，而 1115 就是上面那个 `pc`。到这一步才确认：**`pc` 是这俩插件的父进程，也是那个不停重拉它们的看门狗。** 这直接决定了后面的屏蔽思路——光删进程没用，得对付 pc。

再确认一下本体到底是哪个可执行文件（别被符号链接或脚本骗了）：

```sh
readlink /proc/1967/exe                          # → /usr/sbin/ufwmg，是个 ELF
cat /proc/1967/cmdline | tr '\0' ' '; echo       # 完整命令行
hexdump -C /usr/sbin/ufwmg | head                # 7f 45 4c 46 = \x7fELF，ARM 32-bit
```

顺手记几个"没有 `ps -w`"时常用的替代手段：

```sh
cat /proc/<pid>/cmdline | tr '\0' ' '; echo      # 完整命令行（不截断）
cat /proc/<pid>/status | grep PPid               # 父进程 PID
readlink /proc/<pid>/exe                         # 真正的可执行文件路径
cat /proc/<pid>/maps                             # 加载了哪些 so
```

---

## 5. 屏蔽策略：换成"桩"，而不是删

**直接删二进制 / 注释启动行**踩过坑：

- 注释掉 `rcS` 第 339 行（`. /etc/init.d/cuinform-rcS`）→ pc 找不到 loader，**疯狂 grep / 重拉**（respawn 风暴）。
- 注释掉 `S01userconfig` 第 87 行（`. /etc/default/S01userconfig_ctcc_smart`）→ 所有 overlay / 存储没了 → **telnet 起不来、开机卡死**（幸好网页 shell 还能用）。

> 救场用的 sed（恢复注释）：
> ```sh
> sed -i '87s/^#//' /etc/default/S01userconfig
> sed -i '339s/^#//' /etc/init.d/rcS
> ```

教训：pc 是看门狗，**进程不在就重拉**。所以正确姿势是——

> **保留进程位，但让它什么都不干。** 把官方二进制替换成一个"桩"：进程照样存在（骗过 pc，不触发重拉风暴），但插件逻辑永不执行。

### 5.1 第一版：shell 桩

```sh
#!/bin/sh
trap '' TERM INT
while :; do sleep 86400; done
```

能用，但 ps 里很假：

```
{CuInformLoader} /bin/sh /bin/CuInformLoader
```

那个**花括号 `{...}` 是 `comm`（进程名），后面是 `argv[]`（命令行）**。busybox 只在两者不一致时才用 `{comm} argv...` 提示你。脚本是被 `/bin/sh` 解释执行的 → comm=脚本名、argv[0]=`/bin/sh` → 一眼假。

### 5.2 第二版：C 桩（让 ps 显示得和原版一致）

换成**文件名仍叫 `CuInformLoader` 的真 ELF** 后：

```
execve("/bin/CuInformLoader", ...) by pc
  → comm = "CuInformLoader"     （内核取被执行文件的 basename，截到 15 字符）
  → argv = pc 原样传入，程序不去改它
  → ps 显示：CuInformLoader + pc 给的参数   ← 无花括号、无 /bin/sh，和原版二进制一致
```

保证"ps 一致"只需两条：**①可执行文件仍叫原名；②代码里不碰 argv**（不调 `prctl(PR_SET_NAME)`、不改 `argv[0]`）。

---

## 6. 两个 C 桩源码（daemon 版）

最终版：daemon 化（double-fork + setsid），**不忽略信号**（实测 pc 不会来 kill），打印放在 daemon 化之前，保证那行 argv 还能输出到 pc 的 stdout。

### `CuInformLoader.c`

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/types.h>
#include <sys/stat.h>

static void daemonize(void)
{
    pid_t pid;
    int fd;

    pid = fork();
    if (pid < 0) _exit(1);
    if (pid > 0) _exit(0);          /* 第一次 fork：父进程退出 */

    setsid();                        /* 新会话，脱离控制终端 */

    pid = fork();
    if (pid < 0) _exit(1);
    if (pid > 0) _exit(0);          /* 第二次 fork：确保不是会话首进程 */

    chdir("/");
    umask(0);

    fd = open("/dev/null", O_RDWR);  /* stdin/out/err 重定向到 /dev/null */
    if (fd >= 0) {
        dup2(fd, 0); dup2(fd, 1); dup2(fd, 2);
        if (fd > 2) close(fd);
    }
}

int main(int argc, char *argv[])
{
    int i;
    for (i = 0; i < argc; i++) {     /* daemon 化前打印命令行 */
        if (i) putchar(' ');
        fputs(argv[i], stdout);
    }
    putchar('\n');
    fflush(stdout);                  /* 进程不退出，必须手动 flush */

    daemonize();

    for (;;) sleep(86400);           /* 后台永久挂起；信号走默认处置 */
    return 0;
}
```

### `ufwmg.c`

和上面**完全相同**，只是注释里的名字不同（占位 `/usr/sbin/ufwmg`，ps 里对应 `ufwmg service 10 11 12`）。

> ⚠️ daemon 化用的是标准 double-fork，**被 pc 直接拉起的原始 PID 会退出，真正存活的是孙进程**。前提是 pc 把它当 "forking 服务"看待（拉起看到父进程退出 0 即认为成功、不再管）。若 pc 是盯着原始 PID 重拉的，就改成单 fork 或干脆不 fork。本机实测 OK。

---

## 7. 交叉编译工具链：统一用 Linaro 5.5

光猫上没有 gcc，所有东西都得在 PC（我这边是 Debian）上交叉编译。这一节先把工具链定下来，后面编 C 桩、dropbear、mtd-utils 全用同一套。

一开始图省事，直接 `apt install gcc-arm-linux-gnueabi` 装了发行版自带的交叉工具链，编出来的桩跑是能跑，但等到编 dropbear 就翻车了——发行版工具链的 sysroot 是很新的 glibc，编出来的二进制要新版本的 glibc 符号，而光猫只有 glibc 2.26，跑起来直接 `version not found`（具体过程见第 8 节）。

结论是：**交叉编译选工具链，关键看它 sysroot 里的 glibc 版本，得 ≤ 目标设备的 glibc，不能光看 gcc 版本号。** 这台光猫是 glibc 2.26，选 Linaro 5.5-2017.10 正合适，它自带 glibc 2.21，比 2.26 老，编出来的符号光猫向后兼容能跑。

先确认两件事。一是光猫的 glibc 版本：

```sh
ls -l /lib/libc*.so*
# libc.so.6 -> libc-2.26.so       ← glibc 2.26
```

二是原厂二进制的 ABI（软浮点还是硬浮点），照着选对应的工具链：

```sh
file /usr/sbin/ufwmg
readelf -A /usr/sbin/ufwmg | grep -iE 'Tag_ABI_VFP|float'
cat /proc/cpuinfo | grep Features
# Features : half thumb fastmult edsp tls   ← 没有 vfp
```

这台 CPU（Cortex-A9）`Features` 里没有 `vfp`，是软浮点，所以用 `arm-linux-gnueabi`（软浮点）那个变体，编译加 `-mfloat-abi=soft`。如果你的设备有 VFP / `Tag_ABI_VFP_args`，那要换 `arm-linux-gnueabihf`（硬浮点）。

下载并解压 Linaro 5.5：

```sh
wget https://releases.linaro.org/components/toolchain/binaries/5.5-2017.10/arm-linux-gnueabi/gcc-linaro-5.5.0-2017.10-x86_64_arm-linux-gnueabi.tar.xz
tar xf gcc-linaro-5.5.0-2017.10-x86_64_arm-linux-gnueabi.tar.xz
export PATH="$PWD/gcc-linaro-5.5.0-2017.10-x86_64_arm-linux-gnueabi/bin:$PATH"

arm-linux-gnueabi-gcc --version        # gcc 5.5.0
```

编两个 C 桩（静态链接，体积小、无依赖）：

```sh
arm-linux-gnueabi-gcc -static -Os -mcpu=cortex-a9 -mfloat-abi=soft \
    -o CuInformLoader CuInformLoader.c
arm-linux-gnueabi-gcc -static -Os -mcpu=cortex-a9 -mfloat-abi=soft \
    -o ufwmg ufwmg.c
file CuInformLoader        # ELF 32-bit LSB, ARM, statically linked
```

> 桩程序只用到 libc，静态链接没问题。但 dropbear 因为要调 `crypt()` 校验密码，静态链接反而会出兼容问题，那边得动态链接——这是第 8 节的主要内容。

---

## 8. 交叉编译 dropbear（轻量 SSH）

光猫默认只有 telnet，想用 SSH 就自己编个 dropbear。它对外部库依赖很少（zlib 都能禁掉），本以为很快搞定，结果在密码登录上卡了大半天，前后换了三套方案。问题根源就一个：**`crypt()` 这个函数**。密码哈希的生成端（光猫自带的 libc）和验证端（dropbear 链接的 libc）必须用同一套 `crypt()` 实现，否则同样的密码算出来的哈希对不上，怎么都登不进。先把这个结论放这，下面是踩坑的全过程。

### 8.0 先摸清目标平台

```sh
# 光猫上
uname -a
# Linux localhost 4.19.136 ... armv7l GNU/Linux

cat /proc/cpuinfo
# CPU part : 0xc09  → Cortex-A9
# Features : half thumb fastmult edsp tls   ← 没有 vfp

# libc 类型和版本（决定整个编译策略）
ls -l /lib/libc*.so*
# libc.so.6 -> libc-2.26.so      ← glibc 2.26
ls -l /lib/libcrypt*
# libcrypt.so.1 -> libcrypt-2.26.so
```

这台是 **ZTE ZX279127S，Cortex-A9，ARMv7，无 VFP，glibc 2.26**。两个事实决定了后面所有选择：没有 VFP，必须软浮点 `-mfloat-abi=soft`（硬浮点二进制一跑就 `Illegal instruction`）；glibc 2.26 是 2017 年的版本，比主流工具链自带的 glibc 老不少，这是后面 GLIBC 版本坑的根源。

> 补一句编译参数的细节：`-mcpu=cortex-a9` 和 `-march=armv7-a` 不能同时给，会报 `switch conflicts` 警告，留 `-mcpu` 一个就行。

### 8.1 坑一：musl 静态链接，crypt 实现不兼容

最先想到的是 musl 全静态，一个文件丢上去零依赖，最省心。用 [musl.cc](https://musl.cc) 的预编译工具链（省得自己 `musl-cross-make` 编半天）：

```sh
wget https://musl.cc/arm-linux-musleabi-cross.tgz
tar xf arm-linux-musleabi-cross.tgz
export PATH="$PWD/arm-linux-musleabi-cross/bin:$PATH"

./configure --host=arm-linux-musleabi --disable-zlib --disable-shadow \
    CC=arm-linux-musleabi-gcc LDFLAGS="-static" LIBS="-lcrypt" \
    CFLAGS="-mcpu=cortex-a9 -mfloat-abi=soft"
make PROGRAMS="dropbear dropbearkey dbclient scp"
```

编出来了，丢上去跑，前台 `-F -E` 看日志：

```
Bad password attempt for 'root' from ...
```

密码明明是对的（`su` 切换没问题），dropbear 就是说错。折腾了一会才反应过来：**musl 的 `crypt()` 对 `$1$`(MD5)、`$5$`(SHA-256) 的实现，和 glibc 算出来的哈希不一样**。我静态链接了 musl，验证密码用的是 musl 的 `crypt()`，而 `/etc/shadow` 里的哈希当年是用 glibc 的 `crypt()` 生成的，两套实现结果不同，自然对不上。

> 新版 musl（1.2.x）对 `$5$/$6$` 的兼容性据说改善了不少，理论上把 shadow 密码改成 `$6$` 再配新 musl 可能能通。但我没在这条路上继续——既然问题是两端 crypt 不一致，最干净的解法是让 dropbear 直接用光猫自己的 `crypt()`，也就是动态链接 glibc。

### 8.2 坑二：新工具链 GLIBC 版本过高 / libxcrypt 不兼容

既然要动态链接 glibc，先随手抓了个 Linaro 7.5（2019 年的，当时觉得够老了）：

```sh
./configure --host=arm-linux-gnueabi --disable-zlib --disable-shadow \
    LIBS="-lcrypt" CFLAGS="-mcpu=cortex-a9 -mfloat-abi=soft"
# 注意：去掉了 -static，动态链接
```

编出来传上去，连跑都跑不起来：

```
./testcrypt: /lib/libcrypt.so.1: version `XCRYPT_2.0' not found (required by ...)
./testcrypt: /lib/libc.so.6: version `GLIBC_2.34' not found (required by ...)
```

两个问题：`GLIBC_2.34 not found`，二进制要求 glibc ≥ 2.34，光猫只有 2.26，符号解析不了；`XCRYPT_2.0 not found`，现代工具链的 sysroot 用的是 libxcrypt（glibc 2.28 之后把 libcrypt 拆出去了），光猫那个老 libcrypt 没有这个版本符号。

这就引出前面那个结论：**交叉编译看的不是 gcc 版本号，而是工具链 sysroot 里的 glibc 版本。** Linaro 7.5 虽然 gcc 才 7.x，但它打包的 sysroot 是很新的 glibc + libxcrypt，编出来的东西自然要新符号。只看 `gcc --version` 会判断错。

> 中途还试过把光猫的 `libc-2.26.so`/`libcrypt-2.26.so` 拷回来当 sysroot 硬怼，结果撞上 `undefined reference to __snprintf@GLIBC_PRIVATE`——链接器混用了两套 libc 的私有符号，越搞越乱，不建议走这条路。

### 8.3 坑三：`--disable-shadow`，最隐蔽的一个

换上正确的老工具链（见 8.4）之后，`testcrypt` 能跑了、哈希也对上了，但 dropbear 登录还是失败，日志这次变成了：

```
User account 'root' is locked
```

`shadow` 里密码字段明明是正常的 `$5$...`，`passwd -u` 也说没锁定。这种时候别瞎猜，改源码加日志最快。看 `src/svr-authpasswd.c`，报这个错的地方是：

```c
testcrypt = crypt(password, passwdcrypt);   // passwdcrypt 从哪来？
...
if (testcrypt == NULL) {                     // crypt 返回 NULL 才报 locked
    dropbear_log(LOG_WARNING, "User account '%s' is locked", ...);
```

在判断前面插一行，把 dropbear 实际读到的哈希打出来：

```sh
sed -i '/if (testcrypt == NULL) {/i\        dropbear_log(LOG_WARNING, "DEBUG pw_passwd=[%s] testcrypt=[%s]", passwdcrypt ? passwdcrypt : "(null)", testcrypt ? testcrypt : "(null)");' src/svr-authpasswd.c
make PROGRAMS="dropbear"
```

传上去再登一次：

```
DEBUG pw_passwd=[x] testcrypt=[(null)]
User account 'root' is locked
```

`pw_passwd=[x]`，dropbear 读到的密码字段是 `/etc/passwd` 里的占位符 `x`，根本没去读 `/etc/shadow`。于是 `crypt("123456", "x")` 因为 salt 非法返回 `NULL`，报 "locked"（这个错误信息有点误导，实际是 salt 非法，不是账户锁定）。

问题出在前面 `./configure` 那个 **`--disable-shadow`**。我一开始加它，是为了绕开 configure 检测 `crypt()` 失败时报的 `DROPBEAR_SVR_PASSWORD_AUTH requires crypt()` 错误，结果副作用是 dropbear 不读 shadow 了，只认 passwd 里的 `x`。把 `--disable-shadow` 去掉，让它正常走 `getspnam()` 读 `/etc/shadow`，问题就没了。

> configure 报 `requires crypt()` 的正解，不是去 `--disable-shadow`，而是确认 `LIBS="-lcrypt"` 传对、工具链 sysroot 里有 libcrypt。绕过检测只是把问题推到运行期。

### 8.4 正确方案：Linaro 5.5（glibc 2.21）+ 动态链接 + 读 shadow

工具链就是第 7 节那套 Linaro 5.5-2017.10，它自带 glibc 2.21，比光猫的 2.26 老，编出来的符号光猫向后兼容；而且它是老版本，自带的还是 libcrypt（不是 libxcrypt），不会有 8.2 那个 `XCRYPT` 问题。

编译前先确认它的 libcrypt 支持目标哈希算法（这台 shadow 里是 `$5$`）：

```sh
arm-linux-gnueabi-strings $(arm-linux-gnueabi-gcc -print-sysroot)/lib/libcrypt.so.1 \
    | grep -E "sha256|sha512"
# 能看到 sha256-crypt / sha512-crypt → 支持 $5$ 和 $6$
```

编译。关键三点：动态链接（不加 `-static`，让运行期用光猫的 libcrypt）、不加 `--disable-shadow`、软浮点：

```sh
cd dropbear-2024.86
make distclean

./configure \
    --host=arm-linux-gnueabi \
    --disable-zlib \
    LIBS="-lcrypt" \
    CFLAGS="-mcpu=cortex-a9 -mfloat-abi=soft"

make PROGRAMS="dropbear dropbearkey dbclient scp"
arm-linux-gnueabi-strip dropbear dropbearkey dbclient scp
```

编完先看一下 GLIBC 符号版本，这步能提前判断会不会"version not found"：

```sh
arm-linux-gnueabi-objdump -p dropbear | grep -E "GLIBC|XCRYPT"
# 全部 <= GLIBC_2.17，且没有 XCRYPT → 光猫 2.26 向后兼容，没问题
```

### 8.5 用 testcrypt 单独验证 crypt

整个过程里最有用的一个小技巧，是写个十几行的程序单独验证 `crypt()`，把"crypt 实现对不对"从 dropbear 一堆逻辑里隔离出来单测：

```c
/* testcrypt.c */
#include <stdio.h>
#define _GNU_SOURCE
#include <crypt.h>
int main(void) {
    /* 直接抄 /etc/shadow 里 root 那段哈希（含 $5$salt$ 部分） */
    const char *hash = "$5$IN2FNhbihRCYkpmQ$xLCr2gqEHCqzMUx3.q0/He4iGUtZVPwAF5omoXSvV12";
    char *r = crypt("123456", hash);
    printf("result: %s\n", r ? r : "NULL");
    return 0;
}
```

用同一套工具链编，传到光猫跑：

```sh
arm-linux-gnueabi-gcc testcrypt.c -o testcrypt -lcrypt \
    -mcpu=cortex-a9 -mfloat-abi=soft
arm-linux-gnueabi-strip testcrypt
# 传到光猫
./testcrypt
# result: $5$IN2FNhbihRCYkpmQ$xLCr2gqEHCqzMUx3.q0/He4iGUtZVPwAF5omoXSvV12
```

输出的哈希和 `/etc/shadow` 里的完全一致，说明 crypt 链路通了，密码就能验证成功。这步绿了，dropbear 登录基本就稳了。建议每次换工具链或换链接方式后，先拿它打个样，别直接上 dropbear（变量太多不好定位）。

最终登录成功的日志（debug 行还留着，正好对照）：

```
DEBUG pw_passwd=[$5$IN2FNhbihRCYkpmQ$xLCr...] testcrypt=[$5$IN2FNhbihRCYkpmQ$xLCr...]
Password auth succeeded for 'root' from 192.168.1.23
```

两个值一致，登录通过。验证完把 debug 日志删掉重编一次：

```sh
sed -i '/DEBUG pw_passwd/d' src/svr-authpasswd.c
make PROGRAMS="dropbear dropbearkey dbclient scp"
arm-linux-gnueabi-strip dropbear dropbearkey dbclient scp
```

> `lastlog_perform_login: Couldn't stat /var/log/lastlog` 这类警告是光猫没建 lastlog 文件导致的，不影响登录，无视即可。

### 8.6 部署与运行（光猫上）

这台 ZX279127S 的 root 可写（`mount -o remount,rw /`），文件直接按 FHS 放到 `/usr/sbin`、`/usr/bin` 即可：

```sh
# 1) 二进制就位
cp dropbear    /usr/sbin/dropbear
cp dropbearkey /usr/bin/dropbearkey
cp dbclient    /usr/bin/dbclient
cp scp         /usr/bin/scp
chmod +x /usr/sbin/dropbear /usr/bin/dropbearkey /usr/bin/dbclient /usr/bin/scp

# 2) 确认运行期依赖都能解析（不能有 not found）
ldd /usr/sbin/dropbear
# libcrypt.so.1 => /lib/libcrypt.so.1
# libc.so.6     => /lib/libc.so.6

# 3) 生成主机密钥
mkdir -p /etc/dropbear
dropbearkey -t rsa   -f /etc/dropbear/dropbear_rsa_host_key
dropbearkey -t ecdsa -f /etc/dropbear/dropbear_ecdsa_host_key

# 4) 前台调试（-F 不后台，-E 日志到 stderr）
dropbear -r /etc/dropbear/dropbear_rsa_host_key \
         -r /etc/dropbear/dropbear_ecdsa_host_key -p 22 -F -E

# 5) OK 后后台常驻
dropbear -r /etc/dropbear/dropbear_rsa_host_key \
         -r /etc/dropbear/dropbear_ecdsa_host_key -p 22
```

> 若你的固件像 SG631Z 那样给 `/usr`、`/bin` 挂了 overlay，**直接 cp 进去是 tmpfs，重启没了**，得用第 10 节的 bind-mount 写进 rootfs 真身，或把启动命令加进开机脚本（如 `/etc/init.d/rcS`）。主机密钥放 `/etc/dropbear` 这种可持久分区。

### 8.7 小结：crypt 这条链怎么才能通

一句话——**密码哈希的生成端和验证端必须用同一套 `crypt()` 实现**。三个坑本质都是这一条的不同侧面：

| 方案 | crypt 来源 | 结果 |
|---|---|---|
| musl 静态 | musl 自己的 crypt | `$1$/$5$` 算法和 glibc 不一致 → Bad password |
| 新工具链（Linaro 7.5）动态 | sysroot 的 libxcrypt + glibc 2.38 | 二进制要 GLIBC_2.34 / XCRYPT_2.0，光猫 2.26 → version not found |
| 老工具链（Linaro 5.5）动态 + 读 shadow | **运行期光猫自己的 libcrypt-2.26** | ✅ crypt 实现天然一致，哈希匹配 |

外加一个独立的坑：`--disable-shadow` 会让 dropbear 只读 passwd 的占位 `x`，报误导性的 "account is locked"，去掉即可。

---

## 9. 交叉编译 mtd-utils（flash_erase / ubiformat / ubiattach 等）

`mtd-utils` 提供 `flash_erase`、`nanddump`、`mtdinfo`，以及一整套 UBI 卷管理工具（`ubiformat`、`ubiattach`、`ubidetach`、`ubimkvol`、`ubirmvol`、`ubiupdatevol` 等）——也就是第 2 节里那套真正能擦动 NAND/UBI 的工具。

> **为什么非自己编不可**：如第 2 节所述，光猫官方 busybox 里那套 NAND 工具能力不全、擦不干净；只有真·mtd-utils 的 `flash_erase`(nand erase) / `ubiformat` 才能真正擦除、格式化。这是本节的直接动机。

### 9.1 这些工具其实不依赖 zlib/lzo/uuid

网上不少教程一上来就让你交叉编译 zlib、lzo、e2fsprogs(libuuid) 三个依赖。但实际试下来，真正需要这三个依赖的只有 `mkfs.ubifs` 和 `mkfs.jffs2`（构建文件系统镜像那两个）。`flash_erase`、`ubiformat`、`ubiattach`、`ubimkvol` 这些运行期擦除和卷管理工具，只依赖 libmtd/libubi + libc，没有第三方库。

所以如果只是想擦分区、格式化 UBI、建卷、挂载，根本不用编那三个依赖，把两个 mkfs 用 configure 选项关掉就行。

工具链继续用第 8 节那套 Linaro 5.5（glibc 2.21）软浮点，和光猫 ABI / glibc 兼容：

```sh
export PATH="$HOME/gcc-linaro-5.5.0-2017.10-x86_64_arm-linux-gnueabi/bin:$PATH"

wget http://ftp.infradead.org/pub/mtd-utils/mtd-utils-2.1.6.tar.bz2
tar xjf mtd-utils-2.1.6.tar.bz2 && cd mtd-utils-2.1.6
```

### 9.2 configure 报 `missing one or more dependencies`

照默认 `./configure` 走，直接挂：

```
configure: WARNING: cannot find uuid library required for mkfs.ubifs
configure: WARNING: cannot find ZLIB library required for mkfs programs
configure: error: missing one or more dependencies
```

原因是默认要编 `mkfs.ubifs`/`mkfs.jffs2`，找不到 uuid/zlib 就报错中止。解法不是去补依赖，而是把这两个用不到的 mkfs 关掉：

```sh
./configure \
    --host=arm-linux-gnueabi \
    --without-ubifs \
    --without-jffs \
    --without-xattr \
    --without-zstd \
    --without-lzo \
    --without-crypto \
    --disable-tests \
    CC=arm-linux-gnueabi-gcc \
    CFLAGS="-mcpu=cortex-a9 -mfloat-abi=soft -static" \
    LDFLAGS="-static"
```

> 关键是 `--without-ubifs --without-jffs`（关掉依赖 uuid/zlib/openssl 的 mkfs），其余 `--without-*` 是顺手砍掉用不上的可选特性。不同 mtd-utils 版本选项名可能略有出入，编不过时先 `./configure --help | grep -iE 'without|disable'` 对一下准确名字，报 `unrecognized` 的去掉即可。

### 9.3 编译需要的工具

configure 过了之后，**按需逐个编**，不用 `make` 全量（全量可能还会去碰被禁掉的目标）：

```sh
# flash_erase（擦除）
make flash_erase

# 一整套 UBI 工具
make ubiattach ubidetach ubiformat ubimkvol ubirename ubirmvol ubirsvol ubiupdatevol

# 顺手编个 mtdinfo，用来查 NAND 页大小/子页/OOB 等参数（见 9.5）
make mtdinfo
```

如果某个目标报 "No rule to make target"，说明这版 Makefile 把它放在子目录路径下，改带路径写法：

```sh
make ubi-utils/ubiformat ubi-utils/ubiattach ubi-utils/ubimkvol ...
```

strip 全部产物：

```sh
find . -maxdepth 2 -type f -executable \
  \( -name 'flash_erase' -o -name 'ubi*' -o -name 'mtdinfo' -o -name 'nanddump' \) \
  -exec arm-linux-gnueabi-strip {} \;

file flash_erase ubiformat        # ELF 32-bit LSB executable, ARM, statically linked
```

部署：`cp` 到光猫（root 可写直接放 `/usr/sbin` 或 `/usr/bin`），`chmod 755`。静态链接无依赖，传上去就能跑。

### 9.4 如果你确实要 mkfs.ubifs（才需要那套依赖）

只有当你要**离线制作 ubifs 镜像**（而不是在机器上现格式化）时，才需要把 zlib/lzo/uuid 交叉编译成静态库再喂给 configure。这条重路子留个备份，平时用不到：

```sh
export CROSS=arm-linux-gnueabi CC=${CROSS}-gcc PREFIX=$PWD/sysroot
# zlib
./configure --static --prefix=$PREFIX            # zlib 用自带 configure
# lzo / e2fsprogs(libuuid)
./configure --host=$CROSS --enable-static --disable-shared --prefix=$PREFIX CC=$CC
# 然后 mtd-utils 不加 --without-ubifs，改为传依赖路径：
./configure --host=$CROSS --disable-tests --without-xattr CC=$CC \
    CFLAGS="-I$PREFIX/include" LDFLAGS="-L$PREFIX/lib -static" \
    ZLIB_CFLAGS="-I$PREFIX/include" ZLIB_LIBS="-lz" \
    LZO_CFLAGS="-I$PREFIX/include"  LZO_LIBS="-llzo2" \
    UUID_CFLAGS="-I$PREFIX/include" UUID_LIBS="-luuid"
```

> 更省事的路线是直接用 **OpenWrt SDK / buildroot** 出的 `mtd-utils` 包，ABI 天然一致。但对"只要擦盘+格式化"的需求来说，9.2 的 `--without-ubifs` 路子已经够用，没必要上这套。

### 9.5 用之前：先在光猫上查清 NAND 参数

`ubiformat` 默认能自动检测参数，但格式化前最好先搞清这块 NAND 的物理参数，尤其是**子页大小**（影响 VID header offset）。直接读 sysfs，不用猜：

```sh
cat /proc/mtd                              # 分区列表 + name + erasesize
cat /sys/class/mtd/mtd12/writesize         # 页大小，如 2048
cat /sys/class/mtd/mtd12/subpagesize       # 子页大小，如 2048（=页大小说明不分子页）
cat /sys/class/mtd/mtd12/oobsize           # OOB 大小，如 64
cat /sys/class/mtd/mtd12/erasesize         # 擦除块，如 131072 (128KiB)
cat /sys/class/mtd/mtd12/type              # nand / nor
```

或用刚编好的 `mtdinfo`（信息更全）：

```sh
mtdinfo /dev/mtd12
mtdinfo -a            # 所有分区 + NAND 总体参数
```

**实测这台 ZX279127S 的某数据分区：writesize=2048、subpagesize=2048（即不分子页）、oobsize=64、erasesize=128KiB。** 因为 `subpagesize == writesize`，`ubiformat` 不用手动加 `-s`/`-O`，自动检测就对。

### 9.6 擦除 + 格式化 + 建卷 + 挂载（完整流程）

> ⚠️ **擦错分区会变砖**。动手前务必 `cat /proc/mtd` + `cat /sys/class/mtd/mtdN/name` 确认目标是数据分区，别碰 boot/kernel/rootfs。`ubi0` 这种挂在 `/` 的 rootfs **绝对不能格式化**。查某个 ubi 设备对应哪个 mtd：`cat /sys/class/ubi/ubi1/mtd_num`。

```sh
MTD_NUM=12
MTD_DEV=/dev/mtd${MTD_NUM}
VOL_NAME=data
MOUNT_POINT=/mnt/data

# 1) 若该分区已被挂载/attach，先卸载再 detach（否则 ubiformat 报 device busy）
umount ${MOUNT_POINT} 2>/dev/null
ubidetach /dev/ubi_ctrl -m ${MTD_NUM} 2>/dev/null

# 2) 格式化（清空所有数据；subpagesize=writesize，不用 -s/-O，自动检测）
#    必须操作字符设备 /dev/mtdN，不能用块设备 /dev/mtdblockN
ubiformat ${MTD_DEV} -y
#    想彻底重置擦除计数器再加 -e 0（一般不需要，默认保留 EC 利于磨损均衡）

# 3) attach 到 UBI（-d 显式指定设备号，避免和已有 ubi0/ubi1 混淆）
ubiattach /dev/ubi_ctrl -m ${MTD_NUM} -d 2

# 4) 建卷（-m 用满剩余空间；-N 卷名必填，没有默认空名）
ubimkvol /dev/ubi2 -N ${VOL_NAME} -m
#    指定大小则： ubimkvol /dev/ubi2 -N ${VOL_NAME} -s 100MiB

# 5) 挂载（用 设备:卷名 格式；也可用卷号 /dev/ubi2_0）
mkdir -p ${MOUNT_POINT}
mount -t ubifs ubi2:${VOL_NAME} ${MOUNT_POINT}

# 6) 验证
ubinfo /dev/ubi2
mount | grep ubi
df -h ${MOUNT_POINT}
```

几个实操要点：`ubiattach` 后生成的设备号不一定是 `ubi0`，光猫如果已有 rootfs(ubi0)、apps(ubi1) 在用，新 attach 的就接着排（用 `-d 2` 强制指定最清楚）。`ubimkvol` 的 `-N` 卷名是**必填**的，不给会报 `volume name was not specified`，随便起个固定名即可，对功能无影响。挂载既可用 `ubi2:卷名`，也可用卷号 `/dev/ubi2_0`，后者不用记名字。

---

## 10. 持久化落地：bind-mount 改 rootfs 真身

核心难点：目标目录（`/bin`、`/usr`）挂着 overlay，**直接写只进 tmpfs**；而且有的 overlay "繁忙"无法 umount。解决办法是 **bind-mount 把整个 rootfs 再挂一份到别处**，绕过 overlay 直接编辑底层：

```sh
# 1) rootfs 解只读
mount -o remount,rw /

# 2) 把根再 bind 挂到 /tmp/rr —— 这里看到的是“没有 overlay 遮挡的真身”
mkdir -p /tmp/rr && mount --bind / /tmp/rr

# 3) 备份原二进制，再写入桩
cp /tmp/rr/usr/sbin/ufwmg /tmp/rr/usr/sbin/ufwmg.bak
cp /tmp/ufwmg            /tmp/rr/usr/sbin/ufwmg       # 交叉编译好的 C 桩
chmod 755 /tmp/rr/usr/sbin/ufwmg

cp /tmp/rr/bin/CuInformLoader /tmp/rr/bin/CuInformLoader.bak
cp /tmp/CuInformLoader        /tmp/rr/bin/CuInformLoader
chmod 755 /tmp/rr/bin/CuInformLoader

# 4) 落盘并卸载
sync && umount /tmp/rr
```

### CuInform 还要堵住一处"开机重新拷贝"

`/etc/init.d/cuinform-rcS` 里有一句会在每次开机把真身拷回 `/bin`，盖掉你的桩，必须注释掉：

```sh
# 第 43 行：cp -f /CuInform/CuInformLoader /bin/
```

（ufwmg 没有这种"开机重拷"，替换 ELF 即可，省事。）

### 验证 + 让 pc 重拉新桩

```sh
head -c 4 /usr/sbin/ufwmg | xxd          # 7f45 4c46 = \x7fELF，确认不是脚本
head -c 4 /bin/CuInformLoader | xxd

# 杀掉旧进程让 pc 重拉新桩
kill 1980 ; killall ufwmg
killall CuInformLoader

# 复查：稳定停在一个新 PID、不再反复重启 = 成功
ps | grep -E 'ufwmg|CuInform'
```

### 回滚

```sh
mount -o remount,rw /
mkdir -p /tmp/rr && mount --bind / /tmp/rr
cp /tmp/rr/usr/sbin/ufwmg.bak        /tmp/rr/usr/sbin/ufwmg
cp /tmp/rr/bin/CuInformLoader.bak    /tmp/rr/bin/CuInformLoader
# 别忘了把 cuinform-rcS 第 43 行的注释去掉
sync && umount /tmp/rr
reboot
```

---

## 11. 踩坑清单

| 坑 | 现象 | 原因 / 正解 |
|---|---|---|
| 注释 `S01userconfig` 第 87 行 | telnet 起不来、开机卡死 | 它负责所有 overlay/存储，别动；用 `sed -i '87s/^#//'` 救回 |
| 注释 `rcS` 第 339 行 | 后台疯狂 grep loader | pc 找不到 loader 就 respawn 风暴；保留它，改用桩 |
| 直接删/改 `/bin` 里的文件 | 重启还原 | `/bin` 挂 overlay，写进的是 tmpfs；要 bind 到 rootfs |
| busybox NAND 工具擦分区 | "弄不掉"、擦了没效果 | busybox 那套 NAND 工具能力不全；换自编 mtd-utils 的 `flash_erase`/`ubiformat` 即可 |
| dropbear（musl 静态） | 密码对也报 Bad password | musl 的 `crypt()` 和 glibc 算法不一致，哈希对不上；改动态链接 glibc |
| dropbear（新工具链 Linaro 7.5） | 跑不起来 `GLIBC_2.34/XCRYPT_2.0 not found` | sysroot 是新 glibc+libxcrypt；**看 sysroot glibc 版本，别看 gcc 版本号**，换 Linaro 5.5（glibc 2.21） |
| dropbear 报 `account is locked` | shadow 哈希正常也报锁 | `--disable-shadow` 害的：只读 passwd 的占位 `x`，`crypt(pw,"x")` 返回 NULL；去掉该选项走 `/etc/shadow` |
| 定位 dropbear 登录问题 | 日志信息误导 | 写 `testcrypt.c` 单测 `crypt()`；在 `svr-authpasswd.c` 的 `is locked` 前加 `dropbear_log` 打 `pw_passwd` |
| mtd-utils configure | `missing one or more dependencies` | 默认要编 mkfs.ubifs/jffs2 才需要 uuid/zlib；只擦盘的话 `--without-ubifs --without-jffs` 关掉即可，不用编依赖 |
| ubiformat 报 device busy | 格式化失败 | 分区已被 attach/mount；先 `umount` 再 `ubidetach -m N`，且必须操作 `/dev/mtdN` 不是 `mtdblockN` |
| 想二进制 patch `pc` | strings 找不到插件名 | pc 数据驱动，不可 patch；改用桩 |
| `ps -w` | 不支持 | busybox ps 没有 `-w`，用 `/proc/<pid>/cmdline` 等 |
| 插件 kill 了又起来 | 找不到谁拉的 | busybox ps 不显示 PPID；用 `top` 或 `cat /proc/<pid>/status \| grep PPid` 找父进程（这里是 pc）|
| umount `/usr` | 繁忙 | 用 `mount --bind / /tmp/rr` 绕过 |
| shell 桩 ps 很假 | `{X} /bin/sh /bin/X` | 用同名 C ELF，comm 与 argv 自然一致 |

---

## 12. 最后：固件升级会冲掉一切

所有改动都落在 **rootfs（ubifs 可写、持久）** 上。但——

> **固件升级会重写整个 rootfs。** 升级后，两个 C 桩、`cuinform-rcS` 里注释掉的那行、dropbear、mtd-utils 全部会被覆盖、官方插件复活。

升级后照本文流程重做一遍即可：交叉编译 → bind-mount 写 rootfs → 注释开机重拷行 → 让 pc 重拉。把源码和编好的二进制留一份在 PC 上，重做只要几分钟。

---

*本文记录的是对自有设备的离线研究与定制，仅供学习交流。操作有变砖风险，务必先备份、保留 telnet/串口/网页 shell 等回退手段。*
