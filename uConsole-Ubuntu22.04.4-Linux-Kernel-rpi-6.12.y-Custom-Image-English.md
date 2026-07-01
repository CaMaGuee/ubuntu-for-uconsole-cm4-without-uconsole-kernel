# uConsole CM4 + Ubuntu 22.04 + Linux 6.12 Kernel & Wi‑Fi Full Tutorial

This document starts from a completely fresh VM and aims to reproduce the exact same OS environment currently used on the uConsole.

- Base OS: Ubuntu 22.04.4 preinstalled desktop for Raspberry Pi (arm64 + raspi).
- Kernel: Based on the Raspberry Pi `rpi-6.12.y` branch, with local version `-camaguee-uconsole-6.12`.
- uConsole-specific support:
  - LCD panel: `panel-cwu50` (for the CM4 model, 720×1280, MIPI DSI).
  - Backlight: `backlight-ocp8178`.
  - PMU/Battery: AXP20x (`axp20x_battery`, `axp20x_regulator`, and related components).
  - Overlays: `clockworkpi-uconsole`, `clockworkpi-custom-battery`, `clockworkpi-devterm-*`, `audremap`.
- Wi‑Fi: Realtek USB `0bda:b831` (RTL8851BU/RTL8831BU family), supported through the `rtw89` Git driver.

If every step is followed in order from beginning to end, the result is the same kernel, environment, and driver setup as the current uConsole system.

---

## STEP 0. Workflow Overview

1. Prepare the VM: Debian/Ubuntu x86_64 VM with at least 120GB of disk space.
2. Set up the host environment: kernel build tools, arm64 cross compiler, and multi-architecture support.
3. Download and mount the Ubuntu Raspberry Pi image.
4. Clone the Raspberry Pi Linux `rpi-6.12.y` source tree.
5. Generate `.config` using the Ubuntu Raspberry Pi kernel config as the base.
6. Adjust settings to avoid build errors (`SYSTEM_TRUSTED_KEYS`, etc.) and set the local version.
7. Apply uConsole-specific panel, backlight, PMU, and overlay patches.
8. Build kernel `.deb` packages with `make bindeb-pkg`.
9. Install the new kernel and generate initramfs inside the Ubuntu rootfs using chroot.
10. Copy the new kernel and initrd into the `boot/firmware` partition, replace `config.txt`, and deploy overlays.
11. Unmount the image, flash it to microSD, and verify the first boot on the uConsole.
12. Build, install, and test the RTL8851BU Wi‑Fi driver (`rtw89`) inside the uConsole.

Each step contains commands that can be copied and pasted directly.

---

## STEP 1. VM Preparation

### 1.1 VM Specifications

- CPU: 4 cores or more. 8 cores or more is preferable.
- RAM: 4GB or more. 8GB or more is preferable.
- Disk: 120GB or more is recommended. This accounts for kernel source, build artifacts, images, and logs.
- Host OS: Debian 12/13 or Ubuntu 22.04/24.04 x86_64.

### 1.2 When Disk Expansion Is Needed

If the existing VM disk is too small, expand the root partition.

```bash
lsblk
sudo parted /dev/sda

(parted) print           # Check current partition layout
(parted) rm 5            # Remove logical swap partition if present
(parted) rm 2            # Remove extended partition if present
(parted) resizepart 1 100%
(parted) quit

sudo resize2fs /dev/sda1

df -h
```

More than 100GB of free space on `/dev/sda1` is sufficient.

---

## STEP 2. Host Environment Setup (Debian/Ubuntu x86_64)

Create the working root directory.

```bash
mkdir -p ~/uconsole-kernel-work/src
cd ~/uconsole-kernel-work
```

### 2.1 Install Required Packages

```bash
sudo apt update && sudo apt upgrade -y

sudo apt install -y   git build-essential bc bison flex   libssl-dev libelf-dev rsync   debhelper debhelper-compat   gcc-aarch64-linux-gnu binutils-aarch64-linux-gnu   kmod cpio python3   debootstrap qemu-user-static
```

### 2.2 Add arm64 Multi-Architecture Support

`bindeb-pkg` builds arm64 packages, so arm64 architecture support and related libraries must be added.

```bash
sudo dpkg --add-architecture arm64
sudo apt update
sudo apt install -y libssl-dev:arm64 libelf-dev:arm64
```

---

## STEP 3. Download and Mount the Ubuntu 22.04 Raspberry Pi Image

### 3.1 Download and Extract the Image

```bash
cd ~/uconsole-kernel-work

wget https://cdimage.ubuntu.com/releases/22.04/release/ubuntu-22.04.4-preinstalled-desktop-arm64+raspi.img.xz

xz -d ubuntu-22.04.4-preinstalled-desktop-arm64+raspi.img.xz
```

### 3.2 Attach the Loop Device and Mount It

```bash
sudo losetup -P /dev/loop0 ubuntu-22.04.4-preinstalled-desktop-arm64+raspi.img

lsblk /dev/loop0

sudo mkdir -p /mnt/ubuntu/rootfs
sudo mount /dev/loop0p2 /mnt/ubuntu/rootfs

sudo mkdir -p /mnt/ubuntu/rootfs/boot/firmware
sudo mount /dev/loop0p1 /mnt/ubuntu/rootfs/boot/firmware

df -h | grep loop
```

### 3.3 Back Up the Default Ubuntu Kernel Config

```bash
ls /mnt/ubuntu/rootfs/boot/
cp /mnt/ubuntu/rootfs/boot/config-6.8.0-1047-raspi   ~/uconsole-kernel-work/config-ubuntu-raspi
```

---

## STEP 4. Prepare the Raspberry Pi Kernel Source (rpi-6.12.y)

### 4.1 Clone rpi-linux

```bash
cd ~/uconsole-kernel-work/src

git clone --depth=1 --branch rpi-6.12.y   https://github.com/raspberrypi/linux.git rpi-linux

cd rpi-linux
```

`rpi-6.12.y` is the Raspberry Pi Foundation maintained 6.12 branch.

### 4.2 Generate .config from the Ubuntu Config

```bash
cp ~/uconsole-kernel-work/config-ubuntu-raspi .config

make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- olddefconfig
```

---

## STEP 5. Build Error Prevention Settings + localversion

### 5.1 Clear SYSTEM_TRUSTED_KEYS / REVOCATION_KEYS

```bash
cd ~/uconsole-kernel-work/src/rpi-linux

sed -i 's|CONFIG_SYSTEM_TRUSTED_KEYS=.*|CONFIG_SYSTEM_TRUSTED_KEYS=""|' .config
sed -i 's|CONFIG_SYSTEM_REVOCATION_KEYS=.*|CONFIG_SYSTEM_REVOCATION_KEYS=""|' .config

grep 'SYSTEM_TRUSTED_KEYS\|SYSTEM_REVOCATION_KEYS' .config

make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- olddefconfig
```

### 5.2 Set localversion

```bash
echo "-camaguee-uconsole-6.12" > localversion
cat localversion
```

---

## STEP 6. Apply uConsole-Specific Panel / Backlight / PMU / Overlay Patches

```bash
cd ~/uconsole-kernel-work/src/rpi-linux
BASE="https://raw.githubusercontent.com/ak-rex/ClockworkPi-linux/rpi-6.12.y"
```

### 6.1 Add the panel-cwu50 Driver

```bash
wget -O drivers/gpu/drm/panel/panel-cwu50.c   "${BASE}/drivers/gpu/drm/panel/panel-cwu50.c"

wget -O drivers/gpu/drm/panel/panel-cwu50-cm3.c   "${BASE}/drivers/gpu/drm/panel/panel-cwu50-cm3.c"

grep -q 'DRM_PANEL_CWU50' drivers/gpu/drm/panel/Kconfig || cat >> drivers/gpu/drm/panel/Kconfig << 'EOF_P'

config DRM_PANEL_CWU50
	tristate "CWU50 panel"
	depends on OF
	depends on DRM_MIPI_DSI
	depends on BACKLIGHT_CLASS_DEVICE
	help
	  Say Y here if you want to enable support for CWU50 panel.

config DRM_PANEL_CWU50_CM3
	tristate "CWU50_CM3 panel"
	depends on OF
	depends on DRM_MIPI_DSI
	depends on BACKLIGHT_CLASS_DEVICE
	help
	  Say Y here if you want to enable support for CWU50_CM3 panel.
EOF_P

grep -q 'panel-cwu50' drivers/gpu/drm/panel/Makefile || cat >> drivers/gpu/drm/panel/Makefile << 'EOF_PM'
obj-$(CONFIG_DRM_PANEL_CWU50) += panel-cwu50.o
obj-$(CONFIG_DRM_PANEL_CWU50_CM3) += panel-cwu50-cm3.o
EOF_PM

scripts/config --module CONFIG_DRM_PANEL_CWU50
scripts/config --module CONFIG_DRM_PANEL_CWU50_CM3
```

### 6.2 Add the backlight-ocp8178 Driver

```bash
wget -O drivers/video/backlight/ocp8178_bl.c   "${BASE}/drivers/video/backlight/ocp8178_bl.c"

sed -i '/^endif # BACKLIGHT_CLASS_DEVICE/i config BACKLIGHT_OCP8178
	tristate "OCP8178 Backlight Driver"
	depends on GPIOLIB
	help
	  If you have an OCP8178, say Y to enable the backlight driver.
'   drivers/video/backlight/Kconfig

grep -q 'ocp8178_bl' drivers/video/backlight/Makefile ||   echo 'obj-$(CONFIG_BACKLIGHT_OCP8178)		+= ocp8178_bl.o' >> drivers/video/backlight/Makefile

scripts/config --module CONFIG_BACKLIGHT_OCP8178
```

### 6.3 Replace AXP20x PMU / Battery Related Files

```bash
wget -O drivers/power/supply/axp20x_battery.c   "${BASE}/drivers/power/supply/axp20x_battery.c"
wget -O drivers/mfd/axp20x.c   "${BASE}/drivers/mfd/axp20x.c"
wget -O include/linux/mfd/axp20x.h   "${BASE}/include/linux/mfd/axp20x.h"

wget -O drivers/mfd/axp20x-i2c.c    "${BASE}/drivers/mfd/axp20x-i2c.c"
wget -O drivers/mfd/axp20x-rsb.c    "${BASE}/drivers/mfd/axp20x-rsb.c"
wget -O drivers/iio/adc/axp20x_adc.c "${BASE}/drivers/iio/adc/axp20x_adc.c"
wget -O drivers/input/misc/axp20x-pek.c "${BASE}/drivers/input/misc/axp20x-pek.c"
wget -O drivers/power/supply/axp20x_ac_power.c   "${BASE}/drivers/power/supply/axp20x_ac_power.c"
wget -O drivers/power/supply/axp20x_usb_power.c   "${BASE}/drivers/power/supply/axp20x_usb_power.c"
wget -O drivers/regulator/axp20x-regulator.c   "${BASE}/drivers/regulator/axp20x-regulator.c"
```

### 6.4 Add clockworkpi / uconsole / devterm / custom-battery Overlays

```bash
ODIR="arch/arm/boot/dts/overlays"

wget -O ${ODIR}/clockworkpi-uconsole-overlay.dts   "${BASE}/arch/arm/boot/dts/overlays/clockworkpi-uconsole-overlay.dts"
wget -O ${ODIR}/clockworkpi-uconsole-cm3-overlay.dts   "${BASE}/arch/arm/boot/dts/overlays/clockworkpi-uconsole-cm3-overlay.dts"
wget -O ${ODIR}/clockworkpi-uconsole-cm5-overlay.dts   "${BASE}/arch/arm/boot/dts/overlays/clockworkpi-uconsole-cm5-overlay.dts"

wget -O ${ODIR}/clockworkpi-custom-battery-overlay.dts   "${BASE}/arch/arm/boot/dts/overlays/clockworkpi-custom-battery-overlay.dts"

wget -O ${ODIR}/clockworkpi-devterm-overlay.dts   "${BASE}/arch/arm/boot/dts/overlays/clockworkpi-devterm-overlay.dts"
wget -O ${ODIR}/clockworkpi-devterm-cm5-overlay.dts   "${BASE}/arch/arm/boot/dts/overlays/clockworkpi-devterm-cm5-overlay.dts"

grep -q 'clockworkpi-uconsole' ${ODIR}/Makefile || cat >> ${ODIR}/Makefile << 'EOF_OV'
dtbo-$(CONFIG_ARCH_BCM2835) += clockworkpi-uconsole.dtbo
dtbo-$(CONFIG_ARCH_BCM2835) += clockworkpi-uconsole-cm3.dtbo
dtbo-$(CONFIG_ARCH_BCM2835) += clockworkpi-uconsole-cm5.dtbo
dtbo-$(CONFIG_ARCH_BCM2835) += clockworkpi-custom-battery.dtbo
dtbo-$(CONFIG_ARCH_BCM2835) += clockworkpi-devterm.dtbo
dtbo-$(CONFIG_ARCH_BCM2835) += clockworkpi-devterm-cm5.dtbo
EOF_OV
```

### 6.5 Refresh the Final .config

```bash
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- olddefconfig

grep -E 'DRM_PANEL_CWU50|BACKLIGHT_OCP8178' .config
```

---

## STEP 7. Build Kernel .deb Packages (bindeb-pkg)

```bash
cd ~/uconsole-kernel-work/src/rpi-linux

make -j"$(nproc)" ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- bindeb-pkg
```

After the build, check the generated packages.

```bash
cd ~/uconsole-kernel-work
find . -maxdepth 3 -name "*.deb" -ls
```

Four `.deb` files such as `linux-image-6.12.93-camaguee-uconsole-6.12+_..._arm64.deb` are expected.

---

## STEP 8. Install the Kernel into the Ubuntu rootfs (chroot)

```bash
cd ~/uconsole-kernel-work

sudo cp src/linux-image-6.12.93-camaguee-uconsole-6.12+_*.deb         src/linux-headers-6.12.93-camaguee-uconsole-6.12+_*.deb         src/linux-libc-dev_6.12.93-*_arm64.deb         /mnt/ubuntu/rootfs/tmp/

sudo mount -t proc none /mnt/ubuntu/rootfs/proc
sudo mount -t sysfs none /mnt/ubuntu/rootfs/sys
sudo mount --bind /dev /mnt/ubuntu/rootfs/dev
sudo mount --bind /dev/pts /mnt/ubuntu/rootfs/dev/pts

sudo chroot /mnt/ubuntu/rootfs /bin/bash -c "
  cd /tmp &&   dpkg -i linux-image-6.12.93-camaguee-uconsole-6.12+_*.deb &&   dpkg -i linux-headers-6.12.93-camaguee-uconsole-6.12+_*.deb &&   dpkg -i linux-libc-dev_6.12.93-*_arm64.deb
"

sudo chroot /mnt/ubuntu/rootfs /bin/bash -c   "update-initramfs -u -k 6.12.93-camaguee-uconsole-6.12+"

ls /mnt/ubuntu/rootfs/boot/vmlinuz-6.12.93-camaguee-uconsole-6.12+
ls /mnt/ubuntu/rootfs/boot/initrd.img-6.12.93-camaguee-uconsole-6.12+
```

---

## STEP 9. Copy the Kernel and initrd into boot/firmware

```bash
sudo cp /mnt/ubuntu/rootfs/boot/vmlinuz-6.12.93-camaguee-uconsole-6.12+         /mnt/ubuntu/rootfs/boot/firmware/vmlinuz-6.12.93-camaguee-uconsole-6.12+

sudo cp /mnt/ubuntu/rootfs/boot/initrd.img-6.12.93-camaguee-uconsole-6.12+         /mnt/ubuntu/rootfs/boot/firmware/initrd.img-6.12.93-camaguee-uconsole-6.12+

ls -lh /mnt/ubuntu/rootfs/boot/firmware/vmlinuz-6.12.93-camaguee-uconsole-6.12+
ls -lh /mnt/ubuntu/rootfs/boot/firmware/initrd.img-6.12.93-camaguee-uconsole-6.12+
```

---

## STEP 10. Replace config.txt with the uConsole-Specific Version

```bash
sudo cp /mnt/ubuntu/rootfs/boot/firmware/config.txt         /mnt/ubuntu/rootfs/boot/firmware/config.txt.bak

sudo tee /mnt/ubuntu/rootfs/boot/firmware/config.txt > /dev/null << 'EOF_CFG'
[all]
arm_64bit=1
kernel=vmlinuz-6.12.93-camaguee-uconsole-6.12+
initramfs initrd.img-6.12.93-camaguee-uconsole-6.12+ followkernel

ignore_lcd=1
display_auto_detect=0
camera_auto_detect=0

dtoverlay=vc4-kms-v3d
disable_fw_kms_setup=1

dtoverlay=dwc2,dr_mode=host

dtoverlay=clockworkpi-uconsole
dtoverlay=devterm-pmu
dtoverlay=devterm-panel-uc
dtoverlay=devterm-misc
dtoverlay=audremap,pins_12_13

dtparam=spi=on
dtparam=i2c_arm=on
dtparam=audio=on
gpio=10=ip,np

gpu_mem=64
disable_overscan=1

[pi4]
max_framebuffers=2
arm_boost=1

[cm4]
dtoverlay=vc4-kms-v3d-pi4,cma-384
dtoverlay=dwc2,dr_mode=host
EOF_CFG

cat /mnt/ubuntu/rootfs/boot/firmware/config.txt
```

Optionally apply custom battery capacity values.

```text
dtoverlay=clockworkpi-custom-battery,charge_full_design_uah=6700000,energy_full_design_uwh=24790000
```

---

## STEP 11. Check Whether overlays(.dtbo) Exist

```bash
ls /mnt/ubuntu/rootfs/boot/firmware/overlays/ | grep -E 'clockworkpi|devterm|audremap'
```

Run the following if needed.

```bash
sudo cp ~/uconsole-kernel-work/src/rpi-linux/arch/arm/boot/dts/overlays/clockworkpi*.dtbo         /mnt/ubuntu/rootfs/boot/firmware/overlays/ 2>/dev/null || true

sudo cp ~/uconsole-kernel-work/src/rpi-linux/arch/arm/boot/dts/overlays/audremap*.dtbo         /mnt/ubuntu/rootfs/boot/firmware/overlays/ 2>/dev/null || true
```

---

## STEP 12. Check cmdline.txt

```bash
cat /mnt/ubuntu/rootfs/boot/firmware/cmdline.txt
```

Remove `quiet splash` if more boot-time debug logs are needed.

---

## STEP 13. Unmount and Detach the Loop Device

```bash
sudo umount /mnt/ubuntu/rootfs/dev/pts 2>/dev/null || true
sudo umount /mnt/ubuntu/rootfs/dev 2>/dev/null || true
sudo umount /mnt/ubuntu/rootfs/sys 2>/dev/null || true
sudo umount /mnt/ubuntu/rootfs/proc 2>/dev/null || true

sudo umount /mnt/ubuntu/rootfs/boot/firmware
sudo umount /mnt/ubuntu/rootfs
sudo losetup -d /dev/loop0
```

---

## STEP 14. Flash the Image to microSD

```bash
cd ~/uconsole-kernel-work
lsblk   # Confirm which device is the microSD, for example /dev/sdX

sudo dd if=ubuntu-22.04.4-preinstalled-desktop-arm64+raspi.img         of=/dev/sdX bs=4M status=progress conv=fsync

sudo sync
```

---

## STEP 15. Verify the First Boot on the uConsole

Insert the microSD into the uConsole, boot it, and then run the following locally or over SSH.

```bash
uname -r
```

Expected result:

```text
6.12.93-camaguee-uconsole-6.12+
```

Also verify the following.

```bash
dmesg | grep -i "clockworkpi\|devterm\|cwu50\|ocp8178"
lsmod | grep -E "cwu50|ocp8178"

cat /sys/class/power_supply/axp20x-battery/capacity
cat /sys/class/power_supply/axp20x-battery/voltage_now
```

---

## STEP 16. Install the RTL8851BU Wi‑Fi Driver (0bda:b831) Using rtw89

Perform this part inside the uConsole.

```bash
lsusb   # Check for the 0bda:b831 adapter
```

### 16.1 Install Dependencies

```bash
sudo apt update
sudo apt install -y build-essential dkms git iw
```

### 16.2 Use gcc 14 (Recommended)

```bash
sudo apt install -y gcc-14 g++-14

sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-14 14
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-13 13
sudo update-alternatives --set gcc /usr/bin/gcc-14

gcc --version
```

### 16.3 Clone and Build the rtw89 Driver

```bash
mkdir -p ~/src
cd ~/src

git clone https://github.com/morrownr/rtw89.git --depth=1
cd rtw89

make clean modules
sudo make install
sudo make install_fw
```

### 16.4 Load the Modules and Check the Interface

```bash
sudo modprobe rtw89_core_git
sudo modprobe rtw89_usb_git
sudo modprobe rtw89_8851bu_git

lsmod | grep rtw89
ip link show
```

If necessary, reboot once and check `ip link show` again.

---

## STEP 17. Connect to Wi‑Fi (NetworkManager)

```bash
nmcli device wifi list
nmcli device wifi connect "SSID" password "PASSWORD" ifname wlan0
```

After connection, verify the following.

```bash
ip addr show wlan0
ping -c 3 8.8.8.8
```

---

## STEP 18. Final Checklist

1. `uname -r` → `6.12.93-camaguee-uconsole-6.12+`.
2. The LCD displays correctly. Console or GUI access is available.
3. `panel_cwu50` and `ocp8178_bl` modules are loaded.
4. `axp20x-battery` capacity and voltage are reported correctly.
5. `lsusb` shows the `0bda:b831` Realtek adapter.
6. `lsmod | grep rtw89` shows the three Git-based modules: `core_git`, `usb_git`, and `8851bu_git`.
7. `ip link show` lists `wlan0` or a `wlx...` interface.
8. `nmcli` can connect to the access point.

If every item above is satisfied, this document alone is enough to reconstruct the same uConsole OS environment at any time.
