---
layout: post
title: Desktop HVAC Lab - Phase 1
date: 2025-11-26 23:56:47
categories: [blog]
tags: []
excerpt: ""
---
# **Building a Desktop HVAC Lab: Project Overview and Phase One Progress**

I am starting a new hands-on project that has been in the back of my mind for a while. The goal is to build a small HVAC control lab on my desk that behaves like a scaled-down version of the systems I work with in commercial HVAC. I want a simple environment where I can experiment with sensors, actuators, controls logic, and eventually the types of cybersecurity questions that show up in building automation and energy management systems.

The long-term plan is to build something that follows the same pattern used in real equipment:

**Sensors → Controller → Logic → Actuators → Monitoring Interface**

I have a mix of hardware to work with, including ESP32 boards, Arduino clones, a Raspberry Pi Pico, and a Raspberry Pi Zero 2W that will eventually serve as a small “supervisory” device. I have also salvaged several components from retired HVAC equipment, such as a T6613 CO2 sensor and a thermistor module. My aim is to build up this environment step by step, testing real control strategies along the way.

Phase One focused on a simple but important loop: measure temperature and use it to control a fan.

---

## **Phase One: Temperature Measurement and Fan Control**

I started with an ESP32 dev board, a 10k NTC thermistor, and a small 5 V CPU fan that I pulled from an old laptop. The idea was to read temperature from the thermistor and then use PWM to adjust the speed of the fan. This would give me the basic structure of a control loop that behaves like a tiny VAV box or a simple cooling circuit.

The first part was getting reliable temperature readings. I wired the thermistor into a voltage divider and wrote a small sketch to convert the ADC reading to a temperature using the beta coefficient formula. I printed the output to the serial monitor to make sure everything behaved as expected.

The next step was controlling the fan. I wired the CPU fan through an IRLZ44N MOSFET so the ESP32 could drive it with PWM. The fan runs on the 5 V rail of my breadboard power module, and the ESP32 runs from the 3.3 V rail. Both rails share a common ground. The ESP32 drives the MOSFET gate through a 220 ohm resistor, with a 100k pulldown to keep the fan off when the ESP32 is not driving the pin. A 1N4007 diode across the fan protects the circuit from voltage spikes.

Once everything was wired up, the PWM control worked immediately. I could see the duty cycle change in the serial output, and the fan speed matched it physically. This confirmed that the basic output side of the loop was working.

The temperature readings were another story. When I touched the thermistor to warm it, the reported temperature went down instead of up. As a result, the fan slowed down when it should have been speeding up. At first I wondered if this was related to a known quirk on this particular development board where the built-in LED is active-low. After a little troubleshooting, I realized the problem was simpler. I had wired the voltage divider backwards.

The math in my code assumes the divider is arranged like this:

`3.3 V → 10k resistor → ADC node → thermistor → GND`

I had it reversed, so the ADC saw the voltage move in the opposite direction from what the math expected. As a result, the Steinhart calculation inverted the temperatures. I rewired the divider to match the intended layout, and everything lined up. Warming the thermistor raised the temperature reading and increased the fan speed. Cooling it lowered both values.

With that correction, Phase One was complete. I now have a working loop where the ESP32 reads a sensor value, processes it, and drives an actuator with PWM. Everything is monitored through the serial console, and the system behaves predictably.

---

## **What Comes Next**

This first step gives me the core mechanics that HVAC controls are built on. The next phases will add more sensors and more realistic behavior. I plan to bring the salvaged T6613 CO2 sensor online to build a basic demand-controlled ventilation demo. I also want to add servos to simulate dampers, and then bring the Raspberry Pi Zero into the mix as a small EMS-style front end.

Later I will expand the system to explore networked communication, logging, and security. This will create a hands-on platform where I can study how real building control systems behave, and how they break when they are designed without security in mind.

For now, Phase One is up and running. It is a small step, but it forms the foundation for everything else that will follow.
