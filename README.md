# 360° Wireless Radar System — ESP32 + Stepper Motor + HC-SR04

Watch Full Tutorial:- 

A real-time, wireless radar system that continuously scans a full 360° arc, measures distances using an ultrasonic sensor, and visualizes live object positions on a radar-style UI — all over Wi-Fi, with no USB cable involved.

---

## Overview

This is not a fancy sensor fusion project. It's a clean, minimal system that answers one question: *where are objects around me, at every angle?*

The ESP32 drives a stepper motor that rotates an HC-SR04 ultrasonic sensor in a continuous 360° sweep. At regular angular intervals, it reads the distance and transmits `angle,distance` pairs over a TCP socket to a Processing-based radar UI running on a PC. The UI plots those readings in real time, giving you a live radar display.

Everything — motor control, sensing, Wi-Fi server, and data streaming — runs on a single ESP32. No secondary MCU, no extra processing board.

---

## System Architecture

The system is split into three layers:

```
┌──────────────────────────────────┐
│         Sensing Layer            │
│  HC-SR04 → distance in cm        │
│  Stepper → angle in degrees      │
└────────────────┬─────────────────┘
                 │
┌────────────────▼─────────────────┐
│        Processing Layer          │
│  ESP32 calculates angle from     │
│  step count, reads sensor, and   │
│  formats + transmits data        │
└────────────────┬─────────────────┘
                 │  Wi-Fi TCP
┌────────────────▼─────────────────┐
│       Visualization Layer        │
│  Processing (Java) receives      │
│  data and renders radar in       │
│  real time on PC                 │
└──────────────────────────────────┘
```

---

## Hardware Components

| Component | Purpose |
|---|---|
| ESP32 Development Board | Main controller — runs Wi-Fi server, drives motor, reads sensor |
| 28BYJ-48 Stepper Motor | Rotates the sensor platform with precise angular control |
| ULN2003 Driver Board | Drives the stepper coils from ESP32 GPIO signals |
| HC-SR04 Ultrasonic Sensor | Measures distance to objects in the sensor's line of sight |
| LM2596 Buck Converter | Steps down power supply voltage to a safe 5V for the motor |
| 18650 Batteries x 2 | Powers the motor via the buck converter |
| Jumper wires + breadboard | Connections |
| Rotating platform / mount | Holds the HC-SR04 on the motor shaft |

---

## Power System

The 28BYJ-48 stepper motor runs on 5V but draws enough current (especially under load) that powering it directly from the ESP32's 3.3V or USB 5V rail causes voltage drops — which leads to missed steps, unstable Wi-Fi, or ESP32 resets.

The solution here is simple: use a separate power path for the motor.

```
18650 Batteries (In Series Connections)
       │
  LM2596 Buck Converter
       │
    5V Output ──→ ULN2003 VCC + Motor coils
       │
      GND ──→ Common ground with ESP32
```


## Stepper Motor Logic

### Why a stepper motor, not a servo?

Servo motors work using PWM duty cycle — you send a pulse width and the servo *tries* to reach the corresponding angle. In practice, there's mechanical hysteresis, and the angle isn't perfectly repeatable, especially under varying loads. For this application, where object position depends entirely on knowing the *exact* angle the sensor is pointing, that imprecision breaks the mapping.

A stepper motor, on the other hand, moves in discrete, fixed-angle steps. Each electrical pulse moves the rotor by a known amount. No feedback needed. No estimation. You simply count the steps.

### Step angle and resolution

The 28BYJ-48 uses a geared design with a half-step sequence. In half-step mode:

```
Steps per revolution = 4096
Step angle = 360° / 4096 ≈ 0.088° per step
```

In the firmware, the motor advances `STEPS_PER_SEND = 8` steps between each sensor reading. That gives an angular resolution of roughly 0.7° per data point — fine enough for meaningful object detection at close range.

### Angle calculation

The angle is derived purely from the cumulative step count:

```cpp
float angle = fmod((float)totalSteps / STEPS_PER_REV * 360.0, 360.0);
```

`totalSteps` is a running counter that increments or decrements based on direction. The motor reverses when the angle hits 0° or 359°, creating a continuous back-and-forth sweep.

### Half-step sequence

The ULN2003 is driven directly using a lookup table for the 8-phase half-step sequence:

```cpp
const int STEP_SEQ[8][4] = {
  {1,0,0,0}, {1,1,0,0}, {0,1,0,0}, {0,1,1,0},
  {0,0,1,0}, {0,0,1,1}, {0,0,0,1}, {1,0,0,1}
};
```

Each row corresponds to the coil states on IN1–IN4. No stepper library needed.

---

## Ultrasonic Sensing

The HC-SR04 works by emitting a short ultrasonic burst and timing how long it takes for the echo to return. Distance is:

```
distance (cm) = pulse_duration (µs) × 0.01715
```

The `0.01715` factor comes from the speed of sound (~343 m/s) divided by 2 (round-trip), converted to cm/µs.

In the firmware, readings outside the range of 5–50 cm are discarded:

```cpp
float cm = duration * 0.01715;
if (cm < 5.0 || cm > 50.0) return 0.0;
```

The lower bound filters out surface reflections from the motor mount. The upper bound keeps the visualization tight and avoids noise at long ranges.

A 30ms timeout is set on `pulseIn()` so the main loop doesn't stall if no echo returns:

```cpp
long duration = pulseIn(ECHO_PIN, HIGH, 30000);
if (duration == 0) return 0.0;
```

---

## Wireless Communication

### Why TCP over Wi-Fi?

TCP is connection-oriented — once a connection is established, data flows reliably in order, without packet loss. For a radar that needs continuous, correctly-ordered angle+distance pairs, TCP is the right choice. UDP would be faster but adds complexity (you'd need to handle out-of-order or missing packets).

### How it works conceptually

Think of it as a direct pipe between two programs. One end (the ESP32) opens a socket and waits. The other end (the PC running Processing) connects using the ESP32's IP address. Once connected, both sides can send and receive text data freely — like a persistent chat connection, but for sensor data.

### ESP32 as TCP Server

```cpp
WiFiServer server(TCP_PORT);  // TCP_PORT = 8888
server.begin();
```

After connecting to Wi-Fi, the ESP32 starts listening on port 8888. It waits for the first client that connects.

### Processing UI as TCP Client

```java
socket = new Socket(ESP32_IP, PORT);
reader = new BufferedReader(new InputStreamReader(socket.getInputStream()));
writer = new PrintWriter(socket.getOutputStream(), true);
```

Once this runs, the connection is live. `reader` pulls incoming data from the ESP32; `writer` pushes commands back to it.

---

## Connection Flow

```
1. ESP32 boots and connects to Wi-Fi
2. ESP32 starts TCP server on port 8888
3. ESP32 prints its local IP to Serial Monitor
4. User enters that IP in Processing sketch (ESP32_IP variable)
5. Processing sketch starts → socket connects to ESP32
6. Connection established — both sides are now live
```

The ESP32's IP address is printed on boot:

```
Serial.println(WiFi.localIP());
```

Copy that IP into the Processing sketch before running it.

---

## Command Flow

The UI sends plain-text commands terminated with a newline. The ESP32 reads them and reacts:

**Sending a command from Processing:**
```java
writer.println("START");  // Press 'S' key
writer.println("STOP");   // Press 'X' key
```

**Reading and handling on ESP32:**
```cpp
String cmd = client.readStringUntil('\n');
cmd.trim();

if (cmd == "START") {
  motorRunning = true;
  client.println("ACK_START");
}
else if (cmd == "STOP") {
  motorRunning = false;
  client.println("ACK_STOP");
}
```

The motor only moves when `motorRunning` is `true`. If the connection drops, the motor stays in whatever state it was last set to.

---

## Data Format and Transmission

Every time the motor advances `STEPS_PER_SEND` steps, the ESP32 takes a distance reading and sends a line in this format:

```
angle,distance\n
```

**Example output stream:**
```
44.3,0.0
45.1,23.5
45.8,24.0
46.6,0.0
47.4,18.2
```

A distance of `0.0` means either no echo was received within the timeout, or the reading was outside the 5–50 cm filter range. The UI ignores zero-distance readings for plotting — they just advance the sweep line without marking an object.

**Transmission code on ESP32:**
```cpp
String data = String(angle, 1) + "," + String(distance, 1) + "\n";
client.print(data);
```

The `1` in `String(angle, 1)` means one decimal place — keeps the data compact without losing useful precision.

---

## UI Processing (Visualization)

The Processing sketch runs a continuous `draw()` loop, reading incoming data and updating the display each frame.

### Parsing incoming data

```java
String line = reader.readLine();
String[] parts = split(line, ',');
float angle    = float(parts[0]);
float distance = float(parts[1]);
```

### Plotting objects

Each valid `angle,distance` pair is converted to Cartesian coordinates on the radar canvas:

```
x = cx + cos(radians(angle - 90)) × (distance / maxDistance) × maxRadius
y = cy + sin(radians(angle - 90)) × (distance / maxDistance) × maxRadius
```

The `-90` offset rotates the coordinate system so that 0° points upward (north) instead of right.

Objects are stored in a circular buffer. If a new reading arrives at an angle already occupied (within ±2°), it updates the existing entry rather than adding a duplicate. This keeps the display clean as the motor sweeps back and forth over the same region.

### Display elements

- **Concentric rings** — range markers at 12.5 cm, 25 cm, 37.5 cm, 50 cm
- **Radial lines** — angle markers every 30°
- **Sweep line** — green line showing current sensor angle, with a soft glow effect
- **Object dots** — red ellipses at detected object positions
- **HUD text** — current angle and object count

---

## Main Loop

The full scanning cycle, simplified:

```
loop:
  ├── Check for new TCP client connection
  ├── Read any incoming command (START / STOP)
  └── If motorRunning:
        ├── Advance motor by STEPS_PER_SEND steps
        ├── Update totalSteps and calculate angle
        ├── Check and flip direction at 0° / 359°
        ├── Read distance from HC-SR04
        └── Send "angle,distance\n" to connected client
```

This loop runs as fast as the step delay allows. With `STEP_DELAY_US = 1200` µs per step and 8 steps per send, each data point takes roughly 9.6 ms to generate — that's around 100 data points per second at full speed, though real throughput depends on Wi-Fi latency and the TCP stack.

The UI's `draw()` loop runs at Processing's default frame rate (60 fps) and reads whatever data is available in the buffer each frame.

---

## Design Challenges and Solutions

### Wire twist during continuous rotation

The sensor cables twist as the motor rotates. For a full 360° system with continuous back-and-forth motion, this is inevitable without a slip ring.

**Solution used here:** The motor reverses direction before completing a full revolution (at ~359°), keeping the wires from winding up beyond a manageable amount. This trades true 360° rotation for practical reliability.

**Proper fix for future versions:** Add a 4-channel slip ring between the rotating platform and the base. This allows unlimited rotation without wire damage.

### Motor power instability

Powering the stepper from the ESP32's USB 5V caused occasional resets under load.

**Solution:** Separate power rail via LM2596 buck converter. The motor draws current independently from the ESP32, and only GND is shared.

### Data rate vs. angular resolution

Higher step delay = slower sweep = more time per angle = better sensor readings per point.
Lower step delay = faster sweep = more data per second = lower accuracy per point.

`STEPS_PER_SEND = 8` and `STEP_DELAY_US = 1200` was a practical balance — fast enough to feel real-time, slow enough that the sensor has time to complete each measurement cycle properly.

---

## Key Insights

**Stepper position is deterministic.** Unlike closed-loop systems, the firmware never needs to ask "where am I?" — it always knows, because it counted every step from the start. This only holds if no steps are missed (which is why proper motor power matters).

**TCP gives you a reliable ordered stream.** You can treat the data coming in on the Processing side as a sequential list of `angle,distance` values — no packet reordering, no duplicates to filter.

**The sensor's blind zone matters.** The HC-SR04 can't reliably measure below ~2–3 cm or above ~400 cm. The 5–50 cm filter in firmware keeps the radar display useful and noise-free for close-range detection.

**Reversing at edges is a simple and effective design choice.** True 360° rotation would require mechanical changes (slip ring). Reversing at 359° gives you full coverage in both sweep directions with no hardware modifications.

---

## How to Run the Project

### 1. Flash the ESP32

Open `Arduino_IDE_Code.ino` in Arduino IDE. Edit the credentials:

```cpp
const char* SSID     = "Your_WiFi_Name";
const char* PASSWORD = "Your_WiFi_Password";
```

Make sure the board is set to **ESP32 Dev Module** and the correct COM port is selected. Upload the sketch.

### 2. Get the ESP32's IP address

Open Serial Monitor (baud: `115200`). After connecting to Wi-Fi, you'll see:

```
Connected!
ESP32 IP Address: 192.168.x.x
TCP server started on port 8888
```

Copy that IP address.

### 3. Configure the Processing sketch

Open `Processing_UI_CODE.pde`. Update the IP:

```java
String ESP32_IP = "192.168.x.x";  // Paste your ESP32 IP here
```

### 4. Run the Processing sketch

Press the Run button. The window opens and connects to the ESP32 automatically.

### 5. Control the radar

| Key | Action |
|---|---|
| `S` | Send START command — motor begins rotating |
| `X` | Send STOP command — motor stops |
| `C` | Clear all plotted objects from the display |

---

## Possible Improvements

**Slip ring for true 360° rotation**  
Replace the cable run with a 4-channel slip ring. This eliminates the reversal workaround and allows the motor to spin continuously in one direction.

**Persistent object map**  
Currently, the object buffer doesn't age out old readings. Adding a timestamp to each entry and fading/removing points older than a few sweep cycles would make the display more accurate.

**Distance filter refinement**  
A simple moving average or median filter over 3–5 consecutive readings at the same angle would reduce false positives from sensor noise.

**Faster data transport**  
Switching to UDP with a simple sequence number would reduce latency at higher sweep speeds, where TCP's acknowledgment overhead starts to matter.

**Web-based UI**  
Replace the Processing sketch with a browser-based radar using HTML5 Canvas and WebSockets. This removes the need for Processing to be installed on the PC.

**Multiple sensors**  
Mount 2–3 HC-SR04 sensors at fixed angular offsets on the platform. Interleave their readings to increase angular coverage per revolution.


## A Final Note

If you're here to copy-paste and move on — that's fine, but you'll hit problems you won't know how to debug. The angle calculation, the power separation, the TCP server setup — each of these is a small decision that had a reason behind it. If you understand those reasons, you can adapt this to different sensors, different motors, different ranges, or an entirely different platform.

The code is written to be readable. Read it.

---
