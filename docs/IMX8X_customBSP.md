// ...existing code...

# Understanding the Process

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

Reproducibility is the goal: another engineer should be able to clone the layers, select the machine, run the build, and generate the same bootable image.

---

# Setting up your Development Operating System

Recommended host: Ubuntu 22.04 (Desktop or Server).
If you are natively running Ubuntu 22.04, you can skip to the next chapter.

- Download: https://releases.ubuntu.com/jammy/
- VM options: VirtualBox (Windows), UTM (Mac). When using UTM for Yocto builds, emulate x86_64 (not ARM).

If using a VM, configure networking and SSH for convenience:

1. Obtain the VM IP address:
```bash
ip a
```

2. (Optional) Configure a static IP (Netplan on Ubuntu) to avoid address changes.

3. Create an SSH config entry on the host (Mac):
```bash
nano ~/.ssh/config
```
Add:
```text
Host yocto-vm
    HostName 192.168.x.x
    User your-username
    Port 22
```

4. Set correct permissions:
```bash
chmod 600 ~/.ssh/config
```

5. Connect:
```bash
ssh yocto-vm
```

---

# Using the Yocto Environment to Build Your BSP

This guide outlines setting up Yocto Linux v5.0 (scarthgap) on Ubuntu 22.04 and building the PHYTEC i.MX8X BSP.

Resources:
- Documentation: https://scales-docs.readthedocs.io/en/latest/imx_yocto_bsp/

## 1. Building the Developer Kit BSP

### First-time Setup

1. Install Podman:
```bash
sudo apt-get update
sudo apt-get install podman
```

2. Configure git:
```bash
git config --global user.email "your@gmail.com"
git config --global user.name "Your Name"
```

3. Create a working directory:
```bash
mkdir -p ~/BSP-Yocto-NXP-i.MX8X-PD24.1.y/yocto
cd ~/BSP-Yocto-NXP-i.MX8X-PD24.1.y/yocto
```

4. Pull the PHYTEC Podman image:
```bash
podman pull docker.io/phybuilder/yocto-ubuntu-22.04
```

### Configuring the Build

1. Start the Podman container (run this each time you start a new shell session):
```bash
podman run --rm=true -v /home:/home -e USER=$USER --userns=keep-id --workdir=$PWD -it docker.io/phybuilder/yocto-ubuntu-22.04 bash
```

2. Download BSP meta layers (interactive tool). Enter `1` when prompted:
```bash
phyLinux init -p imx8x -r BSP-Yocto-NXP-i.MX8X-PD24.1.y
```
Example output:
```text
1: phycore-imx8x-1: PHYTEC phyCORE i.MX8X
    PCM-942.A2, PCM-065-QP28NESI2.A0
    distro: ampliphy-vendor
    target: -c populate_sdk phytec-qt6demo-image
    target: phytec-headless-image
    target: phytec-qt6demo-image
```

3. Initialize the build environment (run this in the container every time before building):
```bash
source sources/poky/oe-init-build-env
```

4. Edit `conf/local.conf` to accept the NXP EULA and set parallelism:
```bash
vi conf/local.conf
```
Add or uncomment:
```conf
# Uncomment to accept NXP EULA (needed if any NXP / freescale layer is used)
ACCEPT_FSL_EULA = "1"

# Parallelism options - based on CPU count
BB_NUMBER_THREADS ?= "4"
PARALLEL_MAKE ?= "-j 4"
```

5. Start the build:
```bash
cd $BUILDDIR
bitbake phytec-headless-image
```

If you need to clean and rebuild:
```bash
bitbake phytec-headless-image -c cleansstate
```

6. Built images are in:
```bash
$BUILDDIR/deploy-ampliphy-vendor/images
```

---

## 2. Creating the Custom BSP

This section covers common tasks when customizing the i.MX8 BSP:

- Use the bitbake kernel devshell to modify and apply custom device trees.
- Modify the bootloader to include custom device trees.
- Add custom meta-layers and recipes to prebake your programs into the image.
- Use `.bbappend` files to apply patches or add files without modifying vendor recipes.

Recommended workflow:
1. Update the device tree to reflect carrier board changes (GPIOs, pinmux, peripherals).
2. Test changes quickly (devshell, rebuild kernel/DTB as needed).
3. Formalize changes in your custom `meta-` layer (DTS, recipes, patches).
4. Commit metadata so builds are reproducible.

Once the BSP is built from Phytec we now need to create the custom version. Get back into the directory and open Podman

```bash
podman run --rm=true -v /home:/home -e USER=$USER --userns=keep-id --workdir=$PWD -it docker.io/phybuilder/yocto-ubuntu-22.04 bash
```

initialize the build environment

```bash
. sources/poky/oe-init-build-env
```

Once there we have a list of items we need to attend to

1. Create our custom layer by adding all the folders and files
2. Create our custom dts file from the development kit
3. Edit our Make file so our device tree gets compiled
4. Edit the U-boot to use our new device tree
5. Create a patch and bitbake again

# Creating our own layer

## Editing in the Development shell

open an interactive shell within the active build environment inside the selected kernel

```bash
bitbake virtual/kernel -c devshell
```

Now we are going to be creating a copy of or desired device tree.

### How to find your dts

Since we are using the i.MX8X we are looking for the `imx8qxp-phytec-pcm-942.dts` which is in the directory of:

```bash
cd arch/arm64/boot/dts/freescale
```

If there is a different SOM then you can do:

```bash
find . -type f -name "*imx8qxp*"
```

We will be using the development board as a starting point for our custom DTS. We will copy it and then start editing it.

```bash
cp imx8qxp-phytec-pcm-942.dts <YOUR_CUSTOM_DTS>.dts
```

### Creating a Custom device tree

Make sure are in the same directory where your new custom dts is we can edit it with vim

```bash
vim <YOUR_CUSTOM_DTS>.dts
```

Make all your changes for setting up address values for your pin mux and pin declaration.

SIDE NOTE: There are two files we will need to look at for making changes which is our new `dts` and the `imx8qxp-phycore-som-emmc.dtsi` that relates to the our custom hardware.

### Making you custom dts

We will mainly be changing the bottom portion which is the pinmux control. We will need to add in a node which is the declaration and then the fsl which is the actual hex value that will be put into the registers.

- Examples
    
    Here is an example of a node for gpio
    
    ```bash
    &lsio_gpio1 {
    	pinctrl-names = "default";
    	pinctrl-0 = <&pinctrl_gpio1>;
    	
    	gpio-line-names =
            /* 0-7 */
     IMX8QXP_ADC_IN1_LSIO_GPIO1_IO09                 0x06000041 //internal pull down. Active High
                                    IMX8QXP_ADC_IN0_LSIO_GPIO1_IO10                 0x06000041
                                    IMX8QXP_ADC_IN2_LSIO_GPIO1_IO12                 0x06000041
                                    IMX8QXP_ADC_IN5_LSIO_GPIO1_IO13                 0x06000041
                                    IMX8QXP_ADC_IN4_LSIO_GPIO1_IO14                 0x06000041
                                    IMX8QXP_FLEXCAN1_RX_LSIO_GPIO1_IO17             0x06000041
                                    IMX8QXP_FLEXCAN1_TX_LSIO_GPIO1_IO18             0x06000041
                                    IMX8QXP_FLEXCAN2_RX_LSIO_GPIO1_IO19             0x06000041
                                    IMX8QXP_FLEXCAN2_TX_LSIO_GPIO1_IO20             0x06000041
                                    IMX8QXP_MIPI_DSI0_I2C0_SCL_LSIO_GPIO1_IO25      0x06000041
                                    IMX8QXP_MIPI_DSI0_I2C0_SDA_LSIO_GPIO1_IO26      0x06000041
                                    IMX8QXP_MIPI_DSI1_I2C0_SCL_LSIO_GPIO1_IO29      0x06000041
                                    IMX8QXP_MIPI_DSI1_I2C0_SDA_LSIO_GPIO1_IO30      0x06000041
    				
                            >;       "", "", "", "", "", "", "", "",
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
            /* 18-20 (you muxed them too, but didn’t list as hogs — still usable) */
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
    ```
    
    This node is created so we can have a custom pin group and we can identify it.
    
    Example of the fls pin control group
    
    ```bash
    pinctrl_gpio1: gpio1grp {
    	fsl,pins = <
    	  IMX8QXP_ADC_IN1_LSIO_GPIO1_IO09                 0x06000041 //internal pull down. Active High
    	  IMX8QXP_ADC_IN0_LSIO_GPIO1_IO10                 0x06000041
    	  IMX8QXP_ADC_IN2_LSIO_GPIO1_IO12                 0x06000041
    	  IMX8QXP_ADC_IN5_LSIO_GPIO1_IO13                 0x06000041
    	  IMX8QXP_ADC_IN4_LSIO_GPIO1_IO14                 0x06000041
    	  IMX8QXP_FLEXCAN1_RX_LSIO_GPIO1_IO17             0x06000041
    	  IMX8QXP_FLEXCAN1_TX_LSIO_GPIO1_IO18             0x06000041
    	  IMX8QXP_FLEXCAN2_RX_LSIO_GPIO1_IO19             0x06000041
    	  IMX8QXP_FLEXCAN2_TX_LSIO_GPIO1_IO20             0x06000041
    	  IMX8QXP_MIPI_DSI0_I2C0_SCL_LSIO_GPIO1_IO25      0x06000041
    	  IMX8QXP_MIPI_DSI0_I2C0_SDA_LSIO_GPIO1_IO26      0x06000041
    	  IMX8QXP_MIPI_DSI1_I2C0_SCL_LSIO_GPIO1_IO29      0x06000041
    	  IMX8QXP_MIPI_DSI1_I2C0_SDA_LSIO_GPIO1_IO30      0x06000041
    	
    	>;
    };
    ```
    
    This is the based off the [i.MX8 Processor Reference Manual](https://livecsupomona.sharepoint.com/sites/broncospacelab/Shared%20Documents/Forms/AllItems.aspx?id=%2Fsites%2Fbroncospacelab%2FShared%20Documents%2FSCALES%20%2D%20General%2FDocumentation%2FHardware%2Fi%2EMX%208X%2FIMX8DQXPRM%2Epdf&parent=%2Fsites%2Fbroncospacelab%2FShared%20Documents%2FSCALES%20%2D%20General%2FDocumentation%2FHardware%2Fi%2EMX%208X) (pg. 680) and using some of the same values that currently work on the development board.
    
    Here is another one for I2C
    
    ```bash
    # Node
    &i2c0 {
    	#address-cells = <1>;
    	#size-cells = <0>;
    	clock-frequency = <100000>;
    	pinctrl-names = "default";
    	pinctrl-0 = <&pinctrl_lpi2c0>;
    	status = "okay";
    	};
    	
    # Pin Control Group	
    	pinctrl_lpi2c0: lpi0cgrp {
    			fsl,pins = <
    				IMX8QXP_MIPI_CSI0_GPIO0_00_ADMA_I2C0_SCL	0x06000020 
    				IMX8QXP_MIPI_CSI0_GPIO0_01_ADMA_I2C0_SDA	0x06000020
    			>;
    		};
    ```
    

This will all be determined base on your pin muxed values that you can create on: https://pinmux.phytec.com/. Once that is done you can check to see if there is any conflicts with the `imx8qxp-phycore-som-emmc.dtsi` if there are you can remove them from the `dtsi`.

- Current `imx8xqxp-scales-mariner.dts`
    
    ```bash
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
    
- Current edited `imx8qxp-phycore-som-emmc.dtsi`
    
    ```bash
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
    
- Here is the original `imx8qxp-phytec-pcm-942.dts`
    
    ```bash
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
    		ti,fifo-de// SPDX-License-Identifier: GPL-2.0+
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
    };pth = <DP83867_PHYCR_FIFO_DEPTH_8_B_NIB>;
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
    

You can find more information you can refer to:

- [Modifying the BSP](https://docs.phytec.com/projects/yocto-phycore-imx8x/en/latest/developingwithyocto/modifyBSP.html#change-the-linux-kernel-device-tree)
- [linux-phytec-imx/include/dt-bindings/pinctrl/pads-imx8qxp.h](https://github.com/phytec/linux-phytec-imx/blob/v6.6.52-2.2.0-phy/include/dt-bindings/pinctrl/pads-imx8qxp.h)
- [System Controller Firmware 101 - Pad configuration service](https://community.nxp.com/t5/i-MX-Processors-Knowledge-Base/System-Controller-Firmware-101-Pad-configuration-service/ta-p/1124213#:~:text=MX8%20has%20three%20types%20of%20I/Os:%20*,1.8V%20/%203.3V%20I/Os%20Dual%20Voltage%20I/Os)

Once you have created your custom dts we need to edit the `MakeFile` to include our new dts as a device tree blob or `dtb`. 

You will need to add in our new dtb above the original dtb which is around line 534 in the file:

```bash
dtb-$(CONFIG_ARCH_MXC) += <YOUR_CUSTOM_DTS>.dtb
dtb-$(CONFIG_ARCH_MXC) += imx8qxp-phytec-pcm-942.dtb
```

This is where there are two ways of applying these changes. If you want to test this, make sure to save your changes and then exit. Once you have done that you need to force compile the image again because if you just bitbake it will not automatically detect the changes.

- Force compile
    
    ```bash
    cd $BUILDDIR
    bitbake phytec-headless-image -c compile --force
    ```
    
    Then once it has successfully recompiled you can:
    
    ```bash
    bitbake phytec-headless-image
    ```
    
- Create a Patch file
    
    Since we made multiple changes we need to change directories so we can create a patch that changes everything.
    
    ```bash
    cd ~/BSP-Yocto-NXP-i.MX8X-PD24.1.y/yocto/build/tmp-ampliphy-vendor/work-shared/phycore-imx8x-1/kernel-source/
    ```
    
    To create the patch we need to do a few different things. First we need to add everything to git, then we need to create a commit which will become our patch. Make sure to name the patch something makes sense and gives an idea of what it does. Then we need to format the commit into a patch and then we can exit and move that into our custom meta layer. The commands are as follows
    
    ```bash
    git add .
    ```
    
    ```bash
    git commit -m "message goes here"
    ```
    
    ```bash
    git format-patch -1
    ```
    

Now you can exit the development shell, but we will be going right back into it edit the boot loader.

### Boot Loader

```bash
bitbake virtual/bootloader -c devshell
```

There are 2 things we need to change in the `phycore_imx8x.h`, the fdt file and the rate at which the sd card can read. Here is the example of our current.

To edit the header file

```bash
vim include/configs/phycore_imx8x.h 
```

- `phycore_imx8x.h`
    
    ```bash
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
    

To edit the device tree include file

```bash
vim arch/arm/dts/imx8qxp-phycore-som.dtsi
```

- `imx8qxp-phycore-som.dtsi`
    
    ```bash
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
    

To create the patch we need to do a few different things. First we need to add everything to git, then we need to create a commit which will become our patch. Make sure to name the patch something makes sense and gives an idea of what it does. Then we need to format the commit into a patch and then we can exit and move that into our custom meta layer. The commands are as follows

```bash
git add .
```

```bash
git commit -m "message goes here"
```

```bash
git format-patch -1
```

Now that the patch is complete you can start working on creating your own layer.

## Creating your own layer

Once you are in the build environment you need to bitbake your own layer and register the created layer by:

```bash
bitbake-layers create-layer ~/BSP-Yocto-NXP-i.MX8X-PD24.1.y/yocto/sources/<your-layer-name>
```

Make sure to change it to the name you want it to be like meta-scales mariner. Then register it

```bash
bitbake-layers add-layer ~/BSP-Yocto-NXP-i.MX8X-PD24.1.y/yocto/sources/<meta-layer-name>
```

Now we can move on to adding the needed folders and files that we will be editing. You can stay in the build directory but it might be easier to navigate to your new layer

```bash
cd ~/BSP-Yocto-NXP-i.MX8X-PD24.1.y/yocto/sources/<meta-layer-name>
```

we will need to create a couple of different directories and some `.bbappend` files

```bash
mkdir -p recipes-kernel/linux/files conf/machine recipes-bsp/u-boot/files
touch recipes-kernel/linux/linux-phytec-imx_%.bbappend conf/layer.conf recipes-bsp/u-boot/u-boot-phytec-imx_%.bbappend
```

Now we need to copy over the machine configuration file. REMEMBER to change the name of the `.conf` file.

```bash
cp ~/BSP-Yocto-NXP-i.MX8X-PD24.1.y/yocto/sources/meta-phytec/conf/machine/phycore-imx8x-1.conf conf/machine/scales-mariner-1.conf
```

We need to edit the `conf/layer.conf` by changing the file priority to 999

```bash
BBFILE_PRIORITY_<meta-layer-name> = "999"
```

Need to edit the file

```bash
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

Moving out of the `conf` folder we are going to set the structure up for all the`.bbappend` files

```bash
FILESEXTRAPATHS:prepend := "${THISDIR}/files:"
SRC_URI += "file://<YOUR STABLE PATCH>.patch"

COMPATIBLE_MACHINE:append = "|phycore-imx8x-1|scales-mariner-1"
```

Now we need to move over our patch files which will be located in:

```bash
~/BSP-Yocto-NXP-i.MX8X-PD24.1.y/yocto/build/tmp-ampliphy-vendor/work-shared/phycore-imx8x-1/kernel-source/
```

and

```bash
~/BSP-Yocto-NXP-i.MX8X-PD24.1.y/yocto/build/tmp-ampliphy-vendor/work/phycore_imx8x_1-phytec-linux/u-boot-phytec-imx/2024.04-2.2.0-phy18/git
```

so it would look something like

```bash
cp ~/BSP-Yocto-NXP-i.MX8X-PD24.1.y/yocto/build/tmp-ampliphy-vendor/work-shared/phycore-imx8x-1/kernel-source/<YOUR_PATCH>.patch ~/BSP-Yocto-NXP-i.MX8X-PD24.1.y/yocto/sources/meta-scales-mariner/recipes-kernel/linux/files
```

```bash
cp ~/BSP-Yocto-NXP-i.MX8X-PD24.1.y/yocto/build/tmp-ampliphy-vendor/work/phycore_imx8x_1-phytec-linux/u-boot-phytec-imx/2024.04-2.2.0-phy22/git/<YOUR_PATCH>.patch ~/BSP-Yocto-NXP-i.MX8X-PD24.1.y/yocto/sources/meta-scales-mariner/recipes-bsp/u-boot/files/
```

Next you need to edit your `.bbapend` file to include the patch.

- `u-boot-phtect-imx_%.bbappend`
    
    ```bash
    FILESEXTRAPATHS:prepend := "${THISDIR}/files:"
    
    SRC_URI += "file://0001-Creating-the-U-boot-needed-for-scales-board.patch"
    
    COMPATIBLE_MACHINE .= "|scales-mariner-1"
    ```
    
- `linux-phytec-imx_%.bbappend`
    
    ```bash
    FILESEXTRAPATHS:prepend := "${THISDIR}/files:"
    
    SRC_URI += "file://0001-Creating-the-scale-mariner-device-tree.patch"
    
    COMPATIBLE_MACHINE .= "|scales-mariner-1"
    ```
    

### Optional layer

You can also clone a custom layer by navigating to the `~/BSP-Yocto-NXP-i.MX8X-PD24.1.y/yocto/sources/` make sure you are in the correct branch

## Building your custom layer

We are going to be bitbaking the new image but before we do we need to change the `conf/local.conf` to have our custom machine by adding:

```bash
MACHINE ?= "<CUSTOME MACHINE NAME>"
#MACHINE ?= "phycore-imx8x-1"
```

Now we can compile and bitbake to get our image.

```bash
bitbake phytec-headless-image -c cleansstate
```

Once that finishes you can then 

```bash
bitbake phytec-headless-image
```

Once the image is done you can also create the sdk for cross compilation or just get it onto an SD card just like the normal process.

---

# Leviathan 1A Carrier Board BSP

(Place carrier-board-specific notes, DTS overrides, and recipes in your custom layer under `meta-leviathan` or similarly named layer.)

// ...existing code...