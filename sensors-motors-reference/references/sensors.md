# Sensors Reference

---

## HC-SR04 Ultrasonic Distance Sensor

**Protocol:** Digital (trigger pulse + echo timing)
**Voltage:** 5V (3.3V possible with voltage divider on Echo pin for 3.3V MCUs)
**Current:** ~15 mA
**Range:** 2 cm – 400 cm | **Accuracy:** ±3 mm | **Beam angle:** 15°

### Pinout
| Pin | Connect to |
|-----|-----------|
| VCC | 5V |
| GND | GND |
| Trig | Any digital output pin |
| Echo | Any digital input pin (use voltage divider for 3.3V MCUs) |

### How it works
1. Send a 10 µs HIGH pulse on Trig.
2. The sensor emits 8 ultrasonic bursts at 40 kHz.
3. Echo pin goes HIGH for the duration the sound travels to the object and back.
4. Distance = (echo pulse duration in µs) / 58 → cm  |  / 148 → inches

### Arduino code
```cpp
#define TRIG 9
#define ECHO 10

void setup() { Serial.begin(9600); pinMode(TRIG, OUTPUT); pinMode(ECHO, INPUT); }

void loop() {
  digitalWrite(TRIG, LOW); delayMicroseconds(2);
  digitalWrite(TRIG, HIGH); delayMicroseconds(10);
  digitalWrite(TRIG, LOW);
  long duration = pulseIn(ECHO, HIGH);
  float cm = duration / 58.0;
  Serial.println(cm);
  delay(200);
}
```

### MicroPython (ESP32/Pico)
```python
from machine import Pin
import time

trig = Pin(5, Pin.OUT)
echo = Pin(18, Pin.IN)

def distance_cm():
    trig.off(); time.sleep_us(2)
    trig.on();  time.sleep_us(10)
    trig.off()
    while echo.value() == 0: pass
    t1 = time.ticks_us()
    while echo.value() == 1: pass
    t2 = time.ticks_us()
    return time.ticks_diff(t2, t1) / 58.0

while True:
    print(distance_cm(), "cm")
    time.sleep(0.2)
```

### Tips
- Readings below 2 cm or above 400 cm are unreliable — clamp your values.
- Add a 1 kΩ + 2 kΩ voltage divider on the Echo line when using with 3.3V MCUs (ESP32, RP2040).
- Use `pulseIn(ECHO, HIGH, 30000)` to add a timeout and avoid freezing if no echo is received.
- Avoid parallel runs of multiple HC-SR04 units without staggering trigger pulses.

---

## Moisture / Soil Sensor

**Protocol:** Analog (AO) + Digital threshold (DO)
**Voltage:** 3.3V – 5V
**Current:** ~35 mA

### Pinout
| Pin | Description |
|-----|-------------|
| VCC | 3.3V or 5V |
| GND | GND |
| AO  | Analog output (0–1023 on 5V Arduino) |
| DO  | Digital output — HIGH when dry, LOW when moist (threshold set by onboard pot) |

### Reading & interpreting
- **Dry soil:** high ADC value (~700–1023 on Arduino)
- **Wet soil:** low ADC value (~0–300)
- Adjust the blue trimmer pot on the module to set the DO trigger threshold.

### Arduino code
```cpp
#define SOIL_AO A0
#define SOIL_DO 7

void setup() { Serial.begin(9600); pinMode(SOIL_DO, INPUT); }

void loop() {
  int raw = analogRead(SOIL_AO);
  int pct = map(raw, 1023, 0, 0, 100);  // 0=dry, 100=wet
  bool wet = !digitalRead(SOIL_DO);     // DO is active-low
  Serial.print("Moisture: "); Serial.print(pct); Serial.print("% | Wet: "); Serial.println(wet);
  delay(500);
}
```

### Tips
- The probe corrodes over time in soil — use stainless steel probes or capacitive sensors for long-term deployments.
- Capacitive soil sensors (e.g., STEMMA) are more durable and don't corrode.
- Always power the probe only when taking a reading to extend probe life.

---

## pH Sensor

**Protocol:** Analog
**Voltage:** 5V (module), probe output ~0–3.3V mapped to pH 0–14
**Current:** ~5 mA

### Pinout (typical analog pH module)
| Pin | Connect to |
|-----|-----------|
| VCC | 5V |
| GND | GND |
| PO (or AO) | Analog input pin |

### Voltage → pH mapping
pH 7 (neutral) ≈ 2.5V output. Each pH unit ≈ ±0.18V.
- pH < 7 → voltage > 2.5V
- pH > 7 → voltage < 2.5V

### Calibration steps
1. Prepare pH 4.0 and pH 7.0 buffer solutions.
2. Dip the probe in pH 7.0 — note the ADC reading and adjust the module's offset pot until output reads 2.5V.
3. Dip in pH 4.0 — note reading. Use the two points to build a linear map: `pH = m * voltage + b`.

### Arduino code
```cpp
#define PH_PIN A0
float calibration_offset = 0.0;  // adjust after calibration

void setup() { Serial.begin(9600); }

void loop() {
  float voltage = analogRead(PH_PIN) * (5.0 / 1023.0);
  float ph = 3.5 * voltage + calibration_offset;  // rough linear approximation
  Serial.print("pH: "); Serial.println(ph, 2);
  delay(1000);
}
```

### Tips
- Rinse the probe with distilled water between measurements; never wipe dry — pat gently.
- Store the probe in pH 7 buffer or 3 mol/L KCl solution, never in distilled water.
- Allow 1–2 minutes of stabilization after immersing in a new solution.
- Recalibrate regularly — pH probes drift over weeks.

---

## Water Level Sensor

**Protocol:** Analog (AO) + Digital threshold (DO)
**Voltage:** 3.3V – 5V
**Current:** ~20 mA

### Pinout
| Pin | Description |
|-----|-------------|
| VCC | 3.3V–5V |
| GND | GND |
| AO  | Analog (higher value = more water contact) |
| DO  | Digital threshold (adjustable via pot) |

### Notes
- Resistance between exposed traces decreases as more water bridges them → higher ADC reading.
- Use DO with a threshold pot for simple "water reached level X" detection.
- Suitable for non-continuous use; continuous immersion corrodes the traces.

### Arduino code
```cpp
int level = analogRead(A0);
// 0 = dry, ~600+ = fully submerged (varies by sensor)
int pct = map(level, 0, 600, 0, 100);
```

---

## Light Sensors

### LDR (Photoresistor)
**Protocol:** Analog via voltage divider
**Resistance:** ~1 MΩ (dark) → ~100 Ω (bright)

**Wiring (voltage divider):**
```
5V ── LDR ── A0 ── 10kΩ ── GND
```

```cpp
int light = analogRead(A0);  // 0=bright, 1023=dark (with pulldown 10k)
```

**Tip:** LDRs measure relative light level, not calibrated lux. Use BH1750 for accurate lux.

---

### BH1750 (Calibrated Lux Sensor)
**Protocol:** I2C
**Voltage:** 3.3V – 5V
**Range:** 1–65535 lux | **Address:** 0x23 (ADDR=LOW) or 0x5C (ADDR=HIGH)

**Pinout:**
| Pin | Connect to |
|-----|-----------|
| VCC | 3.3V–5V |
| GND | GND |
| SDA | SDA (A4 on Uno) |
| SCL | SCL (A5 on Uno) |
| ADDR | GND (for 0x23) |

```cpp
#include <Wire.h>
#include <BH1750.h>  // install: robtillaart/BH1750
BH1750 lightMeter;
void setup() { Wire.begin(); lightMeter.begin(); }
void loop() {
  float lux = lightMeter.readLightLevel();
  Serial.println(lux);
  delay(1000);
}
```

---

### TEMT6000 (Ambient Light, Phototransistor)
**Protocol:** Analog
**Voltage:** 3.3V–5V
**Wiring:** VCC → 3.3V/5V, GND → GND, SIG → analog pin + 10kΩ pulldown to GND.
Output voltage increases with light intensity.

---

## Temperature Sensors

### DS18B20 (OneWire, Waterproof)
**Protocol:** OneWire (1 data wire)
**Voltage:** 3.0V – 5.5V | **Range:** -55°C to +125°C | **Accuracy:** ±0.5°C

**Wiring:**
```
VCC ── DS18B20 (pin 3) 
GND ── DS18B20 (pin 1)
DATA (pin 2) ── Arduino digital pin + 4.7kΩ pullup to VCC
```

```cpp
#include <OneWire.h>
#include <DallasTemperature.h>
OneWire ow(2); DallasTemperature sensors(&ow);
void setup() { Serial.begin(9600); sensors.begin(); }
void loop() {
  sensors.requestTemperatures();
  Serial.println(sensors.getTempCByIndex(0));
  delay(1000);
}
```

**Tips:**
- The 4.7 kΩ pullup resistor on the data line is mandatory.
- Multiple DS18B20 sensors can share the same data wire (each has a unique 64-bit address).

---

### DHT11 / DHT22 (Temperature + Humidity)
**Protocol:** Single-wire serial (custom)
**Voltage:** 3.3V–5V

| Spec | DHT11 | DHT22 |
|------|-------|-------|
| Temp range | 0–50°C ±2°C | -40–80°C ±0.5°C |
| Humidity | 20–80% ±5% | 0–100% ±2% |
| Sample rate | 1 Hz | 0.5 Hz |
| Cost | cheaper | better accuracy |

**Wiring:** VCC, GND, DATA → digital pin + 10 kΩ pullup to VCC. (Many modules have built-in pullup.)

```cpp
#include <DHT.h>  // install: adafruit/DHT sensor library
DHT dht(2, DHT22);  // pin 2, DHT22 type
void setup() { Serial.begin(9600); dht.begin(); }
void loop() {
  float t = dht.readTemperature();
  float h = dht.readHumidity();
  if (!isnan(t)) { Serial.print(t); Serial.print("°C "); Serial.print(h); Serial.println("%"); }
  delay(2000);
}
```

**MicroPython (DHT22 on ESP32):**
```python
import dht, machine
d = dht.DHT22(machine.Pin(4))
d.measure()
print(d.temperature(), d.humidity())
```

**Tips:**
- Wait at least 2 seconds between readings (1 second for DHT11).
- If you get `nan`, check the pullup resistor and data pin assignment.

---

### LM35 (Analog Temperature)
**Protocol:** Analog
**Voltage:** 4V–30V (output ratiometric to 5V ADC reference)
**Output:** 10 mV/°C | **Range:** -55°C to +150°C

```cpp
float voltage = analogRead(A0) * (5.0 / 1023.0);
float tempC   = voltage * 100.0;  // 10 mV per °C
```

**Tips:** Use the 5V pin as ADC reference for best accuracy. For sub-zero readings, use a negative supply or a voltage offset circuit.

---

## IR Sensors

### TCRT5000 (Reflective / Line Follower)
**Protocol:** Analog (AO) + Digital (DO)
**Voltage:** 3.3V–5V | **Sensing distance:** 1–25 mm

| Pin | Function |
|-----|---------|
| VCC | 3.3V–5V |
| GND | GND |
| DO  | Digital: LOW when surface detected (dark=LOW with typical polarity) |
| AO  | Analog reflectance value |

**Tip:** Threshold is set by onboard pot. White/reflective surface → LOW on DO; Black/absorbing → HIGH on DO (polarity depends on module). Test and adjust pot before deployment.

---

### Obstacle Avoidance IR Module
**Protocol:** Digital
**Voltage:** 3.3V–5V | **Sensing distance:** 2–30 cm (adjustable pot)
- DO = LOW when obstacle detected, HIGH when clear (active low).
- IR LED + phototransistor pair. Adjust pot for desired detection range.

---

## PIR Motion Sensor HC-SR501

**Protocol:** Digital output (active HIGH)
**Voltage:** 5V–12V (typically 5V) | **Current:** ~50 µA standby
**Detection range:** up to 7 m | **Angle:** 120°

### Pinout
| Pin | Description |
|-----|-------------|
| VCC | 5V–12V |
| OUT | Digital output — HIGH (3.3V) when motion detected |
| GND | GND |

### Onboard adjustments (two pots + one jumper)
- **Sensitivity pot:** sets detection range (turn CW = farther)
- **Time pot:** sets how long OUT stays HIGH after trigger (3 sec – 5 min)
- **Jumper:**
  - **H (repeat/retrigger):** OUT stays HIGH while motion continues
  - **L (single trigger):** OUT goes LOW after time delay even if motion persists

### Arduino code
```cpp
#define PIR_PIN 3
void setup() { Serial.begin(9600); pinMode(PIR_PIN, INPUT); delay(30000); } // 30s warmup

void loop() {
  if (digitalRead(PIR_PIN) == HIGH) {
    Serial.println("Motion detected!");
    delay(100);
  }
}
```

**Tips:**
- The sensor needs a 30–60 second warmup period after power-on before readings stabilize.
- Shield the sensor from direct sunlight and heat sources (HVAC vents, lamps).

---

## Pressure / Altitude: BMP280 & BMP180

**Protocol:** I2C (SPI also supported on BMP280)
**Voltage:** 3.3V (use level shifter with 5V Arduinos or get 5V-compatible breakout)
**BMP280 I2C address:** 0x76 (SDO=GND) or 0x77 (SDO=VCC)
**BMP180 I2C address:** 0x77 (fixed)

### Pinout (I2C)
| Module pin | Connect to |
|-----------|-----------|
| VCC / 3V3 | 3.3V |
| GND | GND |
| SDA | SDA |
| SCL | SCL |

```cpp
#include <Wire.h>
#include <Adafruit_BMP280.h>  // or Adafruit_BMP085 for BMP180
Adafruit_BMP280 bmp;
void setup() {
  Serial.begin(9600); Wire.begin();
  bmp.begin(0x76);  // use 0x77 for BMP180
}
void loop() {
  Serial.print(bmp.readTemperature()); Serial.print(" C  ");
  Serial.print(bmp.readPressure() / 100.0); Serial.print(" hPa  ");
  Serial.print(bmp.readAltitude(1013.25)); Serial.println(" m");
  delay(1000);
}
```

**Tips:**
- BMP280 is the newer, more accurate successor to BMP180.
- Pass your local sea-level pressure (in hPa) to `readAltitude()` for accurate altitude.
- BMP280 is 3.3V only — use a level shifter or a 3.3V-to-5V compatible breakout.

---

## Current & Voltage Sensing

### ACS712 (Hall Effect Current Sensor)
**Protocol:** Analog
**Voltage:** 5V | **Variants:** ±5A (185 mV/A), ±20A (100 mV/A), ±30A (66 mV/A)
**Output at 0A:** ~2.5V (Vcc/2)

**Wiring:** Current flows through IP+ and IP- terminals. VCC=5V, GND=GND, OUT=analog pin.

```cpp
float sensitivity = 0.185;  // V/A for ACS712-5A
float vcc = 5.0;
float raw = analogRead(A0) * (vcc / 1023.0);
float current = (raw - vcc / 2.0) / sensitivity;
Serial.println(current);  // Amperes
```

**Tips:**
- The zero-current output is exactly Vcc/2 — calibrate by averaging many readings at 0A.
- Keep current-carrying traces away from signal traces to reduce noise.

---

### INA219 (I2C Current + Voltage + Power)
**Protocol:** I2C | **Address:** 0x40 (default), configurable 0x40–0x4F
**Voltage:** 3.0V–5.5V (supply) | Measures up to 32V bus, ±3.2A

```cpp
#include <Adafruit_INA219.h>
Adafruit_INA219 ina219;
void setup() { Serial.begin(9600); ina219.begin(); }
void loop() {
  float shunt = ina219.getShuntVoltage_mV();
  float bus   = ina219.getBusVoltage_V();
  float mA    = ina219.getCurrent_mA();
  float mW    = ina219.getPower_mW();
  Serial.print(bus); Serial.print("V  "); Serial.print(mA); Serial.print("mA  "); Serial.print(mW); Serial.println("mW");
  delay(500);
}
```

---

## Gas Sensors — MQ Series (MQ-2, MQ-135, etc.)

**Protocol:** Analog (AO) + Digital threshold (DO)
**Voltage:** 5V | **Current:** ~150 mA (heater draws significant current)
**Warmup time:** 20–60 seconds (MQ-2); up to 24 hours for first-use conditioning

| Sensor | Target gases |
|--------|-------------|
| MQ-2   | LPG, propane, H2, smoke, methane |
| MQ-3   | Alcohol/ethanol |
| MQ-5   | LPG, natural gas |
| MQ-7   | Carbon monoxide (CO) |
| MQ-135 | Air quality — CO2, NH3, benzene, smoke |
| MQ-9   | CO, flammable gas |

```cpp
int raw = analogRead(A0);
// Raw value increases with gas concentration
// For calibrated ppm, consult the sensor's datasheet Rs/Ro curve
Serial.print("Raw gas reading: "); Serial.println(raw);
```

**Tips:**
- These sensors need the heater to warm up — wait at least 60 seconds after power-on before trusting readings.
- For accurate ppm conversion, calibrate Rs/Ro ratio in clean air using the datasheet curves.
- MQ sensors consume ~150 mA continuously — don't run them off small LiPo batteries 24/7.
- DO (digital) output + pot gives a simple threshold alarm without needing conversion.

---

## Color & Gesture Sensors

### TCS34725 (RGB Color Sensor)
**Protocol:** I2C | **Address:** 0x29 (fixed)
**Voltage:** 3.3V (5V tolerant on Adafruit breakout) | Has onboard white LED for illumination

```cpp
#include <Wire.h>
#include <Adafruit_TCS34725.h>
Adafruit_TCS34725 tcs = Adafruit_TCS34725(TCS34725_INTEGRATIONTIME_50MS, TCS34725_GAIN_4X);
void setup() { Serial.begin(9600); tcs.begin(); }
void loop() {
  uint16_t r, g, b, c;
  tcs.getRawData(&r, &g, &b, &c);
  Serial.print("R:"); Serial.print(r); Serial.print(" G:"); Serial.print(g);
  Serial.print(" B:"); Serial.print(b); Serial.print(" C:"); Serial.println(c);
  delay(500);
}
```

---

### APDS-9960 (Color + Proximity + Gesture)
**Protocol:** I2C | **Address:** 0x39 (fixed)
**Voltage:** 3.3V (use level shifter with 5V Arduino)
**Capabilities:** RGB color, proximity (0–255), gesture (UP/DOWN/LEFT/RIGHT/NEAR/FAR)

```cpp
#include <SparkFun_APDS9960.h>
SparkFun_APDS9960 apds;
void setup() { Serial.begin(9600); apds.init(); apds.enableGestureSensor(true); }
void loop() {
  if (apds.isGestureAvailable()) {
    switch (apds.readGesture()) {
      case DIR_UP:    Serial.println("UP");    break;
      case DIR_DOWN:  Serial.println("DOWN");  break;
      case DIR_LEFT:  Serial.println("LEFT");  break;
      case DIR_RIGHT: Serial.println("RIGHT"); break;
    }
  }
}
```

---

## Flex & Force Sensors

### FSR (Force Sensitive Resistor)
**Protocol:** Analog via voltage divider
**Resistance:** ~1 MΩ (no force) → ~200 Ω (heavy press)

**Wiring (voltage divider):**
```
5V ── FSR ── A0 ── 10kΩ ── GND
```

```cpp
int raw = analogRead(A0);
float voltage = raw * (5.0 / 1023.0);
float resistance = (5.0 - voltage) / voltage * 10000.0;  // 10k divider
// Lower resistance = more force
Serial.print("Force resistance: "); Serial.println(resistance);
```

---

### Flex Sensor
**Protocol:** Analog via voltage divider
**Resistance:** ~10 kΩ (flat) → ~40 kΩ (90° bend)

**Wiring:** Same voltage divider as FSR. Use 47 kΩ fixed resistor for best range coverage.

```cpp
int raw = analogRead(A0);
int angle = map(raw, 300, 700, 0, 90);  // calibrate these values for your sensor
Serial.print("Bend angle: "); Serial.println(angle);
```

**Tips:**
- Flex sensors are directional — bending in one direction increases resistance; reverse direction has no effect.
- Calibrate the map() range by observing ADC values at known angles.
