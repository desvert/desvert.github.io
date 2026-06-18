---

layout: post 
title: "Honest Sensors: Adding a DS18B20 to env-node-01 and Building a Delta T Panel" 
date: 2026-06-02 
categories: 
  - blog 
tags: 
  - homelab 
  - esp32 
  - mqtt 
  - grafana 
  - sensors 
  - hvac 
excerpt: ""

---

# Honest Sensors: Adding a DS18B20 to `env-node-01` and Building a Delta T Panel

`env-node-01` has been running for months. The thermistor reads, the LDR reads, the heartbeat ticks, and the data flows through MQTT into Telegraf, Prometheus, and Grafana without complaint. But a thermistor calibrated against its own readings isn't really calibrated: it's just internally consistent. To know whether the numbers mean anything, you need a reference.

That's what this post is about. A DS18B20 waterproof temperature sensor from the HVAC testbed parts stock, added to `env-node-01`, an outdoor deployment, and a Delta T panel that turned out to be more useful than expected.

---

## The Calibration Backstory

During the initial `env-node-01` build, the thermistor went through a real calibration process. Not assumed datasheet constants: an HVAC probe from the work van held next to the sensor, two stable data points taken at different temperatures, and a two-point Beta calculation done live.

The math produces three constants that go into the firmware:

```c
#define THERMISTOR_NOMINAL  12474.0f
#define TEMPERATURE_NOMINAL 22.06f
#define BCOEFFICIENT        3120.0f
```

Getting there wasn't clean. The ADC resistance formula was inverted in the first pass: `raw / (ADC_MAX - raw)` instead of the correct `(ADC_MAX - raw) / raw`. That produced readings that were directionally backwards. Once that was fixed, the first Beta estimate was still off because the assumed 10kΩ nominal resistance at 25°C didn't match the actual thermistor. The two-point method resolved it by calculating Beta from measured data rather than assuming it.

The calibration held up across months of runtime. But "held up" based on what? The numbers looked reasonable. The trend lines tracked with ambient changes. That's not the same as confirmed accurate.

---

## The DS18B20

The DS18B20 is a digital temperature sensor: OneWire protocol, no ADC math, no calibration required. Accuracy is ±0.5°C across a wide range. A handful of the waterproof variant came in as part of stocking up the electronics lab for the HVAC testbed project. One of them was the right tool for this job.

Development and initial testing happened on a spare ESP32: a 30-pin USB-C WROOM-32 that was sitting unused. The first hurdle was a CP2102 driver issue on Windows. The board showed up under "Other devices" in Device Manager with no COM port assigned. The fix is the Silicon Labs CP210x driver from silabs.com. Once installed, unplug and replug and it appears under Ports with a COM number.

This board's pin numbering maps silkscreened numbers directly to GPIO numbers with no aliasing. GPIO5 is labeled "5". That's cleaner than some ESP32 variants where D0/D1 labeling creates confusion.

The DS18B20 wired to GPIO5 with a 5kΩ pull-up resistor between 3.3V and the data line. The pull-up placement matters: it's a T-junction, not in series:

```
3.3V  -->  [5kΩ]  -->  junction  -->  GPIO5
                           |
                       data wire
```

Wiring the resistor in series between the data wire and GPIO5 is a common mistake. The sensor won't respond.

![Photo of DS18B20 wired to the ESP32 development board showing the pull-up resistor placement](/assets/images/honest-sensors-wiring-showing-resistor-placement.jpg)

---

## Firmware

The DS18B20 integration used Arduino IDE rather than PlatformIO: a different toolchain than my previous ESP32 work. Four libraries needed:

- PubSubClient (Nick O'Leary)
- ArduinoJson (Benoit Blanchon)
- OneWire (Paul Stoffregen)
- DallasTemperature (Miles Burton)

The firmware adds a second temperature reading alongside the existing thermistor, publishing to a new topic:

```
labnet/sensors/env/01/temperature_out
```

The existing pipeline, Telegraf subscribed to `labnet/sensors/#`, picked it up without any configuration changes. A new metric appeared in Prometheus automatically.

One edge case to handle in firmware: if the DS18B20 is disconnected, the DallasTemperature library returns -127°C. Without a check, that value flows straight into Grafana and spikes the panel. The fix is an explicit check before publishing:

```cpp
float tempC = sensors.getTempCByIndex(0);
if (tempC == DEVICE_DISCONNECTED_C) {
  // publish error payload instead
  doc["error"] = "sensor disconnected";
} else {
  doc["celsius"] = tempC;
  doc["fahrenheit"] = (tempC * 9.0 / 5.0) + 32.0;
}
```

Once the firmware was confirmed working on the development board, the DS18B20 was integrated into `env-node-01` directly. The development board was a test fixture, not a permanent second node.

---

## Side by Side

Both sensors in the same environment, readings stabilized. DS18B20: 22.06°C. Thermistor: 22.06°C.

The exact match is worth a moment. 22.06°C is also `TEMPERATURE_NOMINAL`: the reference temperature used in the thermistor calibration constants. That's coincidence, not circularity; the DS18B20 is reporting its own independent measurement. But it meant the validation result was unambiguous. Two sensors, same reading, same temperature the thermistor was calibrated against. The calibration held.

---

## Outside

With validation done, the DS18B20 went outdoors. The waterproof variant handles it. The Grafana panel for the outdoor reading uses a lower clamp bound of -20°F rather than the 50°F used for the indoor thermistor:

```
clamp_min(clamp_max(mqtt_consumer_fahrenheit{topic="labnet/sensors/env/01/temperature_out"}, 120), -20)
```

The clamp suppresses disconnect spikes on both ends without masking legitimate readings. An unconnected DS18B20 returns -127°C (-196°F), well outside the clamp range, so it shows as -20°F rather than a dramatic spike that distorts the panel scale.

The indoor thermistor climbed slowly as the evening progressed. The outdoor panel dropped from ~72°F to ~62°F.

---

## Delta T

One more panel. Prometheus can do the subtraction directly:

```
mqtt_consumer_fahrenheit{topic="labnet/sensors/env/01/temperature"} - ignoring(topic) mqtt_consumer_fahrenheit{topic="labnet/sensors/env/01/temperature_out"}
```

`ignoring(topic)` tells Prometheus to match the two series despite differing label values and subtract them. Result is positive when indoors is warmer than outdoors, negative when outdoors is warmer. A zero-baseline reference line makes the crossover point readable at a glance.

The panel showed a 14.4°F delta with the gap widening in real time.

![Screenshot of the Delta T Grafana panel showing the 14.4°F gap with trend graph](/assets/images/honest-sensors-grafana-delta-t.webp)

That's a number with meaning beyond the homelab. A 14°F indoor-outdoor differential is exactly the kind of signal a real building automation system uses to make economizer decisions: when the outside air is cool enough relative to the return air temperature, you switch from mechanical cooling to free cooling. The HVAC testbed this parts stock was purchased for will eventually compute that logic. This is the first glimpse of what it will look like.

---

## What Came Out of Building It

The DS18B20 added three things the thermistor alone couldn't provide: independent validation, an outdoor reference, and a differential signal. None of that required changes to the existing pipeline. Telegraf, Prometheus, and Grafana handled a new MQTT topic without any reconfiguration. The infrastructure absorbed it.

The disconnect error handling in firmware is easy to skip and worth not skipping. A single -196°F reading compresses the entire Y-axis of a panel and makes everything else unreadable. The check is four lines of code.

The Delta T panel started as a dashboard curiosity and ended up being the most practically useful output this lab has produced so far.