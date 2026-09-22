# Voice-Controlled Arduino LED

### Version 1 — Voice → Serial → Arduino → LED

A simple project that turns a spoken command into a real hardware action.

A Linux PC listens to your voice, converts it into text, maps it to a simple command, and sends that command to an Arduino Uno over USB serial. The Arduino receives the command and controls an external LED connected to digital pin 8.

> The important idea: **the Arduino never needs to understand human language.**
> The PC handles speech. The Arduino handles hardware.

---

## ✨ What This Project Does

You say:

> "Turn the light on"

The system turns it into:

```text
LIGHT_ON
```

Then sends it through USB serial:

```text
PC → USB Serial → Arduino Uno → D8 → LED
```

The Arduino replies:

```text
LIGHT_STATE:ON
```

The PC displays the response.

A simplified view:

```text
🎤 Your Voice
      ↓
🧠 Speech-to-Text
      ↓
🔎 Command Parser
      ↓
"LIGHT_ON"
      ↓
🔌 USB Serial
      ↓
🤖 Arduino Uno
      ↓
💡 External LED
```

---

## 🎥 Demo

Example:

```text
Listening...

You said: turn the light on

Command: LIGHT_ON

Sending to Arduino...

Arduino: LIGHT_STATE:ON
```

The LED turns on.

When you say:

```text
turn the light off
```

the command becomes:

```text
LIGHT_OFF
```

and the LED turns off.

The project also rejects unrelated sentences instead of sending them to the Arduino.

---

## 🧠 Why This Project?

This is **Version 1** of a larger learning path.

The goal is not to build a complete smart-home system immediately.

The goal is to understand how the layers are connected:

```text
Human
  ↓
Speech
  ↓
Software
  ↓
Protocol
  ↓
Serial Communication
  ↓
Microcontroller
  ↓
Physical Output
```

Later versions can replace the USB cable with networking, MQTT, Node-RED and eventually KNX.

The command itself can stay simple while the communication layer becomes more advanced.

---

## 🏗️ Architecture

![System architecture](diagrams/architecture.png)

```text
🎤 Voice
   ↓
🧠 Speech-to-Text
   ↓
🔎 Command Parser
   ↓
LIGHT_ON / LIGHT_OFF
   ↓
🔌 USB Serial
   ↓
🤖 Arduino Uno R3
   ↓
📍 Digital Pin 8
   ↓
💡 External LED
   ↓
↩️ Serial Response
   ↓
💻 PC
```

The speech layer and hardware layer are intentionally separated.

The Arduino only receives predefined protocol commands.

It does not receive:

```text
"Hey Arduino, could you please turn the light on?"
```

It receives:

```text
LIGHT_ON
```

This makes the hardware side simple and reusable.

---

## ⚙️ How It Works

### 1. 🎤 Voice Input

The microphone captures the spoken command.

For example:

```text
Turn the light on
```

### 2. 🧠 Speech-to-Text

The speech recognition engine converts the audio into text.

Example:

```text
turn the light on
```

The default Version 1 setup uses Google Web Speech through the `SpeechRecognition` Python library.

This requires an internet connection.

### 3. 🔎 Command Parser

Python checks the recognized sentence and maps it to a protocol token:

```text
turn the light on
        ↓
LIGHT_ON
```

and:

```text
turn the light off
        ↓
LIGHT_OFF
```

If the sentence is not recognized as a valid command, nothing is sent to the Arduino.

### 4. 🔌 USB Serial

Python sends:

```text
LIGHT_ON\n
```

to the Arduino at:

```text
9600 baud
8N1
```

### 5. 🤖 Arduino

The Arduino reads the command until the newline character.

For:

```text
LIGHT_ON
```

it executes:

```cpp
digitalWrite(LED_PIN, HIGH);
```

For:

```text
LIGHT_OFF
```

it executes:

```cpp
digitalWrite(LED_PIN, LOW);
```

### 6. 💡 Physical Output

The external LED connected to **D8** turns on or off.

### 7. ↩️ Feedback

The Arduino sends a response back to the PC:

```text
LIGHT_STATE:ON
```

or:

```text
LIGHT_STATE:OFF
```

The PC displays the actual response from the Arduino instead of assuming that the command succeeded.

---

## 🔌 Hardware

### Required

| Component         | Purpose                      |
| ----------------- | ---------------------------- |
| 🤖 Arduino Uno R3 | Microcontroller              |
| 🔌 USB data cable | Power + serial communication |
| 💡 LED            | Physical output              |
| 🧱 220 Ω resistor | LED current limiting         |
| 🔗 Jumper wires   | Connections                  |
| 🟫 Breadboard     | Optional                     |

No relay or mains equipment is required.

This project only controls a low-voltage LED.

---

## 🔧 Wiring

The external LED is connected to **digital pin 8**.

```text
Arduino D8
    │
    │
 [220 Ω]
    │
    ▼
 LED Anode (+)
 LED Cathode (-)
    │
    ▼
   GND
```

### Connections

```text
D8 ─── 220 Ω resistor ─── LED long leg (+)

LED short leg (-) ─────── GND
```

The resistor must be connected in series with the LED.

Do not connect the LED directly between D8 and GND.

The Arduino sketch uses:

```cpp
const int LED_PIN = 8;
```

---

## 💻 Software

### Requirements

* Linux
* Python 3.10+
* Arduino Uno R3
* Arduino IDE or `arduino-cli`
* USB data cable
* Microphone
* Internet connection for Google Web Speech

Python packages:

```text
pyserial
SpeechRecognition
```

PyAudio is optional.

The controller can fall back to `arecord` when PyAudio is not available.

---

## 📦 Installation

### Arch Linux

Install the required system packages:

```bash
sudo pacman -S python python-pip alsa-utils
```

Create the virtual environment:

```bash
python3 -m venv .venv
source .venv/bin/activate
```

Install Python dependencies:

```bash
python -m pip install --upgrade pip
python -m pip install -r pc/requirements.txt
```

Optional PyAudio support:

```bash
python -m pip install -r pc/requirements-pyaudio.txt
```

### Debian / Ubuntu

```bash
sudo apt update
sudo apt install -y python3 python3-pip python3-venv python3-dev \
    portaudio19-dev build-essential alsa-utils
```

Then:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r pc/requirements.txt
```

---

## 🔐 Serial Permissions

On Linux, the user may need access to the serial device.

For Debian/Ubuntu:

```bash
sudo usermod -aG dialout "$USER"
```

Log out and back in afterwards.

On Arch Linux, check the permissions of the detected device if access is denied.

---

## 🔎 Find the Arduino Port

The Arduino may appear as:

```text
/dev/ttyACM0
```

or, depending on the USB-to-serial chip:

```text
/dev/ttyUSB0
```

List available ports:

```bash
python3 pc/voice_controller.py --list-ports
```

You can also check:

```bash
ls -l /dev/ttyACM* /dev/ttyUSB* 2>/dev/null
```

---

## 🤖 Upload the Arduino Code

Open:

```text
arduino/voice_light_controller/voice_light_controller.ino
```

In Arduino IDE select:

```text
Board → Arduino Uno
```

Then select the detected port.

For example:

```text
/dev/ttyUSB0
```

Upload the sketch.

The Arduino should start with:

```text
READY
```

---

## 🧪 Test the Arduino First

Before using voice recognition, test the hardware and serial protocol.

Open Serial Monitor:

```text
Baud: 9600
Line ending: Newline
```

Send:

```text
LIGHT_ON
```

Expected:

```text
LIGHT_STATE:ON
```

The external LED should turn on.

Then:

```text
LIGHT_OFF
```

Expected:

```text
LIGHT_STATE:OFF
```

The LED should turn off.

You can also test:

```text
STATUS
```

Expected:

```text
LIGHT_STATE:OFF
```

or:

```text
LIGHT_STATE:ON
```

depending on the last commanded state.

---

## ⚠️ Important: Serial Monitor

Only **one program** can own the serial port at a time.

Before running Python:

> **Close the Arduino IDE Serial Monitor.**

Otherwise you may get:

```text
/dev/ttyUSB0 is already open
```

Check which process is using the port:

```bash
fuser -v /dev/ttyUSB0
```

---

## 🐍 Test Python Without Voice

Activate the virtual environment:

```bash
source .venv/bin/activate
```

Then:

```bash
python3 pc/voice_controller.py --send LIGHT_ON
```

Expected:

```text
Port: /dev/ttyUSB0
Baud: 9600
Waiting for Arduino READY...
Arduino: READY
```

The LED should turn on.

Test OFF:

```bash
python3 pc/voice_controller.py --send LIGHT_OFF
```

The LED should turn off.

This confirms:

```text
Python → USB Serial → Arduino → LED
```

before introducing speech recognition.

---

## 🎤 Voice Control

Start the controller:

```bash
python3 pc/voice_controller.py
```

Speak after:

```text
Listening...
```

Try:

```text
Turn the light on
```

or:

```text
Turn the light off
```

The complete path becomes:

```text
🎤 Voice
 ↓
🧠 Speech Recognition
 ↓
🔎 Parser
 ↓
LIGHT_ON
 ↓
🔌 USB Serial
 ↓
🤖 Arduino
 ↓
💡 LED
```

---

## 🎙️ Microphone Test

Check available recording devices:

```bash
arecord -l
```

Record three seconds:

```bash
arecord -d 3 -f S16_LE -r 16000 -c 1 /tmp/mic-test.wav
```

Play the recording:

```bash
aplay /tmp/mic-test.wav
```

If you can hear yourself, the microphone is working.

---

## 📡 Serial Protocol

The protocol is intentionally simple.

### PC → Arduino

```text
LIGHT_ON
LIGHT_OFF
STATUS
```

### Arduino → PC

```text
READY
LIGHT_STATE:ON
LIGHT_STATE:OFF
ERROR:UNKNOWN_COMMAND
ERROR:LINE_TOO_LONG
```

Example:

```text
PC → Arduino
LIGHT_ON

Arduino → PC
LIGHT_STATE:ON
```

The Arduino does not parse natural language.

It only understands the protocol.

---

## 🧪 Testing

The recommended testing order is:

```text
1️⃣ Command parser
       ↓
2️⃣ Arduino + LED
       ↓
3️⃣ Serial communication
       ↓
4️⃣ Python command
       ↓
5️⃣ Microphone
       ↓
6️⃣ Speech recognition
```

Test the parser:

```bash
python3 pc/test_parser.py
```

Test parsing directly:

```bash
python3 pc/voice_controller.py --parse "turn the light on"
```

Expected:

```text
Command: LIGHT_ON
```

Test a sentence that should not control the LED:

```bash
python3 pc/voice_controller.py --parse "what time is it"
```

The command should be rejected.

---

## 🗂️ Project Structure

```text
voice-controlled-arduino/
│
├── 📄 README.md
├── 📄 LICENSE
│
├── 🤖 arduino/
│   └── voice_light_controller/
│       └── voice_light_controller.ino
│
├── 🐍 pc/
│   ├── voice_controller.py
│   ├── command_parser.py
│   ├── test_parser.py
│   ├── requirements.txt
│   ├── requirements-pyaudio.txt
│   └── requirements-offline.txt
│
├── 📐 diagrams/
│   ├── architecture.png
│   ├── architecture.svg
│   ├── wiring.png
│   ├── wiring.svg
│   ├── flowchart.png
│   └── flowchart.svg
│
└── 📚 docs/
    ├── protocol.md
    └── video-script.md
```

---

## 🛠️ Troubleshooting

### `/dev/ttyUSB0 is already open`

Close Arduino Serial Monitor.

Then:

```bash
fuser -v /dev/ttyUSB0
```

Only one application should use the port.

### LED does not turn on

Check:

```text
D8 → 220 Ω → LED long leg
LED short leg → GND
```

Also verify the sketch contains:

```cpp
const int LED_PIN = 8;
```

### Arduino is detected but Python cannot access it

Check:

```bash
ls -l /dev/ttyUSB* /dev/ttyACM*
```

Then check permissions and serial-group membership.

### `arecord: command not found`

On Arch:

```bash
sudo pacman -S alsa-utils
```

On Debian/Ubuntu:

```bash
sudo apt install alsa-utils
```

### Speech recognition does not work

Check the microphone:

```bash
arecord -l
```

Then test recording manually.

Remember that the default Google speech engine requires an internet connection.

---

## 🚀 Roadmap

This project is **Version 1** of a larger progression.

### V1 — Fundamentals

```text
🎤 Voice
 ↓
💻 PC
 ↓
🔌 USB Serial
 ↓
🤖 Arduino
 ↓
💡 LED
```

### V2 — Multiple Outputs

```text
Voice
 ↓
Serial
 ↓
Arduino
 ↓
💡 💡 💡 Multiple Outputs
```

### V3 — Network

```text
Voice
 ↓
Wi-Fi
 ↓
ESP32
 ↓
Device
```

### V4 — IoT

```text
Voice
 ↓
MQTT
 ↓
ESP32
 ↓
Device
```

### V5 — Automation

```text
Voice
 ↓
Node-RED
 ↓
MQTT
 ↓
ESP32
 ↓
Device
```

### V6 — Building Automation

```text
Voice
 ↓
Node-RED
 ↓
KNX
 ↓
Actuator
 ↓
💡 Real Light
```

The goal is to build each layer step by step instead of jumping directly into a complex Building Automation system.

---

## 🔮 What Comes Next?

The next versions will gradually introduce:

* 📡 Wi-Fi
* 📬 MQTT
* 🔀 Node-RED
* 🏠 Home Automation
* ⚡ KNX
* 💡 Real lighting actuators
* 🔄 Feedback and closed-loop control

The same basic principle remains:

> **Human language stays at the software layer. Hardware receives simple, well-defined commands.**

---

## 📺 Video

This project is part of the:

### **From Arduino to Building Automation**

A practical progression from basic microcontroller projects to IoT, automation and eventually Building Automation with KNX.

The complete shoot sheet is available here:

[docs/video-script.md](docs/video-script.md)

---

## 👤 Author

**Abdelkhalek Mammeri**

Electronics • IoT • Building Automation • IT/OT

GitHub: [Ab40D](https://github.com/Ab40D)
