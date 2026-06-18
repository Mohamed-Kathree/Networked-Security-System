# 🔒 Networked Security System

> A Raspberry Pi–based emergency alert system that broadcasts real-time UDP alerts to all connected clients when a physical button is pressed, triggers a PWM siren, and logs every event to a persistent file.

![Python](https://img.shields.io/badge/Python-3.x-3776AB?style=flat-square&logo=python&logoColor=white)
![Raspberry Pi](https://img.shields.io/badge/Hardware-Raspberry%20Pi-C51A4A?style=flat-square&logo=raspberrypi&logoColor=white)
![Protocol](https://img.shields.io/badge/Protocol-UDP-1B3A5C?style=flat-square)
![Status](https://img.shields.io/badge/Status-Complete-brightgreen?style=flat-square)

---

## Overview

This system was built as a networked security solution using a Raspberry Pi as the central alarm server. A physical emergency button is wired directly to the Pi's GPIO pins. When pressed, the server simultaneously:

- Broadcasts a timestamped alert to every connected client over UDP
- Activates a PWM buzzer in a rising-and-falling siren pattern
- Appends the event to a persistent log file (`Alerts.txt`)

Any number of client machines on the same network can register with the server and will receive the alert in real time.

---

## System Architecture

```
┌─────────────────────────────────┐
│        Raspberry Pi (Server)    │
│                                 │
│  GPIO 23 ── [Emergency Button]  │
│  GPIO 18 ── [PWM Buzzer]        │
│                                 │
│  UDP Socket  0.0.0.0:12345      │
└────────────┬────────────────────┘
             │  UDP broadcast on button press
     ┌───────┴────────┐
     │                │
┌────▼────┐      ┌────▼────┐
│ Client 1│      │ Client 2│  ...
│ (any IP)│      │ (any IP)│
└─────────┘      └─────────┘
```

---

## How It Works

### Server (`Server.py`) — runs on the Raspberry Pi

**Registration:** When a client sends any UDP message to the server on port 12345, the server registers that client's address and responds with a confirmation. The client is then added to the broadcast list for all future alerts.

**Emergency trigger:** A physical button on GPIO pin 23 is monitored via `gpiozero`. When pressed, the `Emergency()` function fires immediately via an interrupt callback (`button.when_pressed`) — it does not depend on the main loop polling, so response time is near-instant.

**Alert broadcast:** The server sends a timestamped alert message to every registered client simultaneously over UDP.

**Siren:** The buzzer on GPIO pin 18 sweeps its PWM frequency from 500 Hz up to 2000 Hz and back down, repeated three times — producing an audible rising-and-falling siren effect.

**Logging:** Every alert is appended to `Alerts.txt` with a full timestamp (`YYYY-MM-DD HH:MM:SS`).

**Non-blocking loop:** The main loop uses a 0.1 s socket timeout so the server can continue accepting new client registrations without blocking the button interrupt.

### Client (`Client.py`) — runs on any networked machine

On launch, the client sends a registration message to the server. It then enters a blocking receive loop, printing any incoming alert to the console as soon as it arrives. Any number of clients can connect simultaneously.

---

## Technical Details

### Network
| Property | Value |
|---|---|
| Protocol | UDP (SOCK_DGRAM) |
| Server bind address | 0.0.0.0 (all interfaces) |
| Port | 12345 |
| Max message size | 2048 bytes |
| Client discovery | Self-registration on first message |

### GPIO (Raspberry Pi)
| Component | GPIO Pin | Configuration |
|---|---|---|
| Emergency button | GPIO 23 | Pull-up resistor, 300 ms debounce |
| PWM buzzer | GPIO 18 | PWM output, 1000 Hz initial frequency |

### Siren pattern
```
Repeat × 3:
  Sweep frequency 500 Hz → 2000 Hz (step +50 Hz, 10 ms per step)
  Sweep frequency 2000 Hz → 500 Hz (step −50 Hz, 10 ms per step)
```
Total siren duration ≈ 3 × (30 + 30) steps × 10 ms = ~1.8 seconds.

### Alert log format (`Alerts.txt`)
```
Emergency detected at [2023-10-14 09:32:17]
Emergency detected at [2023-10-14 09:45:03]
```

---

## Installation & Setup

### Hardware required
- Raspberry Pi (any model with GPIO)
- Momentary push button
- Active or passive buzzer (PWM-capable)
- Jumper wires

### Wiring
```
Button:  one leg → GPIO 23,  other leg → GND
Buzzer:  positive → GPIO 18, negative  → GND
```

### Software — Raspberry Pi (server)

```bash
# Install dependencies
pip install gpiozero

# Run the server
python Server.py
```

### Software — Client machines

Update the `HOST` variable in `Client.py` to the Raspberry Pi's IP address on your network:

```python
HOST = "172.21.12.32"   # ← replace with your Pi's actual IP
```

Then run:

```bash
python Client.py
```

> The client will register automatically on launch and begin listening for alerts.

---

## File Structure

```
Networked-Security-System/
│
├── Server.py       # Raspberry Pi server — GPIO, UDP socket, alert broadcast
├── Client.py       # Client — registers with server, listens for alerts
└── Alerts.txt      # Auto-generated alert log (created on first trigger)
```

---

## Design Decisions

**UDP over TCP** — UDP is used deliberately. There is no handshake overhead, so alerts are dispatched to all clients with minimal latency the moment the button is pressed. For an emergency alert system, speed of delivery matters more than guaranteed delivery. TCP's connection overhead would introduce unnecessary delay.

**Interrupt-driven button handling** — The button uses `gpiozero`'s `when_pressed` callback rather than polling in the main loop. This means the emergency response fires immediately on the hardware interrupt, independent of the socket loop's 0.1 s timeout cycle.

**Self-registering clients** — Clients register simply by sending any message to the server. The server adds them to the broadcast set automatically, so no pre-configuration or client list management is needed — any machine on the network can join by running the client script.

**Debounce** — The button is configured with a 300 ms debounce (`bounce_time=0.3`) to prevent multiple rapid triggers from a single physical press.

---

## Concepts Demonstrated

- UDP socket programming (`socket.SOCK_DGRAM`)
- Multi-client broadcast over a local network
- Raspberry Pi GPIO — digital input (button) and PWM output (buzzer)
- Hardware interrupt handling via `gpiozero` callbacks
- Non-blocking I/O with socket timeouts
- Persistent event logging
- Embedded systems integration with networked software

---

*Built as part of a software engineering team project (2023) — Team Captain role.*
