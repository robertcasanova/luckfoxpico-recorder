# LuckFox AV Recorder Daemon — Design Document

Revision: 1.1\
Date: 2026-10-07\
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
ffmpeg -hide_banner -nostdin -n -rtsp_transport tcp -i rtsp://localhost:554/live/0 -map 0:v:0 -an -c:v copy -f h264 output.h264
```

Rationale:
- FFmpeg handles RTSP/RTP protocol complexity
- `-c:v copy` means no transcoding (minimal CPU/RAM)
- Raw H.264 output requires no container finalization
- Controlled via process spawn and SIGTERM for clean stop

Select only the first video stream and verify that it is H.264; stream copy cannot convert another codec to H.264. Never overwrite an existing output (`-n`). Launch with an argument vector, not a shell. In the multithreaded daemon, prefer `posix_spawnp()`; a fork-based child must use only async-signal-safe operations before exec. Reap the child exactly once and treat an unexpected exit as a video-track failure, not evidence that recording succeeded. Do not assume FFmpeg automatically reconnects to RTSP.

### 3.2 Audio Capture Strategy

Direct SPI via `/dev/spidevX.Y` with paced reads per the ESP32-H2 protocol:
- WAV header written at start with placeholder size
- Patched on close with actual data size
- Bounded 64KB PCM queue between SPI capture and a WAV writer thread to absorb short SD card stalls
- 64KB stdio write buffer (`setvbuf`) to batch writes; this alone does not decouple capture from blocking SD I/O

The PCM queue holds about 0.74 seconds at 88,200 bytes/second. If it fills, stop the session with an overrun error rather than silently dropping samples or growing memory. Stop and join the SPI producer before draining the queue, joining the writer, and patching the WAV header. The writer exclusively owns WAV access while running; after it is joined, the control loop may finalize/close the file.

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

The architecture sketch shows capture/control paths; the SPI thread feeds the bounded PCM queue and a separate WAV writer thread performs disk I/O.

Targeting <4MB total daemon footprint:

| Component | Size |
|-----------|------|
| SPI RX buffer | 4 KB (4 chunks) |
| PCM queue | 64 KB (bounded) |
| WAV write buffer | 64 KB (setvbuf) |
| UART RX buffer | 256 bytes |
| Thread stacks | Explicit bounded sizes, validated on target |
| FFmpeg process | Measure on target; not part of the daemon's <4MB target |

The 64MB system budget includes the daemon, FFmpeg, existing camera/RTSP services, kernel, and filesystem cache. FFmpeg's footprint is build-dependent; the previous 2-3MB estimate is not a guarantee. Validate peak RSS and available system memory during a one-hour recording. Default pthread stacks can exceed the daemon budget, so include stacks and all buffers in measurements.

---

## 4. SPI Audio Protocol Reference

Expected protocol parameters (the referenced `docs/spi-audio-stream.md` is not present in this repository; confirm against ESP32-H2 firmware before hardware acceptance):

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

Use one full-duplex `SPI_IOC_MESSAGE(1)` transfer per chunk, with 1024 dummy TX bytes, 1024 RX bytes, and CS asserted for the entire transaction. Configure and verify mode 0, 8 bits/word, MSB-first, and speed. The dummy-byte value and any startup/ready handshake must be confirmed with the slave firmware; do not invent command bytes or split a chunk into separate transactions.

Schedule from accumulated sample count (`512 * chunks / 44100` seconds), or carry fractional nanoseconds, to avoid cumulative rounding drift. Retry interrupted absolute sleeps against the same deadline. Count missed deadlines and do not issue uncontrolled bursts of catch-up transactions; sustained pacing loss stops the session. Whether the slave permits catch-up reads remains a firmware validation dependency. All-zero PCM is valid silence, so report zero-filled chunks separately, not as confirmed underflows without a firmware status signal.

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

Configure raw mode with software/hardware flow control and echo disabled. Parse a byte stream, not one command per `read()`: retain fragmented lines, accept multiple lines per read, strip only a terminal CR before LF, and accept exact uppercase commands. Ignore unknown/empty lines. On a line-buffer overflow, discard through the next LF before parsing again. Use a wakeable poll/read loop so shutdown does not depend on another UART byte arriving.

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

Serialize the 44-byte header explicitly in little-endian order, not by writing a native C struct. RIFF size is `36 + data_bytes`; data size is PCM bytes only. Count only successfully written complete samples, enforce 2-byte alignment, and check write, flush, seek, header patch, sync, and close errors. Finalization after disk failure is best-effort and must not be logged as successful if any step fails. Power loss/SIGKILL recovery is outside the initial scope; only graceful shutdown guarantees a patched header.

### 6.3 H.264 File Format

Raw Annex B H.264 bitstream (NAL units with start codes). No container overhead.

Verify captured files contain SPS/PPS and a decodable IDR frame; an arbitrary mid-GOP start may not be immediately playable. Audio and video start independently, and raw H.264 carries no container timestamps. This design does not guarantee sample-accurate A/V synchronization; measure startup skew and drift on target and do not infer synchronization from matching filenames.

### 6.4 Storage Safety

Use one wall-clock timestamp for both files in a session, but monotonic time for elapsed time and timeout. If either name already exists, choose a shared numeric suffix for both; never truncate an earlier recording. Reserve the WAV with exclusive creation and protect video with FFmpeg's no-overwrite option. Run only one recorder instance per hardware device.

Before creating the default output directory, verify `/sdcard` is actually mounted and writable; a missing card must not cause recordings to fill the root filesystem. Recheck before each session. Custom output directories may use other filesystems. Directory/open failures reject the session; runtime ENOSPC/EIO stops both tracks. Preserve nonempty partial recordings and log their finalization status.

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

Validate `--duration` as an integer in the range 1-3600 seconds; reject invalid options before initializing hardware.

---

## 8. State Machine

One control loop owns session transitions; UART callbacks and workers submit events rather than performing start/stop operations themselves. Never hold the state mutex across I/O, sleeps, child waits, or thread joins. Repeated ON while active and OFF while idle are no-ops; ON during STOPPING is ignored, not queued. Startup is serialized and must handle a pending stop/shutdown before completing.

Start only if at least one track can run. A missing or failed capture source disables that track and allows the other to continue; loss of the last track returns through STOPPING to IDLE. Shared storage errors, PCM queue overrun, or sustained pacing loss stop the whole session. Stop is idempotent, cancels capture, finalizes outputs once, and releases all per-session resources before another start. Use `CLOCK_MONOTONIC` for the maximum-duration deadline, starting when session startup begins so connection delays cannot extend the limit.

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
| RTSP stream unavailable / FFmpeg exits | Mark video failed, log exit status, continue audio if active; no implicit retry |
| SD card full | Stop recording, finalize files |
| SD card missing / output unwritable | Reject session; do not fall back to root filesystem |
| SPI read fails | Disable audio, finalize its WAV, continue video if active |
| All capture sources fail | Stop session, return to IDLE with failure logged |
| Audio queue full / sustained pacing loss | Stop session, log data-loss risk |
| UART disconnected | Log warning, continue (CLI still works) |
| SIGTERM/SIGINT | Stop recording gracefully, finalize files, exit |
| Max duration reached | Stop recording automatically |

Handle SIGTERM/SIGINT using `sigwait()` or an async-signal-safe notification to the control loop; do not call logging, mutex, join, or file-finalization APIs in a signal handler. Reset the child's signal mask/dispositions when spawning FFmpeg so it does not inherit blocked termination signals. Request both tracks to stop before waiting for either to finalize. Bound FFmpeg shutdown waits with a monotonic timeout, then SIGKILL and reap if needed. Bound RTSP connection/read waits using options supported by the deployed FFmpeg version. SPI ioctl and filesystem I/O may block in the kernel; validate driver/storage failure behavior on target rather than promising an unconditionally bounded shutdown.

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
