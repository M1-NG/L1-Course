# NVR4.6 编译脚本与 Shell 脚本学习整理

整理日期：2026-10-14
整理对象：`nvr4.6` 工作区，主型号 TS3116K0（芯片平台 `sstar631g`）
目的：把"改一行代码 → 变成一台机器上能装的固件"这条链路上的脚本逐个对上行号，并顺带整理这些脚本用到的 shell 写法。

> 本文只描述脚本本身。代码架构与启动流程见[项目阅读指南](../项目导读/项目阅读指南.md)与 [TS3116K0 本周任务分析](../型号需求/TS3116K0本周任务分析.md)，崩溃定位见仓库根目录的 [stack_debug.md](../../stack_debug.md)。

---

## 1. 全景：脚本分成四层

| 层 | 目录 / 文件 | 职责 | 谁调用 |
| --- | --- | --- | --- |
| ① 入口层 | `build-<型号>.sh`（6 个） | 复制配置、清编、编译、调打包、打日志 | 人工 / Jenkins |
| ② 代码构建层 | `code/Makefile`、`code/common.mak`、各模块 `build`、`CrossConfigure.sh` | 交叉编译出 `edvr` 与各 `.so` | ① 里的 `make -C code` |
| ③ 打包层 | `makefs-<芯片>.sh`（5 个） | 组装 rootfs 目录、生成分区镜像与升级包 | ① |
| ④ 文件系统层 | `code/newfs/buildfs_<芯片>.sh` | 解基础包、strip、打 `rootfs/app/user/sn` 分区镜像 | ③ |

还有一组**公共工具脚本**被上面各层复用，都是根目录下的小文件：

`changeini.sh`、`readini.sh`、`partition_merge.sh`、`usbupgrade.sh`、`changedevtype.sh`、`model/Changeversion.sh`。

一条完整的调用链（以 TS3116K0 为例）：

```
./build-ts3116k0.sh 4.6.1.9 all upg
  ├─ cp menuconfig/TS3116K0.config → code/.config
  ├─ make -C code clean
  ├─ make -C code                              ← code/Makefile 的 all 目标
  │    └─ common.mak 读 .config → CHIP_TYPE/CROSS_COMPILE/FUNC_CFLAGS
  │    └─ 依次编译 libipc → dvrsdk → syssdk → netsdk → egui → guiapp → up
  └─ ./makefs-631g.sh ts3116k0 4.6.1.9 all upg
       ├─ changeini.sh           改 Version.ini 的 BuildDate
       ├─ 拷 edvr/.so/字库/res 到 rootfs/newfs/
       ├─ newfs/buildfs_631g.sh  ← 打 rootfs/app/user/sn 镜像
       ├─ usbupgrade.sh          ← 生成 USB 升级包 restore-*.bin
       ├─ tar + md5firmware      ← 生成网络升级包 update-*.bin
       ├─ build_lang.sh          ← 生成多语言包
       ├─ partition_merge.sh     ← 生成生产母片（Mstarbin / partition_merge）
       └─ 产物复制到 tmp/TS3116K0/（upg 时另存 tmp/Upgrade/TS3116K0/）
```

---

## 2. 第一层：入口脚本 `build-*.sh`

### 2.1 每个入口脚本只做三件事

看最简单的 [build-ts8232c.sh](../../build-ts8232c.sh)：

```sh
mkdir -p ./tmp
cp ./menuconfig/TS8232C6.config ./code/.config          # 第 5 行：选型号配置
make -C code clean; make -C code 2>&1 | tee build.txt    # 第 7 行：清编 + 编译
./makefs-3536c.sh ts8232c $1 $2 2>&1 | tee -a build.txt  # 第 8 行：打包
if [ "$3" = "upg" ]; then ...                            # 第 9-11 行：可选再打一次升级包
ERRORNUM=`grep "error:" build.txt | wc -l`               # 第 19 行：看日志里有没有 error:
```

关键点：`code/.config` **由脚本复制产生**，是 gitignore 的中间产物。改能力开关要改 `menuconfig/*.config`，不要改 `code/.config`；同理型号的芯片/通道/工具链全在 `menuconfig` 里定义，见 [TS3116K0.config](../../menuconfig/TS3116K0.config) 第 9、14、19、24 行。

### 2.2 两种写法，差别不只是好不好看

| 脚本 | 写法 | 后果 |
| --- | --- | --- |
| `build-ts3116k0.sh` | `run_logged` 封装（第 5-21 行） | 每步单独记录退出码，失败 `exit $?` 立刻停 |
| 其余 5 个 | `make ... ; make ...` 直接串 | 编译失败**不会中断**，脚本会继续跑打包，最后靠 `grep "error:"` 判断成功失败 |

[build-ts3116k0.sh:5-21](../../build-ts3116k0.sh#L5-L21) 的 `run_logged` 值得记住，它解决了一个通用问题——**管道会吃掉退出码**：

```sh
run_logged()
{
    status_file=./tmp/build-command.status
    rm -f "$status_file"
    (
        "$@"                                   # 子 shell 里跑真命令
        printf '%s\n' "$?" > "$status_file"    # 退出码写文件，绕开管道
    ) 2>&1 | tee -a build.txt
    ...
    return "$status"
}
```

因为 `cmd | tee log` 的 `$?` 是 `tee` 的退出码，永远是 0，所以必须用临时文件把真实退出码带出来。这是 shell 里最典型的坑，六个入口脚本里只有一个修了。

另外 `build-ts3116k0.sh:38-44` 的 `grep "error:" | wc -l` 判断也很脆：日志里只要有别处打印的 `error:` 字样（比如编译器 warning 文本、脚本 echo）就会误报失败。它只是"横幅提示"，不是可靠的构建结果判断，**最终以退出码和产物为准**。

### 2.3 参数错位（易错点）

型号名占了打包脚本的第一个参数，所以传参整体右移一位：

| 调用方 | 参数 | 被调用方 `makefs-*.sh` | 参数 |
| --- | --- | --- | --- |
| `./build-ts3116k0.sh 4.6.1.9 all upg` | `$1 $2 $3` | `./makefs-631g.sh ts3116k0 4.6.1.9 all upg` | `$1 $2 $3 $4` |

所以：
- `build` 脚本用 `$3 = upg` 判断打第二遍，`makefs` 里对应的是 `$4 = upg`（[makefs-631g.sh:163](../../makefs-631g.sh#L163)、[makefs-631g.sh:191](../../makefs-631g.sh#L191)）；
- `makefs` 里 `$2` 是版本号参数，`def` 表示用 `Version.ini` 里的版本，否则用传入版本（[makefs-631g.sh:28-32](../../makefs-631g.sh#L28-L32)）。

---

## 3. 第二层：`code/Makefile` + `code/common.mak`

### 3.1 `.config` 是唯一的注入点

[code/Makefile:1-7](../../code/Makefile#L1-L7)：

```make
export BUILD_ROOT := $(shell pwd)
include ./common.mak
.EXPORT_ALL_VARIABLES:
```

`common.mak` 第 1 行 `include $(BUILD_ROOT)/.config`，把 menuconfig 生成的 `CONFIG_*` 全部拉进 make 变量空间，再翻译成 C 世界认识的东西：

| `.config` 里的字段 | `common.mak` 转成 | 给谁用 |
| --- | --- | --- |
| `CONFIG_TOOLCHAIN="arm-sstar631-linux"` | `CROSS_COMPILE`、`CC/AR/LD/STRIP` | 第 9-12、122-128 行 |
| `CONFIG_CHIP_TYPE="sstar631g"` | `CHIP_TYPE` → 决定 `UPLIBPATH`、`SDK_LIB_PATH`、CPU 参数 | 第 14-18、106-120 行 |
| `CONFIG_CHANS="mini16"` | `CHANS` → `-D__631g_mini16__` | 第 20-23、118 行 |
| `CONFIG_MAX_CHANNEL` | `-DMAX_CHANNEL_NUM=` | 第 25-27、130-132 行 |
| 每个功能 `CONFIG_ENABLE_*` | `FUNC_CFLAGS += -D...` | 第 141-386 行成片 |

**新增一个功能开关要动两处**：`menuconfig/*.config`（含 `Kconfig`）加字段 + `common.mak` 加 `ifeq` 映射。只加一边，配置里勾了也不会生效。

### 3.2 `all` 目标的顺序是有依赖的

[code/Makefile:14](../../code/Makefile#L14)：

```make
all: showparam libtps_protocol libipc libsn libzram libsignalhandle libbase dvrsdk dvrmodules lib28181 syssdk netsdk egui guiapp up
```

几个必须记住的因果：

- `guiapp: libipc`（第 80 行）——GUI 主程序隐式依赖 `libipc`，所以 代码助手协作指南.md 里说 `make -C code guiapp` 会自动先编 `libipc`；
- `dvrsdk` 里 `rm -rf ./lib_$(CHIP_TYPE)/*`（第 59 行）——**每次全量编译都会清空业务库输出目录**，所以局部重编某个 DVR_SDK 模块后，别指望其它 `.so` 还在；
- `netsdk` 编译完会把 `Bin/*.so` 拷进 `midware/modules/lib_$(CHIP_TYPE)/`（第 75 行），打包时从 `modules` 目录整体取；
- `syssdk: ./build $(CHIP_TYPE) $(CHANS)`（第 71 行）、`egui: ./CrossConfigure.sh $(CHIP_TYPE) $(CHANS)`（第 78 行）——这两个子系统的构建入口都是**脚本**，不是 Makefile。

### 3.3 局部构建与 clean

`make -C code <目标>`（依赖 `code/.config` 已存在）：

| 目标 | 说明 |
| --- | --- |
| `guiapp` | 只重编 GUI 主程序（隐式带 `libipc`） |
| `egui` | 只重编 EGUI 框架（内部会先 `CrossConfigure.sh` 再 `make clean && make`） |
| `dvrsdk` / `syssdk` / `netsdk` / `lib28181` / `up` | 各子系统 |
| `clean` | 第 131 行聚合了各 `*_clean`，但**漏了 `egui_clean`**——`egui_clean` 目标本身在第 108-109 行有定义，只是没被 `clean` 依赖，所以 `make -C code clean` 不会清 EGUI 的产物 |

`lib28181`、`libzram` 受 `ifeq ($(CONFIG_...), y)` 包住（第 51、65-68 行），关掉开关时目标存在但**什么都不做**，不会报错——排查"为什么没编出 lib28181"时先看 `.config`。

### 3.4 业务库是怎么编出来的

`code/midware/DVR_SDK/build` 只有 22 行，核心是：

```sh
dir=`pwd | awk -F\/ '{print $NF}'`    # 第 5 行：目录名即模块名
TARGET=lib$dir.so                     # 第 6 行：MPU → libMPU.so
make clean; make $1                   # 第 18-19 行
```

所以 DVR_SDK 下每个子目录名就是 `.so` 名，产物统一落在 `midware/DVR_SDK/lib_$(CHIP_TYPE)/`。`$1` 透传（`build clean` 就是只清理）。

---

## 4. 第三层：打包脚本 `makefs-*.sh`

以 TS3116K0 用的 [makefs-631g.sh](../../makefs-631g.sh) 为主线（`makefs-931g.sh` 结构一致，多了一个 ts8232c10 分支）。整个脚本可以切成 9 段：

| 段 | 行号 | 做什么 |
| --- | --- | --- |
| 1 版本读取 | 4-12 | `GetVersion()` 用 awk 从 `Version.ini` 抠出三段号，拼成 `4.x.y.z` |
| 2 型号判定 | 14-22 | 决定 `postfix`、`versionfile`、`modelfile` |
| 3 参数处理 | 24-47 | `$2=def` 用 ini 里的版本；`$4=upg` 时时间戳 **+5 分钟** |
| 4 准备目录 | 49-73 | 清 `rootfs/`，拷 `code/newfs`、`tools/update`、`code/res/res` |
| 5 注入版本 | 75-82 | `changeini.sh` 改 BuildDate，拷 `DeviceVersion.ini`、`DvrModel.cfg` |
| 6 拷产物 | 84-126 | `edvr`、各 `.so`、字库、升级程序、`png2bmp`… |
| 7 打镜像 | 128-131 | 调 `newfs/buildfs_631g.sh` |
| 8 升级包 | 133-160 | `upgrade` 包、USB 包、md5、语言包 |
| 9 母片与输出 | 162-197 | `partition_merge.sh` 产线母片，最后整体复制到 `../tmp/<型号大写>/` |

### 4.1 为什么要打两遍（`upg`）

`build-xxx.sh <版本> <参数2> upg` 会调 makefs 两次，第二次多带 `upg`：

- 打包脚本里 `$4 = upg` 时，时间戳取 `date -d "+5 minute"`（[makefs-631g.sh:35-39](../../makefs-631g.sh#L35-L39)）——让"升级包"的构建时间比"整机固件"**晚 5 分钟**，设备端版本比较时升级包天然更新；
- `$4 = upg` 时**跳过** `partition_merge.sh`（第 163 行）——升级不用产线母片；
- 输出目录从 `tmp/<型号>/` 换成 `tmp/Upgrade/<型号>/`（第 191-195 行的 `copy_release_to_package_dir`）。

`copy_release_to_package_dir()` 里有一句注释解释了它存在的理由：Jenkins 只拷 `tmp` 下的目录，而 `edvr.bin`（调试符号包）必须留在型号目录里，不能进固件分区。

### 4.2 `sstar631g` 与 `sstar931g` 打包的差别

- 631g：版本注入前 `export MODEL_PACKAGE_BUILD_TIME=$nowtime`（[makefs-631g.sh:130](../../makefs-631g.sh#L130)），931g 没有；
- 631g 多拷一个 `code/newfs/app/edvr/model_group_registry.ini`（第 97 行）；
- 931g 在 `ts8232c10` 型号上会按内存大小换 `libpkgstream`（[makefs-931g.sh:117-121](../../makefs-931g.sh#L117-L121)：1536k vs 2048k），631g 固定 2048k。

---

## 5. 第四层：`code/newfs/buildfs_<芯片>.sh`

以 [buildfs_931g.sh](../../code/newfs/buildfs_931g.sh) 为例（631g 版同构，可在 `code/newfs/` 下对照）。这个脚本负责"目录树 → 分区镜像"，顺序很讲究：

1. **解基础 rootfs 包**：`tar zxvf rootfs_glibc_<type>_base.tgz`（第 24 行）。基础包是预置的，`rootfs_glibc_931g_base.tgz` 这种；
2. **先存符号，再 strip**（第 28-77 行）。`edvr.bin` 调试包的核心逻辑就在这里：
   - `copy_debug_shared_libs` 在 strip **之前**把 `lib/`、`usr/lib/` 下的 `.so` 完整拷到 `../release/debug_symbols/`；
   - 然后才 `find ... | xargs ${STRIP}`（第 76-77 行）；
   - app 侧的 `edvr` 同理（第 127-141 行），并且会带上 `edvr.map`（第 144-150 行）；
   - 生成 `manifest.txt`（sha256 清单，第 157-169 行），再 `tar zcf ../release/edvr.bin -C ../release debug_symbols`（第 172 行），最后删掉中间目录。
   
   **这条顺序是它能工作的唯一前提**：注释里写得很明白——"源文件若已被上游 strip，无法恢复调试符号"。所以出现"符号对不上"时要先确认 `edvr.bin` 是不是和固件同一次构建出来的；
3. **组装 rootfs 内容**：拷 `fs1_<type>/*`、网页、passwd/group/profile、`EGUI.cfg`，并把应用层只读资源挪到 `app/edvr/`（第 81-97 行）；
4. **打三个分区镜像**，全部走 `mkubiimg_new.sh`：
   - `rootfs.<postfix>`：squashfs，24M（第 114 行）
   - `app.<postfix>`：squashfs，36M（第 194 行）
   - `user.<postfix>` / `user_factory.<postfix>`：ubifs，82M（第 218、223 行）
   - `sn.<postfix>`：ubifs，12M（第 228 行）
   
   `user_factory` 与 `user` 的唯一区别是打了个 `user/NVR_PRODUCTION_AGING.flag` 文件（第 222 行）——产测标记就是这么来的；
5. **app 分区最后才 strip**（第 179 行），因为要先把未 strip 的 `edvr` 留下来做调试包。

`imgcfg/` 目录是 rootfs 上 `/etc` 那一层的内容源（`init.d/rootapp`、`passwd`、`TZ` 等）。设备上 `edvr` 由 [imgcfg/init.d/rootapp:80](../../code/newfs/imgcfg/init.d/rootapp#L80) 的 `./edvr &` 拉起。

---

## 6. 公共工具脚本

### 6.1 `changeini.sh` / `readini.sh`——ini 的写与读

[changeini.sh](../../changeini.sh) 全文就一行，是一行 sed 的经典用法：

```sh
sed -i "/$2/,/^\[/s/$3=.*$/$3=$4/" $1
```

从 `[$2]` 所在行开始、到下一个 `^[` 之前结束的区间内，把 `$3=...` 换成 `$3=$4`。**只在指定 section 内替换**，这就是它比全局 `sed s///` 强的地方。

[readini.sh](../../readini.sh) 用 awk 读：

```sh
awk -F '=' '/\['$SECTION'\]/{a=1}a==1&&$1~/'$ITEM'/{print $2;exit}' $INIFILE
```

`a=1` 是"进入目标 section"的开关，命中 item 就 print 并 `exit`。注意它 `-F '='` 后取 `$2`，所以**值里不能有 `=`**，且不会去空格；`usbupgrade.sh:43` 就是用它取 `UBOOT_VERSION` 的。

`makefs-*.sh` 的 `GetVersion()` 用的是同一个套路（[makefs-631g.sh:7-9](../../makefs-631g.sh#L7-L9)），三行分别抠 Major/Minor/Reversion。

### 6.2 `model/Changeversion.sh`——批量改版本

[Changeversion.sh](../../model/Changeversion.sh) 用 `IFS="."` 把 `4.6.1.9` 拆成数组（第 12-18 行），然后遍历 `model/*.ini` 和 `model/version*` 调 `changeini.sh`。有第二个参数时还会写 `*_base` 三件套（第 35-40 行）——这是**降级基线**：设备端比版本时会拿 `base` 版本做下限，不要随便设。注释里明确写了"主版本号注意范围 `基础版本号 <= 主版本号`，设置错误会造成版本检测认为版本信息错误"。

### 6.3 `partition_merge.sh`——产线母片

- 前半段（第 5-127 行）是**型号 → 分区表**的硬编码映射：每个型号一组 `flash_size` 和各分区大小，按分区顺序排列；
- 后半段（第 132-229 行）先把 `update/parts/*`、`update/tmp/*`、`release/*` 收进 `firmware/`，再按型号调 `../partition_merge`（Hisi 系）或 `tools/mstarbin/Mstarbin`（sstar 系，第 221-228 行）拼母片；
- `partition_merge` 靠**参数个数**决定布局：脚本注释（第 165-166 行）说"涉及到的分区都在输入参数中，不写的分区在程序里做跳过"。所以改分区表时，`partition_merge.sh` 里的参数列表和 `firmware/` 里实际存在的文件**必须同时对得上**；
- `ts3116k0` 在有的分支里被特殊处理（第 146 行跳过 config 分区、第 215 行走 Mstarbin 分支）。

### 6.4 `usbupgrade.sh`——U 盘升级包

把 `update/parts`、`update/tmp`、`release/config*` 凑成 `usbupgrade/` 目录，再用 `tools/usb_upgrade` 工具生成 U 盘升级包。整段是一个大 `if/elif` 按型号给工具传不同参数（第 72-104 行），参数含义：

`-T` 型号 ID（如 ts8232c 是 1023、ts3116k0 是 2047）、`-M` 型号名、`-U/-K/-F/-A/-C/-R/-L/-MODEL` 分别对应 uboot/kernel/rootfs/app/config/user/logoBoot/model 分区文件、`-VER` 版本号、`-NTIME` 构建时间。

型号名与参数不匹配（比如给某个型号传了它没有的分区）是这里最常见的出错方式。

### 6.5 `changedevtype.sh`——跨型号升级包

把一个型号的升级包改造成另一个型号的（例：给 TS8232C 的包改成 TS8225C 能装的包）。做四件事：

1. `tmp/version` 里把 `TYPE_NVR_3536C_K` 换成 `TYPE_NVR_3536C_J`（第 52 行，映射表在第 10-23 行 `GetDevInCodeType`）；
2. 往 `etc/init.d/rootapp` 里插一段"检测到旧型号标记文件就切换"的代码（第 55 行）；
3. `tmp/` 下所有 `*.源型号` 后缀改名成 `*.目标型号`（第 58-64 行）；
4. 重新打包 + `md5firmware` + `md5file`。

它依赖 `tmp/<源型号>/` 下已经存在源型号的升级包，是"重新包装"，不重新编译。

---

## 7. 这些 shell 脚本里的通用写法小结

从 C 视角看，这批脚本反复用到下面这些，值得单独记一笔：

| 写法 | 出现位置 | 说明 |
| --- | --- | --- |
| `awk -F '=' '/\[sec\]/{a=1}a==1&&$1~/"k"/{print $2;exit}'` | `readini.sh`、`GetVersion()` | 读 ini 的标准姿势：`a` 是 section 开关，`exit` 防止重复匹配 |
| `sed -i "/$2/,/^\[/s/$3=.*$/$3=$4/"` | `changeini.sh` | 限定 section 的替换 |
| `pushd / popd` | 各打包脚本、`DVR_SDK/build` | 切目录后必须切回来，脚本里几十处 |
| `cmd | tee -a build.txt` | 所有 `build-*.sh` | 同时上屏和落文件；**代价是 `$?` 变成 `tee` 的** |
| `set -e` | `DVR_SDK/build`、`SysSDK/build` | 出错即停（注意 `run_logged` 那两个脚本没用它，而是手动传退出码） |
| `\`cmd\`` 命令替换 | 通篇 | `nowtime=\`date +%Y%m%d%H%M\``、`` `pwd` `` |
| `tr a-z A-Z` | 各脚本 | 型号名转大写做目录名：`ts3116k0` → `TS3116K0` |
| `find . -name "*.sh" | xargs chmod +x` | `makefs-*` 第 72 行 | 打包前统一加可执行位 |
| `[ "$1" = "x" ]` | 通篇 | POSIX sh 用 `=`（不是 `==`），且变量必须加引号，否则空值会变成语法错误 |
| `if [ $# -gt 1 ]` | `partition_merge.sh:31` | 位置参数个数判断，用于给可选参数设默认值 |

几个真实存在的坑，读脚本时要留意：

1. **管道吞退出码**——只有 `build-ts3116k0.sh` 处理了；
2. **`grep "error:"` 判成败**——存在误判，只看横幅不可靠；
3. **参数错位**——`build-*.sh` 的 `$3` 到 `makefs-*.sh` 变成 `$4`；
4. **旧脚本注释是 GBK/ANSI**——`makefs-*.sh` 里的中文注释在 UTF-8 终端是乱码，不要顺手转码（会产生整文件 diff）。`build-ts3116k0.sh`、`buildfs_931g.sh` 是新写的 UTF-8，正常显示；
5. **`make -C code clean` 会清空 `DVR_SDK/lib_$(CHIP_TYPE)/`**，局部调试时注意。

---

## 8. 常用命令速查

```sh
# 全量构建（在仓库根目录，必须在 Linux/WSL）
./build-ts3116k0.sh 4.6.1.9 all upg     # $1=版本 $2=参数2 $3=upg 时额外打升级包
                                        # 日志：根目录 build.txt；产物：tmp/TS3116K0/、tmp/Upgrade/TS3116K0/

# 局部重编（要求 code/.config 已存在）
make -C code guiapp     # GUI 主程序（隐式带 libipc）
make -C code egui       # EGUI 框架
make -C code dvrsdk     # DVR_SDK 业务库
make -C code netsdk     # 网络库
make -C code syssdk     # 芯片适配层
make -C code up         # 升级程序

# 只跑打包（编译已完成时）
./makefs-631g.sh ts3116k0 def all upg

# 版本号批量修改
./model/Changeversion.sh 4.6.2.0

# 跨型号升级包
./changedevtype.sh ts8232c ts8225c

# 看某型号当前的配置
grep -E "CONFIG_CHIP_TYPE|CONFIG_CHANS|CONFIG_TOOLCHAIN" menuconfig/TS3116K0.config
```

---

## 9. 还没搞清 / 明天要接着看的

脚本这一层已经能画完整条链。剩下要落到代码的是**升级入口与版本校验**，也就是"授权后禁止降级到不支持型号变更的固件"这个需求：

| 要读的文件 | 关注点 |
| --- | --- |
| [CBaseUpdate.cpp:816](../../code/midware/DVR_SDK/SYS/CBaseUpdate.cpp#L816) `check_version()` | 版本比较的判定分支（第 960-1030 行附近的 `uiOldSoftVer`/`uiNewSoftVer`/`uiBaseSofVer` 三值比较），`base` 版本从哪来 |
| [CBaseUpdate.cpp:589,719,760](../../code/midware/DVR_SDK/SYS/CBaseUpdate.cpp#L589) | `version` 文件解包、`g_boardtype` / `BoardFormat` 型号判定 |
| `code/midware/netsdk/TPS/src/uc2/uc2syscontrol.cpp` | 升级请求入口、与授权状态的互斥点 |
| `code/midware/DVR_SDK/MPU/MPU_PUB_API.cpp` | 型号读取、授权读取、能力缓存 |
| `model/TS3116K0Version.ini` 的 `*_base` 字段 | 降级基线的数据来源，对应 `Changeversion.sh` 的第二个参数 |
| `cmake`/`buildfs` 的构建边界 | 支持型号变更（码 33）与不支持（码 27）在打包侧是否有区分 |

要回答的问题：一条升级包从被设备读到被拒绝，中间经过哪几次版本/型号比较？授权标记存在哪里、在哪个环节可读？改动落在 `check_version` 里对新旧固件都成立，还是要配合 `version` 文件的新字段。
