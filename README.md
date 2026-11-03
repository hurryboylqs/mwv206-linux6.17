# mwv206 2.1.13 — 景嘉微 JM7200 显卡驱动（官方版，移植到 Linux 6.17）

## 版本标识

| 项 | 值 |
|---|---|
| 驱动版本 | **2.1.13-0kylin1** |
| 包名 | `mwv206-dkms_2.1.13-0kylin1_all.deb` |
| 来源 | 景嘉微官网下载（KOS desktop release arm64） |
| 上游形态 | DKMS 源码包（`/usr/src/mwv206-2.1.13-0kylin1/`） |
| 同目录的另一个版本 | `../mwv206-1.1.2/`（2020 社区版，保留作备份） |

> 官网同时提供的 `mwv206-dev_2.2.13-0kylin1_arm64.deb` 是**用户态**部分
> （Xorg 驱动 `mwv206_drv.so`、GL 库、`libmwv206api.so`），小系统用 Qt+linuxfb，
> **不需要**，只用本目录的内核驱动源码。

## 为什么要它

机器（如 LEGEND KaiTianM740J-D183 / 飞腾 D2000 + JM7200）的 UEFI 只提供
**影子帧缓冲**（framebuffer 在系统内存 0x78000000，不在显卡 BAR 内）。
grub 阶段靠 UEFI 显示服务刷新，画面正常；`ExitBootServices` 之后没人刷，
内核 `efifb` 写进去**屏幕上没有任何变化** —— 表现为“卡在 Start loading”。

只有真正的显卡驱动（本模块）把显存驱动起来，画面才会出。

## 相对官方 2.1.13 源码的移植改动

### 构建
| 文件 | 改动 | 原因 |
|---|---|---|
| `Makefile` | 注释掉 `CC=gcc` | 否则交叉编译时用宿主机的 x86 gcc（`-mabi=lp64` 报错） |
| `Makefile` | 新增 `ccflags-y += -I$(src)...` | 新内核不把 `EXTRA_CFLAGS` 传给编译器，头文件找不到 |
| `Makefile` | `-Werror` → `-Wno-error` | 新内核 missing-prototypes 等检查更严 |

### 6.x API 变更
| 文件 | 改动 |
|---|---|
| `mwv206hal/mwv206ioctl_display.c` | `hrtimer_init()` + `.function=` → `hrtimer_setup()` |
| `mwv206linuxdev/mwv206_edid.c` | `I2C_CLASS_DDC`（已移除）→ `I2C_CLASS_HWMON` |
| `mwv206linuxdev/mwv206_edid.c` | `drm_edid_block_valid()` → `drm_edid_is_valid()` |
| `mwv206linuxdev/mwv206_fb.c` | `strlcpy` → `strscpy` |
| `mwv206linuxdev/mwv206_pci.c` | `class_create(THIS_MODULE, name)` → `class_create(name)` |
| `mwv206linuxdev/mwv206_proc.c` | `PDE_DATA()` → `pde_data()` |
| 全项目 | `del_timer_sync` → `timer_delete_sync` |

### 已移除的内核接口
| 文件 | 改动 |
|---|---|
| `mwv206linuxdev/mwv206_fb.c` | `mwv206fb_init_early()` 去掉 `alloc_apertures()` / `remove_conflicting_framebuffers()`，改空实现 |

### 关键新增
| 文件 | 改动 |
|---|---|
| `mwv206linuxdev/mwv206_fb.c` | **新增 `mwv206fb_mmap()`** 并挂到 `fb_ops` |

原因：6.x 的 `fb_mmap()` 不再有通用实现，`fb_ops->fb_mmap == NULL` 时直接返回
`-ENODEV`；Qt 的 linuxfb 必须 mmap 显存。而且 PCI 显存**必须用
`pgprot_writecombine` 映射**，按普通 cached 内存映射在 arm64 上会触发 SError panic
（实测：不修它，Qt 一起来系统就挂）。

### DDR 频率表补充
`mwv206hal/mwv206ddr_chip.c`：本机显存频率 **430MHz**，官方表和 1.1.2 一样只收录
266/400/465/533，直接 `unrecognized ddr freq 430!` → `ddr init failure`。
5 处时序 switch 都加了 `case 430:`，**落到 465 的时序**（430 < 465，更宽松安全）。

## 构建

```sh
# 本仓库根目录；默认编 2.1.13
sh scripts/build-mwv206.sh

# 或指定版本（1.1.2 / 2.1.13）
sh scripts/build-mwv206.sh 1.1.2
```

产物：`out/mwv206.ko`，同时同步到 `rootfs/root/lib/` 和
`rootfs/usr/lib/modules/*/extra/`。

⚠ **strip 注意**：内核模块只能用 `strip --strip-debug`（或 `arm64-strip --strip-debug`）。
`strip --strip-all` 会删掉 `.symtab`，加载时报 `module has no symbols (stripped?)`。

## 运行时

`rootfs/etc/init.d/rcS`：

```sh
insmod .../backlight.ko      # 依赖 CONFIG_BACKLIGHT_CLASS_DEVICE
insmod /root/lib/mwv206.ko
```

之后多出一个名字为 `MWV206` 的 fb，rcS 会优先选它：

```sh
for f in /sys/class/graphics/fb[0-9]*; do
    [ "$(cat $f/name)" = "MWV206" ] && FBDEV=/dev/$(basename $f)
done
```

## 与 1.1.2 的差异

- 官方维护（3 年 bug 修复）
- `proc_ops` 已由官方适配（少改一处）
- 新增 `mwv206_shadow_sync` / `scanctrl`（软件影子同步，2.x 特性）
- 本模块 strip --strip-debug 后约 451K

## 已知遗留

- `mwv206_hdmiaudio.c` 在 2.1.13 里是独立编译单元；本内核 `CONFIG_SOUND=n`，
  若编译报错需从 `mwv206-objs` 排除并补 stub（1.1.2 目录里有先例）
- 2D/3D 加速、视频解码未系统验证（CMA 池偏小，`cma=128M` 可改善）
