
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
