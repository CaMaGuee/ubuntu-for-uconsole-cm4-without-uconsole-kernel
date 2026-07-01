# uConsole CM4 + Ubuntu 22.04 + Linux 6.12 커널 & Wi‑Fi 전체 튜토리얼

이 문서는 완전히 새로 설치한 VM 상태에서 시작해서, 지금 사용 중인 uConsole 환경과 동일한 OS를 재현하는 것을 목표로 한다.

- 베이스 OS: Ubuntu 22.04.4 preinstalled desktop for Raspberry Pi (arm64 + raspi).
- 커널: Raspberry Pi `rpi-6.12.y` 브랜치 기반, 로컬 버전 `-camaguee-uconsole-6.12`.
- uConsole 전용 지원:
  - LCD 패널: `panel-cwu50` (CM4 모델용, 720×1280, MIPI DSI).
  - 백라이트: `backlight-ocp8178`.
  - PMU/배터리: AXP20x (`axp20x_battery`, `axp20x_regulator` 등).
  - Overlays: `clockworkpi-uconsole`, `clockworkpi-custom-battery`, `clockworkpi-devterm-*`, `audremap`.
- Wi‑Fi: Realtek USB `0bda:b831` (RTL8851BU/RTL8831BU 계열), `rtw89` Git 드라이버로 지원.

처음부터 끝까지 모든 단계를 순서대로 따라 하면, 지금 이 uConsole과 완전히 같은 커널, 환경, 드라이버를 만들 수 있다.

---

## STEP 0. 전체 흐름 개요

1. VM 준비: Debian/Ubuntu x86_64 VM, 디스크 120GB 이상 확보.
2. 호스트 환경 세팅: 커널 빌드 도구, arm64 크로스 컴파일러, 멀티 아키텍처 설정.
3. Ubuntu 라즈파이 이미지 다운로드 및 마운트.
4. Raspberry Pi Linux `rpi-6.12.y` 소스 클론.
5. Ubuntu 라즈파이 커널 config를 베이스로 `.config` 생성.
6. 빌드 오류 방지용 설정 수정 (`SYSTEM_TRUSTED_KEYS` 등) + localversion 설정.
7. uConsole 전용 패널, 백라이트, PMU, overlay 패치 적용.
8. `make bindeb-pkg`로 커널 `.deb` 빌드.
9. Ubuntu rootfs(chroot)에서 새 커널 설치 및 initramfs 생성.
10. `boot/firmware` 파티션에 새 커널, initrd 복사 + `config.txt` 교체 + overlay 배포.
11. 이미지 언마운트 → microSD에 플래시 → uConsole에서 첫 부팅 검증.
12. uConsole 내부에서 RTL8851BU Wi‑Fi 드라이버(`rtw89`) 빌드, 설치, 접속 테스트.

각 STEP은 그대로 복붙해서 실행하면 되는 명령만 포함한다.

---

## STEP 1. VM 준비

### 1.1 VM 스펙

- CPU: 4코어 이상. 가능하면 8코어 이상.
- RAM: 4GB 이상. 가능하면 8GB 이상.
- 디스크: 120GB 이상 권장. 커널 소스, 빌드 아티팩트, 이미지, 로그 등을 고려한다.
- 호스트 OS: Debian 12/13 또는 Ubuntu 22.04/24.04 x86_64.

### 1.2 디스크 확장이 필요할 때

이미 생성된 VM 디스크가 작다면 루트 파티션을 늘린다.

```bash
lsblk
sudo parted /dev/sda

(parted) print           # 현재 파티션 구조 확인
(parted) rm 5            # 논리 스왑 파티션 제거 (있으면)
(parted) rm 2            # extended 파티션 제거 (있으면)
(parted) resizepart 1 100%
(parted) quit

sudo resize2fs /dev/sda1

df -h
```

`/dev/sda1`에 100GB 이상 여유가 보이면 충분하다.

---

## STEP 2. 호스트 환경 세팅 (Debian/Ubuntu x86_64)

작업 루트 디렉터리를 만든다.

```bash
mkdir -p ~/uconsole-kernel-work/src
cd ~/uconsole-kernel-work
```

### 2.1 기본 패키지 설치

```bash
sudo apt update && sudo apt upgrade -y

sudo apt install -y \
  git build-essential bc bison flex \
  libssl-dev libelf-dev rsync \
  debhelper debhelper-compat \
  gcc-aarch64-linux-gnu binutils-aarch64-linux-gnu \
  kmod cpio python3 \
  debootstrap qemu-user-static
```

### 2.2 arm64 멀티 아키텍처 추가

`bindeb-pkg`는 arm64 패키지를 만들기 때문에 arm64 아키텍처와 해당 라이브러리를 추가한다.

```bash
sudo dpkg --add-architecture arm64
sudo apt update
sudo apt install -y libssl-dev:arm64 libelf-dev:arm64
```

---

## STEP 3. Ubuntu 22.04 라즈파이 이미지 다운로드 & 마운트

### 3.1 이미지 다운로드 및 압축 해제

```bash
cd ~/uconsole-kernel-work

wget https://cdimage.ubuntu.com/releases/22.04/release/ubuntu-22.04.4-preinstalled-desktop-arm64+raspi.img.xz

xz -d ubuntu-22.04.4-preinstalled-desktop-arm64+raspi.img.xz
```

### 3.2 loop 디바이스 연결 및 마운트

```bash
sudo losetup -P /dev/loop0 ubuntu-22.04.4-preinstalled-desktop-arm64+raspi.img

lsblk /dev/loop0

sudo mkdir -p /mnt/ubuntu/rootfs
sudo mount /dev/loop0p2 /mnt/ubuntu/rootfs

sudo mkdir -p /mnt/ubuntu/rootfs/boot/firmware
sudo mount /dev/loop0p1 /mnt/ubuntu/rootfs/boot/firmware

df -h | grep loop
```

### 3.3 Ubuntu 기본 커널 config 백업

```bash
ls /mnt/ubuntu/rootfs/boot/
cp /mnt/ubuntu/rootfs/boot/config-6.8.0-1047-raspi \
  ~/uconsole-kernel-work/config-ubuntu-raspi
```

---

## STEP 4. Raspberry Pi 커널 소스 준비 (rpi-6.12.y)

### 4.1 rpi-linux 클론

```bash
cd ~/uconsole-kernel-work/src

git clone --depth=1 --branch rpi-6.12.y \
  https://github.com/raspberrypi/linux.git rpi-linux

cd rpi-linux
```

`rpi-6.12.y`는 Raspberry Pi 재단이 유지하는 6.12 계열 브랜치다.

### 4.2 Ubuntu config 기반 .config 생성

```bash
cp ~/uconsole-kernel-work/config-ubuntu-raspi .config

make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- olddefconfig
```

---

## STEP 5. 빌드 오류 방지 설정 + localversion

### 5.1 SYSTEM_TRUSTED_KEYS / REVOCATION_KEYS 비우기

```bash
cd ~/uconsole-kernel-work/src/rpi-linux

sed -i 's|CONFIG_SYSTEM_TRUSTED_KEYS=.*|CONFIG_SYSTEM_TRUSTED_KEYS=""|' .config
sed -i 's|CONFIG_SYSTEM_REVOCATION_KEYS=.*|CONFIG_SYSTEM_REVOCATION_KEYS=""|' .config

grep 'SYSTEM_TRUSTED_KEYS\|SYSTEM_REVOCATION_KEYS' .config

make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- olddefconfig
```

### 5.2 localversion 설정

```bash
echo "-camaguee-uconsole-6.12" > localversion
cat localversion
```

---

## STEP 6. uConsole 전용 패널/백라이트/PMU/overlay 패치

```bash
cd ~/uconsole-kernel-work/src/rpi-linux
BASE="https://raw.githubusercontent.com/ak-rex/ClockworkPi-linux/rpi-6.12.y"
```

### 6.1 panel-cwu50 드라이버 추가

```bash
wget -O drivers/gpu/drm/panel/panel-cwu50.c \
  "${BASE}/drivers/gpu/drm/panel/panel-cwu50.c"

wget -O drivers/gpu/drm/panel/panel-cwu50-cm3.c \
  "${BASE}/drivers/gpu/drm/panel/panel-cwu50-cm3.c"

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

### 6.2 backlight-ocp8178 드라이버 추가

```bash
wget -O drivers/video/backlight/ocp8178_bl.c \
  "${BASE}/drivers/video/backlight/ocp8178_bl.c"

sed -i '/^endif # BACKLIGHT_CLASS_DEVICE/i \
config BACKLIGHT_OCP8178\n\
\ttristate "OCP8178 Backlight Driver"\n\
\tdepends on GPIOLIB\n\
\thelp\n\
\t  If you have an OCP8178, say Y to enable the backlight driver.\n' \
  drivers/video/backlight/Kconfig

grep -q 'ocp8178_bl' drivers/video/backlight/Makefile || \
  echo 'obj-$(CONFIG_BACKLIGHT_OCP8178)		+= ocp8178_bl.o' >> drivers/video/backlight/Makefile

scripts/config --module CONFIG_BACKLIGHT_OCP8178
```

### 6.3 AXP20x PMU/배터리 관련 파일 교체

```bash
wget -O drivers/power/supply/axp20x_battery.c \
  "${BASE}/drivers/power/supply/axp20x_battery.c"
wget -O drivers/mfd/axp20x.c \
  "${BASE}/drivers/mfd/axp20x.c"
wget -O include/linux/mfd/axp20x.h \
  "${BASE}/include/linux/mfd/axp20x.h"

wget -O drivers/mfd/axp20x-i2c.c    "${BASE}/drivers/mfd/axp20x-i2c.c"
wget -O drivers/mfd/axp20x-rsb.c    "${BASE}/drivers/mfd/axp20x-rsb.c"
wget -O drivers/iio/adc/axp20x_adc.c "${BASE}/drivers/iio/adc/axp20x_adc.c"
wget -O drivers/input/misc/axp20x-pek.c "${BASE}/drivers/input/misc/axp20x-pek.c"
wget -O drivers/power/supply/axp20x_ac_power.c \
  "${BASE}/drivers/power/supply/axp20x_ac_power.c"
wget -O drivers/power/supply/axp20x_usb_power.c \
  "${BASE}/drivers/power/supply/axp20x_usb_power.c"
wget -O drivers/regulator/axp20x-regulator.c \
  "${BASE}/drivers/regulator/axp20x-regulator.c"
```

### 6.4 clockworkpi/uconsole/devterm/custom-battery overlays 추가

```bash
ODIR="arch/arm/boot/dts/overlays"

wget -O ${ODIR}/clockworkpi-uconsole-overlay.dts \
  "${BASE}/arch/arm/boot/dts/overlays/clockworkpi-uconsole-overlay.dts"
wget -O ${ODIR}/clockworkpi-uconsole-cm3-overlay.dts \
  "${BASE}/arch/arm/boot/dts/overlays/clockworkpi-uconsole-cm3-overlay.dts"
wget -O ${ODIR}/clockworkpi-uconsole-cm5-overlay.dts \
  "${BASE}/arch/arm/boot/dts/overlays/clockworkpi-uconsole-cm5-overlay.dts"

wget -O ${ODIR}/clockworkpi-custom-battery-overlay.dts \
  "${BASE}/arch/arm/boot/dts/overlays/clockworkpi-custom-battery-overlay.dts"

wget -O ${ODIR}/clockworkpi-devterm-overlay.dts \
  "${BASE}/arch/arm/boot/dts/overlays/clockworkpi-devterm-overlay.dts"
wget -O ${ODIR}/clockworkpi-devterm-cm5-overlay.dts \
  "${BASE}/arch/arm/boot/dts/overlays/clockworkpi-devterm-cm5-overlay.dts"

grep -q 'clockworkpi-uconsole' ${ODIR}/Makefile || cat >> ${ODIR}/Makefile << 'EOF_OV'
dtbo-$(CONFIG_ARCH_BCM2835) += clockworkpi-uconsole.dtbo
dtbo-$(CONFIG_ARCH_BCM2835) += clockworkpi-uconsole-cm3.dtbo
dtbo-$(CONFIG_ARCH_BCM2835) += clockworkpi-uconsole-cm5.dtbo
dtbo-$(CONFIG_ARCH_BCM2835) += clockworkpi-custom-battery.dtbo
dtbo-$(CONFIG_ARCH_BCM2835) += clockworkpi-devterm.dtbo
dtbo-$(CONFIG_ARCH_BCM2835) += clockworkpi-devterm-cm5.dtbo
EOF_OV
```

### 6.5 최종 .config 갱신

```bash
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- olddefconfig

grep -E 'DRM_PANEL_CWU50|BACKLIGHT_OCP8178' .config
```

---

## STEP 7. 커널 .deb 빌드 (bindeb-pkg)

```bash
cd ~/uconsole-kernel-work/src/rpi-linux

make -j"$(nproc)" ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- bindeb-pkg
```

빌드 후 다음을 확인한다.

```bash
cd ~/uconsole-kernel-work
find . -maxdepth 3 -name "*.deb" -ls
```

`linux-image-6.12.93-camaguee-uconsole-6.12+_..._arm64.deb` 등 네 개의 `.deb` 파일이 생성된다.

---

## STEP 8. Ubuntu rootfs에 커널 설치 (chroot)

```bash
cd ~/uconsole-kernel-work

sudo cp src/linux-image-6.12.93-camaguee-uconsole-6.12+_*.deb \
        src/linux-headers-6.12.93-camaguee-uconsole-6.12+_*.deb \
        src/linux-libc-dev_6.12.93-*_arm64.deb \
        /mnt/ubuntu/rootfs/tmp/

sudo mount -t proc none /mnt/ubuntu/rootfs/proc
sudo mount -t sysfs none /mnt/ubuntu/rootfs/sys
sudo mount --bind /dev /mnt/ubuntu/rootfs/dev
sudo mount --bind /dev/pts /mnt/ubuntu/rootfs/dev/pts

sudo chroot /mnt/ubuntu/rootfs /bin/bash -c "
  cd /tmp && \
  dpkg -i linux-image-6.12.93-camaguee-uconsole-6.12+_*.deb && \
  dpkg -i linux-headers-6.12.93-camaguee-uconsole-6.12+_*.deb && \
  dpkg -i linux-libc-dev_6.12.93-*_arm64.deb
"

sudo chroot /mnt/ubuntu/rootfs /bin/bash -c \
  "update-initramfs -u -k 6.12.93-camaguee-uconsole-6.12+"

ls /mnt/ubuntu/rootfs/boot/vmlinuz-6.12.93-camaguee-uconsole-6.12+
ls /mnt/ubuntu/rootfs/boot/initrd.img-6.12.93-camaguee-uconsole-6.12+
```

---

## STEP 9. boot/firmware로 커널 & initrd 복사

```bash
sudo cp /mnt/ubuntu/rootfs/boot/vmlinuz-6.12.93-camaguee-uconsole-6.12+ \
        /mnt/ubuntu/rootfs/boot/firmware/vmlinuz-6.12.93-camaguee-uconsole-6.12+

sudo cp /mnt/ubuntu/rootfs/boot/initrd.img-6.12.93-camaguee-uconsole-6.12+ \
        /mnt/ubuntu/rootfs/boot/firmware/initrd.img-6.12.93-camaguee-uconsole-6.12+

ls -lh /mnt/ubuntu/rootfs/boot/firmware/vmlinuz-6.12.93-camaguee-uconsole-6.12+
ls -lh /mnt/ubuntu/rootfs/boot/firmware/initrd.img-6.12.93-camaguee-uconsole-6.12+
```

---

## STEP 10. config.txt를 uConsole 전용으로 교체

```bash
sudo cp /mnt/ubuntu/rootfs/boot/firmware/config.txt \
        /mnt/ubuntu/rootfs/boot/firmware/config.txt.bak

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

선택적으로 배터리 용량 커스텀을 적용한다.

```text
dtoverlay=clockworkpi-custom-battery,charge_full_design_uah=6700000,energy_full_design_uwh=24790000
```

---

## STEP 11. overlays(.dtbo) 존재 여부 확인

```bash
ls /mnt/ubuntu/rootfs/boot/firmware/overlays/ | grep -E 'clockworkpi|devterm|audremap'
```

필요 시 다음을 실행한다.

```bash
sudo cp ~/uconsole-kernel-work/src/rpi-linux/arch/arm/boot/dts/overlays/clockworkpi*.dtbo \
        /mnt/ubuntu/rootfs/boot/firmware/overlays/ 2>/dev/null || true

sudo cp ~/uconsole-kernel-work/src/rpi-linux/arch/arm/boot/dts/overlays/audremap*.dtbo \
        /mnt/ubuntu/rootfs/boot/firmware/overlays/ 2>/dev/null || true
```

---

## STEP 12. cmdline.txt 확인

```bash
cat /mnt/ubuntu/rootfs/boot/firmware/cmdline.txt
```

필요하면 `quiet splash`를 제거해서 디버그 로그를 더 많이 볼 수 있다.

---

## STEP 13. 마운트 해제 및 loop 디바이스 해제

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

## STEP 14. microSD에 이미지 플래시

```bash
cd ~/uconsole-kernel-work
lsblk   # microSD가 /dev/sdX인지 확인

sudo dd if=ubuntu-22.04.4-preinstalled-desktop-arm64+raspi.img \
        of=/dev/sdX bs=4M status=progress conv=fsync

sudo sync
```

---

## STEP 15. uConsole에서 첫 부팅 검증

uConsole에 microSD를 넣고 부팅 후 로컬 또는 SSH에서 실행한다.

```bash
uname -r
```

기대 결과는 다음과 같다.

```text
6.12.93-camaguee-uconsole-6.12+
```

다음도 확인한다.

```bash
dmesg | grep -i "clockworkpi\|devterm\|cwu50\|ocp8178"
lsmod | grep -E "cwu50|ocp8178"

cat /sys/class/power_supply/axp20x-battery/capacity
cat /sys/class/power_supply/axp20x-battery/voltage_now
```

---

## STEP 16. RTL8851BU Wi‑Fi (0bda:b831) 드라이버 설치 (rtw89)

uConsole 내부에서 진행한다.

```bash
lsusb   # 0bda:b831 어댑터 확인
```

### 16.1 의존성 설치

```bash
sudo apt update
sudo apt install -y build-essential dkms git iw
```

### 16.2 (권장) gcc 14 사용

```bash
sudo apt install -y gcc-14 g++-14

sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-14 14
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-13 13
sudo update-alternatives --set gcc /usr/bin/gcc-14

gcc --version
```

### 16.3 rtw89 드라이버 클론 및 빌드

```bash
mkdir -p ~/src
cd ~/src

git clone https://github.com/morrownr/rtw89.git --depth=1
cd rtw89

make clean modules
sudo make install
sudo make install_fw
```

### 16.4 모듈 로드 및 인터페이스 확인

```bash
sudo modprobe rtw89_core_git
sudo modprobe rtw89_usb_git
sudo modprobe rtw89_8851bu_git

lsmod | grep rtw89
ip link show
```

필요하면 한 번 재부팅 후 다시 `ip link show`를 확인한다.

---

## STEP 17. Wi‑Fi 접속 (NetworkManager 기준)

```bash
nmcli device wifi list
nmcli device wifi connect "SSID" password "PASSWORD" ifname wlan0
```

연결 후 다음을 확인한다.

```bash
ip addr show wlan0
ping -c 3 8.8.8.8
```

---

## STEP 18. 최종 체크리스트

1. `uname -r` → `6.12.93-camaguee-uconsole-6.12+`.
2. LCD 화면이 정상 표시됨. 콘솔 또는 GUI 접근 가능.
3. `panel_cwu50`, `ocp8178_bl` 모듈 로드.
4. `axp20x-battery` capacity, voltage 정상 표시.
5. `lsusb`에 `0bda:b831` Realtek 어댑터 표시.
6. `lsmod | grep rtw89`에 Git 버전 모듈 3개(`core_git`, `usb_git`, `8851bu_git`) 표시.
7. `ip link show`에 `wlan0` 또는 `wlx...` 인터페이스 존재.
8. `nmcli`로 AP에 접속 가능.

이 모든 항목이 만족되면, 이 문서만으로 지금의 uConsole OS와 완전히 동일한 환경을 언제든지 재구성할 수 있다.
