# PingPongLauncher

Arduino library for controlling the **Ping Pong Ball Launcher Control PCB** — a two-motor drive + servo-fed ball launcher board. The library talks to the board over a simple binary UART protocol, so it works with any Arduino-compatible board and any `Stream`-based serial interface (`HardwareSerial`, `SoftwareSerial`, USB CDC, etc).

If you're **not** using Arduino, skip to [Talking to the Board Without Arduino](#talking-to-the-board-without-arduino) below — the wire protocol is fully documented there so you can drive the board from Python, a Raspberry Pi, another MCU, or anything else that can push bytes out a UART.

---

## Features

- Start/stop both drive motors with a single speed value
- Trigger a single shot (servo opens, holds, and auto-closes on-board)
- Works with any `Stream` — pass in `Serial1`, a `SoftwareSerial` instance, or anything else that implements `Stream`
- No dependencies beyond the Arduino core

## Hardware

- **Baud rate:** 115200, 8N1
- **Connection:** the board exposes a UART connection for TX/RX — wire it to any UART on your Arduino (or a `SoftwareSerial` pair)
- **Logic levels:** the board's UART input is 5V-tolerant, so a 5V-logic host can drive it directly with no level shifter needed. The board's output only ever drives up to 3.3V — most 5V-logic boards will still read that reliably as a logic high, but if your receiving UART requires a stricter 5V logic-high threshold, add a simple level shifter on that one line to be safe.
- **Power/motors/servo:** wired separately per the board's own documentation; this library only concerns itself with the UART command link

## Installation

1. Download this repository as a ZIP (Code → Download ZIP).
2. In the Arduino IDE: **Sketch → Include Library → Add .ZIP Library...** and select the downloaded file.
3. Restart the IDE if the library doesn't show up under **File → Examples → PingPongLauncher** right away.

## Quick Start

```cpp
#include <PingPongLauncher.h>

// Pass whichever Stream is wired to the board's UART TX/RX connection.
PingPongLauncher launcher(Serial1);

void setup() {
  Serial1.begin(115200); // must match the board's baud rate

  launcher.stopMotors();  // always start from a known state
}

void loop() {
  launcher.startMotors(128); // 0-255
  delay(2000);

  launcher.fire();            // board handles the open/hold/close timing
  delay(1500);

  launcher.stopMotors();
  delay(3000);
}
```

See [`examples/BasicControl`](examples/BasicControl/BasicControl.ino) for a complete sketch, including notes on using `SoftwareSerial` if your board only has one hardware UART.

## API Reference

### `PingPongLauncher(Stream &serial)`
Constructs the controller against any `Stream`-compatible serial interface. You're responsible for calling `.begin(115200)` on that interface yourself before using the library.

### `void stopMotors()`
Stops both drive motors.

### `void startMotors(uint8_t speed)`
Starts both drive motors at `speed` (0–255). The same speed value is applied to both motors.

### `void fire()`
Triggers one shot. The board opens the feed servo, holds it open for ~1 second, and closes it automatically — this call is fire-and-forget.

---

## Talking to the Board Without Arduino

The board doesn't care what's on the other end of the UART — it just needs 115200 baud, 8N1, and a byte stream that follows the frame format below. This section is enough to drive the board from Python, a Raspberry Pi, another microcontroller, a terminal program, or anything else.

### Frame Format

Every command is a short binary frame starting with a fixed sync byte:

```
[ 0xAA ][ command ][ optional parameter bytes... ]
```

| Field     | Size    | Notes                                   |
|-----------|---------|------------------------------------------|
| Sync      | 1 byte  | Always `0xAA`. Frames not starting with this byte are ignored. |
| Command   | 1 byte  | See table below.                        |
| Parameters| 0–1 byte| Only `START_MOTORS` takes a parameter.  |

The board validates frame **length** against the command — a frame with the wrong number of bytes for its command is silently discarded, so make sure you send exactly the right number of bytes and nothing else.

### Commands

| Command        | Value  | Frame (hex)         | Total Size | Description |
|----------------|--------|----------------------|------------|-------------|
| `STOP_MOTORS`  | `0x01` | `AA 01`              | 2 bytes    | Stops both drive motors. |
| `START_MOTORS` | `0x02` | `AA 02 <speed>`      | 3 bytes    | Starts both drive motors at `<speed>` (0–255, applied to both motors). |
| `FIRE`         | `0x03` | `AA 03`              | 2 bytes    | Triggers one shot — servo opens, holds ~1 s, closes automatically. No further action needed from the host. |

There is no acknowledgement, checksum, or response frame — this is a fire-and-forget, host-to-board protocol only.

### Byte Examples

- Stop motors: `AA 01`
- Start motors at half speed (128/255): `AA 02 80`
- Start motors at full speed (255/255): `AA 02 FF`
- Fire one shot: `AA 03`

### Example: Python (pyserial)

```python
import serial
import time

port = serial.Serial("/dev/ttyUSB0", 115200, timeout=1)

def stop_motors():
    port.write(bytes([0xAA, 0x01]))

def start_motors(speed: int):
    speed = max(0, min(255, speed))
    port.write(bytes([0xAA, 0x02, speed]))

def fire():
    port.write(bytes([0xAA, 0x03]))

stop_motors()
start_motors(128)
time.sleep(2)
fire()
time.sleep(1.5)
stop_motors()
```

### Example: Raw C (any UART-capable target)

```c
void send_frame(uint8_t cmd, const uint8_t *params, uint8_t param_len) {
    uart_write_byte(0xAA);
    uart_write_byte(cmd);
    for (uint8_t i = 0; i < param_len; i++) {
        uart_write_byte(params[i]);
    }
}

void stop_motors(void)      { send_frame(0x01, NULL, 0); }
void start_motors(uint8_t s){ send_frame(0x02, &s, 1); }
void fire(void)             { send_frame(0x03, NULL, 0); }
```

### Notes for Non-Arduino Integrators

- Configure your UART for **115200 baud, 8 data bits, no parity, 1 stop bit (8N1)**.
- The board's UART input is **5V-tolerant**, so a 5V-logic host can drive the board's receive line directly. The board's transmit line only outputs up to 3.3V — that's high enough for most 5V-logic receivers to register as a logic high, but add a level shifter on that line if your host's UART needs a stricter 5V threshold.
- Send frames back-to-back is fine; there's no minimum inter-frame delay required by the protocol, but be sensible about not spamming `FIRE` faster than the servo can physically cycle (~1 second per shot).
- Because there's no acknowledgement, treat this as an open-loop control link — if you need confirmation a command was received, that has to be handled at the application/mechanical level (e.g., watching the motors or listening for the servo).

## License

This project is licensed under the PolyForm Noncommercial License 1.0.0. See license tab for more details. 

## Contributing

Issue reporting is welcome. Please open an issue describing the board revision and firmware version you're working against, since the UART protocol may evolve between hardware revisions.
