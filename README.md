
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
# ==================== ESP32-C3 16x16 CLOCK (UDP) ====================
# Sends a live digital clock (HH:MM) every second via UDP
# Works perfectly with the matrix code I gave you (serpentine/zigzag order)
# Zero extra libraries needed!

import socket
import time
from datetime import datetime

# ==================== CHANGE THIS ====================
ESP_IP = "192.168.1.XXX"   # ←←← PUT YOUR ESP32 IP HERE (from Serial Monitor)
UDP_PORT = 4210

# ==================== 3x5 FONT (digits 0-9) ====================
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

def draw_digit(pixels, char, x_off, y_off, color):
    if char not in FONT:
        return
    pattern = FONT[char]
    for dy in range(5):
        row = pattern[dy]
        for dx in range(3):
            if row & (1 << (2 - dx)):          # leftmost bit is highest
                pixels[y_off + dy][x_off + dx] = color

def draw_colon(pixels, x, y_off, color):
    pixels[y_off + 1][x] = color
    pixels[y_off + 3][x] = color

def create_clock_image():
    # 16x16 grid (y, x) - black background
    pixels = [[(0, 0, 0) for _ in range(16)] for _ in range(16)]

    now = datetime.now()
    hour_str = f"{now.hour:02d}"
    min_str = f"{now.minute:02d}"

    color = (0, 255, 220)   # nice bright cyan (feels premium on LEDs)

    y_off = 5               # centered vertically

    # Positions that perfectly fit 16x16:
    draw_digit(pixels, hour_str[0], 0,  y_off, color)   # HH tens
    draw_digit(pixels, hour_str[1], 4,  y_off, color)   # HH units
    draw_colon(pixels, 8, y_off, color)
    draw_digit(pixels, min_str[0], 10, y_off, color)    # MM tens
    draw_digit(pixels, min_str[1], 13, y_off, color)    # MM units (fits exactly 13-15)

    # Convert to UDP byte stream (exact serpentine order your matrix uses)
    data = bytearray()
    for y in range(16):
        if y % 2 == 0:
            xs = range(16)                  # even row: left → right
        else:
            xs = range(15, -1, -1)          # odd row: right → left
        for x in xs:
            r, g, b = pixels[y][x]
            data.extend([r, g, b])

    return data


# ==================== MAIN LOOP ====================
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

print("🚀 Clock sender started! (Ctrl+C to stop)")
print(f"Sending to {ESP_IP}:{UDP_PORT} every second")

try:
    while True:
        data = create_clock_image()
        s.sendto(data, (ESP_IP, UDP_PORT))
        
        current_time = datetime.now().strftime("%H:%M:%S")
        print(f"Sent → {current_time}  (switch to CUSTOM mode on ESP if needed)")
        
        time.sleep(1)          # update every second
except KeyboardInterrupt:
    print("\n🛑 Clock sender stopped.")
```
