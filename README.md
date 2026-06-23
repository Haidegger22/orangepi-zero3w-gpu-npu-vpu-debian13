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

## Порядок действий

Поскольку Orange Pi Zero 3W поставляется с Debian 11 (Bullseye) и BSP-ядром `6.6.98-sun60iw2`, нужно сначала настроить всё на родной системе, а затем обновиться до Debian 13.

```
Debian 11 (сток) → GPU/NPU/VPU → HDMI → Debian 13 → закрепление ядра
```

---

## 📦 Содержание

1. [GPU: PowerVR Vulkan + OpenGL + OpenCL](#1-gpu-powervr-vulkan--opengl--opencl)
2. [NPU: VeriSilicon VIP](#2-npu-verisilicon-vip)
3. [VPU: Cedar Video Engine](#3-vpu-cedar-video-engine)
4. [HDMI: кастомное разрешение 1360x768](#4-hdmi-кастомное-разрешение-1360x768)
5. [Обновление до Debian 13](#5-обновление-до-debian-13)
6. [Закрепление BSP-ядра](#6-закрепление-bsp-ядра)
7. [Бенчмарки](#7-бенчмарки)
8. [Известные проблемы](#8-известные-проблемы)
9. [Источники](#9-источники)

---

## 1. GPU: PowerVR Vulkan + OpenGL + OpenCL

GPU на A733 — **PowerVR B-Series BXM-4-64 MC1** (Imagination Technologies, **НЕ Mali!**).

### 1.1 Что есть в Orange Pi образе

Модуль ядра `pvrsrvkm` уже предустановлен в BSP-ядре. Но userspace неполный: есть OpenGL ES, но нет Vulkan ICD и OpenCL ICD.

### 1.2 Получаем полный DDK из Radxa Cubie A7S

Radxa Cubie A7S использует тот же A733 SoC и поставляет полный PowerVR DDK с Vulkan 1.3 и OpenCL 3.0.

#### Вариант A: Docker-сборка (рекомендуется)

```bash
git clone https://github.com/Incipiens/OrangePiZero3W-GPU-VPU.git
cd OrangePiZero3W-GPU-VPU
./make-tarball.sh   # скачает Radxa образ и создаст tarball'ы
```

#### Вариант B: Tarball'ы вручную (быстрее, если образ уже скачан)

Монтируем образ Radxa и вытаскиваем библиотеки:

```bash
# Качаем и распаковываем образ Radxa A733 (rsdk-r2):
wget https://github.com/radxa-build/radxa-a733/releases/download/rsdk-r2/radxa-a733_bullseye_kde_r2.output_512.img.xz
xz -d radxa-a733_bullseye_kde_r2.output_512.img.xz

# Определяем смещение корневого раздела (обычно раздел 3)
fdisk -l radxa-a733_bullseye_kde_r2.output_512.img
# Пример: раздел 3 начинается с сектора 679936, сектор = 512 байт
# Смещение = 679936 * 512

# Монтируем
sudo mkdir /mnt/radxa
sudo mount -o loop,ro,offset=$((679936 * 512)) radxa-a733_bullseye_kde_r2.output_512.img /mnt/radxa
```

#### 1.2.1 Vulkan

```bash
# Копируем драйвер и ICD
sudo cp -a /mnt/radxa/usr/lib/libVK_IMG.so* /usr/lib/
sudo mkdir -p /usr/share/vulkan/icd.d
sudo cp /mnt/radxa/usr/share/vulkan/icd.d/img_icd.json /usr/share/vulkan/icd.d/
```

#### 1.2.2 OpenCL

```bash
# Библиотека libPVROCL.so уже есть в образе Orange Pi.
# Нужно только создать ICD-конфиг:
sudo mkdir -p /etc/OpenCL/vendors
echo "libPVROCL.so" | sudo tee /etc/OpenCL/vendors/img-pvr.icd
```

#### 1.2.3 EGL + GLES (более свежие версии из Radxa)

```bash
sudo cp -a /mnt/radxa/usr/local/lib/libEGL* /usr/lib/
sudo cp -a /mnt/radxa/usr/local/lib/libGLES* /usr/lib/
sudo cp -a /mnt/radxa/usr/local/lib/libvulkan.so* /usr/lib/
sudo cp -a /mnt/radxa/usr/local/lib/libpvr_mesa_wsi.so* /usr/lib/
```

#### 1.2.4 Обновляем кэш библиотек

```bash
sudo ldconfig
```

### 1.3 Проверка

```bash
# Vulkan
vulkaninfo --summary | grep -E "deviceName|apiVersion"
# PowerVR B-Series BXM-4-64 MC1, Vulkan 1.3.277

# OpenCL
clinfo | grep -E "Device Name|Platform"
# PowerVR, OpenCL 3.0

# OpenGL ES
glxinfo | grep -E "OpenGL vendor|OpenGL renderer"
# Или kmscube/glmark2-es2 для безголового теста
```

---

## 2. NPU: VeriSilicon VIP

**Статус: драйвер ядра работает из коробки ✅**

Модуль `vipcore.ko` встроен в BSP-ядро и загружается автоматически:

```bash
lsmod | grep vipcore
# vipcore

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

## 3. VPU: Cedar Video Engine

**Статус: работает из коробки ✅**

Модуль `sunxi_ve` встроен в BSP-ядро и загружается при старте:

```bash
lsmod | grep sunxi_ve
# sunxi_ve

ls -la /dev/cedar_dev_ve2
```

GStreamer OMX кодеки предустановлены:
```bash
gst-inspect-1.0 | grep omx
# Пример:
# OMX.allwinner.video.decoder.avc    (H.264)
# OMX.allwinner.video.decoder.hevc   (H.265)
# OMX.allwinner.video.decoder.vp8
# OMX.allwinner.video.decoder.vp9
# OMX.allwinner.video.decoder.mpeg2
# OMX.allwinner.video.decoder.mpeg4
# OMX.allwinner.video.encoder.h264
# и другие — всего ~10 кодеков
```

Тест:
```bash
gst-launch-1.0 filesrc location=test.h264 ! h264parse ! omxh264dec ! fakesink
```

---

## 4. HDMI: кастомное разрешение 1360x768

Для телевизора LG с разрешением 1360×768 (EDID не читается по I2C/DDC).

### 4.1 Xorg-конфиг

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

### 4.2 Применить сразу (без перезагрузки)

```bash
DISPLAY=:0 xrandr --newmode "1360x768_60.00" 84.75 1360 1432 1568 1776 768 771 781 798 -hsync +vsync
DISPLAY=:0 xrandr --addmode HDMI-1 "1360x768_60.00"
DISPLAY=:0 xrandr --output HDMI-1 --mode "1360x768_60.00"
```

### 4.3 Отключение DPMS (экран не гаснет)

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

## 5. Обновление до Debian 13

После настройки всех акселераторов на стоковой системе обновляемся до Debian 13 (Trixie).

### 5.1 Меняем sources.list

```bash
sudo sed -i 's/bullseye/trixie/g' /etc/apt/sources.list
```

### 5.2 Обновление

```bash
sudo apt update
sudo apt dist-upgrade -y
```

После этого система становится Debian 13, BSP-ядро (`6.6.98-sun60iw2`) сохраняется.

**Важно:** После dist-upgrade могут остаться пакеты, которые не обновились из-за hold (gstreamer, vlc, glmark2). Для полного обновления нужно снять с них hold:

```bash
sudo apt-mark unhold gstreamer1.0-plugins-good libgstreamer-plugins-base1.0-0
sudo apt dist-upgrade -y
```

Также возможно потребуется разрешить конфликт с кастомным Xorg от Orange Pi (пакет `xserver-xorg-img-bxm`). Рекомендуется удалить его при конфликте:

```bash
sudo dpkg --remove --force-remove-reinstreq xserver-xorg-img-bxm
```

---

## 6. Закрепление BSP-ядра

После обновления закрепляем ядро, чтобы при следующих apt upgrade оно не заменилось на стоковое Debian (в котором нет драйверов GPU/NPU/VPU):

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

## 7. Бенчмарки

Сравнение OPI Zero 3W vs Raspberry Pi 5 vs Raspberry Pi 4:

| Бенчмарк | Pi5 (4×A76 @ 2.4ГГц) | OPI Zero 3W (2+6 @ 2.0ГГц) | Pi4 (4×A72 @ 1.8ГГц) |
|---|---|---|---|
| **7z b (Total MIPS)** | 14 203 🥇 | 10 744 🥈 | 4 943 🥉 |
| **SHA256 16KB (MB/s)** | 1 494 🥇 | 1 270 🥈 | 168 🥉 |
| **RAM** | 8 GB LPDDR4X | **12 GB LPDDR5** 🔥 | 8 GB LPDDR4 |

**Вывод:** OPI Zero 3W выдаёт ~75% производительности Pi5 при вдвое меньшей цене и размере. 12 GB LPDDR5 — весомое преимущество.

---

## 8. Известные проблемы

### 🔊 Звук по HDMI не работает

**Симптом:** PulseAudio/ALSA видят HDMI-выход, звук «идёт» (sink в состоянии RUNNING), но на телевизоре тишина.

**Причина:** Драйвер `sunxi-hdmi` на ядре `6.6.98-sun60iw2` не может прочитать EDID по I2C/DDC с телевизора LG (ошибка `i2c read edid failed`). Без EDID драйвер не активирует аудиоканал — идёт цикл:
```
dw audio unset when hdmi_on(0) audio_on(0)
hdmi drv audio set enable done
hdmi drv audio set disable done  ← через ~20 секунд
```

**Что перепробовано (не помогло):**
- Кастомный EDID через `drm.edid_firmware` в cmdline (не подхватывается sunxi-драйвером)
- EDID override через debugfs (`edid_override`) — не влияет на sysfs
- Обновление libdrm до 2.4.124
- Откат libasound2-data до версии 1.2.4 (как в Debian 11)
- Установка ALSA UCM-конфигов из Radxa образа
- Переключение audio data format в amixer
- Отключение DPMS

**Решение:** На Debian 11 звук работал с этим же кабелем — проблема в связке Debian 13 userland + BSP-ядро 6.6.98-sun60iw2. Варианты:
- Другой HDMI-кабель (с рабочими DDC-линиями)
- Использовать Bluetooth-колонки/наушники (BT 5.4 работает)
- USB-аудиокарта
- Пересобрать BSP-ядро с исправлением sunxi-hdmi

---

## 9. Источники

- [Incipiens/OrangePiZero3W-GPU-VPU](https://github.com/Incipiens/OrangePiZero3W-GPU-VPU) — Docker-сборка GPU+VPU userspace из Radxa
- [Radxa Cubie A7S образы](https://docs.radxa.com/en/cubie/a7s/download) — исходник PowerVR DDK (Vulkan/OpenCL)
- [radxa-build/radxa-a733](https://github.com/radxa-build/radxa-a733) — релизы Radxa образов на GitHub
- [radxa/allwinner-target](https://github.com/radxa/allwinner-target) (ветка `target-a733-v1.4.6`) — deb-пакеты с DDK
- [Настройка GPU/NPU на OPI Zero 3W](https://github.com/Haidegger22/orangepi-zero3w-gpu-pcie) — проблема с битым модулем-предохранителем pvrsrvkm и решением через отложенную загрузку (systemd)
- [Разрешение HDMI 1360×768](https://github.com/Haidegger22/orangepi-zero3w-hdmi-resolution) — Xorg-конфиг для LG TV
- [Статья XDA](https://www.xda-developers.com/orange-pi-zero-3w-beats-raspberry-pi-5-cant-use-half-hardware/) — обзор OPI Zero 3W с GPU-тюнингом
