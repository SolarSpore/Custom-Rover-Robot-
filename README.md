# ESP32 4WD Rover: Modular Robotics Platform

A Wi-Fi-connected, 4-wheel-drive rover built around an ESP32, dual L298N motor drivers, and a growing stack of ROS 2 tooling. This robot is the first hardware node in a broader modular robotics platform aimed at multi-robot control, telemetry, OTA updates, and home automation integration.

Design is loosely inspired by the classic Arduino/ESP32 obstacle-avoiding and line-tracking smartcar chassis builds, reworked around an ESP32 as the main controller and a 4-motor, dual-driver layout instead of a single H-bridge.

---

## Current Status

| System | Status |
|---|---|
| Chassis + 4WD drivetrain | Assembled |
| Dual L298N wiring | Wired and driving |
| ESP32 main controller | Flashed, Wi-Fi connected |
| ESP32 firmware (this repo) | Working: UDP motor control, OTA, failsafe stop |
| Steam Deck to ESP32 link (ROS 2) | In progress |
| Foxglove Studio telemetry view | In progress |
| Head swivel (pan servo) | Not connected |
| Ultrasonic sensors ("eyes") | Not connected |
| IR line/obstacle sensors | Not connected |
| Multi-robot support | Planned |

This is an active work in progress. The rover currently drives (teleop-capable via UDP) but has no onboard sensing yet. Obstacle avoidance, line tracking, and the head/camera swivel from the original inspiration build are all still on the bench.

---

## Hardware

- **Controller:** ESP32 (Wi-Fi enabled, replacing the Arduino Uno used in similar builds)
- **Motor drivers:** 2x L298N Dual H-Bridge
- **Drive:** 4x DC gear motors, independent front/rear left/right control
- **Compute (off-board):** Steam Deck running Linux, ROS 2, and Foxglove Studio
- **Not yet installed:** pan servo (head swivel), HC-SR04 ultrasonic sensors, IR sensors

### Motor Wiring

Each wheel gets its own pair of GPIOs, driven directly as PWM forward/reverse pairs (this assumes the L298N ENA/ENB jumpers are left in place, tied HIGH):

| Wheel | GPIO A | GPIO B | LEDC Channels |
|---|---|---|---|
| Front Left (FL) | 26 | 27 | 0, 1 |
| Front Right (FR) | 18 | 19 | 2, 3 |
| Rear Left (RL) | 21 | 22 | 4, 5 |
| Rear Right (RR) | 23 | 25 | 6, 7 |

PWM frequency is 5000 Hz at 8-bit resolution (0-255), matching the -255..255 command range directly.

---

## Network

| Item | Value |
|---|---|
| UDP port | `4210` |
| UDP protocol | `"FL,FR,RL,RR"`, each value `-255..255` |
| Wi-Fi connect timeout | 8 seconds, then falls back to AP mode |
| Fallback AP | SSID/password set in `config.h` (`ap_ssid` / `ap_password`) |
| mDNS / OTA hostname | set in `config.h` (`mdnsHostname`), resolves as `<mdnsHostname>.local` |
| OTA port | `3232` (ArduinoOTA default) |
| OTA password | set in `config.h` (`ota_password`), must match `platformio.ini`'s `--auth` flag |

**No static IP is configured in firmware.** The board takes whatever address your router's DHCP hands out. If you want the IP to stay stable across reboots (recommended, since OTA in `esp32dev-ota` targets a fixed IP), set a DHCP reservation on your router for the board's MAC address, printed to Serial isn't currently implemented, add `Serial.println(WiFi.macAddress())` in `setupNetwork()` if you need to read it off, or check your router's DHCP client list.

---

## Software Stack

- **Firmware:** ESP32, Arduino framework, built and flashed via PlatformIO (this repo)
- **Middleware:** ROS 2 (running on the Steam Deck, communicating with the ESP32 over Wi-Fi via UDP)
- **Visualization/telemetry:** Foxglove Studio
- **Host OS:** Linux (Steam Deck in desktop mode)

The firmware itself never runs ROS: it only speaks the UDP protocol above. ROS 2 integration lives on the Steam Deck side, in the `rover_udp_bridge` package, which is what Foxglove's Teleop panel and a future Nav2 stack talk to. The goal is a reusable foundation: the same ESP32 + ROS 2 + Foxglove pattern should be able to support additional robots beyond this rover, with shared tooling for control, telemetry, and updates.

---

## Firmware

PlatformIO project layout:

```
.
├── platformio.ini
├── config.h.example   # copy to config.h and fill in real values (gitignored)
└── src/
    └── main.cpp
```

### Setup

```bash
cp config.h.example config.h
# then edit config.h with your real Wi-Fi SSID/password, AP credentials,
# mDNS hostname, and OTA password
```

`config.h` is gitignored on purpose, your Wi-Fi credentials and OTA password never get committed.

### `config.h.example`

```cpp
#pragma once

// ------------------------------------------------------------------
// config.h.example
//
// Copy this file to config.h and fill in real values. config.h is
// gitignored so your Wi-Fi credentials and OTA password never end up
// in version control.
//
//   cp config.h.example config.h
// ------------------------------------------------------------------

// Home Wi-Fi network the rover tries first.
const char* sta_ssid     = "YOUR_WIFI_SSID";
const char* sta_password = "YOUR_WIFI_PASSWORD";

// Fallback access point, started if the home network is unreachable
// at boot.
const char* ap_ssid     = "ESP32-Rover";
const char* ap_password = "CHOOSE_AN_AP_PASSWORD"; // 8+ characters required

// mDNS / OTA hostname -> shows up as <mdnsHostname>.local
const char* mdnsHostname = "esp32rover";

// OTA password. Must match the --auth value in platformio.ini's
// esp32dev-ota environment.
const char* ota_password = "YOUR_OTA_PASSWORD";
```

### `platformio.ini`

```ini
[env:esp32dev]
platform = espressif32
board = esp32dev
framework = arduino
upload_port = COM10
monitor_speed = 115200

[env:esp32dev-ota]
platform = espressif32
board = esp32dev
framework = arduino
upload_protocol = espota
upload_port = 192.168.1.26
monitor_speed = 115200

upload_flags =
    --auth=YOUR_OTA_PASSWORD
```

Replace `YOUR_OTA_PASSWORD` with the real value locally (matching `ota_password` in your `config.h`), don't commit the real password here either. Also update `upload_port` under `esp32dev-ota` if your rover's IP differs, and `upload_port` under `esp32dev` to match your machine's serial port (`COM10` on Windows, `/dev/ttyUSB0` on Linux, `/dev/cu.usbserial-XXXX` on macOS).

### `src/main.cpp`

```cpp
/**********************************************************************
  ESP32 4-Motor UDP Rover Controller + OTA

  Hardware:
    - ESP32
    - 2x L298N
    - 4x DC gear motors
    - Motors: FL, FR, RL, RR

  UDP command format:
    "FL,FR,RL,RR"

  Motor range:
    -255 .. 255

  Failsafe:
    Stops all motors if no valid packet is received for FAILSAFE_MS.

  Network:
    1. Try configured Wi-Fi network.
    2. If unavailable, start fallback access point.
    3. mDNS hostname: esp32rover.local

  OTA:
    - ArduinoOTA
    - OTA hostname: esp32rover
    - OTA password configured in config.h
    - Motors are stopped when an OTA update starts
    - USB remains the recovery method
**********************************************************************/

#include "config.h"

#include <Arduino.h>
#include <WiFi.h>
#include <WiFiUdp.h>
#include <ESPmDNS.h>
#include <ArduinoOTA.h>

// ============================================================
// Network
// ============================================================

const unsigned long STA_TIMEOUT_MS = 8000;

const unsigned int UDP_PORT = 4210;

WiFiUDP udp;

char packetBuffer[128];

// ============================================================
// Motor configuration
// ============================================================

struct MotorPins {
  int pinA;
  int pinB;
};

// FL, FR, RL, RR
MotorPins motors[4] = {
  {26, 27},  // FL
  {18, 19},  // FR
  {21, 22},  // RL
  {23, 25}   // RR
};

// Each motor gets two PWM channels.
//
// Motor 0: channels 0, 1
// Motor 1: channels 2, 3
// Motor 2: channels 4, 5
// Motor 3: channels 6, 7

const int PWM_FREQ = 5000;
const int PWM_RES  = 8;

// ============================================================
// Failsafe
// ============================================================

unsigned long lastPacketMs = 0;

const unsigned long FAILSAFE_MS = 500;

// ============================================================
// Motor control
// ============================================================

void setMotor(int index, int speed) {
  if (index < 0 || index >= 4) {
    return;
  }

  speed = constrain(speed, -255, 255);

  int channelA = index * 2;
  int channelB = index * 2 + 1;

  if (speed > 0) {
    // Forward
    ledcWrite(channelA, speed);
    ledcWrite(channelB, 0);
  }
  else if (speed < 0) {
    // Reverse
    ledcWrite(channelA, 0);
    ledcWrite(channelB, -speed);
  }
  else {
    // Stop
    ledcWrite(channelA, 0);
    ledcWrite(channelB, 0);
  }
}

void stopAll() {
  for (int i = 0; i < 4; i++) {
    setMotor(i, 0);
  }
}

void setupMotors() {
  for (int i = 0; i < 4; i++) {
    int channelA = i * 2;
    int channelB = i * 2 + 1;

    ledcSetup(channelA, PWM_FREQ, PWM_RES);
    ledcSetup(channelB, PWM_FREQ, PWM_RES);

    ledcAttachPin(motors[i].pinA, channelA);
    ledcAttachPin(motors[i].pinB, channelB);
  }

  stopAll();
}

// ============================================================
// UDP command parser
// ============================================================

bool parseCommand(char* command) {
  int values[4] = {0, 0, 0, 0};

  int index = 0;

  char* token = strtok(command, ",");

  while (token != nullptr && index < 4) {
    values[index] = atoi(token);
    index++;

    token = strtok(nullptr, ",");
  }

  // We require exactly four motor values.
  if (index != 4) {
    return false;
  }

  for (int i = 0; i < 4; i++) {
    values[i] = constrain(values[i], -255, 255);
  }

  for (int i = 0; i < 4; i++) {
    setMotor(i, values[i]);
  }

  return true;
}

// ============================================================
// Network setup
// ============================================================

bool setupNetwork() {

  WiFi.mode(WIFI_STA);

  Serial.print("Trying to join Wi-Fi network: ");
  Serial.println(sta_ssid);

  WiFi.begin(sta_ssid, sta_password);

  unsigned long startAttempt = millis();

  while (
    WiFi.status() != WL_CONNECTED &&
    millis() - startAttempt < STA_TIMEOUT_MS
  ) {
    delay(300);
    Serial.print(".");
  }

  Serial.println();

  // ----------------------------------------------------------
  // Normal Wi-Fi mode
  // ----------------------------------------------------------

  if (WiFi.status() == WL_CONNECTED) {

    Serial.println("Wi-Fi connected.");

    Serial.print("IP address: ");
    Serial.println(WiFi.localIP());

  }

  // ----------------------------------------------------------
  // Fallback AP mode
  // ----------------------------------------------------------

  else {

    Serial.println(
      "Home network unavailable."
    );

    Serial.println(
      "Starting fallback access point."
    );

    WiFi.disconnect(true);

    delay(100);

    WiFi.mode(WIFI_AP);

    bool apStarted = WiFi.softAP(
      ap_ssid,
      ap_password
    );

    if (apStarted) {

      Serial.print("Access point started: ");
      Serial.println(ap_ssid);

      Serial.print("Robot IP address: ");
      Serial.println(WiFi.softAPIP());

    }
    else {

      Serial.println(
        "ERROR: Failed to start fallback access point."
      );

      return false;
    }
  }

  // ----------------------------------------------------------
  // mDNS
  // ----------------------------------------------------------

  if (MDNS.begin(mdnsHostname)) {

    Serial.print("mDNS responder started: ");
    Serial.print(mdnsHostname);
    Serial.println(".local");

  }
  else {

    Serial.println(
      "WARNING: mDNS startup failed."
    );

  }

  return true;
}

// ============================================================
// OTA setup
// ============================================================

void setupOTA() {

  // Name shown to OTA tools.
  ArduinoOTA.setHostname(mdnsHostname);

  // Require authentication for OTA updates.
  ArduinoOTA.setPassword(ota_password);

  ArduinoOTA.onStart([]() {

    Serial.println();
    Serial.println("OTA update starting...");

    // Never leave the robot driving during a firmware update.
    stopAll();

  });

  ArduinoOTA.onEnd([]() {

    Serial.println();
    Serial.println("OTA update complete.");

  });

  ArduinoOTA.onProgress([](unsigned int progress, unsigned int total) {

    unsigned int percent =
      (progress * 100U) / total;

    Serial.printf(
      "OTA progress: %u%%\r",
      percent
    );

  });

  ArduinoOTA.onError([](ota_error_t error) {

    Serial.println();
    Serial.print("OTA error: ");

    switch (error) {

      case OTA_AUTH_ERROR:
        Serial.println("Authentication failed");
        break;

      case OTA_BEGIN_ERROR:
        Serial.println("Begin failed");
        break;

      case OTA_CONNECT_ERROR:
        Serial.println("Connection failed");
        break;

      case OTA_RECEIVE_ERROR:
        Serial.println("Receive failed");
        break;

      case OTA_END_ERROR:
        Serial.println("End failed");
        break;

      default:
        Serial.println("Unknown error");
        break;
    }

    // Make absolutely sure the robot is stopped.
    stopAll();

  });

  ArduinoOTA.begin();

  Serial.println("OTA ready.");
  Serial.print("OTA hostname: ");
  Serial.println(mdnsHostname);
  Serial.println("OTA port: 3232");
}

// ============================================================
// Setup
// ============================================================

void setup() {

  Serial.begin(115200);

  delay(500);

  Serial.println();
  Serial.println("==============================");
  Serial.println("ESP32 Rover Booting");
  Serial.println("==============================");

  setupMotors();

  if (!setupNetwork()) {

    Serial.println(
      "Network setup failed."
    );

    stopAll();

    return;
  }

  setupOTA();

  udp.begin(UDP_PORT);

  Serial.print("Listening for UDP commands on port ");
  Serial.println(UDP_PORT);

  stopAll();

  Serial.println("Rover ready.");
}

// ============================================================
// Main loop
// ============================================================

void loop() {

  // OTA must be serviced continuously.
  ArduinoOTA.handle();

  // ----------------------------------------------------------
  // UDP command handling
  // ----------------------------------------------------------

  int packetSize = udp.parsePacket();

  if (packetSize > 0) {

    int length = udp.read(
      packetBuffer,
      sizeof(packetBuffer) - 1
    );

    if (length > 0) {

      packetBuffer[length] = '\0';

      bool validCommand =
        parseCommand(packetBuffer);

      if (validCommand) {

        lastPacketMs = millis();

      }
      else {

        Serial.println(
          "Invalid UDP command received."
        );

        stopAll();
      }
    }
  }

  // ----------------------------------------------------------
  // Motor failsafe
  // ----------------------------------------------------------

  if (millis() - lastPacketMs > FAILSAFE_MS) {

    stopAll();

  }
}
```

**LEDC API note:** this firmware uses `ledcSetup()` + `ledcAttachPin()` + `ledcWrite(channel, duty)`, the classic LEDC API. This requires **arduino-esp32 core 2.x**. Core 3.x removed `ledcSetup`/`ledcAttachPin` in favor of a new `ledcAttach(pin, freq, resolution)` + `ledcWrite(pin, duty)` API, if your PlatformIO environment resolves to core 3.x, this won't compile as-is.

**Uploading:**
```bash
# First flash, over USB:
pio run -e esp32dev -t upload

# Later updates, over Wi-Fi (requires setupOTA() to have already
# succeeded on a previous USB flash):
pio run -e esp32dev-ota -t upload
```
Close any open Serial Monitor before a USB upload, an open port will cause the upload to fail.

---

## Project Roadmap

### Phase 1: Core Drivetrain (current)
- [x] Assemble 4WD chassis
- [x] Wire dual L298N motor drivers to ESP32
- [x] Confirm all four wheels drive independently and correctly
- [x] ESP32 firmware: Wi-Fi, fallback AP, mDNS, OTA, UDP motor protocol, failsafe stop
- [ ] Establish stable Wi-Fi teleop control from Steam Deck via the ROS 2 bridge
- [ ] Verify `cmd_vel` / Twist teleop end-to-end through Foxglove

### Phase 2: Sensing
- [ ] Connect and calibrate ultrasonic sensor(s) for obstacle detection
- [ ] Connect IR sensors for line tracking and edge detection
- [ ] Publish sensor data as ROS 2 topics
- [ ] Visualize live sensor data in Foxglove Studio

### Phase 3: Autonomy Features
- [ ] Implement obstacle avoidance behavior
- [ ] Implement line-following mode
- [ ] Add head swivel (pan servo) for sensor sweep and camera aiming
- [ ] Optional: onboard camera streaming (ESP32-CAM or similar) into Foxglove

### Phase 4: Platform Infrastructure
- [ ] Config management for multiple robots on the same platform
- [ ] Standardized ROS 2 message/topic conventions across robot types
- [ ] Multi-robot control from a single Foxglove/ROS 2 session

### Phase 5: Integration
- [ ] Home automation integration (e.g., Home Assistant bridge)
- [ ] Persistent logging/telemetry storage
- [ ] Web or mobile dashboard for status and control outside of Foxglove

---

## Repo Structure

```
.
├── platformio.ini      # PlatformIO project config (USB + OTA envs)
├── config.h.example    # template for Wi-Fi/AP/OTA config, copy to config.h
├── src/
│   └── main.cpp         # ESP32 firmware: UDP motor control, OTA, failsafe
├── ros2_ws/             # ROS 2 packages run on the Steam Deck / host
├── docs/                # Wiring diagrams, pinouts, photos
├── hardware/            # CAD/STL files, BOM
└── README.md
```

---

## Contributing / Notes to Self

- Double-check ENA/ENB PWM pin assignments before wiring more peripherals. GPIO 6-11 are reserved on ESP32 (connected to onboard flash) and should be avoided.
- When adding the ultrasonic and IR sensors, keep pin choices ADC-safe if using ADC2 pins simultaneously with Wi-Fi (ADC2 is unreliable while Wi-Fi is active on most ESP32 modules).
- `config.h` and the real OTA `--auth` value are gitignored on purpose. If either ever gets committed by accident, rotate the OTA password and Wi-Fi credentials, don't just delete the commit.
- Keep this README's status table updated as each roadmap item lands; it doubles as the project's changelog at a glance.
