# Dual Servo Control

## Teensy Firmware

The Teensy listens for UDP packets on port 8888 and drives two servos based on the received pulse-width values. On boot both servos are set to 500 µs. When a packet arrives, it checks the prefix (`PULSE1:` or `PULSE2:`) to decide which servo to update, then clamps the value to the safe 500–2500 µs range before writing it.

```cpp
#include <NativeEthernet.h>
#include <NativeEthernetUdp.h>
#include <Servo.h>

byte mac[] = {0xDE,0xAD,0xBE,0xEF,0xFE,0xED};
IPAddress ip(192,168,2,1);
unsigned int localPort = 8888;

EthernetUDP Udp;

Servo servo1;
Servo servo2;

const int SERVO1_PIN = 10;
const int SERVO2_PIN = 11;

const int US_MIN = 500;
const int US_MAX = 2500;

int currentUs1 = 500;
int currentUs2 = 500;

int targetUs1 = 500;
int targetUs2 = 500;

void setup() {
  Serial.begin(115200);
  delay(1000);

  servo1.attach(SERVO1_PIN, US_MIN, US_MAX);
  servo2.attach(SERVO2_PIN, US_MIN, US_MAX);

  servo1.writeMicroseconds(currentUs1);
  servo2.writeMicroseconds(currentUs2);

  Ethernet.begin(mac, ip);
  delay(500);

  Udp.begin(localPort);

  Serial.println("Teensy Ready");
  Serial.print("IP: ");
  Serial.println(Ethernet.localIP());
}

void loop() {
  int packetSize = Udp.parsePacket();

  if (packetSize) {
    char buf[32];
    int len = Udp.read(buf, sizeof(buf)-1);
    buf[len] = 0;

    Serial.print("Received: ");
    Serial.println(buf);

    if (strncmp(buf, "PULSE1:", 7) == 0) {
      targetUs1 = constrain(atoi(buf + 7), US_MIN, US_MAX);
      servo1.writeMicroseconds(targetUs1);
    }

    if (strncmp(buf, "PULSE2:", 7) == 0) {
      targetUs2 = constrain(atoi(buf + 7), US_MIN, US_MAX);
      servo2.writeMicroseconds(targetUs2);
    }
  }
}
```

### How it works

- **`servo.attach(pin, US_MIN, US_MAX)`** — tells the Servo library the valid pulse range for this servo. Prevents accidental out-of-range commands from reaching the hardware.
- **`Udp.parsePacket()`** — non-blocking check. Returns the size of any waiting packet, or 0 if nothing has arrived. The loop runs freely without blocking.
- **`strncmp(buf, "PULSE1:", 7)`** — compares only the first 7 characters of the received string to identify which servo the command targets.
- **`atoi(buf + 7)`** — parses the number that starts 7 characters into the string, skipping the `PULSE1:` prefix.
- **`constrain(value, US_MIN, US_MAX)`** — clamps the parsed value to 500–2500 µs so a bad packet can never send a servo out of its safe range.
- **`writeMicroseconds()`** — directly sets the PWM pulse width in microseconds, giving finer resolution than `write()` which works in degrees.

---

## Python Keyboard Controller

The Python script opens a UDP socket and a small pygame window to capture keyboard input. Every 30 ms it reads which keys are held down, adjusts each servo's target value by a big step (40 µs) or fine step (5 µs), clamps the result, and sends two UDP packets — one for each servo.

```python
import socket
import pygame
import time

TEENSY_IP = "192.168.2.1"
TEENSY_PORT = 8888

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

PULSE_MIN = 500
PULSE_MAX = 2500

BIG_STEP = 40
SMALL_STEP = 5

servo1 = 500
servo2 = 500

def clamp(x, lo, hi):
    return lo if x < lo else hi if x > hi else x

def send_servo1(us):
    msg = f"PULSE1:{us}".encode()
    sock.sendto(msg, (TEENSY_IP, TEENSY_PORT))

def send_servo2(us):
    msg = f"PULSE2:{us}".encode()
    sock.sendto(msg, (TEENSY_IP, TEENSY_PORT))

def main():
    global servo1, servo2

    pygame.init()
    screen = pygame.display.set_mode((300, 200))
    pygame.display.set_caption("Servo Keyboard Control")

    print("Servo1: W/S big , A/D small")
    print("Servo2: UP/DOWN big , LEFT/RIGHT small")
    print("ESC to quit")

    running = True

    while running:
        pygame.event.pump()
        keys = pygame.key.get_pressed()

        if keys[pygame.K_ESCAPE]:
            running = False

        # Servo 1 big
        if keys[pygame.K_w]:
            servo1 += BIG_STEP
        if keys[pygame.K_s]:
            servo1 -= BIG_STEP

        # Servo 1 fine
        if keys[pygame.K_d]:
            servo1 += SMALL_STEP
        if keys[pygame.K_a]:
            servo1 -= SMALL_STEP

        # Servo 2 big
        if keys[pygame.K_UP]:
            servo2 += BIG_STEP
        if keys[pygame.K_DOWN]:
            servo2 -= BIG_STEP

        # Servo 2 fine
        if keys[pygame.K_RIGHT]:
            servo2 += SMALL_STEP
        if keys[pygame.K_LEFT]:
            servo2 -= SMALL_STEP

        servo1 = clamp(servo1, PULSE_MIN, PULSE_MAX)
        servo2 = clamp(servo2, PULSE_MIN, PULSE_MAX)

        send_servo1(servo1)
        send_servo2(servo2)

        print(f"S1:{servo1}   S2:{servo2}")

        time.sleep(0.03)

    pygame.quit()

if __name__ == "__main__":
    main()
```

### How it works

- **`socket.SOCK_DGRAM`** — creates a UDP socket. UDP is used instead of TCP because it is fire-and-forget with no connection overhead, which suits real-time servo control where low latency matters more than guaranteed delivery.
- **`pygame.event.pump()`** — must be called every loop to process the pygame internal event queue. Without it, `get_pressed()` returns stale data and the OS may consider the window unresponsive.
- **`get_pressed()`** — returns a snapshot of every key's current state. Holding a key down triggers the step every iteration (every 30 ms), which gives the feeling of continuous movement.
- **`BIG_STEP = 40` / `SMALL_STEP = 5`** — two step sizes give both quick range traversal (W/S, ↑/↓) and precise fine-tuning (A/D, ←/→) without needing an analog stick.
- **`clamp()`** — applied after every key read so the servo value can never exceed 500–2500 µs regardless of how long a key is held.
- **`time.sleep(0.03)`** — sets the update rate to ~33 Hz. Lower values send more packets and feel more responsive; higher values reduce CPU and network load. 30 ms is a practical balance for keyboard control.
- **`f"PULSE1:{us}".encode()`** — formats the command as a plain ASCII string matching the prefix the Teensy expects, then encodes it to bytes for the UDP socket.

### Key bindings

| Key | Action |
|---|---|
| `W` | Servo 1 +40 µs |
| `S` | Servo 1 −40 µs |
| `D` | Servo 1 +5 µs |
| `A` | Servo 1 −5 µs |
| `↑` | Servo 2 +40 µs |
| `↓` | Servo 2 −40 µs |
| `→` | Servo 2 +5 µs |
| `←` | Servo 2 −5 µs |
| `ESC` | Quit |
