# Motors Reference

---

## Servo Motors (SG90, MG996R)

**Protocol:** PWM (50 Hz, 20 ms period)
**Control:** Pulse width 500–2500 µs → position 0°–180°
  - 500 µs (0.5 ms) = 0°
  - 1500 µs (1.5 ms) = 90° (center)
  - 2500 µs (2.5 ms) = 180°
  - (Many SG90 modules use 1000–2000 µs in practice — test your unit)

| Spec | SG90 (micro) | MG996R (metal gear) |
|------|-------------|---------------------|
| Weight | 9 g | 55 g |
| Torque | 1.8 kg·cm (4.8V) | 10 kg·cm (6V) |
| Speed | 0.1 s/60° | 0.17 s/60° |
| Voltage | 4.8V–6V | 4.8V–7.2V |
| Gear | Plastic | Metal |
| Best for | Light loads, small projects | Heavy loads, robot arms |

### Pinout (3-wire connector)
| Wire color | Function |
|-----------|---------|
| Brown / Black | GND |
| Red / Center | VCC (4.8V–6V) |
| Orange / Yellow / White | Signal (PWM) |

### Arduino code (using Servo library)
```cpp
#include <Servo.h>
Servo myServo;

void setup() {
  myServo.attach(9);  // PWM pin
}

void loop() {
  myServo.write(0);    delay(1000);  // 0°
  myServo.write(90);   delay(1000);  // 90°
  myServo.write(180);  delay(1000);  // 180°
}
```

### Sweep example
```cpp
for (int pos = 0; pos <= 180; pos++) { myServo.write(pos); delay(15); }
for (int pos = 180; pos >= 0; pos--) { myServo.write(pos); delay(15); }
```

### MicroPython (ESP32)
```python
from machine import Pin, PWM
import time

servo = PWM(Pin(13), freq=50)

def set_angle(angle):
    # Map 0-180° to duty cycle 40-115 (out of 1023) for ESP32
    duty = int(40 + (angle / 180.0) * 75)
    servo.duty(duty)

set_angle(0);   time.sleep(1)
set_angle(90);  time.sleep(1)
set_angle(180); time.sleep(1)
```

### Tips
- **Never power servos from the Arduino 5V pin** for MG996R or multiple servos — draw up to 1A each. Use an external 5V/6V supply with a shared GND.
- SG90 can often be powered from Arduino 5V for a single servo doing light work.
- Servo jitter usually means power supply noise — add a 100 µF capacitor across VCC/GND near the servo.
- `myServo.writeMicroseconds(1500)` gives finer control than `write()` in degrees.
- **360° continuous rotation servos** (FS90R, etc.) use the same signal: 1500 µs = stop, <1500 = CW, >1500 = CCW. They ignore `write()` angle — use `writeMicroseconds()`.

---

## Stepper Motors

### 28BYJ-48 + ULN2003 Driver (Unipolar, 5V)

**Type:** Unipolar, 4-phase
**Voltage:** 5V | **Current:** ~240 mA
**Steps per revolution:** 2048 (full step mode, with 1:64 gearbox) | 512 steps (half-step)
**Gear ratio:** 64:1 (approximate) | **Holding torque:** ~300 g·cm

**ULN2003 driver board pins:**
| Driver pin | Arduino pin |
|-----------|------------|
| IN1 | 8 |
| IN2 | 9 |
| IN3 | 10 |
| IN4 | 11 |
| VCC (+) | 5V |
| GND (-) | GND |

**Step sequence (half-step, 8 steps):**

| Step | IN1 | IN2 | IN3 | IN4 |
|------|-----|-----|-----|-----|
| 1 | 1 | 0 | 0 | 0 |
| 2 | 1 | 1 | 0 | 0 |
| 3 | 0 | 1 | 0 | 0 |
| 4 | 0 | 1 | 1 | 0 |
| 5 | 0 | 0 | 1 | 0 |
| 6 | 0 | 0 | 1 | 1 |
| 7 | 0 | 0 | 0 | 1 |
| 8 | 1 | 0 | 0 | 1 |

```cpp
#include <Stepper.h>
// 2048 steps/rev for 28BYJ-48 in half-step mode
Stepper stepper(2048, 8, 10, 9, 11);  // IN1, IN3, IN2, IN4 (note order!)

void setup() { stepper.setSpeed(15); }  // 15 RPM max reliable

void loop() {
  stepper.step(2048);   // one full revolution CW
  delay(500);
  stepper.step(-2048);  // one full revolution CCW
  delay(500);
}
```

**Tips:**
- Wire order to Stepper library is IN1, IN3, IN2, IN4 (not IN1, IN2, IN3, IN4).
- Max reliable speed is ~15 RPM. Above 20 RPM it loses torque and starts missing steps.
- Turn off the coils after positioning to save power and reduce heat: set all pins LOW.
- The 28BYJ-48 is geared — it's slow but has good torque for its size.

---

### NEMA17 + A4988 / DRV8825 Driver (Bipolar)

**NEMA17 specs:**
- **Voltage:** 12V typical (range varies by model) | **Current/coil:** 1–2 A
- **Step angle:** 1.8°/step = **200 steps/revolution** (full step)
- **Holding torque:** 40–50 N·cm (varies by model)
- **Wire colors (typical):** A coil = 1A (red) + 2A (green); B coil = 1B (blue) + 2B (black)

#### A4988 Driver

**Features:** Up to 2A/coil, microstepping: full, 1/2, 1/4, 1/8, 1/16
**Logic voltage:** 3.3V–5V | **Motor voltage:** 8V–35V

**Wiring:**
| A4988 pin | Connect to |
|-----------|-----------|
| VMOT | 12V motor supply |
| GND (motor) | Motor GND |
| VDD | 5V (logic) |
| GND (logic) | Arduino GND |
| STEP | Arduino digital pin (e.g. 3) |
| DIR | Arduino digital pin (e.g. 4) |
| ENABLE | GND (to enable) or leave floating |
| MS1, MS2, MS3 | Microstepping select (see table below) |
| 1A, 1B | Motor coil A |
| 2A, 2B | Motor coil B |

**Microstepping (A4988):**
| MS1 | MS2 | MS3 | Mode |
|-----|-----|-----|------|
| L | L | L | Full step |
| H | L | L | 1/2 step |
| L | H | L | 1/4 step |
| H | H | L | 1/8 step |
| H | H | H | 1/16 step |

**Current limiting (CRITICAL):**
Set Vref voltage on the trimpot:  `Vref = Imax × 8 × Rsense`
Typical Rsense = 0.1 Ω → for 1A: Vref = 0.8V. Measure at the trimpot center pad with motor powered.

```cpp
#define STEP_PIN 3
#define DIR_PIN  4

void setup() {
  pinMode(STEP_PIN, OUTPUT);
  pinMode(DIR_PIN, OUTPUT);
}

void stepMotor(int steps, bool direction, int speedDelay = 1000) {
  digitalWrite(DIR_PIN, direction ? HIGH : LOW);
  for (int i = 0; i < steps; i++) {
    digitalWrite(STEP_PIN, HIGH); delayMicroseconds(speedDelay);
    digitalWrite(STEP_PIN, LOW);  delayMicroseconds(speedDelay);
  }
}

void loop() {
  stepMotor(200, true,  1000);  // 1 rev CW  at 1000µs/step
  delay(500);
  stepMotor(200, false, 1000);  // 1 rev CCW
  delay(500);
}
```

#### AccelStepper library (recommended for smooth motion)
```cpp
#include <AccelStepper.h>
AccelStepper stepper(AccelStepper::DRIVER, 3, 4); // STEP=3, DIR=4

void setup() {
  stepper.setMaxSpeed(1000);   // steps/sec
  stepper.setAcceleration(500);
}

void loop() {
  stepper.moveTo(1600);  // 8 full revs at 1/8 microstepping
  while (stepper.distanceToGo() != 0) stepper.run();
  delay(500);
  stepper.moveTo(0);
  while (stepper.distanceToGo() != 0) stepper.run();
  delay(500);
}
```

#### DRV8825 vs A4988
| Feature | A4988 | DRV8825 |
|---------|-------|---------|
| Max current | 2 A | 2.5 A |
| Microstepping | up to 1/16 | up to 1/32 |
| Voltage | 8–35V | 8.2–45V |
| Vref formula | Imax × 8 × Rsense | Imax × 5 × Rsense |
| Step/Dir logic | same | same |

**Tips for NEMA17 + drivers:**
- **Always add a 100 µF electrolytic capacitor** across VMOT and GND near the driver to absorb voltage spikes. Missing this cap can kill the driver instantly when disconnecting power.
- Set current limit before connecting the motor — overcurrent causes overheating.
- Never disconnect the motor while the driver is powered — back-EMF can destroy the chip.
- For CNC/3D printing use the AccelStepper library; manual pulses are fine for simple positioning.
- DRV8825 needs a minimum 2 µs pulse on STEP (vs 1 µs for A4988) — adjust `delayMicroseconds` accordingly.

---

## DC Motors — L298N / L293D

### L298N Dual H-Bridge

**Motor voltage:** 5V–35V | **Max current:** 2A/channel (peak 3A)
**Logic voltage:** 5V | **Channels:** 2 independent DC motors (or 1 stepper)
**Current:** Has onboard 5V regulator — can power Arduino from the 5V pin when motor voltage ≥ 7V

**Wiring:**
| L298N terminal | Connect to |
|---------------|-----------|
| +12V / VMS | Motor supply (6V–35V) |
| GND | Common GND |
| 5V (output) | Can power Arduino |
| ENA | PWM pin (speed for motor A); jumper = always-on |
| IN1, IN2 | Direction control motor A |
| OUT1, OUT2 | Motor A terminals |
| ENB | PWM pin (speed for motor B); jumper = always-on |
| IN3, IN4 | Direction control motor B |
| OUT3, OUT4 | Motor B terminals |

**Direction truth table:**
| IN1 | IN2 | Motor A |
|-----|-----|---------|
| H | L | Forward |
| L | H | Reverse |
| H | H | Brake |
| L | L | Free spin (coast) |

```cpp
#define ENA 10  // PWM
#define IN1 9
#define IN2 8

void setup() { pinMode(ENA,OUTPUT); pinMode(IN1,OUTPUT); pinMode(IN2,OUTPUT); }

void motorForward(int speed) {
  analogWrite(ENA, speed);    // 0-255
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
}

void motorReverse(int speed) {
  analogWrite(ENA, speed);
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, HIGH);
}

void motorStop() { analogWrite(ENA, 0); }

void loop() {
  motorForward(200); delay(2000);
  motorStop();       delay(500);
  motorReverse(180); delay(2000);
  motorStop();       delay(500);
}
```

**Tips:**
- The L298N runs hot — add a heatsink. Above 1A continuous, a heatsink is mandatory.
- At motor voltages below 6V, disable the onboard 5V regulator jumper and supply 5V separately to the logic pin — the regulator has a ~2V drop.
- Add flyback diodes if your L298N module doesn't have them (most breakout boards do).

---

### L293D
- Smaller, lower current (600 mA / channel, 1.2A peak).
- Logic and motor voltage can differ: Vss = 5V logic, Vs = motor supply.
- Good for small motors and fans.
- Same IN1/IN2/EN logic as L298N.
- Includes built-in flyback diodes on the D variant (L293D vs L293).

---

## Brushless DC (BLDC) + ESC

**Protocol:** PPM/PWM signal: 1000–2000 µs pulse, 50 Hz (same as servo)
- **1000 µs** = motor off (0% throttle)
- **1500 µs** = ~50% throttle
- **2000 µs** = full throttle (100%)

**ESC calibration (first use):**
1. Power on with signal at 2000 µs (full throttle) → hear two beeps.
2. Reduce to 1000 µs → hear confirmation beeps.
3. ESC is now calibrated.

**ESC arming sequence (every power-on):**
1. Set signal to 1000 µs before powering the ESC.
2. Wait for ESC arming beeps.
3. Slowly increase throttle.

```cpp
#include <Servo.h>  // ESCs use the same PWM as servos
Servo esc;

void setup() {
  esc.attach(9, 1000, 2000);  // pin, min µs, max µs
  esc.writeMicroseconds(1000); // arm the ESC
  delay(2000);
}

void loop() {
  esc.writeMicroseconds(1200);  // slow speed
  delay(3000);
  esc.writeMicroseconds(1000);  // stop
  delay(2000);
}
```

**Tips:**
- **Always arm the ESC at 1000 µs before ramping up** — spinning up without arming can cause sudden full-throttle.
- BLDCs require the ESC to match the motor's KV rating and your battery voltage (2S/3S/4S LiPo).
- For bidirectional BLDC (robots, not drones), use a bidirectional ESC (1000 µs = full reverse, 1500 µs = stop, 2000 µs = full forward).
- Never run a drone ESC without a propeller attached indoors — untethered props are dangerous.

---

## Linear Actuators

**Control options:** PWM (via motor driver), relay (for simple on/off extend/retract), or dedicated controller.
**Common specs:**
- **Stroke length:** 50 mm – 300 mm typical
- **Force:** 20 N – 2000 N
- **Speed:** 5 mm/s – 50 mm/s (inverse relationship with force)
- **Voltage:** 12V most common (24V for high-force industrial)
- **Current:** 0.5A–5A depending on load

### Wiring with L298N (PWM speed control)
Same as DC motor — actuator has two wires that map to OUT1/OUT2.
- **IN1=H, IN2=L** → extend
- **IN1=L, IN2=H** → retract
- **ENA PWM** → speed control (if needed)

### Simple relay control (extend/retract only, no speed control)
```
Relay 1 NO → Actuator+ → 12V
Relay 2 NO → Actuator- → GND
For retract: swap polarity via relay 2
```

```cpp
// Simple extend/retract with L298N, no speed control
#define IN1 7
#define IN2 6

void extend()  { digitalWrite(IN1,HIGH); digitalWrite(IN2,LOW);  }
void retract() { digitalWrite(IN1,LOW);  digitalWrite(IN2,HIGH); }
void stop()    { digitalWrite(IN1,LOW);  digitalWrite(IN2,LOW);  }

void setup() { pinMode(IN1,OUTPUT); pinMode(IN2,OUTPUT); }
void loop() {
  extend();  delay(5000);  // extend for 5 sec (adjust for your stroke length)
  stop();    delay(1000);
  retract(); delay(5000);
  stop();    delay(1000);
}
```

### Position feedback
Higher-end actuators include a built-in potentiometer (typically 0–10 kΩ across stroke).
Wire pot wiper to an analog input to read position:

```cpp
int pos = analogRead(A0);  // 0 = fully retracted, 1023 = fully extended
```

**Tips:**
- Use a motor driver (L298N, VNH5019) rated above the actuator's stall current — stall current can be 5–10× running current.
- Always wire a limit to stop the actuator at end of travel if no internal limit switches are present.
- For heavy loads, add a 0.1–1 A fuse in series for protection.
