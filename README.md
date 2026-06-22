# ubuntu-for-uconsole-cm4-without-uconsole-kernel
A summary of the custom Ubuntu 22.04.4 Linux setup for uConsole CM4, including the Raspberry Pi rpi-6.12.y kernel, hardware support, boot configuration, and RTL8851BU Wi‑Fi setup


# uConsole CM4 커스텀 Ubuntu Linux [ Kor ]

## 프로젝트 소개

이 저장소는 uConsole CM4에서 동작하는 Ubuntu 22.04.4 기반 커스텀 Linux 환경에 대해, 실제로 어떤 작업을 적용했는지 정리한 README입니다.

이 저장소는 OS 이미지 파일 자체를 배포하지 않습니다. 
대신 빌드 방향, 커널 구성, 하드웨어 활성화 요소, 부트 설정, Wi‑Fi 구성 방식 등 재현에 필요한 핵심 내용을 요약해서 제공합니다.

목표는 다음과 같습니다.

- Ubuntu 22.04.4 Raspberry Pi Desktop arm64 기반 환경 사용.
- Raspberry Pi `rpi-6.12.y` 기반 커스텀 커널 적용.
- uConsole CM4에 필요한 패널, 백라이트, PMU, overlay 구성 반영.
- Realtek USB `0bda:b831` 기반 Wi‑Fi 사용 환경 정리.
- 추후 동일한 구성을 다시 만들 수 있도록 구조와 변경점을 명확히 기록.

## 베이스 구성

이 환경은 Ubuntu 22.04.4 preinstalled desktop for Raspberry Pi arm64 이미지를 기반으로 구성했습니다.

핵심 베이스 조건은 다음과 같습니다.

- 베이스 OS: Ubuntu 22.04.4 Raspberry Pi Desktop arm64.
- 부트 경로: `/boot/firmware` 사용 구조.
- 커널 베이스: Raspberry Pi `rpi-6.12.y`.
- 로컬 버전 문자열: `-camaguee-uconsole-6.12`.

## 적용한 작업 요약

### 1. 빌드 환경 준비

새 VM 또는 새 빌드 호스트에서 시작하는 것을 기준으로 구성했습니다.

- x86_64 Debian 환경에서 작업.
- arm64 크로스 컴파일 도구 및 멀티 아키텍처 패키지 구성.
- 커널 빌드, chroot, initramfs 생성에 필요한 패키지 설치.

### 2. Ubuntu Raspberry Pi 이미지 준비

Ubuntu Raspberry Pi 이미지를 다운로드한 뒤 loop 장치로 연결하여 rootfs와 `/boot/firmware`를 각각 마운트하는 방식으로 작업했습니다.

- rootfs 마운트.
- `/boot/firmware` 부트 파티션 마운트.
- Ubuntu 기본 raspi 커널 config를 추출하여 커스텀 커널 `.config`의 시작점으로 사용.

### 3. 커널 소스 준비

커널은 Raspberry Pi `linux` 저장소의 `rpi-6.12.y` 브랜치를 기준으로 구성했습니다.

- Ubuntu raspi 기본 config를 기반으로 `.config` 생성.
- `CONFIG_SYSTEM_TRUSTED_KEYS` 및 `CONFIG_SYSTEM_REVOCATION_KEYS`를 비워 빌드 충돌 가능성 축소.
- `localversion`을 `-camaguee-uconsole-6.12`로 설정하여 결과 커널 버전을 명확히 고정.

### 4. uConsole 하드웨어 지원 반영

uConsole CM4에서 필요한 하드웨어 지원을 커널에 반영했습니다.

적용한 주요 항목은 다음과 같습니다.

- CWU50 패널 드라이버 추가.
- OCP8178 백라이트 드라이버 추가.
- AXP20x PMU 및 배터리 관련 드라이버 반영.
- uConsole/DevTerm 계열 overlay 추가.
- custom battery overlay 반영 가능 구조 포함.

### 5. 커널 빌드 및 rootfs 설치

커널은 `bindeb-pkg` 방식으로 arm64 `.deb` 패키지를 생성하는 경로를 사용했습니다.

- `linux-image`, `linux-headers`, `linux-libc-dev` 패키지 빌드.
- Ubuntu rootfs에 chroot로 진입하여 패키지 설치.
- 대상 커널 버전에 대한 initramfs 생성.
- 생성된 `vmlinuz`와 `initrd.img`를 `/boot/firmware`로 반영.

### 6. /boot/firmware 부트 설정 구성

Ubuntu for Raspberry Pi 구조에 맞춰 `/boot/firmware` 기준으로 부트 설정을 정리했습니다.

주요 반영 내용은 다음과 같습니다.

- `config.txt`에서 커널 및 initramfs 이름을 새 커널 버전에 맞게 고정.
- `ignore_lcd=1`, `display_auto_detect=0`, `camera_auto_detect=0` 적용.
- `dtoverlay=vc4-kms-v3d` 및 CM4 관련 KMS 설정 적용.
- `clockworkpi-uconsole`, `devterm-pmu`, `devterm-panel-uc`, `devterm-misc`, `audremap` overlay 사용.
- `gpu_mem=64`, `spi`, `i2c_arm`, `audio` 등 필요한 부트 파라미터 반영.
- 필요한 `.dtbo` 파일을 `/boot/firmware/overlays`에 배치.

### 7. 첫 부팅 검증 항목 정리

첫 부팅 이후 다음 항목을 검증 대상으로 정리했습니다.

- `uname -r`로 의도한 커널 버전 확인.
- 패널 및 백라이트 관련 모듈 로드 상태 확인.
- 배터리/전원 관련 sysfs 노출 확인.
- overlay 파일 존재 여부 확인.
- LCD 정상 출력 및 콘솔 또는 GUI 진입 가능 여부 확인.

### 8. Wi‑Fi 구성

Wi‑Fi는 Realtek USB `0bda:b831` 기반 RTL8851BU/RTL8831BU 계열을 기준으로 정리했습니다.

적용 방향은 다음과 같습니다.

- `morrownr/rtw89` 기반 Git 드라이버 사용.
- uConsole 내부에서 의존성 설치 후 드라이버 빌드 및 설치.
- `rtw89_core_git`, `rtw89_usb_git`, `rtw89_8851bu_git` 모듈 로드.
- `nmcli` 기준으로 무선 네트워크 연결 확인.

## 반영된 최종 상태

최종적으로 목표한 상태는 다음과 같습니다.

- `uname -r` 결과가 `6.12.93-camaguee-uconsole-6.12+`와 같거나 비슷한 문자로 표시됨.
- LCD 패널이 정상 동작함.
- `panel_cwu50`, `ocp8178_bl` 모듈이 정상 로드됨.
- `axp20x-battery` 관련 sysfs 값이 정상 노출됨.
- `/boot/firmware/overlays` 아래 필요한 overlay 파일이 존재함.
- Realtek USB `0bda:b831` 장치가 인식됨.
- `rtw89` Git 모듈이 정상 로드되고 Wi‑Fi 연결이 가능함.

## 이 저장소에 포함하지 않는 것

이 저장소는 다음 항목을 직접 포함하지 않습니다.

- 완성된 Ubuntu OS 이미지 파일.
- 배포용 microSD 이미지.
- 외부 프로젝트의 전체 소스 코드 미러.

이 저장소는 실제 적용된 작업의 요약과 구성 방향을 정리하는 용도로 사용합니다.

## 참고한 구성 요소

이 프로젝트는 다음과 같은 구성 요소 및 프로젝트를 기반으로 환경을 정리했습니다.

- Raspberry Pi Linux 커널 (`rpi-6.12.y` 계열).
- ClockworkPi uConsole/DevTerm 커널 및 overlay 구성.
- `morrownr/rtw89` 기반 Realtek Wi‑Fi 드라이버 구성.
- Ubuntu for Raspberry Pi 기반 이미지 구조.

## References

- Raspberry Pi Linux
- ClockworkPi 커널/overlay
- morrownr/rtw89
- Ubuntu for Raspberry Pi

## Summary

이 README는 uConsole CM4용 커스텀 Ubuntu Linux 환경에서 실제로 어떤 변경이 적용되었는지를 요약한 문서입니다.

핵심은 Ubuntu 22.04.4 Raspberry Pi Desktop을 베이스로 사용하고, Raspberry Pi `rpi-6.12.y` 커널에 uConsole 하드웨어 지원과 Wi‑Fi 구성을 반영하여 동작 가능한 상태를 만든 점입니다.

기존 uConsole의 패키지 저장소에 접속하여 uConsole 전용 커널을 사용하는 방식은 `Make` 의 수행을 어렵게 하였으며 패키지 및 커널의 업데이트 시도시 의존성이 안정적이지 못하다는 문제점이 있었기 때문에 개발 보드 생태계를 적극 활용하여 uConsole과 동일한 하드웨어를 사용한 `개발 보드` 라는 컨셉으로 해당 하드웨어를 사용할 수 있도록 만들어주는 설정들만 적용하여 Ubuntu와 rpi-kernel을 사용할 수 있도록 구상하였습니다.

OS 이미지 파일 자체는 포함하지 않고, 대신 어떤 작업이 적용되었는지와 어떤 구성 요소를 사용했는지를 추적 가능하게 정리하는 방향으로 구성했습니다.



## License

이 저장소의 내용은 MIT License 하에 제공합니다.
외부 프로젝트와 구성 요소의 소스 코드 및 저작물은 각 프로젝트의 원래 라이선스를 따릅니다.
