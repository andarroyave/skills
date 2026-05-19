---
name: micropython-soc
description: >
  Full workflow for programming SOC boards (ESP32, ESP8266, RP2040, STM32) with MicroPython.
  Covers flashing firmware with esptool or uf2, deploying and managing files with mpremote,
  writing MicroPython scripts for GPIO/I2C/SPI/UART/ADC/PWM peripherals, debugging via serial
  monitor, and installing packages with mip. Use this skill whenever the user mentions MicroPython,
  ESP8266, ESP32, Pico, micropython board, mpremote, esptool, ampy, or wants to blink an LED,
  read a sensor, or write embedded Python code — even if they don't explicitly say "MicroPython skill".
---

# MicroPython SOC Programming

## Workflow overview

1. **Identify the board** — chip family determines the firmware download and flash method
2. **Flash firmware** — get a clean MicroPython runtime onto the board
3. **Connect and verify** — confirm REPL is alive
4. **Write and deploy code** — push files with mpremote
5. **Monitor and debug** — read serial output, iterate

---

## 1. Detect the serial port

```bash
# Linux / macOS
ls /dev/tty* | grep -E 'USB|ACM|usbserial'

# Or let mpremote auto-detect
mpremote connect auto repl
```

Common port names:
- Linux: `/dev/ttyUSB0`, `/dev/ttyACM0`
- macOS: `/dev/cu.usbserial-*`, `/dev/cu.usbmodem*`
- Windows: `COM3`, `COM4`, ...

---

## 2. Flash MicroPython firmware (ESP8266 / ESP32)

### Download firmware
- ESP8266: https://micropython.org/download/ESP8266_GENERIC/
- ESP32: https://micropython.org/download/ESP32_GENERIC/
- RP2040/Pico: download `.uf2` and copy to the drive — no esptool needed

### Install esptool
```bash
pip install esptool
```

### Flash ESP8266
```bash
# Erase flash first (important on re-flash)
esptool.py --port /dev/ttyUSB0 erase_flash

# Flash firmware (.bin file)
esptool.py --port /dev/ttyUSB0 --baud 460800 write_flash --flash_size=detect 0x0 ESP8266_GENERIC-*.bin
```

### Flash ESP32
```bash
esptool.py --chip esp32 --port /dev/ttyUSB0 erase_flash
esptool.py --chip esp32 --port /dev/ttyUSB0 --baud 460800 write_flash -z 0x1000 ESP32_GENERIC-*.bin
```

### Verify
```bash
mpremote connect /dev/ttyUSB0 repl
# You should see the >>> prompt. Ctrl+] to exit.
```

---

## 3. File management with mpremote

`mpremote` is the preferred tool — it handles auto-connect, file sync, and REPL in one command.

```bash
# List files on board
mpremote ls

# Copy a file to the board
mpremote cp main.py :main.py

# Copy entire project directory
mpremote cp -r src/ :

# Run a script without saving it
mpremote run script.py

# Open interactive REPL
mpremote repl

# Execute a one-liner
mpremote exec "import sys; print(sys.implementation)"

# Mount local directory (files auto-sync while connected)
mpremote mount .

# Chain commands
mpremote cp main.py :main.py + reset
```

### Boot files
- `boot.py` — runs once at power-on (put Wi-Fi init here)
- `main.py` — runs after boot.py (your application entry point)

---

## 4. MicroPython patterns

### GPIO — digital I/O
```python
from machine import Pin

led = Pin(2, Pin.OUT)       # ESP8266 built-in LED is GPIO2 (active LOW)
btn = Pin(0, Pin.IN, Pin.PULL_UP)

led.value(0)                # ON (active low)
led.value(1)                # OFF
led.toggle()

# Interrupt
def on_press(pin):
    print("pressed")
btn.irq(trigger=Pin.IRQ_FALLING, handler=on_press)
```

### PWM
```python
from machine import Pin, PWM

pwm = PWM(Pin(14))
pwm.freq(1000)
pwm.duty(512)   # 0–1023 on ESP8266/ESP32
```

### I2C
```python
from machine import I2C, Pin

i2c = I2C(0, scl=Pin(22), sda=Pin(21), freq=400_000)
devices = i2c.scan()        # returns list of addresses
data = i2c.readfrom(0x48, 2)
i2c.writeto(0x48, b'\x01\x00')
```

### SPI
```python
from machine import SPI, Pin

spi = SPI(1, baudrate=1_000_000, polarity=0, phase=0,
          sck=Pin(14), mosi=Pin(13), miso=Pin(12))
cs = Pin(15, Pin.OUT)
cs.value(0)
spi.write(b'\x00')
cs.value(1)
```

### UART
```python
from machine import UART

uart = UART(1, baudrate=9600, tx=17, rx=16)
uart.write("hello\r\n")
if uart.any():
    print(uart.read())
```

### ADC
```python
from machine import ADC, Pin

adc = ADC(Pin(34))          # ESP32: GPIO34 is input-only ADC
adc.atten(ADC.ATTN_11DB)    # 0–3.6V range
val = adc.read()            # 0–4095
```

### Timers
```python
from machine import Timer

tim = Timer(-1)             # virtual timer
tim.init(period=1000, mode=Timer.PERIODIC,
         callback=lambda t: print("tick"))
```

### Wi-Fi (ESP8266/ESP32)
```python
import network

sta = network.WLAN(network.STA_IF)
sta.active(True)
sta.connect("SSID", "password")
while not sta.isconnected():
    pass
print("IP:", sta.ifconfig()[0])
```

### Installing packages with mip
```python
# From REPL on a connected board
import mip
mip.install("urequests")
mip.install("umqtt.simple")
```

Or via mpremote:
```bash
mpremote mip install urequests
```

---

## 5. Debugging

### Serial monitor
```bash
# mpremote built-in (Ctrl+] to exit)
mpremote repl

# Or with screen
screen /dev/ttyUSB0 115200

# Or minicom
minicom -D /dev/ttyUSB0 -b 115200
```

### Soft reset vs hard reset
```python
import machine
machine.soft_reset()   # restarts MicroPython, keeps RAM
machine.reset()        # full hardware reset
```

### Useful diagnostics
```python
import gc, sys, os
print(sys.implementation)   # MicroPython version
print(gc.mem_free())        # free heap
print(os.listdir())         # files on board
```

### Catch exceptions on startup
Wrap `main.py` in a try/except so the board doesn't silently fail:
```python
try:
    run()
except Exception as e:
    import sys
    sys.print_exception(e)
```

---

## 6. Common gotchas

- **ESP8266 GPIO2 LED is active-low** — `Pin(2, OUT).value(0)` turns it ON
- **ESP8266 ADC** has only one channel (A0), input range 0–1V
- **Flash erase before re-flashing** avoids corrupted firmware states
- **Hold BOOT/FLASH button** during esptool flash if it fails to connect
- **`boot.py` crash = bricked loop** — keep boot.py minimal; use `mpremote run` to test before saving
- **File not found on import** — check `os.listdir()` and that the file is in `/` not a subdir
- **Soft reset preserves** connected objects (sockets, peripherals); hard reset clears everything

---

## References

- [MicroPython docs](https://docs.micropython.org/)
- [mpremote docs](https://docs.micropython.org/en/latest/reference/mpremote.html)
- [esptool docs](https://docs.espressif.com/projects/esptool/)
- [MicroPython package index](https://micropython.org/pi/v2/)
