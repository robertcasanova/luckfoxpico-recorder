# LuckFox AV Recorder Daemon — Design Document

Revision: 1.0  
Date: 2026-10-06  
Status: Implementation Plan

---

## 1. Problem Statement

Create a memory-efficient C daemon for LuckFox Pico Mini (RV1103, 64MB RAM) that simultaneously records:

1. **Video** from an existing RTSP stream (`rtsp://localhost:554/live/0`) → raw `.h264` file
2. **Audio** from SPI (ESP32-H2 slave, 16-bit mono PCM @ 44.1kHz) → `.wav` file

Recording is triggered by "ON" message on UART4 (115200 baud) or CLI argument for testing. "OFF" stops recording. Max duration 1 hour with auto-stop. Files saved to `/sdcard/recordings/` with timestamps.

---

## 2. Requirements

| Requirement | Details |
|-------------|---------|
| UART trigger | Listen on `/dev/ttyS4` (UART4, 115200 baud, GPIO24/23 from ESP32-H2) |
| CLI trigger | `--record` argument starts recording immediately (for testing) |
| Video capture | Pull RTSP stream, dump raw H.264 NAL units to file (no re-encoding) |
| Audio capture | SPI master reading 1024-byte chunks @ 11.61ms cadence, write WAV |
| Output files | `/sdcard/recordings/YYYY-MM-DD_HH-MM-SS_video.h264` + `_audio.wav` |
| Memory | Minimize RAM usage — small buffers, streaming writes |
| Duration | Auto-stop at 1 hour, clean finalization of both files |

---

## 3. Technical Approach

### 3.1 Video Capture Strategy

Use FFmpeg as an external process:

```bash
ffmpeg -hide_banner -nostdin -rtsp_transport tcp -i rtsp://localhost:554/live/0 -c:v copy -f h264 output.h264
```

Rationale:
- FFmpeg handles RTSP/RTP protocol complexity
- `-c:v copy` means no transcoding (minimal CPU/RAM)
- Raw H.264 output requires no container finalization
- Controlled via fork/exec and SIGTERM for clean stop

### 3.2 Audio Capture Strategy

Direct SPI via `/dev/spidevX.Y` with paced reads per the ESP32-H2 protocol:
- WAV header written at start with placeholder size
- Patched on close with actual data size
- 64KB write buffer to absorb SD card stalls

### 3.3 Daemon Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Main Process                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  │
│  │ UART Thread │  │ SPI Thread  │  │ FFmpeg Process  │  │
│  │ (trigger)   │  │ (audio)     │  │ (video, child)  │  │
│  └──────┬──────┘  └──────┬──────┘  └────────┬────────┘  │
│         │                │                   │          │
│         └────────────────┴───────────────────┘          │
│                          │                              │
│                   State Machine                         │
│              IDLE ←→ RECORDING ←→ STOPPING              │
└─────────────────────────────────────────────────────────┘
```

### 3.4 Memory Budget

Targeting <4MB total daemon footprint:

| Component | Size |
|-----------|------|
| SPI RX buffer | 4 KB (4 chunks) |
| WAV write buffer | 64 KB (setvbuf) |
| UART RX buffer | 256 bytes |
| FFmpeg process | ~2-3 MB (separate address space) |

---

## 4. SPI Audio Protocol Reference

From `docs/spi-audio-stream.md`:

| Parameter | Value |
|-----------|-------|
| SPI Mode | 0 (CPOL=0, CPHA=0) |
| Bit order | MSB first |
| Bits per word | 8 |
| Clock speed | 10 MHz (8-16 MHz acceptable) |
| Transaction size | 1024 bytes (exactly) |
| Pacing interval | 11,609,977 ns per chunk |
| Audio format | Signed 16-bit PCM, little-endian, mono |
| Sample rate | 44100 Hz |
| Byte rate | 88,200 bytes/second |

**Critical**: Use monotonic clock with absolute deadline scheduling, not relative sleep.

---

## 5. UART Protocol

| Message | Action |
|---------|--------|
| `ON\n` or `ON\r\n` | Start recording |
| `OFF\n` or `OFF\r\n` | Stop recording |

- Device: `/dev/ttyS4`
- Baud: 115200
- Format: 8N1
- Source: ESP32-H2 BLE-UART bridge

---

## 6. File Output

### 6.1 Naming Convention

```
/sdcard/recordings/YYYY-MM-DD_HH-MM-SS_video.h264
/sdcard/recordings/YYYY-MM-DD_HH-MM-SS_audio.wav
```

Example:
```
/sdcard/recordings/2026-10-06_18-30-00_video.h264
/sdcard/recordings/2026-10-06_18-30-00_audio.wav
```

### 6.2 WAV File Format

| Field | Value |
|-------|-------|
| Format | PCM (1) |
| Channels | 1 (mono) |
| Sample rate | 44100 Hz |
| Bits per sample | 16 |
| Byte rate | 88200 |
| Block align | 2 |

Maximum file size for 1 hour: ~317 MB

### 6.3 H.264 File Format

Raw Annex B H.264 bitstream (NAL units with start codes). No container overhead.

---

## 7. Command Line Interface

```
luckfox-recorder [OPTIONS]

Options:
  -h, --help              Show help message
  -r, --record            Start recording immediately (test mode)
  -d, --duration SECS     Max recording duration (default: 3600)
  -D, --daemon            Run as daemon (background)
  -s, --spi-dev PATH      SPI device (default: /dev/spidev0.0)
  -u, --uart-dev PATH     UART device (default: /dev/ttyS4)
  -R, --rtsp-url URL      RTSP stream URL (default: rtsp://localhost:554/live/0)
  -o, --output-dir PATH   Output directory (default: /sdcard/recordings)
```

---

## 8. State Machine

```
         ┌──────────────────────────────────────┐
         │                                      │
         ▼                                      │
      ┌──────┐    ON / --record    ┌───────────┴──┐
      │ IDLE │ ──────────────────► │  RECORDING   │
      └──────┘                     └──────┬───────┘
         ▲                                │
         │                                │ OFF / timeout / error
         │                                ▼
         │                         ┌──────────────┐
         └─────────────────────────┤   STOPPING   │
               finalize complete   └──────────────┘
```

---

## 9. Error Handling

| Condition | Response |
|-----------|----------|
| SPI device not found | Log error, continue without audio |
| FFmpeg not installed | Log error, continue without video |
| RTSP stream unavailable | FFmpeg will retry/fail, log and continue |
| SD card full | Stop recording, finalize files |
| UART disconnected | Log warning, continue (CLI still works) |
| SIGTERM/SIGINT | Stop recording gracefully, finalize files, exit |
| Max duration reached | Stop recording automatically |

---

## 10. Dependencies

- **FFmpeg**: Must be installed on LuckFox (`ffmpeg` in PATH)
- **Linux kernel**: spidev and tty drivers enabled
- **libc**: pthread, standard POSIX APIs

---

## 11. Project Structure

```
luckfox-recorder/
├── docs/
│   ├── DESIGN.md           # This document
│   └── TASKS.md            # Implementation tasks
├── include/
│   ├── config.h            # Configuration constants
│   ├── recorder.h          # State machine and main types
│   ├── uart_trigger.h      # UART listener API
│   ├── spi_audio.h         # SPI audio capture API
│   ├── wav_writer.h        # WAV file writer API
│   └── video_capture.h     # FFmpeg control API
├── src/
│   ├── main.c              # Entry point, CLI, signal handling
│   ├── recorder.c          # State machine implementation
│   ├── uart_trigger.c      # UART listener thread
│   ├── spi_audio.c         # SPI audio capture thread
│   ├── wav_writer.c        # WAV file writer
│   └── video_capture.c     # FFmpeg process control
├── Makefile                # Build system
└── README.md               # Quick start guide
```
