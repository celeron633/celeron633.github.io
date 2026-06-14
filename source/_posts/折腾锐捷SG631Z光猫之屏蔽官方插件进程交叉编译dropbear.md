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

# 折腾中国联通 ZTE SG631Z 光猫：屏蔽官方插件进程 + 交叉编译 dropbear / mtd-utils

> 记录一次对中国联通定制 ZTE SG631Z（OpenWRT 衍生固件，kernel 5.15）光猫的"减负"过程：
> 把厂商塞进来的 DPI / 应用框架插件干掉、搞清楚它为什么"杀不死、删不掉、格式化还原回来"，
> 最后用 C 写了两个 daemon 桩进程替换官方二进制，并交叉编译了 dropbear 和 mtd-utils。
>
> 全程在一台完全离线的设备上完成，靠 telnet 和网页后台的一个 shell。

---

## 0. 环境与目标

- **设备**：ZTE SG631Z ONT（光猫 / 光纤猫）
- **固件**：中国联通定制，`CMS_GIT_BRANCH=UNICOM_2024`，`CMS_SOFT_VERSION=V3.0.2`，kernel 5.15，OpenWRT 衍生
- **架构**：ARM 32-bit（小端）
- **现状**：设备完全离线（没有接入 ISP）
- **手头资源**：
  - 一份从光猫导出的 `/etc` 离线备份（在 Windows 上）
  - telnet 进设备
  - 网页后台自带的一个 web shell（关键时刻救命用）

**目标**：让两个官方插件**永久不再运行**：

1. **CuInformLoader** —— 一个 DPI（深度包检测）插件加载器
2. **ufwmg** —— 一个跑在 LXC 容器里的应用框架插件（容器名 `ufw`，**注意这不是 Linux 那个防火墙 ufw**）

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

## 2. 为什么"格式化 mtd12/mtd13 没用、会被还原"

最早的困惑：把 `mtd12`、`mtd13` 格式化掉，重启又原样回来了。设备明明离线，哪来的"还原源"？

答案在 `/etc/default/S01userconfig_ctcc_smart` 里。

### mtd12（ubifs，`/opt/cu/apps`，运行期暂存）

开机脚本会**自己重建**它：

```sh
if nanddump /dev/mtd12 -p -l 2048 | awk '{if(NR==1)print $0;}' | grep -q '55 42 49 23 01'; then
    echo "  Has found plugin ubifsimg!"
    ubiattach /dev/ubi_ctrl -m 12
else
    echo "  No found plugin ubifsimg!"
    ubiformat /dev/mtd12 -q -e 0       # ← 没有 UBI 头就重新格式化
    ubiattach /dev/ubi_ctrl -m 12
    ubimkvol /dev/ubi1 -n 0 -N plugin_ubifs -m
fi
```

`55 42 49 23` 就是 ASCII 的 `UBI#`（UBI 卷头魔数）。你把它擦了 → 开机检测不到 UBI 头 → 脚本**自动 `ubiformat` 重建一个空的**。而且脚本结尾还会每次开机清空子目录：

```sh
rm -rf /opt/cu/apps/bin/*
rm -rf /opt/cu/apps/usr/*
rm -rf /opt/cu/apps/lib/*
rm -rf /opt/cu/apps/etc/*
```

所以 mtd12 是**运行期 scratch**，怎么改都白搭，开机重置。

### rootfs 是只读的 ubifs

```
ubi0:rootfs on / type ubifs (ro,relatime,assert=read-only,ubi=0,vol=0)
```

要持久改动得先：

```sh
mount -o remount,rw /
```

### 重要更正：mtd12/13"弄不掉"是 busybox 工具的锅，不是真擦不掉

这里要澄清一个我自己一开始也搞错的点。mtd12/13 最初"擦不掉、改不动"，**根因是光猫官方 busybox 里那套 NAND 工具（`nand_write` 之流）能力不全，压根没把内容真正擦干净**。后来换成**自己交叉编译的 mtd-utils**，用里面的 `flash_erase`（nand erase）和 `ubiformat` 那一套，**擦除/格式化完全没问题**（这也正是第 9 节要自己编一套 mtd-utils 的直接动机）。

所以要分两层看，别混为一谈：

| | 擦除这个动作本身 | 擦完会不会"回来" |
|---|---|---|
| **mtd12** | busybox 搞不定；mtd-utils 的 `ubiformat` 一擦一个准 | **会**——下次开机 `S01userconfig_ctcc_smart` 检测到没有 UBI 头(`55 42 49 23`)又自动重建，这是脚本行为、跟工具无关 |
| **mtd13** | 同上，busybox 不行、mtd-utils 行 | **不会**——用 mtd-utils 擦掉后是真的擦掉并保持（没有开机脚本去重写它） |

一句话：**"擦不掉"是工具问题（busybox 不行 → 换 mtd-utils），"擦了又回来"才是脚本问题（仅 mtd12）。** 这两个原因当初被我并到一起了，特此分开。

**结论**：mtd12 别碰（即便擦了开机也自动重建）；真正能持久化的改动都往 **rootfs 真身**上落。

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

用 busybox 的 `ps`（注意 **busybox 的 ps 没有 `-w`**）和 `/proc` 来看。当时抓到的关键输出：

```
PID   USER     COMMAND
1115  root     pc                                  ← 看门狗，一堆进程的父进程
1967  root     ufwmg service 10 11 12              ← ufwmg 本体
1978  root     CuInformLoader                       ← CuInform 本体（父进程是 1115 pc）
1980  root     {ufwmg} [lxc monitor] /opt/cu/framework ufw   ← ufw 的 LXC 容器监控进程
```

确认真正的可执行文件：

```sh
readlink /proc/1967/exe        # → /usr/sbin/ufwmg   （ufwmg 是 ELF）
cat /proc/1967/cmdline | tr '\0' ' '; echo
hexdump -C /usr/sbin/ufwmg | head        # 看到 7f 45 4c 46 = \x7fELF，ARM 32-bit
```

几个常用的"没有 `ps -w`"替代手段：

```sh
ps | grep -E 'ufwmg|CuInform'
cat /proc/<pid>/cmdline | tr '\0' ' '; echo     # 完整命令行
readlink /proc/<pid>/exe                          # 真正的可执行文件路径
cat /proc/<pid>/maps                              # 加载了哪些 so / 哪个 text 段
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

## 7. 交叉编译工具链

设备是 ARM 32-bit，光猫上没有 gcc，全部在 Linux/WSL 上交叉编译。**强烈建议静态链接**，避免和光猫上那套老 libc 对不上。

先确认原二进制的 ABI（软浮点 vs 硬浮点），照着选工具链：

```sh
file /usr/sbin/ufwmg.bak
readelf -A /usr/sbin/ufwmg.bak | grep -iE 'Tag_ABI_VFP|float'
```

- `EABI5` + 无 VFP/硬浮点标记 → 用 **`arm-linux-gnueabi-gcc`**（soft-float）
- 有 `hard-float` / `Tag_ABI_VFP_args` → 用 **`arm-linux-gnueabihf-gcc`**

Debian/Ubuntu 装工具链：

```sh
sudo apt-get install gcc-arm-linux-gnueabi    # 或 gcc-arm-linux-gnueabihf
```

编译两个桩：

```sh
arm-linux-gnueabi-gcc -static -Os -o CuInformLoader CuInformLoader.c
arm-linux-gnueabi-gcc -static -Os -o ufwmg         ufwmg.c
file CuInformLoader        # 应为 ELF 32-bit LSB, ARM, statically linked
```

---

## 8. 交叉编译 dropbear（轻量 SSH）

光猫默认只有 telnet，想要 SSH 就编个 dropbear。它对外部库依赖极少（可禁 zlib），非常适合静态交叉编译。

```sh
wget https://matt.ucc.asn.au/dropbear/releases/dropbear-2022.83.tar.bz2
tar xjf dropbear-2022.83.tar.bz2
cd dropbear-2022.83

# 交叉 + 静态；禁用 zlib 省得再编一个依赖
./configure \
    --host=arm-linux-gnueabi \
    --disable-zlib \
    CC=arm-linux-gnueabi-gcc \
    LDFLAGS="-static"

# MULTI=1 把 dropbear/dbclient/dropbearkey/scp 打成一个多功能单文件，省空间
make PROGRAMS="dropbear dbclient dropbearkey scp" MULTI=1 STATIC=1

file dropbearmulti        # 确认 ARM, statically linked
arm-linux-gnueabi-strip dropbearmulti      # 瘦身（用对应工具链的 strip）
```

### 关键坑：登录一直"password error" —— 要让 dropbear 读 /etc/shadow

第一次编出来死活登不进，**不管密码对不对都报认证失败**。原因是 **dropbear 默认从 `/etc/passwd` 取密码字段做校验，而光猫的 `/etc/passwd` 里密码位只是占位符 `x`，真正的哈希在 `/etc/shadow`**：

```
root:x:0:0:...           ← passwd 里是 x（占位）
root:$6$xxxx$yyyy:....    ← shadow 里才是真哈希
```

于是 `crypt(你输入的密码, "x")` 永远不等于 `"x"` → **怎么输都是"密码错误"**。

解决：**编译时打开 shadow 支持，让它走 `getspnam()` 读 `/etc/shadow`**。dropbear 是在编译期检测 `<shadow.h>` 来决定走不走 shadow 的，所以要确保：

1. 交叉工具链的 sysroot 里有 `shadow.h` —— configure 据此定义 `HAVE_SHADOW_H`；必要时在 `localoptions.h`（放在 `default_options.h` 旁边）里显式开启口令认证相关选项后重新 `./configure && make`。
2. 编完核对一下确实编进去了：
   ```sh
   strings dropbearmulti | grep -i shadow      # 应能看到 /etc/shadow 字样
   ```
3. 再确认目标机 `/etc/shadow` 用的哈希算法（`$1$`=MD5 / `$5$`=SHA-256 / `$6$`=SHA-512）你这份静态 `crypt()` 支持——静态链接 glibc 一般 `$1$/$5$/$6$` 都认；个别精简 libc 只认 `$1$`，那就把口令哈希降级成 MD5 再试。

改成读 shadow 后，用 root 的真实密码即可正常登录。**这一步是整个 dropbear 流程里最容易卡住的地方**，务必先确认再折腾别的。

部署与运行（光猫上）：

```sh
# 1) 传到设备某处，按多功能单文件方式建软链
cp dropbearmulti /usr/sbin/dropbearmulti
ln -sf dropbearmulti /usr/sbin/dropbear
ln -sf dropbearmulti /usr/bin/dbclient
ln -sf dropbearmulti /usr/bin/dropbearkey
ln -sf dropbearmulti /usr/bin/scp

# 2) 生成主机密钥（放到可写、好持久化的位置）
mkdir -p /etc/dropbear
dropbearkey -t rsa -f /etc/dropbear/dropbear_rsa_host_key
dropbearkey -t ed25519 -f /etc/dropbear/dropbear_ed25519_host_key

# 3) 前台调试跑一把（-F 不后台，-E 日志到 stderr）
dropbear -F -E -p 22

# 4) OK 后后台常驻
dropbear -p 22
```

> 注意：`/usr`、`/bin` 都挂了 overlay，**直接 cp 进去是 tmpfs，重启没了**。要持久得用第 9 节的 bind-mount 写进 rootfs，或把启动命令加进开机脚本。主机密钥建议放 `/etc/dropbear` 或别的可持久分区。

---

## 9. 交叉编译 mtd-utils（ubiformat / nanddump / flash_erase 等）

`mtd-utils` 提供 `ubiformat`、`ubiattach`、`ubimkvol`、`nanddump`、`flash_erase`、`mkfs.ubifs` 等——也就是第 2 节里那套操作 NAND/UBI 的工具。

> **为什么非自己编不可**：如第 2 节所述，光猫官方 busybox 里的 NAND 工具（`nand_write` 那套）**擦不动 mtd12/13**；只有这套真·mtd-utils 的 `flash_erase`(nand erase) / `ubiformat` 才能真正擦除、格式化。这是本节的直接动机。

它依赖 **zlib、lzo、e2fsprogs(libuuid)**，比 dropbear 麻烦，要先把依赖也交叉编译成静态库。

> 省事路线：直接用 **OpenWrt SDK / buildroot** 出来的 `mtd-utils` 包，能保证和固件 ABI 一致。下面给手动 autotools 全静态的路子。

```sh
# 统一前缀变量
export CROSS=arm-linux-gnueabi
export CC=${CROSS}-gcc
export PREFIX=$PWD/sysroot          # 把依赖装到这个本地 sysroot

# --- 依赖 1: zlib ---
wget https://zlib.net/zlib-1.3.1.tar.gz && tar xf zlib-1.3.1.tar.gz && cd zlib-1.3.1
CC=$CC ./configure --static --prefix=$PREFIX
make && make install
cd ..

# --- 依赖 2: lzo ---
wget https://www.oberhumer.com/opensource/lzo/download/lzo-2.10.tar.gz
tar xf lzo-2.10.tar.gz && cd lzo-2.10
./configure --host=$CROSS --enable-static --disable-shared --prefix=$PREFIX CC=$CC
make && make install
cd ..

# --- 依赖 3: e2fsprogs（只要 libuuid）---
wget https://mirrors.edge.kernel.org/pub/linux/kernel/people/tytso/e2fsprogs/v1.47.0/e2fsprogs-1.47.0.tar.gz
tar xf e2fsprogs-1.47.0.tar.gz && cd e2fsprogs-1.47.0
./configure --host=$CROSS --enable-static --disable-shared --prefix=$PREFIX CC=$CC
make libs && make install-libs
cd ..

# --- 主体: mtd-utils ---
wget https://infraroot.at/pub/mtd/mtd-utils-2.1.6.tar.bz2
tar xf mtd-utils-2.1.6.tar.bz2 && cd mtd-utils-2.1.6

./configure \
    --host=$CROSS \
    --disable-tests \
    --without-xattr \
    CC=$CC \
    CFLAGS="-I$PREFIX/include" \
    LDFLAGS="-L$PREFIX/lib -static" \
    ZLIB_CFLAGS="-I$PREFIX/include" ZLIB_LIBS="-lz" \
    LZO_CFLAGS="-I$PREFIX/include"  LZO_LIBS="-llzo2" \
    UUID_CFLAGS="-I$PREFIX/include" UUID_LIBS="-luuid"

make

# 产物（静态）
file ubiformat nanddump flash_erase mkfs.ubifs
${CROSS}-strip ubiformat nanddump flash_erase mkfs.ubifs ...
```

部署：把需要的几个工具 `cp` 到设备（同样注意 overlay/持久化问题），`chmod 755` 即可。

> 排错提示：configure 报找不到 zlib/lzo/uuid，多半是 `*_CFLAGS` / `*_LIBS` 没传对，或依赖没装进同一个 `$PREFIX`。全静态时务必确认每个依赖都 `--enable-static --disable-shared`。

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

## 11. 踩坑清单（血泪总结）

| 坑 | 现象 | 原因 / 正解 |
|---|---|---|
| 注释 `S01userconfig` 第 87 行 | telnet 起不来、开机卡死 | 它负责所有 overlay/存储，别动；用 `sed -i '87s/^#//'` 救回 |
| 注释 `rcS` 第 339 行 | 后台疯狂 grep loader | pc 找不到 loader 就 respawn 风暴；保留它，改用桩 |
| 直接删/改 `/bin` 里的文件 | 重启还原 | `/bin` 挂 overlay，写进的是 tmpfs；要 bind 到 rootfs |
| busybox `nand_write` 擦 mtd12/13 | "弄不掉"、擦了没效果 | busybox 那套 NAND 工具能力不全；换自编 mtd-utils 的 `flash_erase`/`ubiformat` 即可 |
| 格式化 mtd12 | 重启被重建 | 即便擦成功，开机脚本检测无 UBI 头又自动 `ubiformat`（仅 mtd12，mtd13 擦了能保持）|
| dropbear 登录 | 不管密码对错都"password error" | 默认读 `/etc/passwd`(占位 `x`)；要编进 shadow 支持走 `/etc/shadow` |
| 想二进制 patch `pc` | strings 找不到插件名 | pc 数据驱动，不可 patch；改用桩 |
| `ps -w` | 不支持 | busybox ps 没有 `-w`，用 `/proc/<pid>/cmdline` 等 |
| umount `/usr` | 繁忙 | 用 `mount --bind / /tmp/rr` 绕过 |
| shell 桩 ps 很假 | `{X} /bin/sh /bin/X` | 用同名 C ELF，comm 与 argv 自然一致 |

---

## 12. 最后：固件升级会冲掉一切

所有改动都落在 **rootfs（ubifs 可写、持久）** 上。但——

> **固件升级会重写整个 rootfs。** 升级后，两个 C 桩、`cuinform-rcS` 里注释掉的那行、dropbear、mtd-utils 全部会被覆盖、官方插件复活。

升级后照本文流程重做一遍即可：交叉编译 → bind-mount 写 rootfs → 注释开机重拷行 → 让 pc 重拉。把源码和编好的二进制留一份在 PC 上，重做只要几分钟。

---

*本文记录的是对自有设备的离线研究与定制，仅供学习交流。操作有变砖风险，务必先备份、保留 telnet/串口/网页 shell 等回退手段。*
