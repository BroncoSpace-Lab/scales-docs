# How to Create and Maintain a Custom BSP for the IMX8X
By Luca Lanzillotta, John Williams and John Pollak

# Understanding the BSP

When working with Yocto, Ubuntu, an i.MX8 BSP, and a custom carrier board, the main goal is to make the software accurately reflect the real hardware. The SoM may already be supported by the vendor BSP, but once the carrier board changes, Linux also has to be updated so it understands the new wiring, peripherals, GPIO usage, buses, and power behavior. In practice, this means you are constantly connecting three things: the schematic, the device tree, and the Yocto build system.

Ubuntu is the host machine where all of this work happens. It is not the embedded OS on the target board, but the environment where you install dependencies, run BitBake, edit device tree files, and manage layers and recipes. For example, when building an image for an i.MX8 board, Ubuntu provides the tools needed to compile U-Boot, the Linux kernel, the DTBs, and the root filesystem.

The BSP is the starting point that gives Yocto awareness of the processor family and supported board platform. It usually includes the bootloader setup, kernel support, machine configuration, device trees, and vendor layers needed to get a reference system booting. The BSP gets you to “Linux can run on this hardware family,” while your custom work gets you to “Linux can run correctly on our board.”

Yocto is a build framework that uses metadata to describe how the embedded Linux system should be assembled. Instead of manually copying files into a build tree, you define recipes, appends, patches, and configuration so the system can be rebuilt the same way every time.

Layers keep responsibilities separated and maintainable. Vendor layers usually contain BSP support, community layers provide common packages, and your custom layer holds product-specific changes like patches, custom recipes, DTS modifications, and configuration updates.

Device trees are critical: the kernel needs them to know which peripherals exist, which pins they use, which buses are enabled, and how devices are connected. The difference between `.dts` and `.dtsi` is organizational: `.dtsi` files contain shared SoC/SoM definitions, while `.dts` files describe a specific board variant.

Pin multiplexing is important on i.MX8 devices. Enabling a controller in the device tree is not enough; the associated pads must be configured in the `pinctrl` section. Recipes and `.bbappend` files are how you make changes part of the Yocto build in a clean, repeatable way.

A good workflow:
- Start from the hardware change.
- Determine affected software layer(s).
- Test quickly (e.g., in a devshell).
- Formalize changes in your meta-layer so they are reproducible.

Reproducibility is the goal: another engineer should be able to clone the layers, select the machine, run the build, and generate the same bootable image. The step-by-step roadmap below follows exactly this workflow, applied to the i.MX8X.

---

# Setting up your Development Operating System

Recommended host: Ubuntu 22.04 (Desktop or Server).
If you are natively running Ubuntu 22.04, you can skip to the next chapter.

- Download: https://releases.ubuntu.com/jammy/
- VM options: VirtualBox (Windows), UTM (Mac). When using UTM for Yocto builds, emulate x86_64 (not ARM).

<details>
<summary>Optional: SSH into your VM from the host</summary>

If you're running Ubuntu in a VM, setting up SSH saves you from working inside the VM's own window every time.

1. Get the VM's IP address:
```bash
ip a
```

2. (Optional) Configure a static IP (Netplan on Ubuntu) so the address doesn't change between reboots.

3. Add an entry to your host machine's SSH config (Mac example):
```bash
nano ~/.ssh/config
```
```text
Host yocto-vm
    HostName 192.168.x.x
    User your-username
    Port 22
```

4. Set correct permissions on the config file:
```bash
chmod 600 ~/.ssh/config
```

5. Connect:
```bash
ssh yocto-vm
```

</details>

---

# Using the Yocto Environment to Build Your BSP

This guide covers customizing Yocto Linux v5.0 (scarthgap) on Ubuntu 22.04 to build a carrier-board-specific version of the PHYTEC i.MX8X BSP.

Resources:
- Documentation: https://scales-docs.readthedocs.io/en/latest/imx_yocto_bsp/

!!! note
    Before customizing anything, build and boot the unmodified PHYTEC developer-kit image by following [Building the BSP](imx_yocto_bsp.md#building-the-bsp) in the base Yocto BSP guide. Everything below assumes you already have a working build environment at `~/BSP-Yocto-NXP-i.MX8X-PD24.1.y/yocto` and that `bitbake phytec-headless-image` succeeds against the unmodified reference machine.

## Creating the Custom BSP

Customizing the BSP for a new carrier board comes down to five steps:

1. **Create your custom meta-layer** — a place to hold your machine configuration, patches, and recipe changes so they survive a clean rebuild.
2. **Customize the kernel device tree** — describe the carrier board's GPIOs, pin muxing, and peripherals, then patch the kernel.
3. **Customize the bootloader** — point U-Boot at your new device tree and any board-specific settings.
4. **Wire your patches into the layer** — attach the kernel and U-Boot patches to your layer via `.bbappend` files.
5. **Build with your custom machine** — point the build at your machine instead of PHYTEC's reference machine.

Each step below assumes you're back in the container and build environment:
```bash
podman run --rm=true -v /home:/home -e USER=$USER --userns=keep-id --workdir=$PWD -it docker.io/phybuilder/yocto-ubuntu-22.04 bash
```
```bash
source sources/poky/oe-init-build-env
```

### Step 1: Create your custom meta-layer

Create and register your layer — replace `<your-layer-name>` with something descriptive, e.g. `meta-scales-mariner`:
```bash
bitbake-layers create-layer ~/BSP-Yocto-NXP-i.MX8X-PD24.1.y/yocto/sources/<your-layer-name>
bitbake-layers add-layer ~/BSP-Yocto-NXP-i.MX8X-PD24.1.y/yocto/sources/<your-layer-name>
```

Move into the layer and lay out the directories and `.bbappend` files you'll need:
```bash
cd ~/BSP-Yocto-NXP-i.MX8X-PD24.1.y/yocto/sources/<your-layer-name>
mkdir -p recipes-kernel/linux/files conf/machine recipes-bsp/u-boot/files
touch recipes-kernel/linux/linux-phytec-imx_%.bbappend conf/layer.conf recipes-bsp/u-boot/u-boot-phytec-imx_%.bbappend
```

Base your machine configuration on PHYTEC's reference machine, then rename it to your own machine name:
```bash
cp ~/BSP-Yocto-NXP-i.MX8X-PD24.1.y/yocto/sources/meta-phytec/conf/machine/phycore-imx8x-1.conf conf/machine/<your-machine-name>.conf
```

Set your layer's priority in `conf/layer.conf` — 999 keeps it high enough to take precedence over the vendor layers:
```bash
BBFILE_PRIORITY_<meta-layer-name> = "999"
```

Now open the machine `.conf` you copied and adjust it for your board:

<details>
<summary><code>conf/machine/&lt;your-machine-name&gt;.conf</code></summary>

```
#@TYPE: Machine
#@NAME: phycore-imx8x-1
#@DESCRIPTION: PHYTEC phyCORE i.MX8X
#@ARTICLENUMBERS: PCM-942.A2, PCM-065-QP28NESI2.A0

BOARD_TYPE := "phycore-imx8x-1"

MACHINEOVERRIDES =. "mx8qxp:mx8qxpc0:"

IMX_DEFAULT_BSP = "nxp"

require conf/machine/include/imx-base.inc
include conf/machine/include/phyimx8x.inc

MACHINE_FEATURES += " emmc pci can wifi bluetooth"

KERNEL_DEVICETREE = " \
        freescale/<YOUR DEVICE TREE BLOB NAME HERE>.dtb \
"

UBOOT_MAKE_TARGET = "u-boot.bin"
UBOOT_SUFFIX = "bin"
UBOOT_CONFIG ??= "sd"
UBOOT_CONFIG[sd] = "phycore-imx8x_defconfig,sdcard"
UBOOT_CONFIG[fspi] = "phycore-imx8x_defconfig"

UBOOT_ENTRYPOINT = "0x96000000"
UBOOT_DTB_LOADADDRESS = "0x83100000"
UBOOT_DTBO_LOADADDRESS = "0x83200000"
UBOOT_RD_LOADADDRESS = "0xA0000000"

LOADADDR = ""

# Set Serial console
SERIAL_CONSOLES = "115200;ttyAMA0"

USE_VT = "0"

IMAGE_BOOTLOADER = "imx-boot"

# Set imx-mkimage boot target
IMXBOOT_TARGETS = "${@bb.utils.contains('UBOOT_CONFIG', 'fspi', 'flash_flexspi', 'flash', d)}"

IMX_DEFAULT_BOOTLOADER = "u-boot-phytec-imx"

# Set u-boot DTB
UBOOT_DTB_NAME = "imx8qxp-phycore-kit.dtb"

# Set ATF platform name and load address
ATF_PLATFORM = "imx8qx"

IMX_BOOT_SOC_TARGET = "iMX8QX"
```

</details>

Leave `KERNEL_DEVICETREE` pointing at a placeholder for now:
```
KERNEL_DEVICETREE = " \
        freescale/<YOUR DEVICE TREE BLOB NAME HERE>.dtb \
"
```
You'll fill in the real name once you've created your custom `.dts` in the next step.

### Step 2: Customize the kernel device tree

Open a devshell inside the kernel source, using the same build environment:
```bash
bitbake virtual/kernel -c devshell
```

#### Find and copy the base device tree

The i.MX8X device trees live in:
```bash
cd arch/arm64/boot/dts/freescale
```
For the phyCORE i.MX8X, the PHYTEC reference board is `imx8qxp-phytec-pcm-942.dts`. On a different SoM, search for it instead:
```bash
find . -type f -name "*imx8qxp*"
```
Copy it as the starting point for your custom board:
```bash
cp imx8qxp-phytec-pcm-942.dts <your-custom-dts-name>.dts
```
Then edit it:
```bash
vim <your-custom-dts-name>.dts
```
You'll also want `imx8qxp-phycore-som-emmc.dtsi`, which holds SoM-level definitions shared across boards — some of your changes may belong there instead of in your board's `.dts`.

#### Edit the pin muxing

Enabling a peripheral in the device tree isn't enough on its own — its pads also have to be configured in a `pinctrl` group. Keep two things separate:

- The **node** for a peripheral (a GPIO bank, an I2C bus, etc.) references its `pinctrl` group by phandle, and — for GPIO banks — lists friendly names for each line in `gpio-line-names`.
- The **`pinctrl` group** itself holds the actual electrical configuration for each pad: which function it's muxed to, pull-up/down, drive strength, as an `fsl,pins` list of hex values.

For example, naming a few GPIO1 lines on the node:
```
&lsio_gpio1 {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_gpio1>;

	gpio-line-names =
		/* 0-8: unused */
		"", "", "", "", "", "", "", "", "",
		/* 9-10: named */
		"mariner_gpio09", "mariner_gpio10",
		/* 11: unused */
		"",
		/* 12-14: named */
		"mariner_gpio12", "mariner_gpio13", "mariner_gpio14",
		/* 15-31: remaining pins */
		"", "", "mariner_gpio17", "mariner_gpio18", "mariner_gpio19",
		"mariner_gpio20", "", "", "", "", "mariner_gpio25", "mariner_gpio26",
		"", "", "mariner_gpio29", "mariner_gpio30", "";
};
```
and configuring those same pins' pad settings in their `pinctrl` group:
```
pinctrl_gpio1: gpio1grp {
	fsl,pins = <
		IMX8QXP_ADC_IN1_LSIO_GPIO1_IO09             0x06000041 /* internal pull-down, active high */
		IMX8QXP_ADC_IN0_LSIO_GPIO1_IO10             0x06000041
		IMX8QXP_FLEXCAN1_RX_LSIO_GPIO1_IO17         0x06000041
		IMX8QXP_FLEXCAN1_TX_LSIO_GPIO1_IO18         0x06000041
	>;
};
```
The hex value comes from the i.MX8 Processor Reference Manual's pad control register description (pg. 680 in PHYTEC's copy) — reuse whatever already works on the development board unless you have a specific reason to change it.

The same node/group split applies to any other peripheral, e.g. I2C:
```
&i2c0 {
	#address-cells = <1>;
	#size-cells = <0>;
	clock-frequency = <100000>;
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_lpi2c0>;
	status = "okay";
};

pinctrl_lpi2c0: lpi0cgrp {
	fsl,pins = <
		IMX8QXP_MIPI_CSI0_GPIO0_00_ADMA_I2C0_SCL 0x06000020
		IMX8QXP_MIPI_CSI0_GPIO0_01_ADMA_I2C0_SDA 0x06000020
	>;
};
```

Work out your pin assignments on PHYTEC's [pinmux tool](https://pinmux.phytec.com/), then check the result against `imx8qxp-phycore-som-emmc.dtsi` for conflicts — if a pin is already claimed there for something your board doesn't use, remove it from the `.dtsi` rather than leaving two conflicting definitions.

For reference, here is the full state of the files used to build the SCALES carrier board, and the unmodified vendor file they started from:

<details>
<summary>Current custom device tree: <code>imx8xqxp-scales-mariner.dts</code></summary>

```
// SPDX-License-Identifier: GPL-2.0+
/*
 * Copyright (C) 2018 PHYTEC Messtechnik GmbH
 *
 * Copyright (C) 2019-2021 PHYTEC America, LLC
 */

/dts-v1/;

#include "imx8qxp-phycore-som-emmc.dtsi"
#include <dt-bindings/leds/common.h>

/ {
	model = "PHYTEC i.MX8QX PCM-942";
	compatible = "phytec,imx8qxp-pcm-942", "phytec,imx8qxp-phycore-som", "fsl,imx8qxp";

	chosen {
		bootargs = "console=ttyLP0,115200 earlycon=lpuart32,0x5a060000,115200";
		stdout-path = &lpuart0;
	};

	
	reg_1p8v: regulator-1p8v {
		compatible = "regulator-fixed";
		regulator-name = "1P8V";
		regulator-min-microvolt = <1800000>;
		regulator-max-microvolt = <1800000>;
		regulator-always-on;
	};

	reg_3p3v: regulator-3p3v {
		compatible = "regulator-fixed";
		regulator-name = "3P3V";
		regulator-min-microvolt = <3300000>;
		regulator-max-microvolt = <3300000>;
		regulator-always-on;
	};

	reg_pcieb: regulator-pcie {
		compatible = "regulator-fixed";
		regulator-name = "mpcie_3v3";
		regulator-min-microvolt = <3300000>;
		regulator-max-microvolt = <3300000>;
		regulator-always-on;
	};
};

&i2c1 {
	#address-cells = <1>;
	#size-cells = <0>;
	clock-frequency = <100000>;
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_lpi2c1>;
	status = "okay";

	tps6598x: tps6598x@3f {
		compatible = "ti,tps6598x";
		reg = <0x3f>;
	};

	i2c_rtc: rtc@52 {
		compatible = "microcrystal,rv3028";
		reg = <0x52>;
		pinctrl-names = "default";
		pinctrl-0 = <&pinctrl_i2crtc>;
		interrupt-parent = <&lsio_gpio0>;
		interrupts = <25 IRQ_TYPE_LEVEL_LOW>;
		backup-switchover-mode = <0x1>;
		status = "disabled";
	};

	eeprom_cb: eeprom@51 {
		compatible = "atmel,24c32";
		pagesize = <32>;
		reg = <0x51>;
		status = "okay";
	};

	sgtl5000: codec@a {
		#sound-dai-cells = <0>;
		compatible = "fsl,sgtl5000";
		reg = <0xa>;
		assigned-clocks = <&clk IMX_SC_R_AUDIO_PLL_0 IMX_SC_PM_CLK_PLL>,
				<&clk IMX_SC_R_AUDIO_PLL_0 IMX_SC_PM_CLK_SLV_BUS>,
				<&clk IMX_SC_R_AUDIO_PLL_0 IMX_SC_PM_CLK_MST_BUS>,
				<&mclkout0_lpcg 0>;
		assigned-clock-rates = <786432000>, <49152000>, <12000000>, <12000000>;
		clocks = <&mclkout0_lpcg 0>;
		clock-names = "mclk";
		VDDA-supply = <&reg_3p3v>;
		VDDIO-supply = <&reg_3p3v>;
		VDDD-supply = <&reg_1p8v>;
	};

	xio: gpio@20 {
		compatible = "nxp,pcf8574a";
		reg = <0x20>;
		gpio-controller;
		#gpio-cells = <2>;
	};
};

&lpuart0 {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_lpuart0>;
	status = "okay";
};

&dc0_prg1 {
	status = "disabled";
};

&dc0_prg2 {
	status = "disabled";
};

&dc0_prg3 {
	status = "disabled";
};

&dc0_prg4 {
	status = "disabled";
};

&dc0_prg5 {
	status = "disabled";
};

&dc0_prg6 {
	status = "disabled";
};

&dc0_prg7 {
	status = "disabled";
};

&dc0_prg8 {
	status = "disabled";
};

&dc0_prg9 {
	status = "disabled";
};

&dc0_dpr1_channel1 {
	status = "disabled";
};

&dc0_dpr1_channel2 {
	status = "disabled";
};

&dc0_dpr1_channel3 {
	status = "disabled";
};

&dc0_dpr2_channel1 {
	status = "disabled";
};

&dc0_dpr2_channel2 {
	status = "disabled";
};

&dc0_dpr2_channel3 {
	status = "disabled";
};

&dpu1 {
	status = "disabled";
};

&usbphy1 {
	status = "okay";
};

&usdhc2 {
	pinctrl-names = "default", "state_100mhz", "state_200mhz";
	pinctrl-0 = <&pinctrl_usdhc2_sd>, <&pinctrl_usdhc2_gpio>;
	pinctrl-1 = <&pinctrl_usdhc2_sd>, <&pinctrl_usdhc2_gpio>;
	pinctrl-2 = <&pinctrl_usdhc2_sd>, <&pinctrl_usdhc2_gpio>;
	bus-width = <4>;
	no-1-8-v;
	no-uhs;
	no-sd-highspeed;             
	max-frequency = <25000000>; 
	cd-gpios = <&lsio_gpio4 22 GPIO_ACTIVE_LOW>;
	status = "okay";
};

&fec2 {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_fec2>;
	phy-mode = "rgmii-id";
	phy-handle = <&ethphy1>;
	fsl,magic-packet;
	nvmem-cells = <&fec_mac1>;
	nvmem-cell-names = "mac-address";
	status = "okay";
};

&mdio {
	ethphy1: ethernet-phy@3 {
		compatible = "ethernet-phy-ieee802.3-c22";
		reg = <3>;
		ti,rx-internal-delay = <DP83867_RGMIIDCTL_2_50_NS>;
		ti,tx-internal-delay = <DP83867_RGMIIDCTL_2_50_NS>;
		ti,fifo-depth = <DP83867_PHYCR_FIFO_DEPTH_8_B_NIB>;
		ti,led-0-active-low;
		ti,led-2-active-low;
		enet-phy-lane-no-swap;
	};
};

&ldb1_phy {
	status = "disabled";
};

&ldb2_phy {
	status = "disabled";
};

&ldb1 {
	status = "disabled";
};

&ldb2 {
	status = "disabled";
};

&dc0_pc {
	status = "disabled";
};

&phyx1 {
	fsl,refclk-pad-mode = <IMX8_PCIE_REFCLK_PAD_INPUT>;
	status = "okay";
};

&lpspi2 {
	fsl,spi-num-chipselects = <1>;
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_lpspi2>;
	status = "okay";

	spidev2_0: spi2@0 {
		reg = <0>;
		compatible = "rohm,dh2228fv";
		spi-max-frequency = <10000000>;
	};
};

&lsio_gpio0{
	usb_port_select-hog {
		gpio-hog;
		gpios = <30 GPIO_ACTIVE_HIGH>;
		output-low;
		line-name = "USB port select GPIO";
	};
};

&lsio_gpio1 {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_gpio1>;
	
	gpio-line-names =
		/* 0-7 */
		"", "", "", "", "", "", "", "",
		/* 8 */
		"",
		/* 9-10: named */
		"mariner_gpio09", "mariner_gpio10",
		/* 11 */
		"",
		/* 12-14: named */
		"mariner_gpio12", "mariner_gpio13", "mariner_gpio14",
		/* 15-16 */
		"", "",
		/* 17: named */
		"mariner_gpio17",
		/* 18-20 (you muxed them too, but didn't list as hogs — still usable) */
		"mariner_gpio18", "mariner_gpio19", "mariner_gpio20",
		/* 21-24 */
		"", "", "", "",
		/* 25-26: */
		"mariner_gpio25", "mariner_gpio26",
		/* 27-28 */
		"", "",
		/* 29-30: */
		"mariner_gpio29", "mariner_gpio30",
		/* 31 */
		"";
};	

/* New Added pins */
&i2c0 {
	#address-cells = <1>;
	#size-cells = <0>;
	clock-frequency = <100000>;
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_lpi2c0>;
	status = "okay";
};

&i2c3 {
	#address-cells = <1>;
	#size-cells = <0>;
	clock-frequency = <100000>;
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_lpi2c3>;
	status = "okay";
};

&lpuart2 {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_lpuart2>;
	status = "okay";
};

&iomuxc {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_hog>;

	scales-mariner-1 {

		/* For GPIO */
		pinctrl_hog: hoggrp {
			fsl,pins = <
				IMX8QXP_SPI0_CS1_LSIO_GPIO1_IO07	0x06000021
			>;
		};	
		pinctrl_i2crtc: i2crtcgrp {
			fsl,pins = <
				IMX8QXP_SAI0_TXD_LSIO_GPIO0_IO25	0x06000021
			>;
		};

		pinctrl_lpi2c1: lpi1cgrp {
			fsl,pins = <
				IMX8QXP_USB_SS3_TC1_ADMA_I2C1_SCL	0x06000020
				IMX8QXP_USB_SS3_TC3_ADMA_I2C1_SDA	0x06000020
			>;
		};

		pinctrl_fec2: fec2grp {
			fsl,pins = <
				IMX8QXP_ESAI0_SCKR_CONN_ENET1_RGMII_TX_CTL	0x06000020
				IMX8QXP_ESAI0_FSR_CONN_ENET1_RGMII_TXC		0x06000020
				IMX8QXP_ESAI0_TX4_RX1_CONN_ENET1_RGMII_TXD0	0x06000020
				IMX8QXP_ESAI0_TX5_RX0_CONN_ENET1_RGMII_TXD1	0x06000020
				IMX8QXP_ESAI0_FST_CONN_ENET1_RGMII_TXD2		0x06000020
				IMX8QXP_ESAI0_SCKT_CONN_ENET1_RGMII_TXD3	0x06000020
				IMX8QXP_ESAI0_TX0_CONN_ENET1_RGMII_RXC		0x06000020
				IMX8QXP_SPDIF0_TX_CONN_ENET1_RGMII_RX_CTL	0x06000020
				IMX8QXP_SPDIF0_RX_CONN_ENET1_RGMII_RXD0		0x06000020
				IMX8QXP_ESAI0_TX3_RX2_CONN_ENET1_RGMII_RXD1	0x06000020
				IMX8QXP_ESAI0_TX2_RX3_CONN_ENET1_RGMII_RXD2	0x06000020
				IMX8QXP_ESAI0_TX1_CONN_ENET1_RGMII_RXD3		0x06000020
			>;
		};

		pinctrl_lpuart0: lpuart0grp {
			fsl,pins = <
				IMX8QXP_UART0_RX_ADMA_UART0_RX	0x0600002c
				IMX8QXP_UART0_TX_ADMA_UART0_TX	0x0600002c
			>;
		};
		
		pinctrl_lpuart2: lpuart2grp {
			fsl,pins = <
				IMX8QXP_UART2_TX_ADMA_UART2_TX	0x0600002c
				IMX8QXP_UART2_RX_ADMA_UART2_RX	0x0600002c
			>;
		};

		pinctrl_usdhc2_gpio: usdhc2gpiogrp {
			fsl,pins = <
				IMX8QXP_USDHC1_CD_B_LSIO_GPIO4_IO22	0x06000020
			>;
		};

		pinctrl_usdhc2_sd: usdhc2sdgrp {
			fsl,pins = <
				IMX8QXP_USDHC1_CLK_CONN_USDHC1_CLK	0x06000040
				IMX8QXP_USDHC1_CMD_CONN_USDHC1_CMD	0x06000060
				IMX8QXP_USDHC1_DATA0_CONN_USDHC1_DATA0	0x06000060
				IMX8QXP_USDHC1_DATA1_CONN_USDHC1_DATA1	0x06000060
				IMX8QXP_USDHC1_DATA2_CONN_USDHC1_DATA2	0x06000060
				IMX8QXP_USDHC1_DATA3_CONN_USDHC1_DATA3	0x06000060
			>;
		};
 
		pinctrl_lpspi2: lpspi2grp {
			fsl,pins = <
				IMX8QXP_SPI2_SCK_ADMA_SPI2_SCK	0x600004c
				IMX8QXP_SPI2_SDO_ADMA_SPI2_SDO	0x600004c
				IMX8QXP_SPI2_SDI_ADMA_SPI2_SDI	0x600004c
				IMX8QXP_SPI2_CS0_ADMA_SPI2_CS0	0x600004c
			>;
		};

		/* Updated Device Tree */

		// I2C0
		pinctrl_lpi2c0: lpi0cgrp {
			fsl,pins = <
				IMX8QXP_MIPI_CSI0_GPIO0_00_ADMA_I2C0_SCL	0x06000020 
				IMX8QXP_MIPI_CSI0_GPIO0_01_ADMA_I2C0_SDA	0x06000020
			>;
		};

		// I2C3 
		pinctrl_lpi2c3: lpi3cgrp {
			fsl,pins = <
				IMX8QXP_SPI3_CS1_ADMA_I2C3_SCL	0x06000020
				IMX8QXP_MCLK_IN1_ADMA_I2C3_SDA	0x06000020
			>;
		};	

		//GPIO1
		pinctrl_gpio1: gpio1grp {
			fsl,pins = <
				IMX8QXP_ADC_IN1_LSIO_GPIO1_IO09			0x06000041
				IMX8QXP_ADC_IN0_LSIO_GPIO1_IO10			0x06000041
				IMX8QXP_ADC_IN2_LSIO_GPIO1_IO12			0x06000041
				IMX8QXP_ADC_IN5_LSIO_GPIO1_IO13			0x06000041
				IMX8QXP_ADC_IN4_LSIO_GPIO1_IO14			0x06000041
				IMX8QXP_FLEXCAN1_RX_LSIO_GPIO1_IO17		0x06000041
				IMX8QXP_FLEXCAN1_TX_LSIO_GPIO1_IO18		0x06000041
				IMX8QXP_FLEXCAN2_RX_LSIO_GPIO1_IO19		0x06000041
				IMX8QXP_FLEXCAN2_TX_LSIO_GPIO1_IO20		0x06000041
				IMX8QXP_MIPI_DSI0_I2C0_SCL_LSIO_GPIO1_IO25	0x06000041
				IMX8QXP_MIPI_DSI0_I2C0_SDA_LSIO_GPIO1_IO26	0x06000041
				IMX8QXP_MIPI_DSI1_I2C0_SCL_LSIO_GPIO1_IO29	0x06000041
				IMX8QXP_MIPI_DSI1_I2C0_SDA_LSIO_GPIO1_IO30	0x06000041
			>;
		};
	};
};
```

</details>

<details>
<summary>Edited SoM include file: <code>imx8qxp-phycore-som-emmc.dtsi</code></summary>

```
// SPDX-License-Identifier: GPL-2.0+
/*
 * Copyright (C) 2018 PHYTEC Messtechnik GmbH
 *
 * Copyright (C) 2019-2021 PHYTEC America, LLC
 */

#include "../freescale/imx8qxp.dtsi"
#include <dt-bindings/gpio/gpio.h>
#include <dt-bindings/net/ti-dp83867.h>

/ {
	model = "Phytec phyCORE-i.MX8X";
	compatible = "phytec,imx8qxp-pcm065", "fsl,imx8qxp";

	/* Make the I2C RTC default */
	aliases {
		rtc0 = &i2c_rtc;
		rtc1 = &rtc;
	};

	reserved-memory {
		#address-cells = <2>;
		#size-cells = <2>;
		ranges;

		gpu_reserved: gpu_reserved@880000000 {
			no-map;
			reg = <0x8 0x80000000 0 0x10000000>;
		};

		decoder_boot: decoder-boot@84000000 {
			reg = <0 0x84000000 0 0x2000000>;
			no-map;
		};

		encoder_boot: encoder-boot@86000000 {
			reg = <0 0x86000000 0 0x200000>;
			no-map;
		};

		decoder_rpc: decoder-rpc@0x92000000 {
			reg = <0 0x92000000 0 0x100000>;
			no-map;
		};

		dsp_reserved: dsp@92400000 {
			reg = <0 0x92400000 0 0x1000000>;
			no-map;
		};
		dsp_reserved_heap: dsp_reserved_heap {
			reg = <0 0x93400000 0 0xef0000>;
			no-map;
		};
		dsp_vdev0vring0: vdev0vring0@942f0000 {
			reg = <0 0x942f0000 0 0x8000>;
			no-map;
		};

		dsp_vdev0vring1: vdev0vring1@942f8000 {
			reg = <0 0x942f8000 0 0x8000>;
			no-map;
		};

		dsp_vdev0buffer: vdev0buffer@94300000 {
			compatible = "shared-dma-pool";
			reg = <0 0x94300000 0 0x100000>;
			no-map;
		};

		encoder_rpc: encoder-rpc@0x94400000 {
			reg = <0 0x94400000 0 0x700000>;
			no-map;
		};
	};

	reg_adc_vref_1v8: adc_vref_1v8 {
		compatible = "regulator-fixed";
		regulator-name = "ADC_VREFH";
		regulator-min-microvolt = <1800000>;
		regulator-max-microvolt = <1800000>;
	};
};

&fec1 {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_fec1>;
	phy-mode = "rgmii-id";
	phy-handle = <&ethphy0>;
	nvmem-cells = <&fec_mac0>;
	nvmem-cell-names = "mac-address";
	fsl,magic-packet;
	status = "okay";

	mdio: mdio {
		#address-cells = <1>;
		#size-cells = <0>;

		ethphy0: ethernet-phy@1 {
			compatible = "ethernet-phy-ieee802.3-c22";
			reg = <1>;
			ti,rx-internal-delay = <DP83867_RGMIIDCTL_2_00_NS>;
			ti,tx-internal-delay = <DP83867_RGMIIDCTL_2_00_NS>;
			ti,fifo-depth = <DP83867_PHYCR_FIFO_DEPTH_8_B_NIB>;
			ti,led-0-active-low;
			ti,led-2-active-low;
			enet-phy-lane-no-swap;
		};
	};
};

&flexspi0 {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_flexspi0>;
	status = "okay";

	flash0: mt35xu512aba@0 {
		reg = <0>;
		#address-cells = <1>;
		#size-cells = <1>;
		compatible = "jedec,spi-nor";
		spi-max-frequency = <29000000>;
		spi-tx-bus-width = <4>;
		spi-rx-bus-width = <4>;
	};
};

&i2c_mipi_csi0 {
	#address-cells = <1>;
	#size-cells = <0>;
	clock-frequency = <100000>;
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_i2c_mipi_csi0>;
	status = "okay";

	i2c_mipi_csi0_eeprom: eeprom@56 {
		compatible = "atmel,24c32";
		pagesize = <32>;
		reg = <0x56>;
		status = "okay";
	};
};

&irqsteer_csi0 {
	status = "okay";
};

&usdhc1 {
	pinctrl-names = "default", "state_100mhz", "state_200mhz";
	pinctrl-0 = <&pinctrl_usdhc1>;
	pinctrl-1 = <&pinctrl_usdhc1_100mhz>;
	pinctrl-2 = <&pinctrl_usdhc1_200mhz>;
	bus-width = <8>;
	no-sd;
	no-sdio;
	non-removable;
	status = "okay";
};

&thermal_zones {
	pmic-thermal0 {
		polling-delay-passive = <250>;
		polling-delay = <2000>;
		thermal-sensors = <&tsens IMX_SC_R_PMIC_0>;

		trips {
			pmic_alert0: trip0 {
				temperature = <110000>;
				hysteresis = <2000>;
				type = "passive";
			};

			pmic_crit0: trip1 {
				temperature = <125000>;
				hysteresis = <2000>;
				type = "critical";
			};
		};

		cooling-maps {
			map0 {
				trip = <&pmic_alert0>;
				cooling-device =
					<&A35_0 THERMAL_NO_LIMIT THERMAL_NO_LIMIT>,
					<&A35_1 THERMAL_NO_LIMIT THERMAL_NO_LIMIT>,
					<&A35_2 THERMAL_NO_LIMIT THERMAL_NO_LIMIT>,
					<&A35_3 THERMAL_NO_LIMIT THERMAL_NO_LIMIT>,
					<&imx8_gpu_ss 0 1>;
			};
		};
	};
};

&iomuxc {
	pcm065 {
		pinctrl_fec1: fec1grp {
			fsl,pins = <
				IMX8QXP_ENET0_MDC_CONN_ENET0_MDC			0x06000020
				IMX8QXP_ENET0_MDIO_CONN_ENET0_MDIO			0x06000060
				IMX8QXP_ENET0_RGMII_TX_CTL_CONN_ENET0_RGMII_TX_CTL	0x06000020
				IMX8QXP_ENET0_RGMII_TXC_CONN_ENET0_RGMII_TXC		0x06000020
				IMX8QXP_ENET0_RGMII_TXD0_CONN_ENET0_RGMII_TXD0		0x06000020
				IMX8QXP_ENET0_RGMII_TXD1_CONN_ENET0_RGMII_TXD1		0x06000020
				IMX8QXP_ENET0_RGMII_TXD2_CONN_ENET0_RGMII_TXD2		0x06000020
				IMX8QXP_ENET0_RGMII_TXD3_CONN_ENET0_RGMII_TXD3		0x06000020
				IMX8QXP_ENET0_RGMII_RXC_CONN_ENET0_RGMII_RXC		0x06000020
				IMX8QXP_ENET0_RGMII_RX_CTL_CONN_ENET0_RGMII_RX_CTL	0x06000020
				IMX8QXP_ENET0_RGMII_RXD0_CONN_ENET0_RGMII_RXD0		0x06000020
				IMX8QXP_ENET0_RGMII_RXD1_CONN_ENET0_RGMII_RXD1		0x06000020
				IMX8QXP_ENET0_RGMII_RXD2_CONN_ENET0_RGMII_RXD2		0x06000020
				IMX8QXP_ENET0_RGMII_RXD3_CONN_ENET0_RGMII_RXD3		0x06000020
			>;
		};

		pinctrl_flexspi0: flexspi0grp {
			fsl,pins = <
				IMX8QXP_QSPI0A_DATA0_LSIO_QSPI0A_DATA0		0x0600004c
				IMX8QXP_QSPI0A_DATA1_LSIO_QSPI0A_DATA1		0x0600004c
				IMX8QXP_QSPI0A_DATA2_LSIO_QSPI0A_DATA2		0x0600004c
				IMX8QXP_QSPI0A_DATA3_LSIO_QSPI0A_DATA3		0x0600004c
				IMX8QXP_QSPI0A_DQS_LSIO_QSPI0A_DQS		0x0600004c
				IMX8QXP_QSPI0A_SS0_B_LSIO_QSPI0A_SS0_B		0x0600004c
				IMX8QXP_QSPI0A_SS1_B_LSIO_QSPI0A_SS1_B		0x0600004c
				IMX8QXP_QSPI0A_SCLK_LSIO_QSPI0A_SCLK		0x0600004c
				IMX8QXP_QSPI0B_SCLK_LSIO_QSPI0B_SCLK		0x0600004c
				IMX8QXP_QSPI0B_DATA0_LSIO_QSPI0B_DATA0		0x0600004c
				IMX8QXP_QSPI0B_DATA1_LSIO_QSPI0B_DATA1		0x0600004c
				IMX8QXP_QSPI0B_DATA2_LSIO_QSPI0B_DATA2		0x0600004c
				IMX8QXP_QSPI0B_DATA3_LSIO_QSPI0B_DATA3		0x0600004c
				IMX8QXP_QSPI0B_DQS_LSIO_QSPI0B_DQS		0x0600004c
				IMX8QXP_QSPI0B_SS0_B_LSIO_QSPI0B_SS0_B		0x0600004c
				IMX8QXP_QSPI0B_SS1_B_LSIO_QSPI0B_SS1_B		0x0600004c
			>;
		};

		pinctrl_i2c_mipi_csi0: i2c_mipi_csi0 {
			fsl,pins = <
				IMX8QXP_MIPI_CSI0_I2C0_SCL_MIPI_CSI0_I2C0_SCL 0xc2000020
				IMX8QXP_MIPI_CSI0_I2C0_SDA_MIPI_CSI0_I2C0_SDA 0xc2000020
			>;
		};

		pinctrl_usdhc1: usdhc1grp {
			fsl,pins = <
				IMX8QXP_EMMC0_CLK_CONN_EMMC0_CLK		0x06000041
				IMX8QXP_EMMC0_CMD_CONN_EMMC0_CMD		0x00000021
				IMX8QXP_EMMC0_DATA0_CONN_EMMC0_DATA0		0x00000021
				IMX8QXP_EMMC0_DATA1_CONN_EMMC0_DATA1		0x00000021
				IMX8QXP_EMMC0_DATA2_CONN_EMMC0_DATA2		0x00000021
				IMX8QXP_EMMC0_DATA3_CONN_EMMC0_DATA3		0x00000021
				IMX8QXP_EMMC0_DATA4_CONN_EMMC0_DATA4		0x00000021
				IMX8QXP_EMMC0_DATA5_CONN_EMMC0_DATA5		0x00000021
				IMX8QXP_EMMC0_DATA6_CONN_EMMC0_DATA6		0x00000021
				IMX8QXP_EMMC0_DATA7_CONN_EMMC0_DATA7		0x00000021
				IMX8QXP_EMMC0_STROBE_CONN_EMMC0_STROBE		0x06000041
				IMX8QXP_EMMC0_RESET_B_CONN_EMMC0_RESET_B	0x00000021
			>;
		};

		pinctrl_usdhc1_100mhz: usdhc1grp100mhz {
			fsl,pins = <
				IMX8QXP_EMMC0_CLK_CONN_EMMC0_CLK		0x06000040
				IMX8QXP_EMMC0_CMD_CONN_EMMC0_CMD		0x00000020
				IMX8QXP_EMMC0_DATA0_CONN_EMMC0_DATA0		0x00000020
				IMX8QXP_EMMC0_DATA1_CONN_EMMC0_DATA1		0x00000020
				IMX8QXP_EMMC0_DATA2_CONN_EMMC0_DATA2		0x00000020
				IMX8QXP_EMMC0_DATA3_CONN_EMMC0_DATA3		0x00000020
				IMX8QXP_EMMC0_DATA4_CONN_EMMC0_DATA4		0x00000020
				IMX8QXP_EMMC0_DATA5_CONN_EMMC0_DATA5		0x00000020
				IMX8QXP_EMMC0_DATA6_CONN_EMMC0_DATA6		0x00000020
				IMX8QXP_EMMC0_DATA7_CONN_EMMC0_DATA7		0x00000020
				IMX8QXP_EMMC0_STROBE_CONN_EMMC0_STROBE		0x06000040
				IMX8QXP_EMMC0_RESET_B_CONN_EMMC0_RESET_B	0x00000020
			>;
		};

		pinctrl_usdhc1_200mhz: usdhc1grp200mhz {
			fsl,pins = <
				IMX8QXP_EMMC0_CLK_CONN_EMMC0_CLK		0x06000040
				IMX8QXP_EMMC0_CMD_CONN_EMMC0_CMD		0x00000020
				IMX8QXP_EMMC0_DATA0_CONN_EMMC0_DATA0		0x00000020
				IMX8QXP_EMMC0_DATA1_CONN_EMMC0_DATA1		0x00000020
				IMX8QXP_EMMC0_DATA2_CONN_EMMC0_DATA2		0x00000020
				IMX8QXP_EMMC0_DATA3_CONN_EMMC0_DATA3		0x00000020
				IMX8QXP_EMMC0_DATA4_CONN_EMMC0_DATA4		0x00000020
				IMX8QXP_EMMC0_DATA5_CONN_EMMC0_DATA5		0x00000020
				IMX8QXP_EMMC0_DATA6_CONN_EMMC0_DATA6		0x00000020
				IMX8QXP_EMMC0_DATA7_CONN_EMMC0_DATA7		0x00000020
				IMX8QXP_EMMC0_STROBE_CONN_EMMC0_STROBE		0x06000040
				IMX8QXP_EMMC0_RESET_B_CONN_EMMC0_RESET_B	0x00000020
			>;
		};
	};
};
```

</details>

<details>
<summary>Original vendor file, for comparison: <code>imx8qxp-phytec-pcm-942.dts</code></summary>

```
// SPDX-License-Identifier: GPL-2.0+
/*
 * Copyright (C) 2018 PHYTEC Messtechnik GmbH
 *
 * Copyright (C) 2019-2021 PHYTEC America, LLC
 */

/dts-v1/;

#include "imx8qxp-phycore-som-emmc.dtsi"
#include <dt-bindings/leds/common.h>

/ {
	model = "PHYTEC i.MX8QX PCM-942";
	compatible = "phytec,imx8qxp-pcm-942", "phytec,imx8qxp-phycore-som", "fsl,imx8qxp";

	chosen {
		bootargs = "console=ttyLP0,115200 earlycon=lpuart32,0x5a060000,115200";
		stdout-path = &lpuart0;
	};

	leds {
		compatible = "gpio-leds";
		pinctrl-names = "default";
		pinctrl-0 = <&pinctrl_leds>;

		led-0 {
			label = "User LED";
			color = <LED_COLOR_ID_GREEN>;
			gpios = <&lsio_gpio0 28 GPIO_ACTIVE_HIGH>;
			linux,default-trigger = "heartbeat";
		};
	};

	reg_1p8v: regulator-1p8v {
		compatible = "regulator-fixed";
		regulator-name = "1P8V";
		regulator-min-microvolt = <1800000>;
		regulator-max-microvolt = <1800000>;
		regulator-always-on;
	};

	reg_3p3v: regulator-3p3v {
		compatible = "regulator-fixed";
		regulator-name = "3P3V";
		regulator-min-microvolt = <3300000>;
		regulator-max-microvolt = <3300000>;
		regulator-always-on;
	};

	reg_bt: regulator-bt {
		status = "disabled";
		compatible = "regulator-fixed";
		pinctrl-names = "default";
		pinctrl-0 = <&pinctrl_bt_en>;
		regulator-name = "bt";
		gpio = <&lsio_gpio4 20 GPIO_ACTIVE_HIGH>;
		enable-active-high;
		regulator-min-microvolt = <3300000>;
		regulator-max-microvolt = <3300000>;
		regulator-always-on;
	};

	reg_pcieb: regulator-pcie {
		compatible = "regulator-fixed";
		regulator-name = "mpcie_3v3";
		regulator-min-microvolt = <3300000>;
		regulator-max-microvolt = <3300000>;
		regulator-always-on;
	};

	sdio_pwrseq: sdio-pwrseq {
		status = "disabled";
		compatible = "mmc-pwrseq-simple";
		pinctrl-names = "default";
		pinctrl-0 = <&pinctrl_wifi_en>;
		reset-gpios = <&lsio_gpio4 19 GPIO_ACTIVE_LOW>;
	};

	sound {
		compatible = "simple-audio-card";
		simple-audio-card,format = "i2s";
		simple-audio-card,bitclock-master = <&dailink0_master>;
		simple-audio-card,frame-master = <&dailink0_master>;
		simple-audio-card,name = "imx8qxp-sgtl5000";

		simple-audio-card,cpu {
			sound-dai = <&sai1>;
		};

		dailink0_master: simple-audio-card,codec {
			sound-dai = <&sgtl5000>;
			clocks = <&mclkout0_lpcg 0>;
		};
	};
};

&i2c1 {
	#address-cells = <1>;
	#size-cells = <0>;
	clock-frequency = <100000>;
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_lpi2c1>;
	status = "okay";

	tps6598x: tps6598x@3f {
		compatible = "ti,tps6598x";
		reg = <0x3f>;
	};

	i2c_rtc: rtc@52 {
		compatible = "microcrystal,rv3028";
		reg = <0x52>;
		pinctrl-names = "default";
		pinctrl-0 = <&pinctrl_i2crtc>;
		interrupt-parent = <&lsio_gpio0>;
		interrupts = <25 IRQ_TYPE_LEVEL_LOW>;
		backup-switchover-mode = <0x1>;
		status = "okay";
	};

	eeprom_cb: eeprom@51 {
		compatible = "atmel,24c32";
		pagesize = <32>;
		reg = <0x51>;
		status = "okay";
	};

	sgtl5000: codec@a {
		#sound-dai-cells = <0>;
		compatible = "fsl,sgtl5000";
		reg = <0xa>;
		assigned-clocks = <&clk IMX_SC_R_AUDIO_PLL_0 IMX_SC_PM_CLK_PLL>,
				<&clk IMX_SC_R_AUDIO_PLL_0 IMX_SC_PM_CLK_SLV_BUS>,
				<&clk IMX_SC_R_AUDIO_PLL_0 IMX_SC_PM_CLK_MST_BUS>,
				<&mclkout0_lpcg 0>;
		assigned-clock-rates = <786432000>, <49152000>, <12000000>, <12000000>;
		clocks = <&mclkout0_lpcg 0>;
		clock-names = "mclk";
		VDDA-supply = <&reg_3p3v>;
		VDDIO-supply = <&reg_3p3v>;
		VDDD-supply = <&reg_1p8v>;
	};

	xio: gpio@20 {
		compatible = "nxp,pcf8574a";
		reg = <0x20>;
		gpio-controller;
		#gpio-cells = <2>;
	};
};

&lpuart0 {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_lpuart0>;
	status = "okay";
};

&lpuart1 {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_lpuart1>;
	status = "okay";
};

&gpu_3d0 {
	status = "okay";
};

&imx8_gpu_ss {
	status = "okay";
};

&dc0_prg1 {
	status = "okay";
};

&dc0_prg2 {
	status = "okay";
};

&dc0_prg3 {
	status = "okay";
};

&dc0_prg4 {
	status = "okay";
};

&dc0_prg5 {
	status = "okay";
};

&dc0_prg6 {
	status = "okay";
};

&dc0_prg7 {
	status = "okay";
};

&dc0_prg8 {
	status = "okay";
};

&dc0_prg9 {
	status = "okay";
};

&dc0_dpr1_channel1 {
	status = "okay";
};

&dc0_dpr1_channel2 {
	status = "okay";
};

&dc0_dpr1_channel3 {
	status = "okay";
};

&dc0_dpr2_channel1 {
	status = "okay";
};

&dc0_dpr2_channel2 {
	status = "okay";
};

&dc0_dpr2_channel3 {
	status = "okay";
};

&dpu1 {
	status = "okay";
};

&usbphy1 {
	status = "okay";
};

&usbotg1 {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_usbotg1>;
	dr_mode = "host";
	srp-disable;
	hnp-disable;
	adp-disable;
	power-active-high;
	disable-over-current;
	status = "okay";
};

&usb3_phy {
	status = "okay";
};

&usbotg3 {
	status = "okay";
};

&usbotg3_cdns3 {
	dr_mode = "host";
	usb-role-switch;
	status = "okay";
};

&usdhc2 {
	pinctrl-names = "default", "state_100mhz", "state_200mhz";
	pinctrl-0 = <&pinctrl_usdhc2_sd>, <&pinctrl_usdhc2_gpio>;
	pinctrl-1 = <&pinctrl_usdhc2_sd>, <&pinctrl_usdhc2_gpio>;
	pinctrl-2 = <&pinctrl_usdhc2_sd>, <&pinctrl_usdhc2_gpio>;
	bus-width = <4>;
	no-1-8-v;
	cd-gpios = <&lsio_gpio4 22 GPIO_ACTIVE_LOW>;
	status = "okay";
};

&fec2 {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_fec2>;
	phy-mode = "rgmii-id";
	phy-handle = <&ethphy1>;
	fsl,magic-packet;
	nvmem-cells = <&fec_mac1>;
	nvmem-cell-names = "mac-address";
	status = "okay";
};

&mdio {
	ethphy1: ethernet-phy@3 {
		compatible = "ethernet-phy-ieee802.3-c22";
		reg = <3>;
		ti,rx-internal-delay = <DP83867_RGMIIDCTL_2_50_NS>;
		ti,tx-internal-delay = <DP83867_RGMIIDCTL_2_50_NS>;
		ti,fifo-depth = <DP83867_PHYCR_FIFO_DEPTH_8_B_NIB>;
		ti,led-0-active-low;
		ti,led-2-active-low;
		enet-phy-lane-no-swap;
	};
};

&flexcan2 {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_flexcan2>;
	status = "okay";
};

&i2c0_mipi_lvds0 {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_i2c0_mipi_lvds0>;
	clock-frequency = <100000>;
	status = "disabled";
};

&i2c0_mipi_lvds1 {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_i2c0_mipi_lvds1>;
	clock-frequency = <100000>;
	status = "disabled";
};

&ldb1_phy {
	status = "disabled";
};

&ldb2_phy {
	status = "disabled";
};

&ldb1 {
	status = "disabled";
};

&ldb2 {
	status = "disabled";
};

&dc0_pc {
	status = "disabled";
};

&phyx1 {
	fsl,refclk-pad-mode = <IMX8_PCIE_REFCLK_PAD_INPUT>;
	status = "okay";
};

&pcieb{
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_pcieb>;
	reset-gpio = <&lsio_gpio4 0 GPIO_ACTIVE_LOW>;
	vpcie-supply = <&reg_pcieb>;
	ext_osc = <1>;
	status = "okay";
};

&sai1 {
	#sound-dai-cells = <0>;
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_sai1>;
	status = "okay";
};

&lpspi2 {
	fsl,spi-num-chipselects = <1>;
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_lpspi2>;
	status = "okay";

	spidev2_0: spi2@0 {
		reg = <0>;
		compatible = "rohm,dh2228fv";
		spi-max-frequency = <10000000>;
	};
};

&lpspi3 {
	fsl,spi-num-chipselects = <1>;
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_lpspi3>;
	status = "okay";

	spidev3_0: spi3@0 {
		reg = <0>;
		compatible = "rohm,dh2228fv";
		spi-max-frequency = <10000000>;
	};
};

&lsio_gpio0{
	usb_port_select-hog {
		gpio-hog;
		gpios = <30 GPIO_ACTIVE_HIGH>;
		output-low;
		line-name = "USB port select GPIO";
	};
};

&iomuxc {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_hog>;

	pcm942 {
		pinctrl_hog: hoggrp {
			fsl,pins = <
				IMX8QXP_SAI1_RXC_LSIO_GPIO0_IO30       0x06000021	 /* USBOTG1 Port Select */
				IMX8QXP_MCLK_OUT0_ADMA_ACM_MCLK_OUT0   0x0600004c	 /* Audio Clock*/
				IMX8QXP_SAI1_RXFS_LSIO_GPIO0_IO31      0x06000021	 /* CAN Fault */
				IMX8QXP_SPI0_CS1_LSIO_GPIO1_IO07       0x06000021	 /* ETH1 OR Gate */
			>;
		};

		pinctrl_leds: leds1grp {
			fsl,pins = <
				IMX8QXP_SAI0_TXFS_LSIO_GPIO0_IO28      0x06000021	 /* User LED */
			>;
		};

		pinctrl_i2crtc: i2crtcgrp {
			fsl,pins = <
				IMX8QXP_SAI0_TXD_LSIO_GPIO0_IO25        0x06000021
			>;
		};

		pinctrl_fec2: fec2grp {
			fsl,pins = <
				IMX8QXP_ESAI0_SCKR_CONN_ENET1_RGMII_TX_CTL	   0x06000020
				IMX8QXP_ESAI0_FSR_CONN_ENET1_RGMII_TXC		   0x06000020
				IMX8QXP_ESAI0_TX4_RX1_CONN_ENET1_RGMII_TXD0	   0x06000020
				IMX8QXP_ESAI0_TX5_RX0_CONN_ENET1_RGMII_TXD1	   0x06000020
				IMX8QXP_ESAI0_FST_CONN_ENET1_RGMII_TXD2		   0x06000020
				IMX8QXP_ESAI0_SCKT_CONN_ENET1_RGMII_TXD3	   0x06000020
				IMX8QXP_ESAI0_TX0_CONN_ENET1_RGMII_RXC		   0x06000020
				IMX8QXP_SPDIF0_TX_CONN_ENET1_RGMII_RX_CTL	   0x06000020
				IMX8QXP_SPDIF0_RX_CONN_ENET1_RGMII_RXD0		   0x06000020
				IMX8QXP_ESAI0_TX3_RX2_CONN_ENET1_RGMII_RXD1	   0x06000020
				IMX8QXP_ESAI0_TX2_RX3_CONN_ENET1_RGMII_RXD2	   0x06000020
				IMX8QXP_ESAI0_TX1_CONN_ENET1_RGMII_RXD3		   0x06000020
			>;
		};

		pinctrl_flexcan2: flexcan1grp {
			fsl,pins = <
				IMX8QXP_UART2_TX_ADMA_FLEXCAN1_TX		0x10000021
				IMX8QXP_UART2_RX_ADMA_FLEXCAN1_RX		0x10000021
			>;
		};

		pinctrl_lpi2c1: lpi1cgrp {
			fsl,pins = <
				IMX8QXP_USB_SS3_TC1_ADMA_I2C1_SCL	0x06000020
				IMX8QXP_USB_SS3_TC3_ADMA_I2C1_SDA	0x06000020
			>;
		};

		pinctrl_lpuart0: lpuart0grp {
			fsl,pins = <
				IMX8QXP_UART0_RX_ADMA_UART0_RX	0x0600002c
				IMX8QXP_UART0_TX_ADMA_UART0_TX	0x0600002c
			>;
		};

		pinctrl_lpuart1: lpuart1grp {
			fsl,pins = <
				IMX8QXP_UART1_TX_ADMA_UART1_TX	   0x06000020
				IMX8QXP_UART1_RX_ADMA_UART1_RX	   0x06000020
			>;
		};

		pinctrl_lpuart1_rtscts: lpuart1rtsctsgrp {
			fsl,pins = <
				IMX8QXP_UART1_RTS_B_ADMA_UART1_RTS_B     0x06000020
				IMX8QXP_UART1_CTS_B_ADMA_UART1_CTS_B     0x06000020
			>;
		};

		pinctrl_usdhc2_gpio: usdhc2gpiogrp {
			fsl,pins = <
				IMX8QXP_USDHC1_CD_B_LSIO_GPIO4_IO22	0x06000020
			>;
		};

		pinctrl_usdhc2_sd: usdhc2sdgrp {
			fsl,pins = <
				IMX8QXP_USDHC1_CLK_CONN_USDHC1_CLK	0x06000040
				IMX8QXP_USDHC1_CMD_CONN_USDHC1_CMD	0x06000060
				IMX8QXP_USDHC1_DATA0_CONN_USDHC1_DATA0	0x06000060
				IMX8QXP_USDHC1_DATA1_CONN_USDHC1_DATA1	0x06000060
				IMX8QXP_USDHC1_DATA2_CONN_USDHC1_DATA2	0x06000060
				IMX8QXP_USDHC1_DATA3_CONN_USDHC1_DATA3	0x06000060
			>;
		};

		pinctrl_usdhc2_wifi: usdhc2wifigrp {
			fsl,pins = <
				IMX8QXP_USDHC1_CLK_CONN_USDHC1_CLK      0x06000040
				IMX8QXP_USDHC1_CMD_CONN_USDHC1_CMD      0x06000020
				IMX8QXP_USDHC1_DATA0_CONN_USDHC1_DATA0  0x06000020
				IMX8QXP_USDHC1_DATA1_CONN_USDHC1_DATA1  0x06000020
				IMX8QXP_USDHC1_DATA2_CONN_USDHC1_DATA2  0x06000020
				IMX8QXP_USDHC1_DATA3_CONN_USDHC1_DATA3  0x06000020
			>;
		};

		pinctrl_usbotg1: usbotg1 {
			fsl,pins = <
				IMX8QXP_USB_SS3_TC0_CONN_USB_OTG1_PWR	   0x00000021
			>;
		};

		pinctrl_i2c0_mipi_lvds0: mipi_lvds0_i2c0_grp {
			fsl,pins = <
				IMX8QXP_MIPI_DSI0_I2C0_SCL_MIPI_DSI0_I2C0_SCL	   0x06000020
				IMX8QXP_MIPI_DSI0_I2C0_SDA_MIPI_DSI0_I2C0_SDA	   0x06000020
			>;
		};

		pinctrl_i2c0_mipi_lvds1: mipi_lvds1_i2c0_grp {
			fsl,pins = <
				IMX8QXP_MIPI_DSI1_I2C0_SCL_MIPI_DSI1_I2C0_SCL	   0x06000020
				IMX8QXP_MIPI_DSI1_I2C0_SDA_MIPI_DSI1_I2C0_SDA	   0x06000020
			>;
		};

		pinctrl_lvds0: lvds0grp {
			fsl,pins = <
				IMX8QXP_MIPI_DSI0_GPIO0_01_LSIO_GPIO1_IO28 0x06000021
			>;
		};

		pinctrl_lvds1: lvds1grp {
			fsl,pins = <
				IMX8QXP_MIPI_DSI1_GPIO0_01_LSIO_GPIO2_IO00 0x06000021
			>;
		};

		pinctrl_pcieb: pcieagrp{
			fsl,pins = <
				IMX8QXP_PCIE_CTRL0_PERST_B_LSIO_GPIO4_IO00    0x06000021
				IMX8QXP_PCIE_CTRL0_CLKREQ_B_LSIO_GPIO4_IO01   0x06000021
				IMX8QXP_PCIE_CTRL0_WAKE_B_LSIO_GPIO4_IO02     0x04000021
			>;
		};

		pinctrl_sai1: sai1grp {
			fsl,pins = <
				IMX8QXP_FLEXCAN1_TX_ADMA_SAI1_RXD     0x06000040
				IMX8QXP_FLEXCAN1_RX_ADMA_SAI1_TXD     0x06000060
				IMX8QXP_FLEXCAN0_TX_ADMA_SAI1_TXFS    0x06000040
				IMX8QXP_FLEXCAN0_RX_ADMA_SAI1_TXC     0x06000040
			>;
		};

		pinctrl_lpspi2: lpspi2grp {
			fsl,pins = <
				IMX8QXP_SPI2_SCK_ADMA_SPI2_SCK          0x600004c
				IMX8QXP_SPI2_SDO_ADMA_SPI2_SDO          0x600004c
				IMX8QXP_SPI2_SDI_ADMA_SPI2_SDI          0x600004c
				IMX8QXP_SPI2_CS0_ADMA_SPI2_CS0          0x600004c
			>;
		};

		pinctrl_lpspi3: lpspi3grp {
			fsl,pins = <
				IMX8QXP_SPI3_SCK_ADMA_SPI3_SCK          0x600004c
				IMX8QXP_SPI3_SDO_ADMA_SPI3_SDO          0x600004c
				IMX8QXP_SPI3_SDI_ADMA_SPI3_SDI          0x600004c
				IMX8QXP_SPI3_CS0_ADMA_SPI3_CS0          0x600004c
				IMX8QXP_SPI3_CS1_ADMA_SPI3_CS1          0x600004c
			>;
		};

		pinctrl_bt_en: btengpiogrp {
			fsl,pins = <
				IMX8QXP_USDHC1_VSELECT_LSIO_GPIO4_IO20	0x06000021
			>;
		};

		pinctrl_wifi_en: wifiengpiogrp {
			fsl,pins = <
				IMX8QXP_USDHC1_RESET_B_LSIO_GPIO4_IO19	0x06000021
			>;
		};
	};
};
```

</details>

More information:

- [Modifying the BSP](https://docs.phytec.com/projects/yocto-phycore-imx8x/en/latest/developingwithyocto/modifyBSP.html#change-the-linux-kernel-device-tree)
- [linux-phytec-imx/include/dt-bindings/pinctrl/pads-imx8qxp.h](https://github.com/phytec/linux-phytec-imx/blob/v6.6.52-2.2.0-phy/include/dt-bindings/pinctrl/pads-imx8qxp.h)
- [System Controller Firmware 101 - Pad configuration service](https://community.nxp.com/t5/i-MX-Processors-Knowledge-Base/System-Controller-Firmware-101-Pad-configuration-service/ta-p/1124213#:~:text=MX8%20has%20three%20types%20of%20I/Os:%20*,1.8V%20/%203.3V%20I/Os%20Dual%20Voltage%20I/Os)

#### Wire the new DTS into the kernel build

Add your new dtb above the reference board's dtb in the kernel's device tree Makefile (around line 534):
```bash
vim arch/arm64/boot/dts/freescale/Makefile
```
```
dtb-$(CONFIG_ARCH_MXC) += <your-custom-dts-name>.dtb
dtb-$(CONFIG_ARCH_MXC) += imx8qxp-phytec-pcm-942.dtb
```

#### Test your changes

Save and exit the devshell, then force a recompile — a plain `bitbake` won't notice devshell edits on its own:
```bash
cd $BUILDDIR
bitbake phytec-headless-image -c compile --force
bitbake phytec-headless-image
```
If you need to start clean instead:
```bash
bitbake phytec-headless-image -c cleansstate
```

#### Create a patch

Once you're happy with the result, turn your kernel changes into a patch so they survive a clean rebuild and can be tracked in your layer:
```bash
cd ~/BSP-Yocto-NXP-i.MX8X-PD24.1.y/yocto/build/tmp-ampliphy-vendor/work-shared/phycore-imx8x-1/kernel-source/
git add .
git commit -m "<describe what this patch does>"
git format-patch -1
```
`git format-patch` writes a `.patch` file to the current directory — you'll move it into your layer in Step 4. Use a descriptive commit message; it becomes the patch's filename.

Exit the devshell once the patch is created — you'll open a different one next, for the bootloader.

### Step 3: Customize the bootloader

Open a devshell for the bootloader:
```bash
bitbake virtual/bootloader -c devshell
```
Two files need changes: the board header, for the device tree filename and SD card read rate, and the SoM's device tree include file.

Edit the header:
```bash
vim include/configs/phycore_imx8x.h
```
<details>
<summary><code>phycore_imx8x.h</code></summary>

```
/* SPDX-License-Identifier: GPL-2.0+ */
/*
 * Copyright (C) 2023 PHYTEC America, LLC
 *
 * This program is free software; you can redistribute it and/or modify
 * it under the terms of the GNU General Public License version 2 as
 * published by the Free Software Foundation.
 */

#ifndef __PHYCORE_IMX8X_H
#define __PHYCORE_IMX8X_H

#include <linux/sizes.h>
#include <asm/arch/imx-regs.h>

#include "imx_env.h"

#ifdef CONFIG_SPL_BUILD
#define CFG_MALLOC_F_ADDR		0x00138000

/*
 * 0x08081000 - 0x08180FFF is for m4_0 xip image,
 * So 3rd container image may start from 0x8181000
 */
#define CFG_SYS_UBOOT_BASE 0x08181000
#endif

#ifdef CONFIG_AHAB_BOOT
#define AHAB_ENV "sec_boot=yes\0"
#else
#define AHAB_ENV "sec_boot=no\0"
#endif

/* Boot M4 */
#define M4_BOOT_ENV \
	"m4_0_image=m4_0.bin\0" \
	"loadm4image_0=fatload mmc ${mmcdev}:${mmcpart} ${loadaddr} ${m4_0_image}\0" \
	"m4boot_0=run loadm4image_0; dcache flush; bootaux ${loadaddr} 0\0" \

#define CFG_MFG_ENV_SETTINGS \
	CFG_MFG_ENV_SETTINGS_DEFAULT \
	"initrd_addr=0x83100000\0" \
	"initrd_high=0xffffffffffffffff\0" \
	"emmc_dev=0\0" \
	"sd_dev=1\0" \

/* Initial environment variables */
#define CFG_EXTRA_ENV_SETTINGS		\
	CFG_MFG_ENV_SETTINGS \
	M4_BOOT_ENV \
	AHAB_ENV \
	"script=boot.scr\0" \
	"image=Image\0" \
	"splashimage=0x9e000000\0" \
	"console=ttyLP0\0" \
	"fdt_addr=0x83000000\0"			\
	"fdto_addr=0x83100000\0" \
	"bootenv_addr=0x83200000\0" \
	"fdt_high=0xffffffffffffffff\0"		\
	"cntr_addr=0x98000000\0"			\
	"cntr_file=os_cntr_signed.bin\0" \
	"boot_fdt=try\0" \
	"fdt_file=imx8qxp-scales-mariner.dtb\0" \ /*This was changed*/
	"bootenv=bootenv.txt\0" \
	"mmc_load_bootenv=fatload mmc ${mmcdev}:${mmcpart} ${bootenv_addr} ${bootenv}\0" \
	"ipaddr=192.168.3.11\0" \
	"serverip=192.168.3.10\0" \
	"netmask=255.255.255.0\0" \
	"ip_dyn=no\0" \
	"mmcpart=1\0" \
	"mmcroot=2\0" \
	"mmcautodetect=yes\0" \
	"mmcargs=setenv bootargs console=${console},${baudrate} " \
		"root=/dev/mmcblk${mmcdev}p${mmcroot} fsck.repair=yes rootwait rw\0" \
	"loadbootscript=fatload mmc ${mmcdev}:${mmcpart} ${loadaddr} ${script};\0" \
	"bootscript=echo Running bootscript from mmc ...; " \
		"source\0" \
	"loadimage=fatload mmc ${mmcdev}:${mmcpart} ${loadaddr} ${image}\0" \
	"loadfdt=fatload mmc ${mmcdev}:${mmcpart} ${fdt_addr} ${fdt_file}\0" \
	"mmc_load_overlay=fatload mmc ${mmcdev}:${mmcpart} ${fdto_addr} ${overlay}\0" \
	"mmc_apply_overlays=fdt address ${fdt_addr}; "  \
		"if test ${no_overlays} = 0; then " \
			"for overlay in $overlays; " \
			"do; " \
				"if run mmc_load_overlay; then " \
					"fdt resize ${filesize}; " \
					"fdt apply ${fdto_addr}; " \
				"fi; " \
			"done;" \
		"fi;\0 " \
	"loadcntr=fatload mmc ${mmcdev}:${mmcpart} ${cntr_addr} ${cntr_file}\0" \
	"auth_os=auth_cntr ${cntr_addr}\0" \
	"boot_os=booti ${loadaddr} - ${fdt_addr};\0" \
	"mmcboot=echo Booting from mmc ...; " \
		"if test ${no_bootenv} = 0; then " \
			"if run mmc_load_bootenv; then " \
				"env import -t ${bootenv_addr} ${filesize}; " \
			"fi; " \
		"fi; " \
		"run mmcargs; " \
		"if run loadfdt; then " \
			"run mmc_apply_overlays; " \
			"booti ${loadaddr} - ${fdt_addr}; " \
		"else " \
			"echo WARN: Cannot load the DT; " \
		"fi;\0 " \
	"nfsroot=/nfsroot\0" \
	"netargs=setenv bootargs console=${console},${baudrate} root=/dev/nfs ip=${nfsip} " \
		"nfsroot=${serverip}:${nfsroot},v4,tcp\0" \
	"net_load_bootenv=${get_cmd} ${bootenv_addr} ${bootenv}\0" \
	"net_load_overlay=${get_cmd} ${fdto_addr} ${overlay}\0" \
	"net_apply_overlays=fdt address ${fdt_addr}; " \
		"if test ${no_overlays} = 0; then " \
			"for overlay in $overlays; " \
			"do; " \
				"if run net_load_overlay; then " \
					"fdt resize ${filesize}; " \
					"fdt apply ${fdto_addr}; " \
				"fi; " \
			"done;" \
		"fi;\0 " \
	"netboot=echo Booting from net ...; " \
		"if test ${ip_dyn} = yes; then " \
			"setenv nfsip dhcp; " \
			"setenv get_cmd dhcp; " \
		"else " \
			"setenv nfsip ${ipaddr}:${serverip}::${netmask}::eth0:on; " \
			"setenv get_cmd tftp; " \
		"fi; " \
		"if test ${no_bootenv} = 0; then " \
			"if run net_load_bootenv; then " \
				"env import -t ${bootenv_addr} ${filesize}; " \
			"fi; " \
		"fi; " \
		"run netargs; " \
		"${get_cmd} ${loadaddr} ${image}; " \
		"if ${get_cmd} ${fdt_addr} ${fdt_file}; then " \
			"run net_apply_overlays; " \
			"booti ${loadaddr} - ${fdt_addr}; " \
		"else " \
			"echo WARN: Cannot load the DT; " \
		"fi;\0" \

#define CONFIG_BOOTCOMMAND \
	   "mmc dev ${mmcdev}; if mmc rescan; then " \
		   "if run loadbootscript; then " \
			   "run bootscript; " \
		   "else " \
			   "if test ${sec_boot} = yes; then " \
				   "if run loadcntr; then " \
					   "run mmcboot; " \
				   "else run netboot; " \
				   "fi; " \
		   "else " \
			   "if run loadimage; then " \
				   "run mmcboot; " \
			   "else run netboot; " \
			   "fi; " \
		   "fi; " \
		   "fi; " \
	   "else booti ${loadaddr} - ${fdt_addr}; fi"

/* Link Definitions */

/* On LPDDR4 board, USDHC1 is for eMMC, USDHC2 is for SD on CPU board */

#define CFG_SYS_SDRAM_BASE		0x80000000
#define PHYS_SDRAM_1			0x80000000
#define PHYS_SDRAM_2			0x880000000
#define PHYS_SDRAM_1_SIZE		0x80000000	/* 2 GB */
#define PHYS_SDRAM_2_SIZE		0x00000000	/* 0 GB */

/* Misc configuration */
#define PHY_ANEG_TIMEOUT 20000

#endif /* __PHYCORE_IMX8X_H */
```

</details>

Edit the device tree include:
```bash
vim arch/arm/dts/imx8qxp-phycore-som.dtsi
```
<details>
<summary><code>imx8qxp-phycore-som.dtsi</code></summary>

```
// SPDX-License-Identifier: GPL-2.0+
/*
 * Copyright (C) 2023 PHYTEC America, LLC
 *
 * This program is free software; you can redistribute it and/or modify
 * it under the terms of the GNU General Public License version 2 as
 * published by the Free Software Foundation.
 */

/dts-v1/;

#include "fsl-imx8qxp.dtsi"
#include <dt-bindings/net/ti-dp83867.h>

/ {
	model = "PHYTEC phyCORE-i.MX8X";
	compatible = "phytec,imx8qxp-phycore-som", "fsl,imx8qxp";

	aliases {
		i2c9 = &i2c0_csi0;
	};

	chosen {
		bootargs = "console=ttyLP0,115200 earlycon";
		stdout-path = &lpuart0;
	};

	regulators {
		compatible = "simple-bus";
		#address-cells = <1>;
		#size-cells = <0>;

		reg_usdhc2_vmmc: usdhc2_vmmc {
			compatible = "regulator-fixed";
			regulator-name = "SD1_SPWR";
			regulator-min-microvolt = <3000000>;
			regulator-max-microvolt = <3000000>;
			enable-active-high;
			off-on-delay-us = <3480>;
		};
	};
};

&fec1 {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_fec1>;
	phy-mode = "rgmii-id";
	phy-handle = <&ethphy0>;
	fsl,magic-packet;
	status = "okay";

	mdio: mdio {
		#address-cells = <1>;
		#size-cells = <0>;

		ethphy0: ethernet-phy@1 {
			compatible = "ethernet-phy-ieee802.3-c22";
			reg = <1>;
			ti,rx-internal-delay = <DP83867_RGMIIDCTL_2_00_NS>;
			ti,tx-internal-delay = <DP83867_RGMIIDCTL_2_00_NS>;
			ti,fifo-depth = <DP83867_PHYCR_FIFO_DEPTH_8_B_NIB>;
			ti,led-0-active-low;
			ti,led-2-active-low;
			enet-phy-lane-no-swap;
		};
	};
};

&i2c0_csi0 {
	#address-cells = <1>;
	#size-cells = <0>;
	clock-frequency = <100000>;
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_i2c0_csi0>;
	status = "okay";

	i2c_eeprom_som: eeprom@56 {
		compatible = "atmel,24c32";
		pagesize = <32>;
		reg = <0x56>;
		status = "okay";
	};
};

&iomuxc {
	imx8qxp-pcm065 {

		pinctrl_fec1: fec1grp {
			fsl,pins = <
				SC_P_ENET0_MDC_CONN_ENET0_MDC                   0x06000020
				SC_P_ENET0_MDIO_CONN_ENET0_MDIO                 0x06000020
				SC_P_ENET0_RGMII_TX_CTL_CONN_ENET0_RGMII_TX_CTL 0x06000021
				SC_P_ENET0_RGMII_TXC_CONN_ENET0_RGMII_TXC       0x06000021
				SC_P_ENET0_RGMII_TXD0_CONN_ENET0_RGMII_TXD0     0x06000021
				SC_P_ENET0_RGMII_TXD1_CONN_ENET0_RGMII_TXD1     0x06000021
				SC_P_ENET0_RGMII_TXD2_CONN_ENET0_RGMII_TXD2     0x06000021
				SC_P_ENET0_RGMII_TXD3_CONN_ENET0_RGMII_TXD3     0x06000021
				SC_P_ENET0_RGMII_RXC_CONN_ENET0_RGMII_RXC       0x06000021
				SC_P_ENET0_RGMII_RX_CTL_CONN_ENET0_RGMII_RX_CTL 0x06000021
				SC_P_ENET0_RGMII_RXD0_CONN_ENET0_RGMII_RXD0     0x06000061
				SC_P_ENET0_RGMII_RXD1_CONN_ENET0_RGMII_RXD1     0x06000061
				SC_P_ENET0_RGMII_RXD2_CONN_ENET0_RGMII_RXD2     0x06000061
				SC_P_ENET0_RGMII_RXD3_CONN_ENET0_RGMII_RXD3     0x06000061
			>;
		};

		pinctrl_i2c0_csi0: i2c0csi0grp {
			fsl,pins = <
				SC_P_MIPI_CSI0_I2C0_SCL_MIPI_CSI0_I2C0_SCL 0x06000020
				SC_P_MIPI_CSI0_I2C0_SDA_MIPI_CSI0_I2C0_SDA 0x06000020
			>;
		};

		pinctrl_lpuart0: lpuart0grp {
			fsl,pins = <
				SC_P_UART0_RX_ADMA_UART0_RX	0x06000020
				SC_P_UART0_TX_ADMA_UART0_TX	0x06000020
			>;
		};

		pinctrl_usdhc1: usdhc1grp {
			fsl,pins = <
				SC_P_EMMC0_CLK_CONN_EMMC0_CLK		0x06000041
				SC_P_EMMC0_CMD_CONN_EMMC0_CMD		0x00000021
				SC_P_EMMC0_DATA0_CONN_EMMC0_DATA0	0x00000021
				SC_P_EMMC0_DATA1_CONN_EMMC0_DATA1	0x00000021
				SC_P_EMMC0_DATA2_CONN_EMMC0_DATA2	0x00000021
				SC_P_EMMC0_DATA3_CONN_EMMC0_DATA3	0x00000021
				SC_P_EMMC0_DATA4_CONN_EMMC0_DATA4	0x00000021
				SC_P_EMMC0_DATA5_CONN_EMMC0_DATA5	0x00000021
				SC_P_EMMC0_DATA6_CONN_EMMC0_DATA6	0x00000021
				SC_P_EMMC0_DATA7_CONN_EMMC0_DATA7	0x00000021
				SC_P_EMMC0_STROBE_CONN_EMMC0_STROBE	0x06000041
				SC_P_EMMC0_RESET_B_CONN_EMMC0_RESET_B	0x00000021
			>;
		};

		pinctrl_usdhc2_gpio: usdhc2gpiogrp {
			fsl,pins = <
				SC_P_USDHC1_RESET_B_LSIO_GPIO4_IO19	0x00000021
				SC_P_USDHC1_WP_LSIO_GPIO4_IO21		0x00000021
				SC_P_USDHC1_CD_B_LSIO_GPIO4_IO22	0x00000021
			>;
		};

		pinctrl_usdhc2: usdhc2grp {
			fsl,pins = <
				SC_P_USDHC1_CLK_CONN_USDHC1_CLK		0x06000040
				SC_P_USDHC1_CMD_CONN_USDHC1_CMD		0x00000060
				SC_P_USDHC1_DATA0_CONN_USDHC1_DATA0	0x00000060
				SC_P_USDHC1_DATA1_CONN_USDHC1_DATA1	0x00000060
				SC_P_USDHC1_DATA2_CONN_USDHC1_DATA2	0x00000060
				SC_P_USDHC1_DATA3_CONN_USDHC1_DATA3	0x00000060
				SC_P_USDHC1_VSELECT_CONN_USDHC1_VSELECT	0x00000060
			>;
		};

		pinctrl_flexspi0: flexspi0grp {
			fsl,pins = <
				SC_P_QSPI0A_DATA0_LSIO_QSPI0A_DATA0	0x06000021
				SC_P_QSPI0A_DATA1_LSIO_QSPI0A_DATA1	0x06000021
				SC_P_QSPI0A_DATA2_LSIO_QSPI0A_DATA2	0x06000021
				SC_P_QSPI0A_DATA3_LSIO_QSPI0A_DATA3	0x06000021
				SC_P_QSPI0A_DQS_LSIO_QSPI0A_DQS		0x06000021
				SC_P_QSPI0A_SS0_B_LSIO_QSPI0A_SS0_B	0x06000021
				SC_P_QSPI0A_SS1_B_LSIO_QSPI0A_SS1_B	0x06000021
				SC_P_QSPI0A_SCLK_LSIO_QSPI0A_SCLK	0x06000021

				/*
				 * For QSPI instead of OSPI, comment out the
				 * following lines.
				 */
				SC_P_QSPI0B_SCLK_LSIO_QSPI0B_SCLK	0x06000021
				SC_P_QSPI0B_DATA0_LSIO_QSPI0B_DATA0	0x06000021
				SC_P_QSPI0B_DATA1_LSIO_QSPI0B_DATA1	0x06000021
				SC_P_QSPI0B_DATA2_LSIO_QSPI0B_DATA2	0x06000021
				SC_P_QSPI0B_DATA3_LSIO_QSPI0B_DATA3	0x06000021
				SC_P_QSPI0B_DQS_LSIO_QSPI0B_DQS		0x06000021
				SC_P_QSPI0B_SS0_B_LSIO_QSPI0B_SS0_B	0x06000021
				SC_P_QSPI0B_SS1_B_LSIO_QSPI0B_SS1_B	0x06000021
			>;
		};
	};
};

&A35_0 {
	bootph-all;
};

&lpuart0 {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_lpuart0>;
	status = "okay";
};

&usdhc1 {
	pinctrl-names = "default", "state_100mhz", "state_200mhz";
	pinctrl-0 = <&pinctrl_usdhc1>;
	pinctrl-1 = <&pinctrl_usdhc1>;
	pinctrl-2 = <&pinctrl_usdhc1>;
	bus-width = <8>;
	non-removable;
	status = "okay";
};

&usdhc2 {
	pinctrl-names = "default", "state_100mhz", "state_200mhz";
	pinctrl-0 = <&pinctrl_usdhc2>, <&pinctrl_usdhc2_gpio>;
	pinctrl-1 = <&pinctrl_usdhc2>, <&pinctrl_usdhc2_gpio>;
	pinctrl-2 = <&pinctrl_usdhc2>, <&pinctrl_usdhc2_gpio>;
	bus-width = <4>;
	cd-gpios = <&gpio4 22 GPIO_ACTIVE_LOW>;
	wp-gpios = <&gpio4 21 GPIO_ACTIVE_LOW>;
	vmmc-supply = <&reg_usdhc2_vmmc>;
	no-1-8-v;
	no-uhs;
	no-sd-highspeed;
	max-frequency = <25000000>; /* This was added*/
	status = "okay";
};

&flexspi0 {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_flexspi0>;
	status = "okay";

	flash0: mt35xu512aba@0 {
		reg = <0>;
		#address-cells = <1>;
		#size-cells = <1>;
		compatible = "jedec,spi-nor";
		spi-max-frequency = <29000000>;
		spi-nor,ddr-quad-read-dummy = <8>;
	};
};
```

</details>

Create the patch the same way as in [Create a patch](#create-a-patch) — `git add`, `git commit`, `git format-patch -1` — but from U-Boot's own work directory, not the kernel's. Find it first, since the exact version suffix in the path changes between BSP releases:
```bash
find ~/BSP-Yocto-NXP-i.MX8X-PD24.1.y/yocto/build/tmp-ampliphy-vendor/work -maxdepth 4 -type d -path '*u-boot-phytec-imx*/git'
```
Then `cd` into the directory it finds and repeat the same three commands:
```bash
git add .
git commit -m "<describe what this patch does>"
git format-patch -1
```

Now that both patches exist, you can start wiring them into your layer.

### Step 4: Wire your patches into the layer

Copy both patches into the recipe folders you created in Step 1:
```bash
cp ~/BSP-Yocto-NXP-i.MX8X-PD24.1.y/yocto/build/tmp-ampliphy-vendor/work-shared/phycore-imx8x-1/kernel-source/<your-kernel>.patch \
   ~/BSP-Yocto-NXP-i.MX8X-PD24.1.y/yocto/sources/<your-layer-name>/recipes-kernel/linux/files/

cp <path-to-uboot-work-dir>/<your-uboot>.patch \
   ~/BSP-Yocto-NXP-i.MX8X-PD24.1.y/yocto/sources/<your-layer-name>/recipes-bsp/u-boot/files/
```

Then point each `.bbappend` at its patch:
```
FILESEXTRAPATHS:prepend := "${THISDIR}/files:"
SRC_URI += "file://<YOUR PATCH>.patch"
COMPATIBLE_MACHINE:append = "|<your-machine-name>"
```
`FILESEXTRAPATHS` tells BitBake to look in your layer's `files/` folder, `SRC_URI` adds the patch to the recipe, and `COMPATIBLE_MACHINE` makes the recipe apply to your custom machine. Here's how that looked for the SCALES carrier board:

<details>
<summary><code>linux-phytec-imx_%.bbappend</code></summary>

```
FILESEXTRAPATHS:prepend := "${THISDIR}/files:"

SRC_URI += "file://0001-Creating-the-scale-mariner-device-tree.patch"

COMPATIBLE_MACHINE .= "|scales-mariner-1"
```

</details>

<details>
<summary><code>u-boot-phytec-imx_%.bbappend</code></summary>

```
FILESEXTRAPATHS:prepend := "${THISDIR}/files:"

SRC_URI += "file://0001-Creating-the-U-boot-needed-for-scales-board.patch"

COMPATIBLE_MACHINE .= "|scales-mariner-1"
```

</details>

### Step 5: Build with your custom machine

Point the build at your machine instead of PHYTEC's reference machine:
```bash
vim conf/local.conf
```
```
MACHINE ?= "<your-machine-name>"
#MACHINE ?= "phycore-imx8x-1"
```
Then rebuild:
```bash
bitbake phytec-headless-image -c cleansstate
bitbake phytec-headless-image
```

Once it succeeds, flash and boot the image the same way as the reference build — see [Flashing and Booting the Board](imx_yocto_bsp.md#flashing-and-booting-the-board) — and build the SDK the same way if you need cross-compilation, see [Setting up the SDK](imx_yocto_bsp.md#setting-up-the-sdk).

## Using an existing custom layer

If a `meta-` layer for your carrier board already exists — for example, the SCALES Leviathan carrier board's `meta-scales-leviathan` — you don't need to repeat Steps 1-5 from scratch. Follow [Installing the `meta-scales-leviathan` Layer in the Base i.MX8QXP PHYTEC BSP](imx_yocto_bsp.md#installing-the-meta-scales-leviathan-layer-in-the-base-imx8qxp-phytec-bsp) instead.
