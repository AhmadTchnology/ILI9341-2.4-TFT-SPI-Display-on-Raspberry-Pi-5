# ILI9341 2.4" TFT SPI Display on Raspberry Pi 5

A complete guide to connecting and running an ILI9341 240x320 SPI TFT display as the primary desktop display on a Raspberry Pi 5, using the `panel-mipi-dbi-spi` kernel driver.

---

## Hardware

| Component | Details |
|-----------|---------|
| Display | 2.4" ILI9341 TFT LCD SPI 240x320 |
| SBC | Raspberry Pi 5 |
| OS | Raspberry Pi OS Bookworm (64-bit) |
| Driver IC | ILI9341 |
| Interface | SPI0 |

---

## Wiring

| ILI9341 Pin | Pi Physical Pin | GPIO (BCM) |
|-------------|----------------|------------|
| VDD33 | 1 | 3.3V |
| GND | 6 | GND |
| LED | 17 | 3.3V (hardwired) |
| CS | 24 | GPIO 8 (SPI0 CE0) |
| RST | 18 | GPIO 24 |
| DC | 22 | GPIO 25 |
| MOSI (SDA) | 19 | GPIO 10 |
| SCK (SCL) | 23 | GPIO 11 |
| MISO (SDO) | 21 | GPIO 9 |

> **Note:** The LED (backlight) pin is wired directly to 3.3V (pin 17) for always-on backlight. GPIO control of backlight is not used.

---

## Software Stack

| Component | Purpose |
|-----------|---------|
| `panel-mipi-dbi-spi` | Kernel DRM driver for the display |
| `mipi-dbi-cmd` | Tool to generate binary firmware init sequence |
| `/lib/firmware/panel.bin` | Binary init commands for ILI9341 |
| `wlr-randr` | Wayland output configuration |
| `labwc` | Wayland compositor (default on Pi OS Bookworm) |

---

## Why Not fbcp-ili9341?

`fbcp-ili9341` is **not compatible with Raspberry Pi 5**. It was built on the VideoCore DispmanX API which is unavailable on Pi 5. It also uses 32-bit ARM compiler flags (`-marm`, `-mfloat-abi=hard`) that fail on the Pi 5's 64-bit architecture.

The correct modern approach is the `panel-mipi-dbi-spi` kernel driver which is built into the Pi OS kernel.

---

## Step 1 — Enable SPI

```bash
sudo raspi-config
# Interface Options → SPI → Enable
sudo reboot
```

Verify:
```bash
ls /dev/spi*
# Should show /dev/spidev0.0  /dev/spidev0.1  /dev/spidev10.0
```

---

## Step 2 — Get the mipi-dbi-cmd Tool

This tool converts a human-readable init sequence into the binary firmware file the kernel driver loads.

```bash
wget https://raw.githubusercontent.com/notro/panel-mipi-dbi/main/mipi-dbi-cmd
chmod +x ~/mipi-dbi-cmd
```

---

## Step 3 — Create the ILI9341 Init Sequence

Save as `~/ili9341.txt`:

```
command 0x01
delay 128
command 0xEF 0x03 0x80 0x02
command 0xCF 0x00 0xC1 0x30
command 0xED 0x64 0x03 0x12 0x81
command 0xE8 0x85 0x00 0x78
command 0xCB 0x39 0x2C 0x00 0x34 0x02
command 0xF7 0x20
command 0xEA 0x00 0x00
command 0xC0 0x23
command 0xC1 0x10
command 0xC5 0x3E 0x28
command 0xC7 0x86
command 0x36 0xE8
command 0x37 0x00
command 0x3A 0x55
command 0xB1 0x00 0x18
command 0xB6 0x08 0x82 0x27
command 0xF2 0x00
command 0x26 0x01
command 0xE0 0x0F 0x31 0x2B 0x0C 0x0E 0x08 0x4E 0xF1 0x37 0x07 0x10 0x03 0x0E 0x09 0x00
command 0xE1 0x00 0x0E 0x14 0x03 0x11 0x07 0x31 0xC1 0x48 0x08 0x0F 0x0C 0x31 0x36 0x0F
command 0x11
delay 128
command 0x29
delay 50
```

> **Key command:** `0x36 0xE8` sets the ILI9341 memory access control to **landscape mode natively** — no software rotation needed, which means the mouse works correctly without any input transformation.

---

## Step 4 — Generate Binary Firmware

```bash
~/mipi-dbi-cmd ~/panel.bin ~/ili9341.txt
sudo mkdir -p /lib/firmware
sudo cp ~/panel.bin /lib/firmware/panel.bin
```

---

## Step 5 — Blacklist the Old fbtft Driver

The old `fb_ili9341` staging driver conflicts with `panel-mipi-dbi`. Blacklist it:

```bash
sudo nano /etc/modprobe.d/blacklist-fbtft.conf
```

Add:
```
blacklist fb_ili9341
blacklist fbtft
```

---

## Step 6 — Configure /boot/firmware/config.txt

```bash
sudo nano /boot/firmware/config.txt
```

Add at the bottom:

```ini
dtparam=spi=on
dtoverlay=mipi-dbi-spi,spi0-0,speed=8000000,fps=60
dtparam=width=320,height=240
dtparam=reset-gpio=24,dc-gpio=25
dtparam=write-only
```

> **SPI Speed Note:** Higher speeds (32MHz, 64MHz) caused flickering due to signal integrity issues with jumper wires. 8MHz proved stable. If you use short direct wires you may be able to increase this. Always use even divisors.

> **Landscape mode:** `width=320,height=240` matches the native landscape orientation set by the `0x36 0xE8` command in the init sequence.

---

## Step 7 — Reboot

```bash
sudo reboot
```

---

## Step 8 — Verify Driver Loaded

```bash
dmesg | grep -i -E "mipi|panel"
```

Expected output:
```
[drm] Initialized panel-mipi-dbi 1.0.0 for spi0.0 on minor X
panel-mipi-dbi-spi spi0.0: [drm] fb0: panel-mipi-dbid frame buffer device
```

Also verify the framebuffer:
```bash
ls /dev/fb*          # should show /dev/fb0
fbset -fb /dev/fb0   # should show 320x240 16bpp
```

And the Wayland output:
```bash
WAYLAND_DISPLAY=wayland-0 wlr-randr
# Should show SPI-1 at 320x240 60Hz
```

---

## Step 9 — Scale the Desktop (Optional)

The default desktop UI is designed for higher resolutions. Scale it down to fit more on the 320x240 screen:

```bash
WAYLAND_DISPLAY=wayland-0 wlr-randr --output SPI-1 --scale 0.5
```

This makes the compositor render at 640x480 and scale down to 320x240, fitting much more on screen.

To make it permanent:

```bash
nano ~/.config/labwc/autostart
```

Add:
```bash
wlr-randr --output SPI-1 --scale 0.5 &
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `GPIO busy` error in Python | Kernel driver owns the GPIO pins | Use kernel driver approach, don't use Python SPI alongside it |
| `fb_ili9341: error -EINVAL: buswidth is not set` | Old fbtft driver loading | Blacklist `fb_ili9341` and `fbtft` |
| White/flickering screen | SPI speed too high | Lower `speed=` in config.txt |
| Constant flicker at all speeds | Power/signal noise | Use shorter wires or add 100µF capacitor between VCC and GND |
| Image cropped | Desktop resolution mismatch | Use `wlr-randr --scale 0.5` |
| Mouse axes swapped | Software rotation applied | Use native landscape via `0x36 0xE8` instead of `wlr-randr --transform` |
| `fbcp-ili9341` fails to compile | Pi 5 is 64-bit, fbcp uses 32-bit ARM flags | Use `panel-mipi-dbi-spi` kernel driver instead |

---

## File Summary

| File | Purpose |
|------|---------|
| `/lib/firmware/panel.bin` | ILI9341 binary init sequence |
| `/boot/firmware/config.txt` | Kernel overlay configuration |
| `/etc/modprobe.d/blacklist-fbtft.conf` | Prevents old driver conflicts |
| `~/.config/labwc/autostart` | Wayland autostart (scale, etc.) |
| `~/mipi-dbi-cmd` | Tool to regenerate panel.bin |
| `~/ili9341.txt` | Human-readable init sequence source |

---

## Display Orientation Notes

The `0x36` register (Memory Access Control) controls the display orientation:

| Value | Orientation |
|-------|-------------|
| `0x48` | Portrait (240x320) |
| `0xE8` | Landscape (320x240) ← used here |
| `0x88` | Portrait flipped |
| `0x28` | Landscape flipped |

Changing this value in `ili9341.txt` and regenerating `panel.bin` is all that's needed to change orientation — no software rotation required.

---

## References

- [panel-mipi-dbi wiki](https://github.com/notro/panel-mipi-dbi/wiki)
- [mipi-dbi-spi overlay README](https://github.com/raspberrypi/linux/blob/rpi-6.6.y/arch/arm/boot/dts/overlays/README)
- [ILI9341 Datasheet](https://cdn-shop.adafruit.com/datasheets/ILI9341.pdf)
- [Adafruit CircuitPython ILI9341](https://github.com/adafruit/Adafruit_CircuitPython_ILI9341)

##

### Made With ❤️ By AhmadTchnology
