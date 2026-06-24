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
Debian 11 (сток) → GPU/NPU/VPU → HDMI → закрепление ядра → Debian 13
```

---

## 📦 Содержание

1. [GPU: PowerVR Vulkan + OpenGL + OpenCL](#1-gpu-powervr-vulkan--opengl--opencl)
2. [NPU: VeriSilicon VIP](#2-npu-verisilicon-vip)
3. [VPU: Cedar Video Engine](#3-vpu-cedar-video-engine)
4. [HDMI: кастомное разрешение 1360x768](#4-hdmi-кастомное-разрешение-1360x768)
5. [Аудио (HDMI)](#5-аудио-hdmi)
6. [Закрепление BSP-ядра](#6-закрепление-bsp-ядра)
7. [Обновление до Debian 13](#7-обновление-до-debian-13)
8. [Бенчмарки](#8-бенчмарки)
9. [Известные проблемы](#9-известные-проблемы)
10. [Источники](#10-источники)

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


## 5. Аудио (HDMI)

**Важно:** Драйвер `sunxi-hdmi` не имеет аппаратного регулятора громкости (Master/PCM). Регулировка возможна только программно через PulseAudio или на стороне телевизора.

### 5.1 ALSA

`/etc/asound.conf`:
```
defaults.pcm.card 0
defaults.pcm.device 0
defaults.ctl.card 0
```

### 5.2 PulseAudio — default sink

Добавить в `/etc/pulse/default.pa`:
```
# HDMI sink для sunxi
load-module module-alsa-sink device=hw:0,0 sink_name=hdmi
```

Перезапуск PulseAudio:
```bash
pulseaudio -k
sleep 1
pulseaudio --start
```

### 5.3 Иконка громкости в трее

**Вариант A: pasystray (через PulseAudio, рекомендуется)**

```bash
sudo apt install -y pasystray
```

`~/.config/autostart/pasystray.desktop`:
```
[Desktop Entry]
Type=Application
Name=PulseAudio System Tray
Exec=pasystray
X-GNOME-Autostart-enabled=true
```

**Вариант B: volumeicon-alsa (через ALSA, если PulseAudio не работает)**

```bash
sudo apt install -y volumeicon-alsa
```

`~/.config/volumeicon/volumeicon`:
```
[Alsa]
card=default
control=Master
logarithmic_scale=true

[StatusIcon]
stepsize=3
lmb_slider=true
mmb_mute=true
show_sound_level=true

[Hotkeys]
up_enabled=true
down_enabled=true
mute_enabled=true
up=XF86AudioRaiseVolume
down=XF86AudioLowerVolume
mute=XF86AudioMute
```

### 5.4 HDMI как default sink при входе в Xfce

```bash
mkdir -p ~/.config/autostart
```

`~/.config/autostart/hdmi-audio.desktop`:
```
[Desktop Entry]
Type=Application
Name=Set HDMI as Default Audio Sink
Exec=pactl set-default-sink hdmi
X-GNOME-Autostart-enabled=true
```

---

## 6. Закрепление BSP-ядра

Перед обновлением до Debian 13 закрепляем BSP-ядро, чтобы при dist-upgrade оно не заменилось на стоковое Debian (в котором нет драйверов GPU/NPU/VPU):

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

## 7. Обновление до Debian 13

После настройки всех акселераторов и закрепления ядра обновляемся до Debian 13 (Trixie).

### 6.1 Меняем sources.list

```bash
sudo sed -i 's/bullseye/trixie/g' /etc/apt/sources.list
```

### 6.2 Обновление

```bash
sudo apt update
sudo apt dist-upgrade -y
```

После этого система становится Debian 13, BSP-ядро (`6.6.98-sun60iw2`) остаётся на месте (hold).

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

## 8. Бенчмарки

Сравнение OPI Zero 3W vs Raspberry Pi 5 vs Raspberry Pi 4:

| Бенчмарк | Pi5 (4×A76 @ 2.4ГГц) | OPI Zero 3W (2+6 @ 2.0ГГц) | Pi4 (4×A72 @ 1.8ГГц) |
|---|---|---|---|
| **7z b (Total MIPS)** | 14 203 🥇 | 10 744 🥈 | 4 943 🥉 |
| **SHA256 16KB (MB/s)** | 1 494 🥇 | 1 270 🥈 | 168 🥉 |
| **RAM** | 8 GB LPDDR4X | **12 GB LPDDR5** 🔥 | 8 GB LPDDR4 |

**Вывод:** OPI Zero 3W выдаёт ~75% производительности Pi5 при вдвое меньшей цене и размере. 12 GB LPDDR5 — весомое преимущество.

---

## 9. Известные проблемы

### 🔊 Звук по HDMI

#### На Debian 11 (штатная система)

Звук по HDMI работает из коробки. После обновления до Debian 13 может перестать работать на некоторых телевизорах.

#### На Debian 13 (после обновления)

**Симптом:** PulseAudio/ALSA видят HDMI-выход, звук «идёт» (sink в состоянии RUNNING), но на телевизоре тишина.

**Причина:** Драйвер `sunxi-hdmi` на ядре `6.6.98-sun60iw2` не может прочитать EDID по I2C/DDC с некоторых телевизоров (ошибка `i2c read edid failed`). Без EDID драйвер не активирует аудиоканал — идёт цикл включения/отключения звука.

**Что перепробовано (не помогло):**
- Кастомный EDID через `drm.edid_firmware` в cmdline (не подхватывается sunxi-драйвером)
- EDID override через debugfs (`edid_override`) — не влияет на sysfs
- Обновление libdrm, ALSA UCM-конфиги
- Переключение audio data format в amixer

**Решение:**
- Использовать Bluetooth-колонки/наушники (BT 5.4 работает)
- USB-аудиокарта
- Пересобрать BSP-ядро с исправлением sunxi-hdmi
- Либо оставаться на Debian 11 для звука по HDMI

---

## 10. Источники

- [Incipiens/OrangePiZero3W-GPU-VPU](https://github.com/Incipiens/OrangePiZero3W-GPU-VPU) — Docker-сборка GPU+VPU userspace из Radxa
- [Radxa Cubie A7S образы](https://docs.radxa.com/en/cubie/a7s/download) — исходник PowerVR DDK (Vulkan/OpenCL)
- [radxa-build/radxa-a733](https://github.com/radxa-build/radxa-a733) — релизы Radxa образов на GitHub
- [radxa/allwinner-target](https://github.com/radxa/allwinner-target) (ветка `target-a733-v1.4.6`) — deb-пакеты с DDK
- [Настройка GPU/NPU на OPI Zero 3W](https://github.com/Haidegger22/orangepi-zero3w-gpu-pcie) — проблема с битым модулем-предохранителем pvrsrvkm и решением через отложенную загрузку (systemd)
- [Разрешение HDMI 1360×768](https://github.com/Haidegger22/orangepi-zero3w-hdmi-resolution) — Xorg-конфиг для LG TV
- [Статья XDA](https://www.xda-developers.com/orange-pi-zero-3w-beats-raspberry-pi-5-cant-use-half-hardware/) — обзор OPI Zero 3W с GPU-тюнингом
