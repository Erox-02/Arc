# ARC

ARC is a small wheeled robot designed to explore how much perception and interactive behavior can be achieved on a resource-constrained MCU without using a camera, Raspberry Pi, cloud services, or a large AI model. Because why would you need 5GHz of silicon when you can do this with a 8$ chip lol.

The project is intentionally hardware/software balanced: custom embedded firmware (rust btw), 4x micro phone instead of a cam , motor control, power management, and procedural animation (ts really hard gotta hire a face designer) all run locally on a ESP32-S3 , No external compute(i mean no bigger pc uses wifi to process , esp is all mighty), no cloud, no "AI" slop in disguise of a mini cute bot.

---

## Status

### Early development (read: wires everywhere)

 The final mechanical design is intentionally left open while the electronics and perception systems are being developed. Translation: I'm still figuring out how to fit all this stuff in a cute shell without it looking like a rats nest lmao and even before that when forge gonna fund me .

---

## Goals

ARC should be able to:

- Wander autonomously 
- Detect and localize sounds (i didnt use 4 mics just for show)
- React to a wake word/name (As is use arch btw , i named it arc)
- Turn toward sounds and investigate them (curious little thing)
- Detect nearby obstacles (no wall-bumping)
- Avoid obstacles while wandering (vl53l5cx is the mvp)
- Detect rapidly approaching objects (spidey sense but for a cute bot)
- Escape when appropriate (like horror games!)
- Remain still if it was called before an approaching object was detected (brave little guy)
- Detect when it is picked up, tilted, rotated, or shaken (stop manhandling it!)
- React through procedural display animations 
- React to capacitive touch (pet it!)
- Enter a low-power sleep state after inactivity (zzz)
- Wake when called or interacted with (rise and shine)
- Display charging and battery-related states (feed me)
- Perform all core behavior locally on the ESP32-S3 (no cloud nonsense)

---

# Hardware

## Main controller

### ESP32-S3 N16R8

The main controller is an ESP32-S3 development board using:

- dual-core Xtensa LX7 (only 2 brain cell still better than me lol)
- 240 MHz maximum clock (not the fastest but works)
- 16 MB flash (enough for firmware and then some)
- 8 MB PSRAM (for bigger buffers yay)
- 512 KB on-chip SRAM (for the real-time ...stuff)

The project uses `no_std` Rust with `esp-hal`. Because C is for people who don't like memory safety (i am a rustacean).

---

## Tdoa

### 4× INMP441

Four identical digital mems microphones form a spatial audio array.
The microphones are arranged symmetrically around the body (like ears but 4 , well not like youre thinking but technically are ears)

The system uses the microphones for two related tasks:

1. Wake word and noise detection.
2. Approximate sound-source pin pointing .

Arc do not require high-quality audio recording (i am not making a podcast here). 

The important properties are:
- synchronized sampling 
- consistent microphone geometry (no lopsided ears)
- low-latency acquisition (fast ears)

---

# Sound localization

ARC uses Time Difference of Arrival (TDOA). Fancy term for "sound arrives at different ears at different times" and it is used that to figure out where it came from. Like echo but without the clicking.

The four audio channels are captured into memory before processing.

```text
    4× microphones  <-- Aud inp
          |
         I²S        <-- data piped
          |
     audio buffer   <-- aud gets sampled
          |
  signal processing <-- math happens here
          |
  cross-correlation <-- compare waves 
          |
       <del>t          <-- time difference
          |
  direction estimat  <-- avg source location
```

Processing doesn't need to happen simultaneously (it is not in a hurry). The important requirement is that the captured samples maintain their known timing relationship. Because if they don't, the whole thing falls apart.

A sample index can represent time:

```
t = n × Ts
```

where:

- n = sample index (which one it is at)
- Ts = sampling period (how often it is measuring)

Therefore individual timestamps don't need to be stored for every sample. It just keep track of where it is. Efficiency btw.

---

# High lev view

ARC combines several low-bandwidth sensors instead of relying on a camera. Because cameras(like solo leveling) are expensive and power-hungry and honestly overrated for what i am doing.

## Microphones

Provide:

* sound detection (something's happening!)
* wake-word detection (did someone say my name?)
* approximate sound direction (where did that come from?)

## MPU6500

Provides:

* acceleration (whoa i am moving)
* angular velocity 
* pickup detection (someone grabbed me!)
* tilt detection (i cant stand rn dont do it)
* rotation detection (helicopter helicopter)
* shake detection (stop shaking me lmao)

---

## VL53L5CX

ARC uses a single vl53l5cx ToF imager mounted near the display as its forward-facing "eye". It's like having one really good eye instead of two mediocre ones(yeah i first considered vl53l0x thts why).

Unlike a single-point ToF sensor, the VL53L5CX provides a low-resolution depth map divided into multiple zones. We get a whole grid of distance measurements instead of just one number.

The sensor is used for:

- obstacle detection (something's in front!)
- collision avoidance (don't hit the wall)
- detecting approaching objects (it's getting closer!)
- estimating which side of ARC is more open (where should I go?)
- local spatial awareness (what's around me)

The sensor's limited field of view is intentional. Rather than adding multiple sensors to obtain a wider field of view, ARC can physically rotate to acquire additional observations. Because it has wheels and knows how to use them.

Conceptually:

```Js
                    VL53L5CX       <-- eye is here
                        |
                        v
                  +----------+
                  |   ARC    |   <-- body
                  +----------+
                    \  |  /
                     \ | /       <-- limited view
                      \|/
                    sensor FoV
```

ARC can rotate approximately 57° and perform another scan. This allows multiple partially overlapping depth observations to cover a much larger angular region, approaching the desired ~120° coverage while using only one depth sensor. it can cover more area by moving! Revolutionary!

The robot therefore uses **movement as part of its sensing system**. Instead of continuously adding hardware to increase sensor coverage, ARC can change its orientation to acquire information that was previously outside the sensor's field of view. It's called being clever with what you have.

---

## Capacitive touch

The ESP32-S3 touch capability is used for interaction. The top of ARC acts as a touch surface (like a big button).

Possible events:

* touch (boop)
* tap (tap tap)
* double tap (two times!)
* long press (holding it)

---

# Behavior System

ARC is event-driven rather than being a collection of independent sensor-to-action `if` statements. Because that would be chaotic and we like organized chaos.

Sensors produce events and some example events:

```C
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
IDLE      (nothing happening rn)
```

---

# Autonomous wandering

When awake and not interacting with something, ARC randomly wanders. Because standing still ,is boring as hell.

```css
        +-----------+
        |  WANDER   |   
        +-----+-----+
              |
        choose move  
              |
         move fwd    
              |
         check ToF   
        /           \
   obstacle        clear  
      |              |
    turn         continue 
      |              |
      +------+-------+
             |
           wander    
```

Movement direction and duration are generated pseudo-randomly (which is fancy for "random enough" , i am not using rand here fools thts for std ). The ToF sensors always have priority over wandering (safety first).

---

# Sound investigation

When ARC detects a sound:

```rust
  SOUND_DETECTED   
        |
       "?"         (curious head tilt)
        |
    estimate dir    
        |
  turn to tht dir  
        |
  listen again(if possible)
        |
      confident     
       /     \
  Hell yea    Nah
      |         \
investigate | search
```

The display will be visually representing the current perception state in other words , confused/happy/curious/ESCAPEEEE etc .

---

# Escape behavior

Obstacle avod and escape feature are twins but diff .Normal obstacle avoidance prevents arc to stumble upton static object while wandering(speed normal , curosity high) but escape behavior is only activates when something approaches arc without calling arc , arc will simply go ahead crank up the pwm tht was holded back all the time and full 500rpm on a small bot and sayonara .

The tof sensors are the 

```
v ≈ <del>d / <del>t    //some high school maths wait it is a readme why am i commenting?
```

A rapidly decreasing distance (something coming closer) can therefore trigger an `OBJECT_APPROACHING` event.

```bash
OBJECT_APPROACH   (somethings coming!)
        |
  was arc called? (did someone ask for me?)
       /          \
     yes           no
      |             |
    STAY         ESCAPE   (nope nope nope)
      |             |
  tilt anim     turn away
                   |
              high speed
              movement   (run!)
```

The high-speed escape behavior uses the available headroom of the 500 RPM N20 motors and PWM control. It can scurry away surprisingly fast for such a tiny thing.

---

# Motion behavior

ARC uses two N20 3 V 500 RPM geared motors. Each motor has an independent BL5612 H-bridge.

```zsh
ESP32-S3
   |
   +--- PWM/dir ---> BL5612 ---> Motor L
   |
   +--- PWM/dir ---> BL5612 ---> Motor R
```

PWM allows different movement profiles:

```python
WANDER       --> low PWM    (chill)
INVESTIGATE  --> medium PWM (curious)
ESCAPE       --> high PWM   (panic)
```

The actual usable PWM ranges will be determined experimentally (which is engineering-speak for "we'll figure it out by trying things and seeing what happens").

---

# Expressive display

ARC uses a 1.44" 128×128 ST7735 display. It's small but mighty.

The display represents ARC's internal state rather than simply showing debug information. Because showing "UART: 115200 baud" is boring.

Possible states:

```ruby
SLEEP          (zzz)
LISTENING      (ears perked)
CONFUSED       (??)
CURIOUS        (ooh what's that)
SEARCHING      (looking for something)
INVESTIGATING  (getting closer)
ESCAPING       (scaredy cat)
PICKED_UP      (I'm being held!)
TILTED         (whoa)
DIZZY          (too much spinning)
CHARGING       (feeding time)
LOW_BATTERY    (I'm hungry)
```

Animations should be procedural where practical. The objective is to make sensor values influence animation parameters instead of storing a separate animation for every possible situation. Because that would take up all the memory and nobody wants that.

---

# Audio feedback

A small passive buzzer provides non-verbal feedback. ARC does not require speech (no "hello world" from the robot).

Examples:

```text
sound detected  --> short chirp   (I heard you!)
searching       --> questioning tone (where did it go?)
source found    --> confirmation tone (found it!)
error           --> warning tone  (something's wrong)
charging        --> charging pattern (nom nom nom)
```

---

# Sleep / Wake

ARC enters a sleep/idle state after approximately one minute without meaningful interaction. Because batteries don't last forever.

```go 
ACTIVE          (awake and ready)
  |
  v
inactivity      (nothing's happening)
  |
  v
DROWSY          (getting sleepy)
  |
  v
SLEEP           (zzz)
```

Wake sources may include:

- wake word/name 
- touch (boop)
- putting to charge

A random sound cant wake arc but calling 'ARC!!!!' will do the work just fine . 

---

# Persistent data

No SD card is planned (too chunky). ARC uses the ESP32-S3's onboard flash for small persistent data such as:

* configuration (how I like to behave)
* calibration values (sensor corrections)
* preferences (things I remember)
* wake-word/model data (my name and how to recognize it)
* device state (what I was doing)

Large external storage is intentionally avoided to keep the physical design small and cheap.

---

# Power

ARC uses a 1S 3.7 V 950 mAh LiPo battery. A TP4056 is used for charging.

Power architecture still needs to be finalized (read: I'm still figuring out voltage regulation). 

The motor and logic power paths should be treated carefully because motor current changes can introduce supply noise. Motors are noisy little buggers.

The final design should include appropriate:

* regulation (smooth power)
* decoupling (no noise)
* battery monitoring (how much juice is left)
* power switching (on/off)
* motor supply filtering (quiet motors)

---

# Gpio's

| GPIO        | Component               | Notes                       |
|-------------|-------------------------|-----------------------------|
| GPIO1       | I²S DIN (Mics 1+2)      | L/R channel pair            |
| GPIO2       | I²S DIN (Mics 3+4)      | L/R channel pair            |
| GPIO3       | Buzzer + Touch          | Time-multiplexed            |
| GPIO4       | I²S SCK                 | Shared bit clock            |
| GPIO5       | I²S WS                  | Shared frame sync           |
| GPIO6       | LEFT DIR                | Motor direction             |
| GPIO7       | LEFT PWM                | Motor speed                 |
| GPIO8       | I²C SDA                 | Sensors + ADS1115           |
| GPIO9       | I²C SCL                 | Sensors + ADS1115           |
| GPIO10      | TFT DC                  | SPI command pin             |
| GPIO11      | TFT MOSI                | SPI data                    |
| GPIO12      | TFT SCK                 | SPI clock                   |
| GPIO13      | RIGHT DIR               | Motor direction             |
| GPIO14      | RIGHT PWM               | Motor speed                 |

> these gpio pins arent esp wise i just made a draft , for the real mudesign esp32 s3 , it does fine as all , but for diff boards it is a problem (like hell i care)

### I²C Bus

| Device      | Address | Notes                       |
|-------------|---------|-----------------------------|
| VL53L5CX    | 0x29    | ToF imager                  |
| MPU6500     | 0x68    | IMU                         |
| ADS1115     | 0x48    | ADC for battery & charging  | //actually i wd have used the bare ic but in india , the assembeled breakout boards costs way cheaper 

### ADS1115 Channels

| Channel     | Function                | Notes                       |
|-------------|-------------------------|-----------------------------|
| A0          | Battery voltage         | Voltage divider             |
| A1          | TP4056 CHRG             | Charging status             |
| A2          | Touch points? (maybe)   | more touch points           |
| A3          | Same as a2              | same as a2           |

> reminder most of these diagrams(except the broken ones) were generated by deepseek , huh you thought i drew them myself , lmao [laughing emoji] 
---

# Memory Strategy

The ESP32-S3 provides:

* internal SRAM for latency-sensitive work (fast stuff)
* PSRAM for larger buffers and less latency-critical data (big stuff)
* flash for firmware and persistent configuration (permanent stuff)

Audio acquisition should prioritize predictable memory usage. No sudden allocation spikes.
Larger temporary buffers can use PSRAM if required (because i have it, i paid for it , why not).

---

# Last section

Sorry i ran out of things to tell about so at least know , ***I use arch btw*** 
>matane

```toml
Author = Dipanjan
Designation = Arch user
Status = alive
```