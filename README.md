# ARC

> An autonomous, reactive creature built from an ESP32-S3 and custom Rust firmware.

ARC is a small wheeled robot designed to explore how much perception and
interactive behavior can be achieved on a resource-constrained MCU without
using a camera, Raspberry Pi, cloud services, or a large AI model.

The project is intentionally hardware/software balanced:
custom embedded firmware, signal processing, sensor fusion, motor control,
power management, and procedural animation all run locally on the ESP32-S3.

---

## Status

### Early development

The hardware architecture is currently being assembled and tested.
The final mechanical design is intentionally left open while the electronics
and perception systems are being developed.

---

## Goals

ARC should be able to:

- Wander autonomously.
- Detect and localize sounds.
- React to a wake word/name.
- Turn toward sounds and investigate them.
- Detect nearby obstacles.
- Avoid obstacles while wandering.
- Detect rapidly approaching objects.
- Escape when appropriate.
- Remain still if it was called before an approaching object was detected.
- Detect when it is picked up, tilted, rotated, or shaken.
- React through procedural display animations.
- React to capacitive touch.
- Enter a low-power sleep state after inactivity.
- Wake when called or interacted with.
- Display charging and battery-related states.
- Perform all core behavior locally on the ESP32-S3.

---

# Hardware

## Main controller

### ESP32-S3 N16R8

The main controller is an ESP32-S3 development board using:

- dual-core Xtensa LX7
- 240 MHz maximum clock
- 16 MB flash
- 8 MB PSRAM
- 512 KB on-chip SRAM

The project uses `no_std` Rust with `esp-hal`.

---

## Audio perception

### 4× INMP441

Four identical digital MEMS microphones form a spatial audio array.

The microphones are arranged symmetrically around the body.

The system uses the microphones for two related tasks:

1. Wake-word/noise detection.
2. Approximate sound-source localization.

The project does not require high-quality audio recording.

The important properties are:
- synchronized sampling
- consistent microphone geometry
- known microphone positions
- low-latency acquisition

---

# Sound localization

ARC uses Time Difference of Arrival (TDOA).

The four audio channels are captured into memory before processing.

```text
4× microphones
       ↓
     I²S
       ↓
   DMA / SRAM
       ↓
  audio buffer
       ↓
  signal processing
       ↓
cross-correlation
       ↓
      Δt
       ↓
direction estimation
```
(drawn by deepseek)

Processing does not need to happen simultaneously.

The important requirement is that the captured samples maintain their
known timing relationship.

A sample index can represent time:

```
t = n × Ts
```

where:

- `n` = sample index
- `Ts` = sampling period

Therefore individual timestamps do not need to be stored for every sample.

---

# Perception

ARC combines several low-bandwidth sensors instead of relying on a camera.

## Microphones

Provide:

* sound detection
* wake-word detection
* approximate sound direction

## MPU6500

Provides:

* acceleration
* angular velocity
* orientation estimation
* pickup detection
* tilt detection
* rotation detection
* shake detection

---


## VL53L5CX

ARC uses a single VL53L5CX ToF imager mounted near the display as its
forward-facing "eye".

Unlike a single-point ToF sensor, the VL53L5CX provides a low-resolution
depth map divided into multiple zones.

The sensor is used for:

- obstacle detection
- collision avoidance
- detecting approaching objects
- estimating which side of ARC is more open
- local spatial awareness

The sensor's limited field of view is intentional.

Rather than adding multiple sensors to obtain a wider field of view, ARC
can physically rotate to acquire additional observations.

Conceptually:

                    VL53L5CX
                        │
                        ▼
                  ┌──────────┐
                  │    ARC   │
                  └──────────┘
                    ╲  │  ╱
                     ╲ │ ╱
                      ╲│╱
                    sensor FoV


ARC can rotate approximately 57° and perform another scan.

This allows multiple partially overlapping depth observations to cover a
much larger angular region, approaching the desired ~120° coverage while
using only one depth sensor.

The robot therefore uses **movement as part of its sensing system**.

Instead of continuously adding hardware to increase sensor coverage, ARC
can change its orientation to acquire information that was previously
outside the sensor's field of view.

---


## Capacitive touch

The ESP32-S3 touch capability is used for interaction.

The top of ARC acts as a touch surface.

Possible events:

* touch
* tap
* double tap
* long press

---

# Behavior System

ARC is event-driven rather than being a collection of independent
sensor-to-action `if` statements.

Sensors produce events.

```text
sensor
  ↓
event
  ↓
world state
  ↓
behavior engine
  ↓
action
```

Example events:

```text
SOUND_DETECTED
SOUND_LOCALIZED
OBSTACLE_DETECTED
OBJECT_APPROACHING
CALLED_BY_NAME
TOUCH
PICKED_UP
TILTED
SHAKEN
CHARGING
LOW_BATTERY
IDLE
```

---

# Autonomous wandering

When awake and not interacting with something, ARC randomly wanders.

```text
             ┌─────────────┐
             │    WANDER   │
             └──────┬──────┘
                    ↓
              choose movement
                    ↓
                move forward
                    ↓
                check ToF
               /          \
          obstacle       clear
             ↓              ↓
           turn          continue
             └──────┬───────┘
                    ↓
                  wander
```

Movement direction and duration can be generated pseudo-randomly.

The ToF sensors always have priority over wandering.

---

# Sound investigation

When ARC detects a sound:

```text
SOUND_DETECTED
      ↓
     "?"
      ↓
estimate direction
      ↓
turn toward source
      ↓
listen again
      ↓
confidence sufficient?
    /          \
  yes           no
   ↓             ↓
investigate    search
```

The display can visually represent the current perception state.

---

# Escape behavior

Obstacle avoidance and escape are separate behaviors.

Normal obstacle avoidance prevents ARC from driving into stationary objects.

Escape behavior responds to something approaching ARC.

The ToF sensors can measure changes in distance over time:

```
v ≈ Δd / Δt
```

A rapidly decreasing distance can therefore trigger an
`OBJECT_APPROACHING` event.

```text
OBJECT_APPROACHING
        ↓
was ARC recently called?
      /       \
    yes        no
     ↓          ↓
   STAY       ESCAPE
     ↓          ↓
 tilt        turn away
 animation      ↓
             high-speed
              movement
```

The high-speed escape behavior uses the available headroom of the
500 RPM N20 motors and PWM control.

---

# Motion behavior

ARC uses two N20 3 V 500 RPM geared motors.

Each motor has an independent BL5612 H-bridge.

```text
ESP32-S3
   │
   ├── PWM / direction → BL5612 → Motor L
   │
   └── PWM / direction → BL5612 → Motor R
```

PWM allows different movement profiles:

```text
WANDER       → low PWM
INVESTIGATE  → medium PWM
ESCAPE       → high PWM
```

The actual usable PWM ranges will be determined experimentally.

---

# Expressive display

ARC uses a 1.44" 128×128 ST7735 display.

The display represents ARC's internal state rather than simply showing
debug information.

Possible states:

```text
SLEEP
LISTENING
CONFUSED
CURIOUS
SEARCHING
INVESTIGATING
ESCAPING
PICKED_UP
TILTED
DIZZY
CHARGING
LOW_BATTERY
```

Animations should be procedural where practical.

The objective is to make sensor values influence animation parameters
instead of storing a separate animation for every possible situation.

---

# Audio feedback

A small passive buzzer provides non-verbal feedback.

ARC does not require speech.

Examples:

```text
sound detected  → short chirp
searching       → questioning tone
source found    → confirmation tone
error            → warning tone
charging         → charging pattern
```

---

# Sleep / Wake

ARC enters a sleep/idle state after approximately one minute without
meaningful interaction.

```text
ACTIVE
  ↓
inactivity
  ↓
DROWSY
  ↓
SLEEP
```

Wake sources may include:

* wake word/name
* touch
* other configured sensor events

A random sound should not necessarily wake ARC.

The intended interaction is:

```text
"ARC!"
  ↓
wake
  ↓
display activation
  ↓
ACTIVE
```

---

# Persistent data

No SD card is planned.

ARC uses the ESP32-S3's onboard flash for small persistent data such as:

* configuration
* calibration values
* preferences
* wake-word/model data
* device state

Large external storage is intentionally avoided to keep the physical design
small.

---

# Power

ARC uses a 1S 3.7 V 950 mAh LiPo battery.

A TP4056 is used for charging.

Power architecture still needs to be finalized.

The motor and logic power paths should be treated carefully because motor
current changes can introduce supply noise.

The final design should include appropriate:

* regulation
* decoupling
* battery monitoring
* power switching
* motor supply filtering

---

# Firmware Architecture

The firmware is written in Rust without the standard library.

```text
src/
├── main.rs
│
├── hal/
│
├── drivers/
│   ├── audio/
│   ├── tof/
│   ├── imu/
│   ├── display/
│   ├── motor/
│   ├── touch/
│   └── buzzer/
│
├── perception/
│   ├── audio_detection/
│   ├── tdoa/
│   ├── motion/
│   └── proximity/
│
├── behavior/
│   ├── events/
│   ├── world_state/
│   └── state_machine/
│
├── animation/
│
└── power/
```

The exact module structure may change during development.

---

# Memory Strategy

The ESP32-S3 provides:

* internal SRAM for latency-sensitive work
* PSRAM for larger buffers and less latency-critical data
* flash for firmware and persistent configuration

Audio acquisition should prioritize predictable memory usage.

Example:

```text
I²S / DMA
    ↓
SRAM buffer
    ↓
DSP
    ↓
direction result
```

Larger temporary buffers can use PSRAM if required.

---

# Development Plan

## Phase 1 — Audio

* Connect four INMP441 microphones.
* Verify I²S acquisition.
* Capture synchronized samples.
* Implement buffering.
* Implement basic filtering.
* Implement cross-correlation.
* Estimate direction.

## Phase 2 — Behavior

* Event system.
* World-state representation.
* State machine.
* Random wandering.
* Sound investigation.
* Escape behavior.

## Phase 3 — Display

* ST7735 driver.
* Basic renderer.
* Procedural eyes/expressions.
* Sensor-driven animations.

## Phase 4 — Motion

* BL5612 motor drivers.
* N20 motors.
* PWM control.
* Differential steering.
* ToF obstacle avoidance.

## Phase 5 — IMU / Interaction

* MPU6500.
* Pickup detection.
* Tilt/rotation detection.
* Touch interaction.

## Phase 6 — Power

* TP4056.
* Battery monitoring.
* Sleep/wake.
* Charging behavior.

## Phase 7 — Integration

Run all systems together and optimize:

* CPU usage
* memory usage
* latency
* power consumption
* motor noise
* audio localization accuracy

---

# Design Principles

### 1. Local btw

ARC should operate without:

* cloud services
* remote inference
* Raspberry Pi
* external computer

### 2. Cheapest possible components

Sensors should provide the minimum information needed for behavior.

A camera is unnecessary if ARC only needs to know:

> "Something is approaching."

A high-quality microphone is unnecessary if ARC only needs:

> "Was that ARC?" and "Where did it come from?"

### 3. Deterministic where possible

Real-time systems should favor predictable:

* memory usage
* timing
* sensor acquisition
* motor control

---

# Current BOM

| Component       | Qty |
| --------------- | --: |
| ESP32-S3 N16R8  |   1 |
| INMP441         |   4 |
| VL53L5CX        |   1 |
| MPU6500         |   1 |
| ST7735 1.44"    |   1 |
| N20 3 V 500 RPM |   2 |
| BL5612          |   2 |
| 950 mAh 1S LiPo |   1 |
| TP4056          |   1 |
| Passive buzzer  |   1 |

Current listed component cost is approximately ₹3,068 excluding the TP4056
price.

Wheels will be designed/3D printed separately.

---

# Why this project?

ARC is intentionally not a conventional "ESP32 robot".

The interesting engineering problem is building a system that can perceive,
reason about its immediate environment, maintain state, and react naturally
while operating entirely on a small MCU.

The project combines:

* embedded Rust
* `no_std`
* digital signal processing
* spatial audio
* sensor fusion
* real-time systems
* motor control
* power management
* procedural graphics
* low-power operation
* embedded ML where it is actually useful

The goal is not to make the most powerful robot.

The goal is to make the most interesting system possible from a very
limited amount of hardware.

<EOS>