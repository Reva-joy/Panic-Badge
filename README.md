Demo video - https://drive.google.com/file/d/15SsNjb7OKQsSOXbOdoBUXPjAK4xVujoP/view?usp=drivesdk 


Panic Badge – Smart Safety Device

The Panic Badge is an innovative wearable safety device designed to provide discreet and instant panic alerts amid rising crime rates. Unlike existing solutions that depend on smartphones or visible panic buttons, this badge enables silent, tap-pattern-based activation and ensures immediate communication of distress signals through location sharing and audio playback.

The device is especially useful for women, children, and the elderly, offering a quick and reliable way to call for help without drawing unwanted attention.The Panic Badge addresses these issues by providing a portable, discreet, and tap-activated emergency alert system.

Key Features

Tap Pattern Activation – Capacitive touch sensor detects predefined tap sequences.

Real-Time GPS Tracking – Location automatically shared during distress.

4G LTE Communication – Sends SMS alerts and calls emergency contacts.

Pre-recorded Audio Playback – Plays distress message for public awareness.

Compact & Portable – Designed like an ID badge for daily wear.

Battery Management System (BMS) – Safe charging and power regulation for reliable use.

"Serial communication Protocol (UART) is used "
Hardware Components

Renesas RA6E2 Microcontroller – Controls all modules.

Capacitive Touch Sensor (HW-139) – Panic trigger via tap patterns.

Quectel EC200U (4G LTE Module) – SMS/call communication.

Neo-6M GPS Module – Provides real-time latitude & longitude.

ISD1820 Audio Playback Module – Plays pre-recorded alarm/distress audio.

18650 Li-ion Battery with TP4056 Charger & BMS – Rechargeable safe power source.

RT9048 LDO Regulator – Converts 3.7 V battery to stable 3.3 V.

System Architecture Flow:

User performs a tap pattern on the badge.

RA6E2 MCU validates the pattern.

If valid:

Sends GPS coordinates via SMS to emergency contacts.

Plays a pre-recorded distress audio message.

Ensures discreet yet effective alert system.


