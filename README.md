# ESPRESSObin Ultra Bootloader Builder

This is inspired by the [mox-boot-builder](https://gitlab.nic.cz/turris/mox-boot-builder) project. The [Turris MOX](https://www.turris.com/en/mox/overview/) shares the same Marvell Armada 3720 SoC as the [Globalscale ESPRESSObin Ultra](https://globalscaletechnologies.com/product/espressobin-ultra/).

This started as a fork of the bootloader build scripts from [Globalscale's repositories](https://github.com/globalscaletechnologies). It heavily references [their build instructions](https://espressobin.net/espressobin-ultra-build-instruction/) and the ARM Trusted Firmware [build instructions](https://trustedfirmware-a.readthedocs.io/en/v2.10/plat/marvell/armada/build.html). Major changes:

* Upgrade build host from Ubuntu 18.04 to 20.04 (22.04 does not currently produce stable images)
* Move from Globalscale's mv-ddr-marvell repository to Marvell's
* Move from Globalscale's ARM Trusted Firmware repo to ARM's and upgrade to v2.10

## Build Host

The directions here are for a fully upgraded Ubuntu Server 20.04 virtual machine using a base image from [osboxes.org](https://www.osboxes.org/ubuntu-server/#ubuntu-server-20-04-4-vbox). Upgrade all packages, resolve any issues, and restart the VM which should leave you with Ubuntu 20.04.6.

__Note__: AFAIK the build process for osboxes images is not open source. If that's a problem in your scenario, install Ubuntu Server 20.04 from scratch. It's also not required to use Ubuntu--any Linux distro should work provided the required build dependencies are satisfied and the _same version_ is used as that in Ubuntu 20.04.6.

Make sure all build dependencies are installed:
```
sudo apt install make binutils build-essential gcc g++ \
bash patch gzip bzip2 perl tar cpio zlib1g-dev \
gawk ccache gettext libssl-dev libncurses5 minicom git \
bison flex device-tree-compiler gcc-arm-linux-gnueabi
```
Many of these are likely already installed in Ubuntu, but this is based on Globalscale's guide. It would be nice to specify exact packages and versions at some point so that any properly set up Linux distribution can run the build script.

Note: While the Armada 3720 uses 64-bit ARMv8 processors, `gcc-arm-linux-gnueabi` provides a 32-bit ARM cross-compiler which is used to compile a part of the firmware meant to run on an internal Cortex-M3 coprocessor.

The 64-bit ARM (aarch64) cross-compiler we get from Linaro per Globalscale. We extract it into a directory named `toolchain`:
```
wget https://releases.linaro.org/components/toolchain/binaries/7.3-2018.05/aarch64-linux-gnu/gcc-linaro-7.3.1-2018.05-x86_64_aarch64-linux-gnu.tar.xz
mkdir toolchain
tar -xvf gcc-linaro-7.3.1-2018.05-x86_64_aarch64-linux-gnu.tar.xz -C toolchain/
```
This toolchain is fairly old and likely a limiting factor in the ability to upgrade the build host OS.

## Download Repositories

__IMPORTANT:__ if you follow the build directions on the manufacturer's page and not the directions below you will encounter errors. In particular you want to use the `-gti` suffixed branches across all of the manufacturer's repos and not just some of them.

```
git clone https://github.com/globalscaletechnologies/A3700-utils-marvell.git -b A3700_utils-armada-18.12.0-gti
git clone https://github.com/MarvellEmbeddedProcessors/mv-ddr-marvell
git clone https://github.com/globalscaletechnologies/u-boot-marvell.git -b u-boot-2018.03-armada-18.12-gti
```
Note that it seems in some cases git is called within the builds for at least two of the above projects. As a result, they need to be git repos and not just pure source code. Source: https://trustedfirmware-a.readthedocs.io/en/latest/plat/marvell/index.html

## Patching A3700 Utils

Globalscale added support for building "secondary" images in their fork of A3700 utils in the `buildtim.sh` script. That feature is not supported by upstream ARM Trusted Firmware. The `a3700.patch` file can be applied to the A3700-utils-marvell repo to remove that feature and allow builds to complete without error.

## Patching U-Boot
U-Boot will fail to compile on newer GCC compilers unless patched. If that happens, see [here](https://github.com/BPI-SINOVOIP/BPI-M4-bsp/issues/4#issuecomment-1296184876) for a fix.

## Configuring CPU Clock Speed

In the `build_bootloader()` function, there is a `cpu_speed` variable. This variable is the CPU clock speed in MHz and can take the integer values 800, 1000, or 1200. While the ESPRESSObin Ultra has mixed advertising claiming speeds either up to 1 or 1.2Ghz,the [Armbian forums](https://www.armbian.com/espressobin/) suggest a 1.2Ghz clock speed is the source of stability problems in ESPRESSObin V7s. It's unclear if this is also a problem for the ESPRESSObin Ultra (seems likely), nor whether it is a problem that needs to be resolved in firmware.

The [Linux kernel has explicitly disabled 1.2Ghz](https://github.com/torvalds/linux/blob/master/drivers/cpufreq/armada-37xx-cpufreq.c#L106-L109) as a speed for this device, and the max speed they seem to allow via dynamic power management is 1GHz.

To make things even more confusing, the factory bootloader image runs at 800MHz, and when Linux boots it does not seem able to scale the clock speed up. Meanwhile build instructions on the manufacturer's website, as well as the repo I forked to start this project, build 1Ghz and 1.2Ghz versions.

In practice 1GHz builds are stable and Linux won't run any faster so 1Ghz is what I use.

## Building
```
source build.sh
build_bootloader
```
Output is saved in the `build` directory.

## Flashing
Copy the `flash.bin` file from the directory with the build timestamp that you want to flash onto a USB flash drive and run `bubt flash.bin spi usb` from u-boot to flash. If your device can't make it to the u-boot prompt, you'll need to boot a known stable image via UART and then use bubt u-boot to flash a stable image. Don't use the WtpDownloader tool from Marvell. It sucks. Use [mox-imager](https://gitlab.nic.cz/turris/mox-imager) instead.
