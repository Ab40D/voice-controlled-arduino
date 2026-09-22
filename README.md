# Voice-Controlled Arduino LED

**Version 1 — speech, then a serial token, then an LED.**

A Linux PC listens on a microphone, turns a short phrase into a
protocol command, and sends that command to an Arduino Uno R3 over
the USB cable you already use for upload. The Uno turns an LED on or
off and answers with the LED state.

The spoken sentence never crosses the serial port.

This version stops at the cable. It does not use MQTT, Wi-Fi,
Bluetooth, Node-RED, Home Assistant, KNX, or a cloud home-automation
service. Those appear only as a roadmap.

```text
Human → PC microphone → speech-to-text → command parser
      → USB serial → Arduino Uno R3 → GPIO → LED
      ← serial status ←
```

## Demo

Device first, then voice. Close the Arduino Serial Monitor before
the Python program opens the port.

```text
Listening...

You said: turn the light on

Command: LIGHT_ON

Sending to Arduino...

Arduino: LIGHT_STATE:ON
```

The L LED on the Uno is on after that reply, and off after
`LIGHT_OFF`. A sentence that is not a light command is printed and
not sent.

A three-minute shoot sheet is in [docs/video-script.md](docs/video-script.md).

## Architecture

![Version 1 architecture. Speech stays on the PC. Only LIGHT_ON or LIGHT_OFF crosses USB serial to the Uno.](diagrams/architecture.png)

What happens when you say "Turn the light on."

1. The microphone captures the phrase through the Linux audio stack
   (PipeWire or PulseAudio, then ALSA).
2. Speech-to-text returns text, typically `turn the light on`. The
   default engine sends that audio to Google's Web Speech endpoint
   and needs internet. Vosk can do this step offline.
3. The parser normalizes case and punctuation and matches a phrase.
   It emits the token `LIGHT_ON`. If the phrase is not a command, the
   path stops here.
4. Python writes the ASCII bytes `LIGHT_ON\n` to the USB serial port
   at 9600 8N1.
5. The Uno's USB bridge (ATmega16U2) delivers those bytes to the
   ATmega328P UART.
6. The sketch reads until newline, matches the token, and executes
   `digitalWrite` on pin 13, `HIGH`.
7. Pin 13 sources current through the onboard resistor and the L LED
   turns on.
8. The sketch writes `LIGHT_STATE:ON`. Python prints that line.

The PC does not infer success from the write. It prints the board's
reply, or a timeout.

Status is not a sensor reading. `STATUS` asks the sketch for the
state it last commanded. Version 1 has no light sensor, and the
return path is still useful: it distinguishes "the write was queued"
from "the sketch accepted the command."

![Runtime flow, from listening through the serial reply and back to listening.](diagrams/flowchart.png)

Speech stays in `pc/command_parser.py`. The sketch in
`arduino/voice_light_controller/` compares tokens only. That split is
the design.

## Hardware

You already have the Uno R3 and a USB cable. For the first build,
buy nothing.

| Part | Version 1 | Notes |
| --- | --- | --- |
| Arduino Uno R3 | required | ATmega328P. Clones work if the cable enumerates a serial port. |
| USB data cable | required | Charge-only cables power the board and create no `/dev/ttyACM*` port. |
| Onboard L LED | used by default | Digital pin 13, `LED_BUILTIN`. Already on the board. |
| LED, any common color | optional | Only for the external circuit below. |
| 220 Ω resistor, 1/4 W | optional | Required if you add that LED. 330 Ω is also fine. |
| Breadboard and two jumper wires | optional | Only for the external LED. |

Do not add a relay, transistor, or mains device for this version.
The load is an LED.

Pin 13 is active high. The onboard LED is dimmer than a discrete LED
because the board already has a series resistor, usually about 1 kΩ.
That is an advantage here: there is nothing to wire backwards.

Ignore the ON LED (power) and the TX/RX LEDs (serial traffic). The
controlled LED is marked **L**.

## Wiring

![Built-in LED, and the optional external LED on pin 8 with a 220 ohm resistor.](diagrams/wiring.png)

### Built-in LED

| Arduino | Connection |
| --- | --- |
| USB | Linux PC, data cable |
| Pin 13 (`LED_BUILTIN`) | Onboard L LED. No wire. |
| GND, 5V, D0, D1 | Not used |

Leave this line in the sketch:

```cpp
const int LED_PIN = LED_BUILTIN;
```

### Optional external LED

Use this only after the onboard LED works. It makes polarity and the
resistor visible, which the onboard LED hides.

| From | To |
| --- | --- |
| D8 | one leg of a 220 Ω resistor |
| other leg of the resistor | LED anode, the longer leg |
| LED cathode, the shorter leg, flat side of the rim | GND |

Any GND pin is the same node. Do not use D0 or D1. They are the UART
behind the USB port, and a load there can corrupt the protocol.

Electrically the resistor can sit on either side of the LED. The LED
direction cannot. A backwards LED stays dark and is not harmed at
5 V. A missing resistor can damage the pin.

Why 220 Ω: the pin is 5 V, a red LED drops about 2 V, and a sensible
current is 10–15 mA. `(5 − 2) / 0.015 = 200 Ω`. 220 Ω gives about
14 mA, under the 20 mA you should stay below and far under the
40 mA absolute maximum. A blue or white LED drops more voltage, so
the same resistor makes it dimmer and is still safe. Do not go below
150 Ω.

Then change one line and upload again:

```cpp
const int LED_PIN = 8;
```

```text
D8 ----[ 220 ohm ]----|>|---- GND
                      LED
                   anode   cathode
                   long    short
```

## Software requirements

| Software | Requirement |
| --- | --- |
| PC | Linux. Commands below are Debian/Ubuntu; the port names are the same on other distributions. |
| Python | 3.10 or newer, including 3.13. SpeechRecognition 3.17 declares that range. |
| Arduino IDE 2.x, or arduino-cli | To compile and upload the sketch. |
| Internet | Only for the default Google speech engine, and to install packages. |
| Microphone | Built-in or USB. Test it before blaming the sketch. |

Python packages are `pyserial` and `SpeechRecognition`. PyAudio is
optional. Vosk is optional and offline.

## Installation

```bash
sudo apt update
sudo apt install -y python3 python3-pip python3-venv python3-dev \
    portaudio19-dev build-essential alsa-utils

cd voice-controlled-arduino
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r pc/requirements.txt
python -m pip install -r pc/requirements-pyaudio.txt
```

If the PyAudio install fails, continue. The controller falls back to
`arecord`. Do not use `sudo pip`, and do not use
`--break-system-packages`. Current Debian and Ubuntu refuse system-wide
pip (PEP 668). The virtualenv is the fix.

Serial permission, once per user. Log out and back in afterwards:

```bash
sudo usermod -aG dialout "$USER"
```

Upload the sketch. In Arduino IDE: **File → Open**
`arduino/voice_light_controller/voice_light_controller.ino`,
board **Arduino Uno**, port `/dev/ttyACM0` or the port from
`--list-ports`, then **Upload**. The folder name must match the
sketch name. It already does.

With arduino-cli, if you prefer a terminal:

```bash
arduino-cli core install arduino:avr
arduino-cli compile --fqbn arduino:avr:uno arduino/voice_light_controller
arduino-cli upload -p /dev/ttyACM0 --fqbn arduino:avr:uno arduino/voice_light_controller
```

## Configuration

| Item | Value |
| --- | --- |
| Baud | 9600. `BAUD_RATE` in the sketch, `--baud` on the PC. |
| Framing | 8N1. pyserial and `Serial.begin` both default to it. |
| Line ending | PC sends `\n`. In Serial Monitor set **Newline** or **Both NL & CR**. |
| Port | Auto-detected, or `--port /dev/ttyACM0`. Override with `VOICE_ARDUINO_PORT`. |
| Speech language | `en-US`. Override with `--language` or `VOICE_LANGUAGE`. |
| LED pin | `LED_BUILTIN` unless you wired D8. |

Find the port:

```bash
python3 pc/voice_controller.py --list-ports
ls -l /dev/ttyACM* /dev/ttyUSB* 2>/dev/null
```

An official Uno R3 appears as `/dev/ttyACM0` (USB CDC, VID `2341`).
A CH340 clone usually appears as `/dev/ttyUSB0`.

## Usage

Activate the virtualenv in every new terminal. Close Serial Monitor
first. Only one process may own the port.

```bash
source .venv/bin/activate
python3 pc/voice_controller.py --list-ports
python3 pc/voice_controller.py --send LIGHT_ON
python3 pc/voice_controller.py --send LIGHT_OFF
python3 pc/voice_controller.py
```

Accepted speech includes `turn the light on`, `turn the light off`,
`light on`, `light off`, and the close variants in
`pc/command_parser.py`. Add phrases there. Do not add them to the
sketch.

Text backup, useful on camera if the microphone fails:

```bash
python3 pc/voice_controller.py --mode text
```

Stop the voice loop with Ctrl+C.

## Serial protocol

Full rules, reset behavior, and byte dumps: [docs/protocol.md](docs/protocol.md).

| Direction | Line | Meaning |
| --- | --- | --- |
| PC → Uno | `LIGHT_ON` | LED on |
| PC → Uno | `LIGHT_OFF` | LED off |
| PC → Uno | `STATUS` | Report state, do not change it |
| Uno → PC | `READY` | Sketch started after reset |
| Uno → PC | `LIGHT_STATE:ON` | LED is on |
| Uno → PC | `LIGHT_STATE:OFF` | LED is off |
| Uno → PC | `ERROR:UNKNOWN_COMMAND` | LED unchanged |
| Uno → PC | `ERROR:LINE_TOO_LONG` | LED unchanged |

USB serial here is a virtual COM port. The PC opens a device file.
Bytes go to the 16U2, then to the 328P's UART. Baud rate is the
symbol rate both ends must agree on. Flow control is off.

Opening the port resets the Uno, because DTR is wired to RESET. The
controller reads immediately and waits up to 4 seconds for `READY`.
It does not sleep and then flush the buffer. That pattern deletes
the ready line.

The supported client is `pc/voice_controller.py`. The reduced
exchange, if you want to see the rules without the microphone:

```python
import time
import serial

ser = serial.Serial("/dev/ttyACM0", 9600, timeout=2)
deadline = time.monotonic() + 4
while time.monotonic() < deadline:
    if ser.readline().decode("utf-8", errors="replace").strip() == "READY":
        break
ser.write(b"LIGHT_ON\n")
print(ser.readline().decode().strip())
```

Do not run that snippet while this program, or Serial Monitor, has
the port open.

## Speech-to-text

Default: **Google Web Speech** through the `SpeechRecognition`
library (`recognize_google`).

Use it for Version 1 because a short command demo needs accuracy more
than it needs privacy, and it needs no model download. It is an
unofficial endpoint, the library's built-in key can be revoked, and
there is no service agreement. That is acceptable for a local
prototype and a bad choice for a product. Audio leaves the machine.
The program prints that fact when it starts.

| Engine | Network | Install | When to use it |
| --- | --- | --- | --- |
| Google Web Speech | required | already in `requirements.txt` | Version 1 demo. Best accuracy for short English phrases. |
| Vosk | none | `requirements-offline.txt` plus a model | Offline English, and later a French model. Weaker on accents. |
| Whisper | none | not a dependency of this version | Future path if French and Algerian Darija have to share one engine. Heavier, and Darija will still be imperfect. |

Language codes for the Google engine include `en-US`, `fr-FR`, and
`ar-DZ`. `ar-DZ` is Arabic (Algeria) as that service defines it. It
is not a promise about everyday Algerian Darija. Version 1 ships
English phrases, plus a few French phrases so `fr-FR` is not a dead
end. Add Darija phrases only after you have seen the transcript your
engine actually returns.

Microphone check:

```bash
arecord -l
arecord -d 3 -f S16_LE -r 16000 -c 1 /tmp/mic-test.wav
aplay /tmp/mic-test.wav
python3 pc/voice_controller.py --list-mics
```

Speak after the program prints `Listening...`, about a forearm's
length from the microphone. The first run asks for a quiet moment so
it can measure the room.

Offline English:

```bash
python -m pip install -r pc/requirements-offline.txt
mkdir -p "$HOME/vosk-models" && cd "$HOME/vosk-models"
wget https://alphacephei.com/vosk/models/vosk-model-small-en-us-0.15.zip
unzip vosk-model-small-en-us-0.15.zip
python3 pc/voice_controller.py --engine vosk \
  --vosk-model "$HOME/vosk-models/vosk-model-small-en-us-0.15"
```

## Testing

Do these in order. Do not start with the microphone.

| Step | Command or action | Expected |
| --- | --- | --- |
| Parser | `python3 pc/test_parser.py` | All tests OK. No board required. |
| Parse one phrase | `python3 pc/voice_controller.py --parse "turn the light on"` | `Command: LIGHT_ON` |
| Reject one phrase | `python3 pc/voice_controller.py --parse "what time is it"` | `not recognized`, exit code 2 |
| Upload | IDE or arduino-cli upload | Done uploading. L LED may flash during reset, then stays off. |
| Serial Monitor | 9600, Newline, send `LIGHT_ON` | `LIGHT_STATE:ON`. L LED on. TX flickers. ON LED was already on. |
| Serial Monitor | `LIGHT_OFF` | `LIGHT_STATE:OFF`. L LED off. |
| Serial Monitor | `nope` | `ERROR:UNKNOWN_COMMAND`. LED unchanged. |
| Serial Monitor | `STATUS` | `LIGHT_STATE:OFF` if you just turned it off. LED unchanged. |
| Close Monitor | — | Port is free. |
| Python, no voice | `python3 pc/voice_controller.py --send LIGHT_ON` | `Arduino: READY` then `Arduino: LIGHT_STATE:ON`. L LED on. |
| Python, no voice | `--send LIGHT_OFF` then `--send NOPE` | State off, then `ERROR:UNKNOWN_COMMAND`. |
| Microphone | `arecord` test above | You hear yourself. |
| Speech | `python3 pc/voice_controller.py`, then "turn the light on" | Transcript, `LIGHT_ON`, `LIGHT_STATE:ON`, L LED on. |
| Speech off | "turn the light off" | `LIGHT_STATE:OFF`, L LED off. |
| Speech reject | "what time is it" | Not recognized. Nothing new on the LED. |
| Text backup | `--mode text`, type `turn the light on` | Same serial result as voice. |

## Troubleshooting

| Symptom | What is wrong | Fix |
| --- | --- | --- |
| No `/dev/ttyACM*` or `/dev/ttyUSB*` | Charge-only cable, bad socket, or the board is not an Uno-class CDC/USB-serial device | Try another cable and USB port. Power LED on is not proof of data. |
| `Permission denied` on `/dev/ttyACM0` | User is not in `dialout` | `sudo usermod -aG dialout "$USER"`, then log out and back in. Temporary: `sudo chmod a+rw /dev/ttyACM0`. |
| Port busy, or Python and the IDE both fail | Serial Monitor, a second Python process, or ModemManager has the port | Close Serial Monitor. `fuser -v /dev/ttyACM0`. If ModemManager grabs ACM devices: `sudo systemctl stop ModemManager`. |
| CH340 clone never stays as `/dev/ttyUSB0` | `brltty` claims some CH340 IDs | `systemctl status brltty`. Removing `brltty` fixes it if you do not use a braille display. Official Unos use `ttyACM` and usually avoid this. |
| Wrong port | Auto-detect saw two adapters | `--list-ports`, then `--port` the Uno. |
| Garbled replies, or every line is an error | Baud mismatch, or two programs reading | Both sides at 9600. One owner. Serial Monitor line ending on Newline. |
| First command does nothing, later ones work | Command was sent during the bootloader, or `READY` was flushed | Use this repository's client. It reads immediately and waits for `READY`. |
| `externally-managed-environment` | pip was aimed at system Python | Use the virtualenv in Installation. |
| `PyAudio` fails to build | Missing PortAudio headers | `sudo apt install portaudio19-dev python3-dev build-essential`, then reinstall. Or skip it and use `--input arecord`. |
| `arecord` finds nothing | Mic muted, or PipeWire has no ALSA route | Unmute in the desktop sound settings. `sudo apt install pipewire-alsa`. Pass `--alsa-device plughw:0,0` from `arecord -l`. |
| Speech service error | No internet, or Google's endpoint refused the request | Check the network. Or switch to `--engine vosk`. |
| Heard the wrong words | Noise, wrong device, wrong language | `--list-mics`, `--mic`, speak after `Listening...`. `--energy` or `--threshold` if the room is loud. |
| LED never changes, serial replies are correct | You are watching TX, RX, or ON | Watch L. TX should flicker when the reply is sent. |
| L LED blinks once a second and ignores commands | The Blink example is still loaded | Upload `voice_light_controller.ino` again. |
| External LED dark, onboard test worked | Backwards LED, wrong pin, or pin not changed in the sketch | Long leg toward the resistor. `LED_PIN` must be `8`. Resistor in series either side. |
| External LED always on | Wired to 5V instead of D8 | The anode path must start at D8, not at 5V. |

## Project structure

```text
voice-controlled-arduino/
├── README.md
├── LICENSE
├── arduino/
│   └── voice_light_controller/
│       └── voice_light_controller.ino
├── pc/
│   ├── voice_controller.py
│   ├── command_parser.py
│   ├── test_parser.py
│   ├── requirements.txt
│   ├── requirements-pyaudio.txt
│   ├── requirements-offline.txt
│   └── README.md
├── diagrams/
│   ├── architecture.png
│   ├── architecture.svg
│   ├── wiring.png
│   ├── wiring.svg
│   ├── flowchart.png
│   └── flowchart.svg
└── docs/
    ├── protocol.md
    └── video-script.md
```

The sketch lives in its own folder because the Arduino IDE requires
the folder name and the `.ino` name to match. `command_parser.py` is
separate so the phrase list can be tested without a board.
`build_diagrams.py` regenerates the figures if you change them.

## Future roadmap

Not implemented. Listed so the boundary in Version 1 has somewhere
to go.

| Version | Path | What changes |
| --- | --- | --- |
| 1, this repository | Voice → serial → Arduino Uno | Perception on the PC. One LED. USB only. |
| 2 | Voice → network → ESP32 | Same token, different transport. Still no broker. |
| 3 | Voice → MQTT → ESP32 | The token becomes a payload. The device still does not parse English. |
| 4 | Voice → Node-RED → MQTT → device | A flow owns routing. The sketch stays small. |
| 5 | Voice → Node-RED → KNX → actuator → a real light | The LED is replaced by a lighting actuator. The command still starts as a token, not as a sentence on the bus. |

Each step should keep the rule this version exists to prove: language
is allowed to be messy only on the side that has the microphone.

## Scope

One LED. One local USB port. English commands, with a small French
phrase list ready for a French recognizer. No authentication on the
serial port. No mains switching. The default speech engine sends
audio to Google. This is a prototype of a boundary, not a home
installation.

## License

[MIT](LICENSE). Replace the copyright line with your name before you
publish.
