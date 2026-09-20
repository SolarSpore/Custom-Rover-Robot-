# ESP32 4WD Rover

A Wi-Fi-connected, 4-wheel-drive rover built around an ESP32 and dual L298N motor drivers. This is the first hardware node of the **Robotics Platform** (see the platform README at the repository root). This README covers the robot itself: hardware, wiring, network protocol, and firmware. The ROS 2 control side (gamepad teleop, the UDP bridge, Foxglove) is documented in the platform README.

Design is loosely inspired by the classic Arduino/ESP32 obstacle-avoiding and line-tracking smartcar chassis builds, reworked around an ESP32 as the main controller and a 4-motor, dual-driver layout instead of a single H-bridge.

---

## Current Status

| System | Status |
|---|---|
| Chassis + 4WD drivetrain | Assembled |
| Dual L298N wiring | Wired and driving |
| ESP32 main controller | Flashed, Wi-Fi connected |
| ESP32 firmware (this directory) | Working: UDP motor control, OTA, failsafe stop |
| Control link from a host (UDP) | Working, driven from a Steam Deck via ROS 2 (see platform README) |
| Head swivel (pan servo) | Not connected |
| Ultrasonic sensors ("eyes") | Not connected |
| IR line/obstacle sensors | Not connected |

The rover drives but has no onboard sensing yet. Obstacle avoidance, line tracking, and the head/camera swivel from the original inspiration build are all still on the bench.

---

## Hardware

- **Controller:** ESP32 (Wi-Fi enabled, replacing the Arduino Uno used in similar builds)
- **Motor drivers:** 2x L298N Dual H-Bridge
- **Drive:** 4x DC gear motors, independent front/rear left/right control
- **Not yet installed:** pan servo (head swivel), HC-SR04 ultrasonic sensors, IR sensors

### Motor Wiring

Each wheel gets its own pair of GPIOs, driven directly as PWM forward/reverse pairs (this assumes the L298N ENA/ENB jumpers are left in place, tied HIGH):

| Wheel | GPIO A | GPIO B | LEDC Channels |
|---|---|---|---|
| Front Left (FL) | 23 | 25 | 0, 1 |
| Front Right (FR) | 26 | 27 | 2, 3 |
| Rear Left (RL) | 21 | 22 | 4, 5 |
| Rear Right (RR) | 18 | 19 | 6, 7 |

This table matches how the rover is physically wired. It was verified with a one-channel-at-a-time test (each wheel driven alone over UDP), and the pin rows in `main.cpp` are ordered to match. If you rewire, re-run that test and update the pins. Which L298N board and terminals each pair lands on has not been re-verified.

PWM frequency is 5000 Hz at 8-bit resolution (0-255), matching the -255..255 command range directly.

### Power and motor behavior

- **Charge the batteries first.** A low pack was behind most of the early "dead wheel" and flaky-channel symptoms.
- **L298N low-speed stall.** On this build the gear motors stall below roughly PWM 80-100 (they whine without turning). Steady, consistent motion starts around PWM 100. Whatever sends commands should keep non-zero values above that range (the platform's ROS bridge does this with a `min_pwm` setting).
- Each L298N also drops about 2 V internally, so a partly discharged pack narrows the usable range further.

---

## Network

| Item | Value |
|---|---|
| UDP port | `4210` |
| UDP protocol | `"FL,FR,RL,RR"`, each value `-255..255` |
| Failsafe | All motors stop after 500 ms without a valid packet |
| Wi-Fi connect timeout | 8 seconds, then falls back to AP mode |
| Fallback AP | SSID/password set in `config.h` (`ap_ssid` / `ap_password`) |
| mDNS / OTA hostname | set in `config.h` (`mdnsHostname`), resolves as `<mdnsHostname>.local` |
| OTA port | `3232` (ArduinoOTA default) |
| OTA password | set in `config.h` (`ota_password`), must match `platformio.ini`'s `--auth` flag |
| Wi-Fi power saving | Modem sleep disabled in station mode (`WiFi.setSleep(false)`) |

**No static IP is configured in firmware.** The board takes whatever address your router's DHCP hands out. This rover uses a DHCP reservation on the router (currently `192.168.1.26`), which keeps the address stable across reboots. That matters because `esp32dev-ota` targets a fixed IP. To find the board's MAC for the reservation, check the router's DHCP client list, or add `Serial.println(WiFi.macAddress())` in `setupNetwork()`.

**Why modem sleep is off:** with it enabled, ping to the ESP32 averaged about 51 ms with spikes to 235 ms, and UDP motor commands arrived in bursts (visibly jittery wheels). With it disabled, the average dropped to about 8 ms with a 28 ms worst case.

### Talking to the rover

The firmware never runs ROS: it only speaks the UDP protocol above, and anything that can send that string can drive it. A sender should resend at a steady rate (20 Hz works well), since the failsafe stops the motors after 500 ms of silence.

**Only one sender should transmit at a time.** A second source sending idle stop packets (for example `0,0,0,0` from a test tool) interleaves with the real commands and shows up as motor stutter.

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
    4. Wi-Fi modem sleep is disabled in station mode for steady
       UDP latency.

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
  {23, 25},  // FL
  {26, 27},  // FR
  {21, 22},  // RL
  {18, 19}   // RR
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

    // Disable modem sleep. With sleep enabled the radio naps between
    // beacons and UDP packets arrive in delayed bursts, which shows up
    // as jittery motors.
    WiFi.setSleep(false);

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

When both environments are targeted at once, PlatformIO reports each separately. If no USB cable is attached, `esp32dev` fails while `esp32dev-ota` succeeds, and the OTA result is the one that counts.

---

## Rover Roadmap

Platform-level work (control software, telemetry dashboard, MQTT/home automation, multi-robot support) is tracked in the platform README and roadmap. These items are specific to this robot.

### Phase 1: Core Drivetrain (done)
- [x] Assemble 4WD chassis
- [x] Wire dual L298N motor drivers to ESP32
- [x] Confirm all four wheels drive independently and correctly
- [x] ESP32 firmware: Wi-Fi, fallback AP, mDNS, OTA, UDP motor protocol, failsafe stop
- [x] Stable Wi-Fi control from a host over UDP (Steam Deck via ROS 2)

### Phase 2: Sensing
- [ ] Connect and calibrate ultrasonic sensor(s) for obstacle detection
- [ ] Connect IR sensors for line tracking and edge detection
- [ ] Add wheel encoders (needed for odometry)
- [ ] Publish sensor data to the control system

### Phase 3: Autonomy Features
- [ ] Implement obstacle avoidance behavior
- [ ] Implement line-following mode
- [ ] Add head swivel (pan servo) for sensor sweep and camera aiming
- [ ] Optional: onboard camera streaming (ESP32-CAM or similar)

---

## Repo Structure

```
firmware/esp32-bot/          (this directory)
├── platformio.ini      # PlatformIO project config (USB + OTA envs)
├── config.h.example    # template for Wi-Fi/AP/OTA config, copy to config.h
├── src/
│   └── main.cpp         # ESP32 firmware: UDP motor control, OTA, failsafe
├── docs/                # Wiring diagrams, pinouts, photos
├── hardware/            # CAD/STL files, BOM
└── README.md
```

---

## Contributing / Notes to Self

- Double-check ENA/ENB PWM pin assignments before wiring more peripherals. GPIO 6-11 are reserved on ESP32 (connected to onboard flash) and should be avoided.
- When adding the ultrasonic and IR sensors, keep pin choices ADC-safe if using ADC2 pins simultaneously with Wi-Fi (ADC2 is unreliable while Wi-Fi is active on most ESP32 modules).
- `config.h` and the real OTA `--auth` value are gitignored on purpose. If either ever gets committed by accident, rotate the OTA password and Wi-Fi credentials, don't just delete the commit.
- Keep this README's status table updated as each roadmap item lands.
- **Lessons from bring-up:**
  - A low battery looks like random dead wheels. Charge before debugging.
  - Never run more than one sender to the ESP32. Competing stop packets read as motor stutter.
  - Wi-Fi modem sleep on the ESP32 causes bursty UDP timing. Leave `WiFi.setSleep(false)` in.
  - Test wiring changes one channel at a time over raw UDP with the rover on blocks. That is how the FL/FR/RL/RR pin order was corrected.
