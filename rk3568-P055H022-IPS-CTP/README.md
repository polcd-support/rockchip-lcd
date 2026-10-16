# P055H022-IPS-CTP 面板（HX8394F）（KICKPI K1 / K1B · Rockchip RK3568）

5.5 寸 IPS **720×1280** MIPI-DSI 面板（LCM IC **HX8394F**）在 KICKPI RK3568（K1/K1B）Linux SDK 上的设备树面板配置。

| 项 | 值 |
|---|---|
| 适用平台 | KICKPI RK3568（K1 / K1B），kernel **5.10.160**，Linux SDK |
| 面板 | 5.5" IPS，720×1280，LCM IC **HX8394F**，MIPI-DSI 4 data lane |
| 实现方式 | Rockchip DRM 面板：`compatible = "simple-panel-dsi"` + `panel-init-sequence`——**dtsi 即驱动**，不需要 C 驱动文件 |
| 文件 | `dts/rk3568-kickpi-lcd-mipi0-P055H022-IPS-CTP.dtsi`、`patches/0001-add-panel.patch` |
| 版本 | v1.0（2026-10-16） |

## 1. 文件清单

```
P055H022-IPS-CTP/
├── README.md                                   ← 本文件
├── dts/rk3568-kickpi-lcd-mipi0-P055H022-IPS-CTP.dtsi    ← 面板 dtsi（拷贝到 kernel/arch/arm64/boot/dts/rockchip/）
└── patches/0001-add-panel.patch                ← 新增该 dtsi + 两处板级 dtsi 的 include 改动
```

## 2. 设备树参数

| 项 | 值 |
|---|---|
| `dsi,flags` | `MIPI_DSI_MODE_VIDEO \| MIPI_DSI_MODE_VIDEO_BURST \| MIPI_DSI_MODE_LPM \| MIPI_DSI_MODE_EOT_PACKET`（编译后 `dsi,flags = <0xa03>`） |
| `dsi,format` / `dsi,lanes` | `MIPI_DSI_FMT_RGB888` / `4` |
| `display-timings` | `clock-frequency = <62000000>`，`hactive 720` / `vactive 1280`，`hfront-porch 20` / `hsync-len 4` / `hback-porch 40`，`vfront-porch 9` / `vsync-len 2` / `vback-porch 25`（合 `ht = 784`、`vt = 1316`） |
| 背光 | `backlight_dsi0`：`pwm-backlight` on `&pwm4`（25000 ns），`default-brightness-level = <200>` |
| 供电 | `vcc3v3_lcd0_n`：`regulator-fixed`，`gpio0 RK_PC7` 高有效 |
| 复位 | `reset-gpios = <&gpio3 RK_PA4 GPIO_ACTIVE_LOW>`，`reset-delay-ms = 120` |
| 通路 | `&dsi0` → `dsi0_in_vp1` → `&route_dsi0 connect = <&vp1_out_dsi0>`，`&video_phy0` 打开 |
| 触摸 | 开启 GT911：`gt9xx@5d` 置 `status = "okay"`（`gtp_resolution_x/y = 720/1280`，沿用 SDK 里的 `goodix,cfg-group0`），同时把 `myft5x06@38` 置 `status = "disabled"` |

初始化码（`panel-init-sequence`）格式为 Rockchip 的
`<data_type> <delay_ms> <payload_len> <payload…>`：数据写用 `39`（DCS 长写），
`0x11` / `0x29` 用 `05`（DCS 短写、无参数），`0x36 = 0x11` 用 `15`（DCS 短写、1 参数）。
本包的表 = 厂家码逐字节实现：17 条寄存器写 + `0x11`（255 ms）+ `0x36 = 0x11` + `0x29`（255 ms）。

## 3. 集成步骤

1. 拷贝 dtsi：

```sh
cp dts/rk3568-kickpi-lcd-mipi0-P055H022-IPS-CTP.dtsi <SDK>/kernel/arch/arm64/boot/dts/rockchip/
```

2. 在板级 dtsi 里放开本面板（K1：`rk3568-kickpi-k1.dtsi`；K1B：`rk3568-kickpi-k1b.dtsi`）：

```diff
-#include "rk3568-kickpi-lcd-mipi0-<当前生效的面板>.dtsi"
+#include "rk3568-kickpi-lcd-mipi0-P055H022-IPS-CTP.dtsi"
```

> 同一时刻**只能有一个** `kickpi-lcd-*` dtsi 生效：这些 dtsi 都定义 `dsi0_panel` / `panel_in_dsi` /
> `dsi_out_panel` / `backlight_dsi0` 等同名 label，两个同时生效会报重复定义。
>
> 触摸节点同理：`&i2c1` 上的 `myft5x06@38` 与 `gt9xx@5d` 不能同时 `okay`。

3. 或直接打补丁（含新增 dtsi 与板级 dtsi 改动；**以当前工作树为基准**）：

```sh
cd <SDK> && patch -p1 < patches/0001-add-panel.patch
```

## 4. 编译与验证（本次已实测）

只编设备树（实测命令，两个面板的 dtsi 都编过）：

```sh
cd <SDK>/kernel
TC=<SDK>/prebuilts/gcc/linux-x86/aarch64/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu-
make ARCH=arm64 CROSS_COMPILE=$TC -j8 rockchip/rk3568-kickpi-k1-linux.dtb
```

整机 / 批量：

```sh
cd <SDK>
./build.sh                 # 默认 allsave：kernel + u-boot + rootfs + update.img
./build.sh all_multi_dtb   # 按板级 dtsi 里每个 kickpi-lcd-* 各出一份 dtb
```

### 4.1 核对（本次实测结果）

| 检查项 | 结果 |
|---|---|
| dtb 编译 | `DTC arch/arm64/boot/dts/rockchip/rk3568-kickpi-k1-linux.dtb`，无 error/warning |
| 反编译核对 | `./scripts/dtc/dtc -I dtb -O dts …/rk3568-kickpi-k1-linux.dtb` → `panel-init-sequence` 为本包表内容；`clock-frequency = 0x3b20b80`(62000000)、`hactive 0x2d0`(720)、`vactive 0x500`(1280)；porch/同步值与上表逐项一致；`dsi,flags = <0xa03>`、`dsi,lanes = <0x04>` |

**硬件点亮状态：RK3568 侧尚未上板验证**（无 RK3568 整机/屏实测）。
porch 与 `clock-frequency` 取自 A133/T113 上已点亮的同一套参数；`VIDEO_BURST` 与厂家给该面板的 RK 参考配置一致。

## 5. 常见问题

| 现象 | 原因 / 处理 |
|---|---|
| 编译报 label 重复 / `dsi0_panel` 重定义 | 同时生效了两个 `kickpi-lcd-*` dtsi，只留一个 |
| 改了 dtsi 但 `rk3568-kickpi-k1-linux.dtb` 没更新 | Kbuild 用 `.dts.d` 依赖文件；新增 include 后确认 make 重新执行了 `DTC`（换板级 dtsi 后必然重编） |
| 屏不亮但有背光（上板后） | 依次核对：`dsi,flags`（先试带 `VIDEO_BURST`，再试去掉）、porch 与 `clock-frequency`、`reset-gpios` 极性/延时、背光 `pwms` 通道、`route_dsi0 connect` 指向的 vp |
| 触摸无反应 | 确认 `gt9xx`/`ft5x06` 只开一个，且 INT/复位脚与实际接线一致 |

## 6. 许可

dtsi 沿用 Rockchip/KICKPI SDK 的 `SPDX-License-Identifier: (GPL-2.0+ OR MIT)`；初始化码来自面板厂资料，仅供本面板适配使用。
