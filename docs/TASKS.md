# LuckFox AV Recorder — Implementation Tasks

This document tracks the implementation tasks for the LuckFox AV Recorder daemon.

---

## Task 1: Project Skeleton and Build System

**Status**: Pending

### Deliverables
- [ ] Create directory structure: `src/`, `include/`, `docs/`
- [ ] Create `Makefile` for cross-compilation with LuckFox toolchain
- [ ] Create `main.c` with argument parsing (getopt_long)
- [ ] Implement signal handlers (SIGINT, SIGTERM)
- [ ] Create `config.h` with default constants

### CLI Options to Implement
```
-h, --help              Show help message
-r, --record            Start recording immediately (test mode)
-d, --duration SECS     Max recording duration (default: 3600)
-D, --daemon            Run as daemon (background)
-s, --spi-dev PATH      SPI device (default: /dev/spidev0.0)
-u, --uart-dev PATH     UART device (default: /dev/ttyS4)
-R, --rtsp-url URL      RTSP stream URL (default: rtsp://localhost:554/live/0)
-o, --output-dir PATH   Output directory (default: /sdcard/recordings)
```

### Acceptance Criteria
- Cross-compile succeeds (or native compile for structure verification)
- Program starts, prints config, exits cleanly on SIGTERM

---

## Task 2: UART Listener for Trigger Commands

**Status**: Pending

### Deliverables
- [ ] Create `uart_trigger.h` with API declarations
- [ ] Create `uart_trigger.c` with implementation
- [ ] Implement UART initialization (`/dev/ttyS4`, 115200, 8N1, non-canonical)
- [ ] Create UART reader thread
- [ ] Parse "ON" and "OFF" messages
- [ ] Update shared state (pthread mutex-protected)
- [ ] Handle UART errors gracefully

### API Design
```c
typedef void (*uart_callback_t)(const char *command, void *user_data);

int uart_trigger_init(const char *device, uart_callback_t callback, void *user_data);
void uart_trigger_start(void);
void uart_trigger_stop(void);
void uart_trigger_cleanup(void);
```

### Acceptance Criteria
- Daemon logs "Recording triggered by UART: ON"
- Daemon logs "Recording stopped by UART: OFF"
- Thread exits cleanly on shutdown

---

## Task 3: WAV File Writer Module

**Status**: Pending

### Deliverables
- [ ] Create `wav_writer.h` with API declarations
- [ ] Create `wav_writer.c` with implementation
- [ ] Write 44-byte WAV header with placeholder sizes
- [ ] Implement streaming write
- [ ] Patch RIFF and data chunk sizes on close
- [ ] Use `setvbuf()` for 64KB write buffer

### API Design
```c
typedef struct wav_writer wav_writer_t;

wav_writer_t *wav_open(const char *path, int sample_rate, int channels, int bits_per_sample);
int wav_write(wav_writer_t *wav, const void *data, size_t size);
int wav_close(wav_writer_t *wav);
uint32_t wav_get_bytes_written(wav_writer_t *wav);
```

### WAV Header Structure (44 bytes)
```
Offset  Size  Field
0       4     "RIFF"
4       4     file size - 8
8       4     "WAVE"
12      4     "fmt "
16      4     16 (fmt chunk size)
20      2     1 (PCM format)
22      2     1 (channels)
24      4     44100 (sample rate)
28      4     88200 (byte rate)
32      2     2 (block align)
34      2     16 (bits per sample)
36      4     "data"
40      4     data size (patched on close)
44      ...   PCM data
```

### Acceptance Criteria
- Output file passes `file` and `ffprobe` validation
- WAV plays back correctly with proper duration

---

## Task 4: SPI Audio Capture Thread

**Status**: Pending

### Deliverables
- [ ] Create `spi_audio.h` with API declarations
- [ ] Create `spi_audio.c` with implementation
- [ ] Implement SPI initialization (`/dev/spidev0.0`, mode 0, 10MHz)
- [ ] Create SPI reader thread with paced reads
- [ ] Use monotonic clock with absolute deadline scheduling
- [ ] Integrate with WAV writer
- [ ] Track underflow (all-zero chunks) for diagnostics

### API Design
```c
typedef void (*spi_data_callback_t)(const uint8_t *data, size_t size, void *user_data);

int spi_audio_init(const char *device);
void spi_audio_set_callback(spi_data_callback_t callback, void *user_data);
void spi_audio_start(void);
void spi_audio_stop(void);
void spi_audio_cleanup(void);
uint32_t spi_audio_get_underflow_count(void);
```

### Pacing Implementation
```c
// Period: 512 samples / 44100 Hz = 11,609,977 ns
#define CHUNK_BYTES     1024
#define PERIOD_NS       11609977LL

struct timespec next_deadline;
clock_gettime(CLOCK_MONOTONIC, &next_deadline);

while (running) {
    spi_read(fd, buffer, CHUNK_BYTES);
    callback(buffer, CHUNK_BYTES, user_data);
    
    add_ns(&next_deadline, PERIOD_NS);
    sleep_until(&next_deadline);
}
```

### Acceptance Criteria
- Audio capture with ESP32-H2 produces correct timing
- Recorded WAV duration matches wall-clock time
- Underflow count stays near zero with proper pacing

---

## Task 5: FFmpeg Video Capture Control

**Status**: Pending

### Deliverables
- [ ] Create `video_capture.h` with API declarations
- [ ] Create `video_capture.c` with implementation
- [ ] Implement `video_start()` with fork/exec
- [ ] Track child PID
- [ ] Implement `video_stop()` with SIGTERM, timeout, SIGKILL fallback
- [ ] Implement `video_is_running()` status check

### API Design
```c
int video_start(const char *rtsp_url, const char *output_path);
int video_stop(void);
int video_is_running(void);
pid_t video_get_pid(void);
```

### FFmpeg Command
```bash
ffmpeg -hide_banner -nostdin -loglevel warning \
       -rtsp_transport tcp -i rtsp://localhost:554/live/0 \
       -c:v copy -f h264 /sdcard/recordings/2026-10-06_18-30-00_video.h264
```

### Acceptance Criteria
- FFmpeg starts and receives RTSP stream
- `.h264` file is playable with `ffplay`
- Clean shutdown with SIGTERM

---

## Task 6: Recording Session State Machine

**Status**: Pending

### Deliverables
- [ ] Create `recorder.h` with state machine types
- [ ] Create `recorder.c` with implementation
- [ ] Implement state machine: IDLE → RECORDING → STOPPING → IDLE
- [ ] Implement `recorder_start()` — generate filenames, start video, open WAV
- [ ] Implement `recorder_stop()` — stop video, close WAV, log summary
- [ ] Handle max duration timer (1 hour)
- [ ] Wire UART callbacks and CLI `--record` flag

### API Design
```c
typedef enum {
    RECORDER_STATE_IDLE,
    RECORDER_STATE_RECORDING,
    RECORDER_STATE_STOPPING
} recorder_state_t;

int recorder_init(const char *output_dir, const char *rtsp_url, int max_duration_sec);
int recorder_start(void);
int recorder_stop(void);
recorder_state_t recorder_get_state(void);
void recorder_cleanup(void);
```

### State Transitions
| Current State | Event | Next State | Actions |
|---------------|-------|------------|---------|
| IDLE | ON / --record | RECORDING | Generate filenames, start video, open WAV, enable SPI |
| RECORDING | OFF | STOPPING | Stop video, stop SPI |
| RECORDING | timeout | STOPPING | Stop video, stop SPI |
| RECORDING | error | STOPPING | Log error, stop video, stop SPI |
| STOPPING | finalize done | IDLE | Close WAV, log summary |

### Acceptance Criteria
- Full start/stop cycle via UART and CLI
- Both files created with correct timestamps
- Max duration auto-stop works

---

## Task 7: Integration, Error Handling, and Polish

**Status**: Pending

### Deliverables
- [ ] Add daemonization option (`--daemon`)
- [ ] Implement graceful shutdown on SIGTERM/SIGINT
- [ ] Handle edge cases (SPI not found, FFmpeg missing, RTSP unavailable)
- [ ] Add periodic status logging (every 30s during recording)
- [ ] Create output directory if missing
- [ ] Create `README.md` with quick start guide

### Daemonization Steps
1. Fork, parent exits
2. `setsid()` to create new session
3. Fork again, parent exits
4. `chdir("/")` or working directory
5. Close stdin/stdout/stderr
6. Open `/dev/null` for fd 0, 1, 2
7. Redirect logs to syslog

### Status Log Format
```
[recorder] Recording: 00:05:30, audio: 29.1 MB, video: running, underflows: 0
```

### Acceptance Criteria
- Daemon runs in background
- UART triggers recording successfully
- Files saved to `/sdcard/recordings/` with timestamps
- Clean shutdown on signal finalizes files
- Graceful handling of missing hardware

---

## Summary

| Task | Description | Dependencies |
|------|-------------|--------------|
| 1 | Project skeleton and build system | None |
| 2 | UART listener | Task 1 |
| 3 | WAV file writer | Task 1 |
| 4 | SPI audio capture | Task 1, Task 3 |
| 5 | FFmpeg video capture | Task 1 |
| 6 | Recording state machine | Task 2, Task 4, Task 5 |
| 7 | Integration and polish | Task 6 |
