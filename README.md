# Wireless Shed Door Alarm (two sheds, one indoor LCD)

Three ESP32 boards talk **directly** with ESP-NOW (no Wi-Fi router).

- **Shed 1 sensor** — `SHED_ID 1` in `shed-sensor.ino`
- **Shed 2 sensor** — same sketch, `SHED_ID 2` on the second ESP32
- **House LCD alarm** — one indoor controller, 16x2 LCD, one siren

Arming is done **in the house**. You can arm both sheds, or only one (so you can work in shed 1 while shed 2 stays protected). Unplugging a shed board cannot mute the house; that shed goes **DEAD** on the LCD and the siren sounds.

## Parts

| Qty | Part | Role |
| --- | --- | --- |
| 3 | ESP32 Dev Module | Shed 1, shed 2, house |
| 2 | Magnetic reed switch + magnet | One door per shed |
| 2 | SW-420 vibration / shock module | Prying / forced entry |
| 1 | I2C 16x2 LCD (PCF8574 backpack) | Indoor display, usually 0x27 or 0x3F |
| 1 | Active buzzer or 5 V siren | House sounder |
| 1 | Optional 12 V siren + relay/MOSFET | Louder house alarm |
| 3 | USB 5 V supplies | One at each end |
| 4 | Momentary buttons | House: Select, Arm, Mute, Test |
| 1 | Green LED + resistor | Armed / link |
|  | Optional lid tamper switches | Shed boxes |

Range is typically tens of metres. Metal sheds block radio — mount each shed ESP32 near a wooden wall or window.

## Arduino IDE setup

1. Install **esp32** by Espressif in Boards Manager.
2. Install **LiquidCrystal I2C** by Frank de Brabander.
3. Board: **ESP32 Dev Module**, Serial 115200.
4. Open `shed-sensor/shed-sensor.ino`. Leave `#define SHED_ID 1`, upload to the first shed ESP32.
5. Change that line to `#define SHED_ID 2`, upload to the second shed ESP32.
6. Open `house-alarm/house-alarm.ino` and upload to the indoor ESP32.
7. Set `ALARM_SECRET` to the same private number in **both** `protocol.h` files.

The two `protocol.h` files must stay identical. The two shed boards must **not** share the same `SHED_ID`.

## Wiring

### Each shed sensor (identical except SHED_ID)

```
Reed switch  (magnet on the door, switch on the frame)
  one side -> GPIO 15
  other    -> GND
  Door shut with magnet present should pull GPIO 15 LOW.

SW-420 vibration
  VCC -> 3.3V
  GND -> GND
  DO  -> GPIO 4

Optional lid tamper (off until TAMPER_ENABLED = true)
  GPIO 5 -> plunger -> GND

First radio test without a reed: jumper GPIO 15 to GND (door "shut").
Onboard LED (GPIO 2) blinks on each send.
```

### House LCD controller

```
I2C LCD
  SDA -> GPIO 21
  SCL -> GPIO 22
  VCC -> 5V (or 3.3V if your backpack wants 3.3V)
  GND -> GND

Active buzzer / 5V siren
  +  -> GPIO 26
  -  -> GND

Green LED
  GPIO 4 -> 220Ω -> LED -> GND

Onboard / red LED is GPIO 2.

Buttons to GND (internal pull-ups):
  SELECT      GPIO 14    ALL -> S1 -> S2
  ARM/DISARM  GPIO 13    hold 1 second (applies to the SELECT focus)
  MUTE        GPIO 15    silences this event
  TEST        GPIO 12    local siren blip
```

Loud 12 V siren: GPIO 26 drives a MOSFET/relay, never the siren current.

## LCD

```
S1:OK   S2:WAIT
>A  1off  2off
```

| Line 1 | Meaning |
| --- | --- |
| OK | Door shut, talking |
| OPEN | Door open |
| PRY | Vibration / forced entry |
| TAMP | Shed box lid |
| DEAD | No heartbeat (power/radio cut) |
| ---- | That shed has not been heard yet |

Line 2 when quiet: `>` marks what ARM will change (`A` = both, `1` / `2` = one shed). `ON` / `off` is armed state.

Line 2 when alarming: `ALM S1 OPEN` or `MUTE S2 DEAD`.

## How to use it

1. Power the **house** first, then both sheds. House Serial should print `Learned shed 1` and `Learned shed 2`. LCD changes `----` to `OK` (jumper GPIO 15 to GND on each shed if the door reads open).
2. Press **SELECT** until `>` is on `A` (both). Hold **ARM** 1 second. Both should show ON.
3. Open shed 1’s door → LCD `S1:OPEN`, siren. Shed 2 stays `OK`.
4. **MUTE** stops the noise. Close the door; the next opening alarms again.
5. To work in shed 1: SELECT until `>1`, hold ARM to disarm S1 only. S2 stays ON.
6. Unplug an armed shed → after ~8 s that row shows **DEAD** and the siren sounds.

Green LED: slow blink = nothing armed, solid = every armed shed is OK, fast blink = waiting, off + red flash = alarm.

## First-time radio check

Shed Serial: `TX shed 1 BEAT` or `TX shed 2 BEAT` every 2 seconds. House: `RX S1` / `RX S2`.

If the house stays quiet:

- All three boards are ESP32s
- `ALARM_SECRET` matches
- Shed IDs are 1 and 2, not both 1
- Try all three in the same room first

## Limits

DIY deterrent, not a certified alarm. `ALARM_SECRET` only stops casual collisions. Still use locks, lighting, and common sense.
