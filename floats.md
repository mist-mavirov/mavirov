# 🌊 Autonomous Profiling Float — Pressure-Based Depth Control System

An Arduino-based **vertical profiling float** designed to perform autonomous underwater missions, hold target depths using **PID control**, log pressure data, and transmit the recorded profile back to a ground station over a wireless serial link.

This project is built around an **MS5837 underwater pressure sensor**, a **brushless ESC-driven thruster**, and a **state-machine-based mission controller** that allows fully automated multi-target dive profiles.

---

## 📌 Table of Contents

- [Overview](#-overview)
- [Features](#-features)
- [Hardware Requirements](#-hardware-requirements)
- [Software Requirements](#-software-requirements)
- [Wiring Diagram](#-wiring-diagram)
- [How It Works](#-how-it-works)
- [Mission State Machine](#-mission-state-machine)
- [PID Control](#-pid-control)
- [Serial Communication Protocol](#-serial-communication-protocol)
- [Configuration](#-configuration)
- [Usage](#-usage)
- [Data Logging](#-data-logging)
- [Tuning Guide](#-tuning-guide)
- [Full Source Code](#-full-source-code)
- [Future Improvements](#-future-improvements)
- [License](#-license)

---

## 🔍 Overview

This float is designed for **MATE ROV–style float missions** or any underwater profiling task that requires:

1. Descending to a target depth (pressure).
2. Holding that depth for a defined duration.
3. Moving to a second target depth.
4. Holding again.
5. Returning to the surface (start pressure).
6. Transmitting time-stamped pressure data to a ground station.

The float is configurable remotely from a **Python ground station** that can upload PID gains, hold time, and tolerance values without re-flashing the Arduino.

---

## ✨ Features

- ✅ **Autonomous multi-target dive profile** (2 target depths + return)
- ✅ **PID-based pressure control** for stable depth holding
- ✅ **State machine architecture** for predictable mission flow
- ✅ **Wireless configuration** via serial radio (`Serial1`)
- ✅ **Onboard data logging** (up to 120 samples)
- ✅ **Post-mission data dump** to ground station
- ✅ **Live serial debug output** through USB (`Serial`)
- ✅ **PWM clamping & integral anti-windup** for safety
- ✅ **Tolerance-based hold detection** to confirm depth lock

---

## 🛠 Hardware Requirements

| Component | Purpose |
|----------|---------|
| Arduino Mega / Arduino with 2 hardware serials | Main microcontroller |
| MS5837-30BA Pressure Sensor | Depth / pressure measurement (I²C) |
| Brushless ESC + Thruster (e.g., T200) | Vertical movement |
| Wireless serial module (HC-12, LoRa, XBee, etc.) | Ground station communication |
| Battery pack (LiPo recommended) | Power source |
| Waterproof enclosure | Pressure hull for electronics |
| Buoyancy adjustment (foam / ballast) | Neutral buoyancy tuning |

---

## 💻 Software Requirements

### Arduino Libraries
- [`Servo.h`](https://www.arduino.cc/reference/en/libraries/servo/) — ESC PWM control
- [`Wire.h`](https://www.arduino.cc/reference/en/language/functions/communication/wire/) — I²C communication
- [`MS5837`](https://github.com/bluerobotics/BlueRobotics_MS5837_Library) — Blue Robotics MS5837 driver

Install through the Arduino Library Manager or directly from GitHub.

### Ground Station
A Python script running on a laptop is expected to:
- Send `CFG,...` packets to update PID & tolerance parameters
- Send `START` to begin the mission
- Listen for `FIRST_READING`, `DATA_BEGIN`, CSV samples, and `DATA_END`

---

## 🔌 Wiring Diagram

```
 ┌─────────────────────────────────────────┐
 │             Arduino (Mega)              │
 │                                         │
 │   Pin 9   ─────────────────► ESC Signal │
 │   SDA/SCL ─────────────────► MS5837     │
 │   TX1/RX1 ─────────────────► Radio TX/RX│
 │   USB     ─────────────────► PC Debug   │
 └─────────────────────────────────────────┘
```

- **ESC_PIN = 9** (PWM output to ESC)
- **MS5837** connected via I²C (default address)
- **Serial1** (TX1/RX1) used for the wireless radio link
- **Serial** (USB) used for live debugging

---

## ⚙️ How It Works

1. On boot, the Arduino arms the ESC with a `STOP_PWM = 1500 µs` signal and initializes the MS5837 sensor.
2. The system idles until a `START` command is received from the ground station.
3. When the mission begins:
   - The current pressure is recorded as the **surface reference** (`startPressure`).
   - The float dives toward `target1 = 1260 mbar`.
   - After reaching it (within tolerance), it holds for `holdDuration` ms.
   - It then moves to `target2 = 1100 mbar` and holds again.
   - Finally, it returns to `startPressure`, stops the motor, and waits.
4. After a 5-second settle period, the float transmits the entire logged dataset over the radio.

---

## 🧭 Mission State Machine

```
IDLE
  │  (receives "START")
  ▼
GO_TARGET1 ──► HOLD_TARGET1
                  │
                  ▼
              GO_TARGET2 ──► HOLD_TARGET2
                                │
                                ▼
                          RETURN_START
                                │
                                ▼
                          FINISHED_WAIT  (5s)
                                │
                                ▼
                            FINISHED
```

Each transition is governed by:
- **Reaching target pressure** within `tolerance` mbar
- **Holding stable** for `holdDuration` ms

---

## 🎛 PID Control

The thruster is controlled by a classic PID loop running every **200 ms**:

```
error      = pressureTarget - currentPressure
derivative = (error - prevError) / dt
integral  += error * dt    (with anti-windup)

PID_output = Kp * error + Ki * integral + Kd * derivative
PWM        = BASE_PWM + PID_output
PWM        = constrain(PWM, STOP_PWM, MAX_PWM)
```

### Default Gains
| Parameter | Value |
|-----------|-------|
| Kp | 0.3 |
| Ki | 0.02 |
| Kd | 0.4 |
| BASE_PWM | 1520 µs |
| MAX_PWM  | 1750 µs |
| STOP_PWM | 1500 µs |

### Anti-Windup
The integral term is only accumulated when the PWM output is **not saturated**, preventing wind-up when the motor is already at its maximum or minimum.

---

## 📡 Serial Communication Protocol

The float listens on `Serial1` for the following commands:

### 1. Configuration Packet
```
CFG,<Kp>,<Ki>,<Kd>,<holdDuration_ms>,<tolerance>
```
Example:
```
CFG,0.35,0.015,0.45,5000,40
```
Response:
```
UPLOAD_DONE
```

### 2. Start Mission
```
START
```
Response:
```
FIRST_READING,<startPressure>
```

### 3. End-of-Mission Data Dump
After completing the mission and waiting 5 seconds, the float transmits:
```
DATA_BEGIN
<time_s>,<pressure_mbar>
<time_s>,<pressure_mbar>
...
DATA_END
```

---

## 🔧 Configuration

Editable constants at the top of the sketch:

```cpp
const int STOP_PWM = 1500;     // Motor stop signal
const int BASE_PWM = 1520;     // Neutral bias above stop
const int MAX_PWM  = 1750;     // Max allowed PWM

float target1 = 1260.0;        // First target pressure (mbar)
float target2 = 1100.0;        // Second target pressure (mbar)

const unsigned long CONTROL_INTERVAL = 200;   // PID period (ms)
const unsigned long LOG_INTERVAL     = 3000;  // Logging period (ms)
const unsigned long DATA_SEND_DELAY  = 5000;  // Wait before TX (ms)
```

Runtime-configurable from the ground station:
- `Kp`, `Ki`, `Kd`
- `holdDuration`
- `tolerance`

---

## ▶️ Usage

1. Flash the sketch to the Arduino.
2. Place the float in water with the radio link connected to the ground station.
3. On the ground station:
   - (Optional) Send a `CFG,...` packet to update PID/tolerance values.
   - Send `START` to begin the mission.
4. Monitor live debug output via USB.
5. After the mission completes, the float transmits all logged data.
6. Plot the `time vs pressure` profile on the ground station.

---

## 📊 Data Logging

- The float logs pressure samples every **3 seconds**.
- Maximum buffer size: **120 samples** (~6 minutes of mission time).
- Each sample is stored as:
  ```cpp
  struct LogData {
    float t;         // seconds since mission start
    float pressure;  // mbar
  };
  ```
- The full log is transmitted as CSV between `DATA_BEGIN` and `DATA_END`.

---

## 🎯 Tuning Guide

If the float overshoots or oscillates:

| Symptom | Fix |
|---------|-----|
| Oscillating around target | Reduce `Kp`, increase `Kd` |
| Slow to reach target | Increase `Kp` |
| Steady-state error | Increase `Ki` slightly |
| Drifts past target | Increase `Kd` |
| Never settles within tolerance | Increase `tolerance` or `holdDuration` |
| Motor saturates constantly | Lower `MAX_PWM` or reduce `Kp` |

Tune **Kp first**, then **Kd**, then **Ki** last (in small steps).

---

## 💾 Full Source Code

```cpp
#include <Servo.h>
#include <Wire.h>
#include <MS5837.h>

Servo esc;
MS5837 sensor;

/* ---------- CONFIG ---------- */
const int ESC_PIN = 9;

const int STOP_PWM = 1500;
const int BASE_PWM = 1520;
const int MAX_PWM  = 1750;

const unsigned long CONTROL_INTERVAL = 200;
const unsigned long LOG_INTERVAL     = 3000;
const unsigned long DATA_SEND_DELAY  = 5000; // wait after motor stop

float target1 = 1260.0;
float target2 = 1100.0;

float pressureTarget = 0;
float startPressure = 0;

/* ---------- VARIABLES FROM PYTHON ---------- */
float tolerance = 50.0;
unsigned long holdDuration = 5000;

float Kp = 0.3;
float Ki = 0.02;
float Kd = 0.4;

/* ---------- PID ---------- */
float error = 0;
float prevError = 0;
float integral = 0;

/* ---------- STATE MACHINE ---------- */
enum State {
  IDLE,
  GO_TARGET1,
  HOLD_TARGET1,
  GO_TARGET2,
  HOLD_TARGET2,
  RETURN_START,
  FINISHED_WAIT,
  FINISHED
};

State state = IDLE;

const char* stateName[] = {
  "IDLE","GO_TARGET1","HOLD_TARGET1",
  "GO_TARGET2","HOLD_TARGET2",
  "RETURN_START","FINISHED_WAIT","FINISHED"
};

bool systemActive = false;

unsigned long lastControl = 0;
unsigned long lastLog = 0;
unsigned long holdStartTime = 0;
unsigned long stopTime = 0;
unsigned long startMillis;

/* ---------- LOGGING ---------- */
struct LogData {
  float t;
  float pressure;
};

LogData logData[120];
int logIndex = 0;

/* ================= SETUP ================= */
void setup() {

  Serial.begin(57600);
  Serial1.begin(57600);

  esc.attach(ESC_PIN);
  esc.writeMicroseconds(STOP_PWM);
  delay(3000);

  Wire.begin();

  if (!sensor.init()) {
    Serial.println("MS5837 init failed");
    while(1);
  }

  sensor.setModel(MS5837::MS5837_30BA);
  sensor.setFluidDensity(997);

  Serial.println("SYSTEM READY");
}

/* ================= LOOP ================= */
void loop() {

  /* ---------- RECEIVE COMMANDS ---------- */
  if (Serial1.available()) {

    String cmd = Serial1.readStringUntil('\n');
    cmd.trim();
    cmd.toUpperCase();

    /* FORMAT:
       CFG,Kp,Ki,Kd,holdTime,tolerance
    */
    if (cmd.startsWith("CFG")) {

      int i1 = cmd.indexOf(',');
      int i2 = cmd.indexOf(',', i1+1);
      int i3 = cmd.indexOf(',', i2+1);
      int i4 = cmd.indexOf(',', i3+1);
      int i5 = cmd.indexOf(',', i4+1);

      Kp = cmd.substring(i1+1, i2).toFloat();
      Ki = cmd.substring(i2+1, i3).toFloat();
      Kd = cmd.substring(i3+1, i4).toFloat();
      holdDuration = cmd.substring(i4+1, i5).toInt();
      tolerance = cmd.substring(i5+1).toFloat();

      Serial1.println("UPLOAD_DONE");
      Serial.println("CONFIG UPDATED FROM PYTHON");
      return;
    }

    if (cmd == "START") {

      sensor.read();
      startPressure = sensor.pressure();

      pressureTarget = target1;
      state = GO_TARGET1;

      integral = 0;
      prevError = 0;
      logIndex = 0;

      startMillis = millis();
      systemActive = true;

      Serial1.print("FIRST_READING,");
      Serial1.println(startPressure);

      logData[logIndex++] = {0.0, startPressure};

      Serial.println("MISSION STARTED");
    }
  }

  if (!systemActive) return;

  unsigned long now = millis();

  /* ---------- PID CONTROL ---------- */
  if (now - lastControl >= CONTROL_INTERVAL &&
      state != FINISHED_WAIT &&
      state != FINISHED) {

    lastControl = now;

    sensor.read();
    float pressure = sensor.pressure();

    error = pressureTarget - pressure;
    float dt = CONTROL_INTERVAL / 1000.0;

    float derivative = (error - prevError) / dt;
    float pid = Kp*error + Ki*integral + Kd*derivative;

    int pwm = BASE_PWM + pid;

    if (pwm < MAX_PWM && pwm > STOP_PWM)
      integral += error * dt;

    prevError = error;

    pwm = constrain(pwm, STOP_PWM, MAX_PWM);
    esc.writeMicroseconds(pwm);

    Serial.print("Time: ");
    Serial.print((now - startMillis)/1000.0,2);
    Serial.print(" | P: ");
    Serial.print(pressure,2);
    Serial.print(" | Target: ");
    Serial.print(pressureTarget);
    Serial.print(" | Err: ");
    Serial.print(error);
    Serial.print(" | PWM: ");
    Serial.print(pwm);
    Serial.print(" | State: ");
    Serial.println(stateName[state]);

    switch(state) {

      case GO_TARGET1:
        if (abs(error) <= tolerance) {
          holdStartTime = now;
          state = HOLD_TARGET1;
        }
        break;

      case HOLD_TARGET1:
        if (abs(error) > tolerance)
          holdStartTime = now;
        else if (now - holdStartTime > holdDuration) {
          pressureTarget = target2;
          state = GO_TARGET2;
        }
        break;

      case GO_TARGET2:
        if (abs(error) <= tolerance) {
          holdStartTime = now;
          state = HOLD_TARGET2;
        }
        break;

      case HOLD_TARGET2:
        if (abs(error) > tolerance)
          holdStartTime = now;
        else if (now - holdStartTime > holdDuration) {
          pressureTarget = startPressure;
          state = RETURN_START;
        }
        break;

      case RETURN_START:
        if (abs(error) <= tolerance) {
          esc.writeMicroseconds(STOP_PWM);
          stopTime = millis();
          state = FINISHED_WAIT;
          Serial.println("MOTOR STOPPED");
        }
        break;

      default:
        break;
    }
  }

  /* ---------- WAIT BEFORE DATA SEND ---------- */
  if (state == FINISHED_WAIT) {

    if (millis() - stopTime > DATA_SEND_DELAY) {

      Serial1.println("DATA_BEGIN");

      for(int i=0;i<logIndex;i++){
        Serial1.print(logData[i].t);
        Serial1.print(",");
        Serial1.println(logData[i].pressure);
      }

      Serial1.println("DATA_END");

      state = FINISHED;
      systemActive = false;

      Serial.println("DATA SENT TO GROUND STATION");
    }
  }

  /* ---------- LOGGING ---------- */
  if (now - lastLog >= LOG_INTERVAL && logIndex < 120) {
    lastLog = now;
    sensor.read();
    float pressure = sensor.pressure();
    float t = (now - startMillis)/1000.0;
    logData[logIndex++] = {t, pressure};
  }
}
```

---

## 🚀 Future Improvements

- [ ] Add temperature compensation using MS5837 temperature output
- [ ] Implement EEPROM storage for PID gains
- [ ] Add SD card logging for longer missions
- [ ] Adaptive PID / gain scheduling for different depths
- [ ] Real-time telemetry instead of post-mission dump
- [ ] Battery voltage monitoring & low-voltage cutoff
- [ ] Surface detection via pressure rate-of-change

---



---

## 🙌 Acknowledgements

- [Blue Robotics](https://bluerobotics.com/) for the MS5837 library and reference hardware.
- The MATE ROV community for inspiring float-based mission designs.
- Arduino & open-source contributors.

---

> Built with 💧 for autonomous underwater exploration.
