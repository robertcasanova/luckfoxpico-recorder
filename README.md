# LuckFox AV Recorder

A memory-efficient daemon for LuckFox Pico Mini (RV1103) that simultaneously records video from RTSP and audio from SPI.

## Features

- **Video**: Captures RTSP stream (H.264) from the onboard camera
- **Audio**: Captures PCM audio from ESP32-H2 via SPI
- **Triggers**: UART command ("ON"/"OFF") or CLI argument
- **Output**: Timestamped `.h264` and `.wav` files
- **Memory**: Optimized for 64MB RAM systems

## Requirements

- LuckFox Pico Mini (RV1103) with SD card
- FFmpeg installed (`ffmpeg` in PATH)
- ESP32-H2 with SPI audio streaming firmware
- RTSP stream available (e.g., from onboard camera)

## Building

### Cross-compilation (recommended)

```bash
# Set up LuckFox SDK environment
source /path/to/luckfox-pico/tools/linux/toolchain/arm-rockchip830-linux-uclibcgnueabihf/env.sh

# Build
make
```

### Native compilation (for testing structure)

```bash
make CC=gcc
```

## Usage

### Basic usage (UART trigger mode)

```bash
# Run in foreground, wait for UART commands
./luckfox-recorder

# Run as daemon
./luckfox-recorder --daemon
```

### Test mode (immediate recording)

```bash
# Record for 60 seconds
./luckfox-recorder --record --duration 60
```

### Custom configuration

```bash
./luckfox-recorder \
    --spi-dev /dev/spidev0.0 \
    --uart-dev /dev/ttyS4 \
    --rtsp-url "rtsp://localhost:554/live/0" \
    --output-dir /sdcard/recordings \
    --duration 3600
```

## Command Line Options

| Option | Description | Default |
|--------|-------------|---------|
| `-h, --help` | Show help message | - |
| `-r, --record` | Start recording immediately | Off |
| `-d, --duration SECS` | Max recording duration | 3600 (1 hour) |
| `-D, --daemon` | Run as background daemon | Off |
| `-s, --spi-dev PATH` | SPI device path | `/dev/spidev0.0` |
| `-u, --uart-dev PATH` | UART device path | `/dev/ttyS4` |
| `-R, --rtsp-url URL` | RTSP stream URL | `rtsp://localhost:554/live/0` |
| `-o, --output-dir PATH` | Output directory | `/sdcard/recordings` |

## UART Commands

Send these commands to `/dev/ttyS4` at 115200 baud (8N1):

| Command | Action |
|---------|--------|
| `ON` | Start recording |
| `OFF` | Stop recording |

## Output Files

Files are saved with timestamps:

```
/sdcard/recordings/2026-10-06_18-30-00_video.h264
/sdcard/recordings/2026-10-06_18-30-00_audio.wav
```

## Hardware Wiring

### SPI (ESP32-H2 → LuckFox)

| ESP32-H2 | LuckFox | Signal |
|----------|---------|--------|
| GPIO22 | SCLK | SPI Clock |
| GPIO11 | MISO | Audio Data |
| GPIO12 | CS | Chip Select |
| GND | GND | Ground |

### UART (ESP32-H2 → LuckFox)

| ESP32-H2 | LuckFox | Signal |
|----------|---------|--------|
| GPIO24 | UART4_RX | TX → RX |
| GPIO23 | UART4_TX | RX ← TX |
| GND | GND | Ground |

## Troubleshooting

### No audio recorded
- Check SPI wiring (MISO, SCLK, CS, GND)
- Verify ESP32-H2 firmware is running
- Check `/dev/spidev0.0` exists

### No video recorded
- Verify RTSP stream is accessible: `ffplay rtsp://localhost:554/live/0`
- Check FFmpeg is installed: `ffmpeg -version`

### UART commands not working
- Verify `/dev/ttyS4` exists
- Check wiring (TX↔RX crossover)
- Test with: `echo "ON" > /dev/ttyS4`

## Documentation

- [DESIGN.md](docs/DESIGN.md) — Architecture and technical details
- [TASKS.md](docs/TASKS.md) — Implementation task breakdown

## License

[Add license here]
