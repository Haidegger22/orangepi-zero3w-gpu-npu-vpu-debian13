# Orange Pi Zero 3W — GPU, NPU, VPU, Debian 13 🚀

Полная инструкция по настройке Orange Pi Zero 3W (Allwinner A733) под Debian 13 с работающими акселераторами.

**Характеристики:**
- **SoC:** Allwinner A733 (2×Cortex-A76 @ 2.0 GHz + 6×Cortex-A55 @ 1.8 GHz)
- **RAM:** 12 GB LPDDR5
- **GPU:** PowerVR B-Series BXM-4-64 MC1
- **NPU:** VeriSilicon VIP (3 TOPS)
- **VPU:** Cedar Video Engine (8Kp24 decode, 4Kp30 encode)
- **Сеть:** WiFi 6 + Bluetooth 5.4
- **Хранение:** microSD (eMMC/UFS BGA — опционально, не распаяно)

---

## 📦 Содержание

1. [Обновление до Debian 13](#1-обновление-до-debian-13)
2. [Закрепление BSP-ядра](#2-закрепление-bsp-ядра)
3. [GPU: PowerVR Vulkan + OpenGL](#3-gpu-powervr-vulkan--opengl)
4. [GPU: OpenCL](#4-gpu-opencl)
5. [NPU: VeriSilicon VIP](#5-npu-verisilicon-vip)
6. [VPU: Cedar Video Engine](#6-vpu-cedar-video-engine)
7. [HDMI: кастомное разрешение 1360x768](#7-hdmi-кастомное-разрешение-1360x768)
8. [VPN: установка Happ](#8-vpn-установка-happ)
9. [Бенчмарки](#9-бенчмарки)
10. [Известные проблемы](#10-известные-проблемы)
11. [Источники](#11-источники)

---

## 1. Обновление до Debian 13

Штатный образ Orange Pi идёт на Debian 11 (Bullseye). Обновляемся до Debian 13 (Trixie).

### 1.1 Меняем sources.list

```bash
sudo sed -i 's/bullseye/trixie/g' /etc/apt/sources.list
```

### 1.2 Обновление

```bash
sudo apt update
sudo apt dist-upgrade -y
```

После этого система становится Debian 13, но BSP-ядро Orange Pi (`6.6.98-sun60iw2`) сохраняется.

---

## 2. Закрепление BSP-ядра

Чтобы при следующих обновлениях ядро не заменилось на стоковое Debian (в котором нет драйверов GPU/NPU/VPU), закрепляем пакеты:

```bash
sudo apt-mark hold \
    linux-image-current-sun60iw2 \
    linux-dtb-current-sun60iw2 \
    linux-u-boot-orangepizero3w-current
```

Проверка:
```bash
sudo apt-mark showhold
```

---

## 3. GPU: PowerVR Vulkan + OpenGL

GPU на A733 — **PowerVR B-Series BXM-4-64 MC1** (Imagination Technologies, **НЕ Mali!**).

### 3.1 Что есть в Orange Pi образе

Модуль ядра `pvrsrvkm` уже предустановлен в BSP-ядре. Но userspace неполный: есть OpenGL ES и OpenCL, но нет Vulkan ICD.

### 3.2 Получаем полный DDK из Radxa Cubie A7S

Radxa Cubie A7S использует тот же A733 SoC и поставляет полный PowerVR DDK.

#### Вариант A: Docker-сборка (рекомендуется)

```bash
git clone https://github.com/Incipiens/OrangePiZero3W-GPU-VPU.git
cd OrangePiZero3W-GPU-VPU
# Скачать образ Radxa и запустить сборку
```

#### Вариант B: Tarball'ы (быстрее)

Монтируем образ Radxa и вытаскиваем библиотеки:

```bash
# На другой машине с Docker
# Или качаем образ Radxa:
wget https://github.com/radxa-build/radxa-a733/releases/download/rsdk-r2/radxa-a733_bullseye_kde_r2.output_512.img.xz
xz -d radxa-a733_bullseye_kde_r2.output_512.img.xz

# Монтируем образ
sudo mount -o loop,ro,offset=$((679936 * 512)) radxa-a733_bullseye_kde_r2.output_512.img /mnt

# Копируем Vulkan драйвер и ICD
sudo cp -a /mnt/usr/lib/libVK_IMG.so* /usr/lib/
sudo mkdir -p /usr/share/vulkan/icd.d
sudo cp /mnt/usr/share/vulkan/icd.d/img_icd.json /usr/share/vulkan/icd.d/

# Копируем EGL и GLES (более новые версии)
sudo cp -a /mnt/usr/local/lib/libEGL* /usr/lib/
sudo cp -a /mnt/usr/local/lib/libGLES* /usr/lib/
sudo cp -a /mnt/usr/local/lib/libvulkan.so* /usr/lib/
sudo cp -a /mnt/usr/local/lib/libpvr_mesa_wsi.so* /usr/lib/

# Обновляем кэш библиотек
sudo ldconfig
```

### 3.3 Проверка

```bash
vulkaninfo --summary | grep deviceName
# Должен показать: PowerVR B-Series BXM-4-64 MC1

glxinfo | grep "OpenGL vendor"
# Или kmscube/glmark2-es2 для GLES
```

---

## 4. GPU: OpenCL

Модуль ядра и библиотека `libPVROCL.so` уже есть в Orange Pi образе. Нужно только создать ICD-конфиг:

```bash
sudo mkdir -p /etc/OpenCL/vendors
echo "libPVROCL.so" | sudo tee /etc/OpenCL/vendors/img-pvr.icd
sudo ldconfig
```

Проверка:
```bash
clinfo | grep -E "Device Name|Platform"
```

---

## 5. NPU: VeriSilicon VIP

**Статус: драйвер ядра работает из коробки ✅**

Модуль `vipcore.ko` уже встроен в BSP-ядро и загружается автоматически:

```bash
lsmod | grep vipcore
# vpucore
ls -la /dev/vipcore
# crw-rw-rw- /dev/vipcore
```

dmesg:
```
vipcore, platform driver init
VIPLite driver version 2.0.3.4-AW-2025-10-27
```

**Userspace (TIM-VX):** пока не настроен. Для запуска моделей потребуется собрать [TIM-VX](https://github.com/VeriSilicon/TIM-VX).

---

## 6. VPU: Cedar Video Engine

**Статус: работает из коробки ✅**

Модуль `sunxi_ve` уже встроен в BSP-ядро и загружается при старте:

```bash
lsmod | grep sunxi_ve
# sunxi_ve

ls -la /dev/cedar_dev_ve
# /dev/cedar_dev_ve2
```

GStreamer OMX кодеки предустановлены:
```bash
gst-inspect-1.0 | grep omx
# OMX.allwinner.video.decoder.avc       (H.264)
# OMX.allwinner.video.decoder.hevc      (H.265)
# OMX.allwinner.video.decoder.vp8
# OMX.allwinner.video.decoder.vp9
# OMX.allwinner.video.decoder.mpeg2
# OMX.allwinner.video.decoder.mpeg4
# Всего ~10 кодеков
```

Тест:
```bash
gst-launch-1.0 filesrc location=test.h264 ! h264parse ! omxh264dec ! fakesink
```

---

## 7. HDMI: кастомное разрешение 1360x768

Для телевизора LG с разрешением 1360×768 (EDID не читается по I2C/DDC).

### 7.1 Xorg-конфиг

```bash
sudo mkdir -p /etc/X11/xorg.conf.d
```

`/etc/X11/xorg.conf.d/99-hdmi-1360x768.conf`:
```
Section "Monitor"
    Identifier "HDMI-1"
    Modeline "1360x768_60.00" 84.75 1360 1432 1568 1776 768 771 781 798 -hsync +vsync
    Option "PreferredMode" "1360x768_60.00"
EndSection

Section "Device"
    Identifier "Card0"
    Driver "modesetting"
EndSection

Section "Screen"
    Identifier "Screen0"
    Device "Card0"
    Monitor "HDMI-1"
    DefaultDepth 24
    SubSection "Display"
        Depth 24
        Modes "1360x768_60.00" "1280x720" "1920x1080"
    EndSubSection
EndSection
```

### 7.2 Применить сразу (без перезагрузки)

```bash
DISPLAY=:0 xrandr --newmode "1360x768_60.00" 84.75 1360 1432 1568 1776 768 771 781 798 -hsync +vsync
DISPLAY=:0 xrandr --addmode HDMI-1 "1360x768_60.00"
DISPLAY=:0 xrandr --output HDMI-1 --mode "1360x768_60.00"
```

### 7.3 Отключение DPMS (экран не гаснет)

```bash
DISPLAY=:0 xset -dpms
DISPLAY=:0 xset s off
```

Автозапуск:
```bash
mkdir -p ~/.config/autostart
```

`~/.config/autostart/disable-dpms.desktop`:
```
[Desktop Entry]
Type=Application
Name=Disable DPMS
Exec=bash -c 'sleep 3 && xset -display :0 -dpms && xset -display :0 s off'
X-GNOME-Autostart-enabled=true
```

---

## 8. VPN: установка Happ

Happ — клиент для прокси/VPN (замена FlClashX).

### Установка через GitHub

```bash
wget 'https://github.com/Happ-proxy/happ-desktop/releases/download/2.18.1/Happ.linux.arm64.deb'
# Если dpkg не поддерживает zstd — конвертируем:
mkdir happ-deb && cd happ-deb
ar x ../Happ.linux.arm64.deb
zstd -d control.tar.zst -o control.tar
zstd -d data.tar.zst -o data.tar
xz -z -6 control.tar
xz -z -6 data.tar
ar rcs ../Happ.linux.arm64-repack.deb debian-binary control.tar.xz data.tar.xz
sudo dpkg -i ../Happ.linux.arm64-repack.deb
```

Или просто распаковать data.tar напрямую:
```bash
mkdir happ-deb && cd happ-deb
ar x ../Happ.linux.arm64.deb
zstd -d data.tar.zst -o data.tar
sudo tar -xf data.tar -C /
```

Проверка: `happd --version` (ответ: `happd 2.18.1`).

---

## 9. Бенчмарки

Сравнение OPI Zero 3W vs Raspberry Pi 5 vs Raspberry Pi 4:

| Бенчмарк | Pi5 (4×A76 @ 2.4ГГц) | OPI Zero 3W (2+6 @ 2.0ГГц) | Pi4 (4×A72 @ 1.8ГГц) |
|---|---|---|---|
| **7z b (Total MIPS)** | 14 203 🥇 | 10 744 🥈 | 4 943 🥉 |
| **SHA256 16KB (MB/s)** | 1 494 🥇 | 1 270 🥈 | 168 🥉 |
| **RAM** | 8 GB LPDDR4X | **12 GB LPDDR5** 🔥 | 8 GB LPDDR4 |

**Вывод:** OPI Zero 3W выдаёт ~75% производительности Pi5 при вдвое меньшей цене и размере. 12 GB LPDDR5 — весомое преимущество.

---

## 10. Известные проблемы

### 🔊 Звук по HDMI не работает

**Симптом:** PulseAudio/ALSA видят HDMI-выход, звук «идёт» (sink в состоянии RUNNING), но на телевизоре тишина.

**Причина:** Драйвер `sunxi-hdmi` на ядре `6.6.98-sun60iw2` не может прочитать EDID по I2C/DDC с телевизора LG (ошибка `i2c read edid failed`). Без EDID драйвер не активирует аудиоканал — идёт цикл:
```
dw audio unset when hdmi_on(0) audio_on(0)
hdmi drv audio set enable done
hdmi drv audio set disable done  ← через ~20 секунд
```

**Что перепробовано (не помогло):**
- Кастомный EDID через `drm.edid_firmware` в cmdline — не подхватывается sunxi-драйвером
- EDID override через debugfs (`edid_override`) — не влияет на sysfs
- Обновление libdrm до 2.4.124
- Откат libasound2-data до версии 1.2.4 (как в Debian 11)
- Установка ALSA UCM-конфигов из Radxa образа
- Переключение audio data format
- Отключение DPMS

**Решение:** На Debian 11 звук работал с этим же кабелем — проблема в связке Debian 13 userland + BSP-ядро 6.6.98-sun60iw2. Требуется либо:
- Другой HDMI-кабель (с рабочими DDC-линиями)
- Использовать Bluetooth-колонки (BT 5.4 работает)
- USB-аудиокарта
- Пересобрать BSP-ядро с исправлением sunxi-hdmi

---

## 11. Источники

- [Incipiens/OrangePiZero3W-GPU-VPU](https://github.com/Incipiens/OrangePiZero3W-GPU-VPU) — Docker-сборка GPU+VPU userspace
- [Radxa Cubie A7S образы](https://docs.radxa.com/en/cubie/a7s/download) — исходник PowerVR DDK
- [radxa-build/radxa-a733](https://github.com/radxa-build/radxa-a733) — релизы Radxa образов
- [radxa/allwinner-target](https://github.com/radxa/allwinner-target) (ветка `target-a733-v1.4.6`) — пакеты с DDK
- [Happ-proxy/happ-desktop](https://github.com/Happ-proxy/happ-desktop) — VPN-клиент Happ
- [Статья XDA](https://www.xda-developers.com/orange-pi-zero-3w-beats-raspberry-pi-5-cant-use-half-hardware/) — обзор с GPU-тюнингом
