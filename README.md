
# Switch to Tetris
curl http://ESP-IP/api/mode?set=tetris

# Switch to Custom mode
curl http://ESP-IP/api/mode?set=custom

# Change brightness (0-255)
curl http://ESP-IP/api/brightness?val=50

# Check status
curl http://ESP-IP/api/status

curl http://ESP-IP/api/mode?set=gameoflife
curl http://ESP-IP/api/mode?set=tetris     # back to tetris


# Python code for send custom images
```python
import socket
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

data = bytearray()
for y in range(16):
    for x in range(16):
        idx = (y * 16 + (15 - x)) if (y % 2 == 1) else (y * 16 + x)   # zigzag matching your matrix
        # Example: red checkerboard
        r = 255 if (x + y) % 2 == 0 else 0
        g = 0
        b = 0 if (x + y) % 2 == 0 else 255
        data.extend([r, g, b])

s.sendto(data, ("YOUR_ESP_IP", 4210))
print("Image sent!")
```

# Python code for send clock to matrix
```python
# ==================== ESP32-C3 16x16 LIVE CLOCK (UDP) ====================
# • HH : MM  on top (3 +1 +3 +1 + : +1 +3 +1 +3)
# • SS       on bottom
# • One random bright color changes EVERY SECOND
# • Vertical: 2 empty + 5 font + 2 empty + 5 font + 2 empty
# • Sends every second

import socket
import time
from datetime import datetime
import random

# ==================== CHANGE THIS ====================
ESP_IP = "192.168.1.XXX"   # ←← YOUR ESP32 IP
UDP_PORT = 4210

# ==================== 3x5 FONT ====================
FONT = {
    '0': [0b111, 0b101, 0b101, 0b101, 0b111],
    '1': [0b010, 0b110, 0b010, 0b010, 0b111],
    '2': [0b111, 0b001, 0b111, 0b100, 0b111],
    '3': [0b111, 0b001, 0b111, 0b001, 0b111],
    '4': [0b101, 0b101, 0b111, 0b001, 0b001],
    '5': [0b111, 0b100, 0b111, 0b001, 0b111],
    '6': [0b111, 0b100, 0b111, 0b101, 0b111],
    '7': [0b111, 0b001, 0b001, 0b001, 0b001],
    '8': [0b111, 0b101, 0b111, 0b101, 0b111],
    '9': [0b111, 0b101, 0b111, 0b001, 0b111],
}

def draw_digit(grid, char, x_off, y_off, color):
    if char not in FONT:
        return
    pattern = FONT[char]
    for dy in range(5):
        row = pattern[dy]
        for dx in range(3):
            if row & (1 << (2 - dx)):
                if 0 <= y_off + dy < 16 and 0 <= x_off + dx < 16:
                    grid[y_off + dy][x_off + dx] = color

def draw_colon(grid, x, y_off, color):
    if 0 <= y_off + 1 < 16:
        grid[y_off + 1][x] = color
    if 0 <= y_off + 3 < 16:
        grid[y_off + 3][x] = color

def create_clock_image(color):
    grid = [[(0, 0, 0) for _ in range(16)] for _ in range(16)]

    now = datetime.now()
    h1, h2 = f"{now.hour:02d}"
    m1, m2 = f"{now.minute:02d}"
    s1, s2 = f"{now.second:02d}"

    # Top part: HH : MM   starting at row 2
    y_top = 2
    draw_digit(grid, h1, 0,  y_top, color)     # 0-2
    draw_digit(grid, h2, 4,  y_top, color)     # 4-6   (1 space gap)
    draw_colon(grid, 8, y_top, color)          # colon at x=8
    draw_digit(grid, m1,9, y_top, color)      # 10-12
    draw_digit(grid, m2,13, y_top, color)      # 14-16 (fits exactly)

    # Bottom part: SS     starting at row 2+5+2 = 9
    y_bottom = 9
    # Center SS roughly (total width 3+1+3 = 7 → start at x=4 or 5)
    draw_digit(grid, s1, 5,  y_bottom, color)  # 5-7
    draw_digit(grid, s2, 9,  y_bottom, color)  # 9-11  (nice spacing)

    # Mirror flip (horizontal) to correct zigzag matrix orientation
    mirrored = [[grid[y][15 - x] for x in range(16)] for y in range(16)]

    # Convert to UDP byte stream (serpentine/zigzag order)
    data = bytearray()
    for y in range(16):
        xs = range(16) if y % 2 == 0 else range(15, -1, -1)
        for x in xs:
            r, g, b = mirrored[y][x]
            data.extend([r, g, b])

    return data


# ==================== MAIN LOOP ====================
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

print("🚀 CLOCK HH:MM top / SS bottom - color changes EVERY SECOND")
print(f"Sending to {ESP_IP}:{UDP_PORT}")

try:
    while True:
        # New random bright color every second
        r = random.randint(120, 255)
        g = random.randint(120, 255)
        b = random.randint(120, 255)
        color = (r, g, b)

        data = create_clock_image(color)
        s.sendto(data, (ESP_IP, UDP_PORT))

        now_str = datetime.now().strftime("%H:%M:%S")
        print(f"Sent {now_str}  RGB={color}")

        time.sleep(1.0)
except KeyboardInterrupt:
    print("\n🛑 Stopped.")
```
