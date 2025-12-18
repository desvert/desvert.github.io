---
layout: post
title: "Phase 2: Building the Primary HVAC Controller"
date: 2025-12-17 21:39:14
categories: [blog]
tags: []

---

## Building the Primary HVAC Controller

Over the last couple of weeks, I’ve been working on a small project that blends my HVAC background with my growing interest in embedded systems and OT security. I’m building a miniature “desktop HVAC system” that behaves like the control loops I work with in the field, just scaled down to a breadboard and a handful of sensors.

Phase 1 was all about the basics: wiring a thermistor to an ESP32, reading temperature data, and controlling a small CPU fan with PWM. It worked, and it proved that the idea was viable.

Phase 2 is where things start to look like an actual control system.

---

## **The Goal of This Phase**

My objective was to build a _primary HVAC controller_, a device that:

- Reads multiple environmental sensors
- Makes decisions based on temperature, CO₂, and occupancy
- Drives outputs like a fan, a damper actuator, and an occupancy indicator
- Logs its behavior in structured data for later analysis
- Serves as the “brain” that will eventually report to a Pi Zero dashboard
    

This mirrors a simplified version of how building automation systems (BAS) behave in commercial HVAC: a central controller, some sensor modules, and actuators making decisions every few seconds.

---

## **Hardware Setup**

The controller is built on an Arduino Uno, which handles all real-time control. For this phase I wired up:

### **Inputs**

- **Thermistor (10k NTC)** for backup temperature sensing
- **BME280** for temperature, humidity, and barometric pressure
- **T6613 CO₂ sensor** salvaged from old equipment
- **PIR motion sensor** for occupancy detection
    

### **Outputs**

- **PWM fan control** using an IRLZ44N MOSFET
- **Servo damper actuator** (the “outdoor air damper” of this mini-system)
- **Occupancy LED** on the Arduino’s built-in D13 pin
    

Even with a small breadboard, the system ended up feeling like a compact VAV controller: temperature in, CO₂ in, occupancy in, airflow and ventilation out.

---

## **Creating a Single Configuration Block**

Before writing the control logic, I wanted to avoid hard-coding thresholds everywhere. In real HVAC systems, technicians tune setpoints, deadbands, and damper limits all the time.

So I created a configuration section at the top of the sketch:

```cpp
const float COOL_SETPOINT_F   = 74.0f;
const float COOL_DEADBAND_F   = 2.0f;

const int FAN_DUTY_LOW        = 80;
const int FAN_DUTY_MED        = 160;
const int FAN_DUTY_HIGH       = 255;

const int CO2_LOW_PPM         = 800;
const int CO2_MED_PPM         = 1200;

const int DAMPER_LOW_DEG      = 45;
const int DAMPER_MED_DEG      = 90;
const int DAMPER_HIGH_DEG     = 120;

const unsigned long OCCUPANCY_HOLD_MS = 30000;
```

This section lets me adjust system behavior in one place. Want a different setpoint? A tighter deadband? A more aggressive CO₂ response? It’s a single edit instead of a rewrite.

---

## **Adding Occupancy Logic**

Most real spaces don’t flip between occupied and unoccupied instantly. People move; sensors flicker; a single footstep shouldn’t shut down ventilation.

So I implemented a **30-second occupancy hold**:

```cpp
if (pirState == HIGH) {
  lastMotionMs = now;
}
occupied = (pirState == HIGH) ||
           (now - lastMotionMs < OCCUPANCY_HOLD_MS);
```

This gives the system a more realistic feel: ventilation stays active briefly after motion stops, just like in conference rooms and corridors.

---

## **Environmental Control Logic**

Once the sensor readings were coming in cleanly, I built the logic that drives the fan and damper.

### **Temperature → Fan**

The main temperature source is the BME280, with the thermistor used as a fallback. The logic is straightforward:

- Below setpoint–deadband → **fan off**
- Within deadband → **medium speed**
- Above setpoint+deadband → **full speed**
    

This maps closely to basic cooling-only VAV logic.

---

### **CO₂ → Damper Position**

Ventilation demand increases with CO₂ levels:

- < 800 ppm → damper at 45°
- 800–1200 ppm → damper at 90°
- > 1200 ppm → damper at 120°, and fan forced to at least medium

This is the same concept used in demand-controlled ventilation (DCV). In my small setup, the actuator is just a servo, but it behaves convincingly like a damper motor.

---

## **Structured JSON Output**

Every five seconds, the controller prints a complete snapshot of sensors and outputs:

```json
{
  "sensors": {
    "thermistorF": 74.39,
    "bmeTempF": 75.38,
    "bmeHumidity": 34.22,
    "co2ppm": 842,
    "pir": true,
    "indoorTempF": 75.38
  },
  "outputs": {
    "fanDuty": 160,
    "damper2Angle": 90,
    "occupied": true
  }
}
```

This format is perfect for:

- Logging to a file
- Feeding into my future Pi Zero dashboard
- Analyzing trends over time
- Simulating building automation point trending
    

I also set up a simple Python script to write each JSON line to a log file with timestamps.

---

## **Where This Is Heading Next**

The primary controller is now stable and behaving like a small HVAC controller should. Upcoming phases will build on this:

- The Pi Zero will act as the “supervisor,” receiving JSON from the Arduino
- Sensors on the ESP32 will join the network via Wi-Fi
- A web dashboard will visualize real-time system behavior
- I’ll intentionally introduce OT-security vulnerabilities for practice
- Eventually I’ll simulate a closed-loop “space” with heat gain, fresh-air dilution, and control response
    

This is becoming much more than a hobby project. It’s turning into a functional learning tool for HVAC controls and (eventually) cybersecurity testing.
