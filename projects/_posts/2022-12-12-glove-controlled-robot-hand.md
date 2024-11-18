---
layout: post
title: Glove-controlled robot hand
image: 
  path: /assets/img/projects/robothand_circuit.jpg
description: >
  We built a robot hand from scratch!
sitemap: false
hide_last_modified: true
---

# Glove-controlled robot hand
## Background
This is a final project for a microcontroller class I took during my senior year. Given the challenge to build something interesting with a Raspberry Pi Pico, me and two other teammates settled to the idea of building our own robot hand. The best thing about robot hands is that they can never get tired or injured. They can help us with repetitive and mundane tasks and can operate in conditions that human cannot stand. These characteristics make them perfect for completing tasks in extreme conditions. But the challenge of using a robot hand is that it is very difficult to operate it exactly the way we want. More specifically, we cannot control its fingers like we control ours, which limits its ability to complete delicate tasks. This led us to think if we can build a robot hand that can precisely follow our movements.

Inspired by this idea, our final project is to build a robot hand that can mimic human fingers’ movements. To accomplish this goal, we designed a control glove that collects curvature data from a set of flex sensors and transmits this data via a transceiver to the robot hand. The microcontroller connected to the robot hand then maps this curvature data to the corresponding duty cycle and thus controls the servo movement using PWM. We will discuss the technical details in the following sections.

## High level design
The purpose of the project is to build a robot hand whose actions could be controlled wirelessly by a sensor glove. The implementation consists of three major parts: assembling a 3D-printed robot arm whose gesture would be controlled by servos, soldering flex sensors to fingers of the sensor glove which would be connected to GPIO ports of a RP2040 microcontroller, and establishing wireless communication between the glove and the robot arm with nRF2401 transceivers. The required hardware components are listed as following and the 3D printing arm parts prototype could be referenced in the following website:

<table>
<center>
  <tr>
    <th>Components</th>
    <th>Quantity</th>
    <th>Shopping Website</th>
  </tr>
  <tr>
    <td>Raspberry Pi Pico</td>
    <td>2</td>
    <td>From lab</td>
  </tr>
  <tr>
    <td>nRF24L01 + Transceiver</td>
    <td>2 </td>
    <td><a class="highlight-link" href="https://www.amazon.com/HiLetgo-NRF24L01-Wireless-Transceiver-Module/dp/B00LX47OCY/ref=asc_df_B00LX47OCY/?tag=hyprod-20&linkCode=df0&hvadid=380013417597&hvpos=&hvnetw=g&hvrand=15043867926934858616&hvpone=&hvptwo=&hvqmt=&hvdev=c&hvdvcmdl=&hvlocint=&hvlocphy=9005779&hvtargid=pla-815756879405&psc=1&tag=&ref=&adgrpid=77922879259&hvpone=&hvptwo=&hvadid=380013417597&hvpos=&hvnetw=g&hvrand=15043867926934858616&hvqmt=&hvdev=c&hvdvcmdl=&hvlocint=&hvlocphy=9005779&hvtargid=pla-815756879405" target="_blank"       rel="noreferrer">Amazon</a></td>
  </tr>
  <tr>
    <td>Servo motor</td>
    <td>5</td>
    <td><a class="highlight-link" href="https://www.amazon.com/ETMall-Digital-Helicopter-Compatible-Raspberry/dp/B08CH2SJLR/ref=asc_df_B08CH2SJLR/?tag=hyprod-20&linkCode=df0&hvadid=459643686207&hvpos=&hvnetw=g&hvrand=15996783978418188023&hvpone=&hvptwo=&hvqmt=&hvdev=c&hvdvcmdl=&hvlocint=&hvlocphy=9005779&hvtargid=pla-947377955673&psc=1" target="_blank"       rel="noreferrer">Amazon</a></td>
  </tr>
    <tr>
    <td>Flex sensor</td>
    <td>3</td>
    <td><a class="highlight-link" href="https://www.adafruit.com/product/182?gclid=Cj0KCQjwqoibBhDUARIsAH2OpWhIXkzbfrneTXbraNZmYftoxgLD-aamazqFndEY7rLEM5Z75axPEasaAroxEALw_wcB" target="_blank"       rel="noreferrer">Adafruit</a></td>
  </tr>
  <tr>
    <td>Glove</td>
    <td>1</td>
    <td><a class="highlight-link" href="https://www.amazon.com/-/es/DEX-FIT-resistentes-el%C3%A1stico-inteligente/dp/B07HHZ4PRZ/ref=sr_1_43_mod_primary_new?__mk_es_US=%C3%85M%C3%85%C5%BD%C3%95%C3%91&crid=1PFRXFN4KW460&keywords=electrical%2Bglove&qid=1667437203&qu=eyJxc2MiOiI0LjAxIiwicXNhIjoiMy43MyIsInFzcCI6IjIuODcifQ%3D%3D&sbo=RZvfv%2F%2FHxDF%2BO5021pAnSA%3D%3D&sprefix=electrical%2Bglove%2Caps%2C105&sr=8-43&th=1&language=en_US" target="_blank"       rel="noreferrer">Amazon</a></td>
  </tr>
</center>
</table>

In particular, three flex sensors have been used to measure the amount of deflection of the sensor glove’s fingers. As flex sensors function as variable resistors, we created a voltage divider circuit with 10k resistors for each of the flex sensors. To be more specific, the GND end of the all flex sensors are connected to the GND port of the Raspberry Pi Pico, and the + 3.3 V from the Raspberry Pi are connected to the main positive voltage wire of the voltage divider circuits. The wire from the other end of each flex sensor is connected to separate Analog-to-Digital GPIO ports – 26, 27, and 28 – to reflect how much each flex sensor bends. The specific connection could be seen in Figure 1. 