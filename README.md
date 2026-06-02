# ILI9341 2.4" TFT SPI Display on Raspberry Pi 5

A complete guide to connecting and running an ILI9341 240x320 SPI TFT display as the primary desktop display on a Raspberry Pi 5, using the `panel-mipi-dbi-spi` kernel driver. Also covers XPT2046 touchscreen setup and calibration.

---

## Hardware

| Component | Details |
|-----------|---------|
| Display | 2.4" ILI9341 TFT LCD SPI 240x320 |
| Touch Controller | XPT2046 (ADS7846 compatible) |
| SBC | Raspberry Pi 5 |
| OS | Raspberry Pi OS Bookworm (64-bit) |
| Driver IC | ILI9341 |
| Interface | SPI0 |

---

## Wiring — Display Only

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

## Wiring — Display + Touch (XPT2046)

The touch controller has its own separate pin header on the board. Even though T_CLK, T_DIN, and T_DO go to the same Pi GPIO pins as the display SCK/MOSI/MISO, you still need separate physical wires from the touch header to those Pi pins.

| Pin | Pi Physical Pin | GPIO (BCM) | Note |
|-----|----------------|------------|------|
| VDD33 | 1 | 3.3V | |
| GND | 6 | GND | |
| LED | 17 | 3.3V | hardwired |
| CS (display) | 24 | GPIO 8 (CE0) | |
| RST | 18 | GPIO 24 | |
| DC | 22 | GPIO 25 | |
| MOSI / T_DIN | 19 | GPIO 10 | shared |
| SCK / T_CLK | 23 | GPIO 11 | shared |
| MISO / T_DO | 21 | GPIO 9 | shared |
| T_CS | **26** | GPIO 7 | see note below |
| T_IRQ | 7 | GPIO 4 | |

> **T_CS Note:** Connecting T_CS to GPIO 7 (CE1) caused SPI conflicts. Grounding T_CS permanently is the reliable solution for this display. This works because the display and touch controller are on separate SPI chip selects at the kernel level (spi0.0 and spi0.1).

> **Shared pins:** T_CLK, T_DIN, T_DO share the same Pi GPIO pins as display SCK, MOSI, MISO. You need two wires going into pins 19, 21, and 23 — one from the display header and one from the touch header. Use a breadboard or twist wires together.

---

## Software Stack

| Component | Purpose |
|-----------|---------|
| `panel-mipi-dbi-spi` | Kernel DRM driver for the display |
| `mipi-dbi-cmd` | Tool to generate binary firmware init sequence |
| `/lib/firmware/panel.bin` | Binary init commands for ILI9341 |
| `ads7846` | Kernel touch driver for XPT2046 |
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

# Touchscreen Setup (XPT2046 / ADS7846)

If your display board has a touch controller (XPT2046 chip), follow these additional steps. The XPT2046 is fully compatible with the Linux `ads7846` kernel driver.

---

## Touch Step 1 — Verify Touch Chip is Present

Look at the back of your display board near the touch pins (T_CLK, T_CS, T_DIN, T_DO, T_IRQ). There should be a small black IC chip labeled `XPT2046` or `2046`. If the chip is missing, touch will not work regardless of software configuration.

---

## Touch Step 2 — Wire the Touch Controller

The touch header pins are separate from the display header even though some signals are shared. You need physical wires from the touch header to the Pi:

| Touch Pin | Pi Physical Pin | GPIO | Note |
|-----------|----------------|------|------|
| T_CLK | 23 | GPIO 11 | same pin as display SCK |
| T_DIN | 19 | GPIO 10 | same pin as display MOSI |
| T_DO | 21 | GPIO 9 | same pin as display MISO |
| T_CS | **26** | GPIO 7 | see note below |
| T_IRQ | 7 | GPIO 4 | |

For pins 23, 19, and 21 you need two wires going into the same Pi pin. Use a breadboard or twist the wires together.

---

## Touch Step 3 — Add ads7846 Overlay to config.txt

```bash
sudo nano /boot/firmware/config.txt
```

Add this line **after** the mipi-dbi-spi lines:

```ini
dtoverlay=ads7846,penirq=4,speed=500000,xohms=60,pmax=255
```

> **Parameters explained:**
> - `penirq=4` — GPIO 4 for the IRQ pin (required)
> - `speed=500000` — 500KHz SPI speed, stable for touch reads
> - `xohms=60` — X plate resistance, typical for XPT2046
> - `pmax=255` — maximum pressure value
> - No `cs=` parameter since T_CS is grounded

Your full config.txt additions should now look like:

```ini
dtparam=spi=on
dtoverlay=mipi-dbi-spi,spi0-0,speed=8000000,fps=60
dtparam=width=320,height=240
dtparam=reset-gpio=24,dc-gpio=25
dtparam=write-only
dtoverlay=ads7846,penirq=4,speed=500000,xohms=60,pmax=255
```

---

## Touch Step 4 — Reboot and Verify

```bash
sudo reboot
```

After reboot verify the touch driver loaded:

```bash
dmesg | grep -i ads
```

Expected output:
```
ads7846 spi0.1: touchscreen, irq 171
input: ADS7846 Touchscreen as /devices/.../input/input1
```

Also verify the input device exists:
```bash
sudo evtest
# Should list ADS7846 Touchscreen as one of the devices
```

Test raw touch events:
```bash
sudo evtest /dev/input/event1
```

Press firmly on the screen — you should see `ABS_X`, `ABS_Y` and `ABS_PRESSURE` events.

---

## Touch Step 5 — Map Touch to Display in labwc

Tell the Wayland compositor to map touch input to the SPI display:

```bash
nano ~/.config/labwc/rc.xml
```

Set the content to:

```xml
<?xml version="1.0"?>
<openbox_config xmlns="http://openbox.org/3.4/rc">
    <touch deviceName="ADS7846 Touchscreen" mapToOutput="SPI-1" mouseEmulation="yes"/>
</openbox_config>
```

Reload labwc:
```bash
WAYLAND_DISPLAY=wayland-0 labwc --reconfigure
```

---

## Touch Step 6 — Calibrate the Touchscreen

The raw touch coordinates (0-4095) need to be mapped to screen pixels. Use `libinput debug-events` to get precise corner values:

```bash
sudo apt install evtest libinput-tools -y
```

Run libinput and touch each corner firmly:
```bash
sudo libinput debug-events --show-keycodes 2>&1 | grep -i touch
```

Touch corners in this order and note the normalized percentage values:
1. Top-left
2. Top-right
3. Bottom-left
4. Bottom-right

You will see output like:
```
TOUCH_DOWN  +0.000s  -1 (0) 89.84/93.55
TOUCH_DOWN  +3.148s  -1 (0) 88.45/ 8.28
TOUCH_DOWN  +6.660s  -1 (0)  9.11/91.28
TOUCH_DOWN  +8.248s  -1 (0)  8.98/ 6.52
```

Convert these percentages to 0.0-1.0 by dividing by 100:
```
Top-left:     tx=0.8984, ty=0.9355
Top-right:    tx=0.8845, ty=0.0828
Bottom-left:  tx=0.0911, ty=0.9128
Bottom-right: tx=0.0898, ty=0.0652
```

Calculate the libinput calibration matrix using these formulas:

```
Since Y raw axis maps to screen X and X raw axis maps to screen Y (axes swapped):

b = (screen_x_right - screen_x_left) / (ty_topright - ty_topleft)
  = (1 - 0) / (0.0828 - 0.9355) = -1.1727

c = screen_x_left - (b * ty_topleft)
  = 0 - (-1.1727 * 0.9355) = 1.0970

d = (screen_y_bottom - screen_y_top) / (tx_bottomleft - tx_topleft)
  = (1 - 0) / (0.0911 - 0.8984) = -1.2387

f = screen_y_top - (d * tx_topleft)
  = 0 - (-1.2387 * 0.8984) = 1.1127

Final matrix: 0 b c d 0 f
            = 0 -1.1727 1.0970 -1.2387 0 1.1127
```

Test the matrix live without rebooting:
```bash
DISPLAY=:0 xinput set-prop "ADS7846 Touchscreen" "libinput Calibration Matrix" 0 -1.1727 1.0970 -1.2387 0 1.1127
```

Touch the screen corners and check accuracy. Fine-tune the `c` and `f` offset values if needed:
- If cursor appears to the **right** of where you pressed → decrease `c`
- If cursor appears to the **left** → increase `c`
- If cursor appears **below** where you pressed → decrease `f`
- If cursor appears **above** → increase `f`

Each 0.01 change moves the cursor approximately 3 pixels.

---

## Touch Step 7 — Save Calibration Permanently

Once the matrix feels accurate, save it as a udev rule:

```bash
sudo nano /etc/udev/rules.d/99-touch-calibration.rules
```

Paste (replace with your calculated values):
```
ACTION=="add|change", KERNEL=="event*", ATTRS{name}=="ADS7846 Touchscreen", ENV{LIBINPUT_CALIBRATION_MATRIX}="0 -1.1727 1.0970 -1.2387 0 1.1127"
```

Reload udev and reboot:
```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
sudo reboot
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
| Touch not detected at all | XPT2046 chip missing on board | Physically check for IC chip near touch pins |
| Touch IRQ fires but no events | T_DO (MISO) wire not connected | Add wire from T_DO to Pi pin 21 |
| Touch works with T_CS grounded only | SPI CE1 conflict | Ground T_CS permanently |
| Touch coordinates wrong | Axes swapped or inverted | Recalculate calibration matrix using libinput debug-events |
| Touch offset from actual press | Calibration matrix offset wrong | Adjust `c` and `f` values in udev rule |
| `ts_calibrate` crops/reverses image | tslib uses raw framebuffer incorrectly | Use libinput matrix method instead |

---

## File Summary

| File | Purpose |
|------|---------|
| `/lib/firmware/panel.bin` | ILI9341 binary init sequence |
| `/boot/firmware/config.txt` | Kernel overlay configuration |
| `/etc/modprobe.d/blacklist-fbtft.conf` | Prevents old driver conflicts |
| `~/.config/labwc/autostart` | Wayland autostart (scale, etc.) |
| `~/.config/labwc/rc.xml` | labwc touch device mapping |
| `/etc/udev/rules.d/99-touch-calibration.rules` | Touch calibration matrix |
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

> **Important:** Using `wlr-randr --transform` for rotation causes mouse/touch coordinate mismatches. Always use native orientation via the `0x36` register instead.

---

## Calibration Matrix Reference

The libinput calibration matrix format is `a b c d e f` where:
```
screen_x = a*touch_x + b*touch_y + c
screen_y = d*touch_x + e*touch_y + f
```

All coordinates are normalized to 0.0-1.0. For the XPT2046 on this display with swapped axes:
- `a=0, e=0` (axes are swapped so diagonal values are zero)
- `b` controls X scaling from raw Y
- `c` controls X offset
- `d` controls Y scaling from raw X
- `f` controls Y offset

---

## References

- [panel-mipi-dbi wiki](https://github.com/notro/panel-mipi-dbi/wiki)
- [mipi-dbi-spi overlay README](https://github.com/raspberrypi/linux/blob/rpi-6.6.y/arch/arm/boot/dts/overlays/README)
- [ILI9341 Datasheet](https://cdn-shop.adafruit.com/datasheets/ILI9341.pdf)
- [Adafruit CircuitPython ILI9341](https://github.com/adafruit/Adafruit_CircuitPython_ILI9341)
- [libinput Calibration via udev](https://wayland.freedesktop.org/libinput/doc/latest/pointer-acceleration.html)
- [ads7846 Kernel Driver](https://www.kernel.org/doc/Documentation/devicetree/bindings/input/ads7846.txt)

---

### Made With ❤️ By AhmadTchnology
