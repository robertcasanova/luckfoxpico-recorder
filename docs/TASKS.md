# LuckFox AV Recorder — Implementation Tasks

This document tracks the implementation tasks for the LuckFox AV Recorder daemon.

Implementation details and acceptance checks below are part of each task's completion criteria. See `DESIGN.md` for cross-module failure policy and hardware assumptions; all tasks remain pending until implemented and verified.

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
- Reject unknown options and malformed/out-of-range duration values (valid range: 1-3600 seconds) before starting threads or opening outputs
- Build with warnings enabled and pthread support; native compilation does not substitute for target ABI/toolchain validation

### Critical Implementation Details
- Process signals in the control loop via `sigwait()` or an async-signal-safe wakeup; signal handlers must not log, take mutexes, or finalize files
- Use monotonic deadlines for duration and shutdown timeouts; wall-clock changes must not affect recording length
- Make initialization failures unwind only successfully initialized resources; failed thread creation must not leave a partially running daemon

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
- Fragmented commands, multiple commands in one read, CRLF, unknown lines, and oversized lines are tested without hardware using a pseudo-terminal

### Critical Implementation Details
- Configure raw 115200 8N1: disable canonical processing, echo, CR/LF translation, and software/hardware flow control
- Buffer until LF, strip an optional terminal CR, and match exact `ON`/`OFF`; discard an oversized line through its next LF to prevent false commands
- Callbacks submit control events; they must not block on recording teardown or update session resources directly
- Use a wakeable poll loop for shutdown; handle `EINTR`, `EAGAIN`, EOF, and disconnect without a busy loop

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
- Header byte offsets and sizes are verified for zero-length, known PCM payload, and interrupted recordings; write/finalization failures are injected

### Critical Implementation Details
- Encode header integers explicitly as little-endian bytes; do not serialize a potentially padded native struct
- RIFF size is `36 + data_bytes`, data size excludes the 44-byte header, and PCM writes must be aligned to 2-byte samples
- Handle short writes and count only complete samples successfully written; reject RIFF size overflow even though the one-hour limit is safely below it
- Check `fflush`, seek/header rewrite, `fsync`, and close results; report best-effort finalization failures rather than success after ENOSPC/EIO
- Keep the `setvbuf` buffer alive until close and close only after all producers have stopped and the writer has drained its queue

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
- [ ] Track zero-filled chunks, missed deadlines, and queue overruns separately; silence is not proof of underflow
- [ ] Decouple SPI reads from SD writes with a bounded 64KB PCM queue and a WAV writer thread

### API Design
```c
typedef void (*spi_data_callback_t)(const uint8_t *data, size_t size, void *user_data);

int spi_audio_init(const char *device);
void spi_audio_set_callback(spi_data_callback_t callback, void *user_data);
void spi_audio_start(void);
void spi_audio_stop(void);
void spi_audio_cleanup(void);
uint32_t spi_audio_get_zero_chunk_count(void);
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
    callback(buffer, CHUNK_BYTES, user_data); // Copy to bounded queue, not disk I/O
    
    add_ns(&next_deadline, PERIOD_NS);
    sleep_until(&next_deadline);
}
```

This sketch omits error handling. Production deadlines must derive from accumulated sample count or carry the fractional nanoseconds rather than repeatedly adding the rounded period. Normalize `timespec` values and retry `clock_nanosleep(CLOCK_MONOTONIC, TIMER_ABSTIME, ...)` on `EINTR` against the same deadline. Do not run a burst of SPI reads merely because multiple deadlines are already past; count lateness and stop on sustained pacing loss.

### Critical Implementation Details
- Configure and read back mode 0, MSB-first, 8 bits/word, and 10MHz; issue one `SPI_IOC_MESSAGE(1)` with exactly 1024 RX bytes and dummy TX bytes, keeping CS asserted throughout
- The referenced `spi-audio-stream.md` is missing: confirm dummy TX value, startup/ready handshake, FIFO behavior, and whether catch-up reads are allowed with ESP32-H2 firmware before hardware acceptance
- A callback receives a reusable RX buffer: copy data into the queue before returning, never retain the pointer
- The 64KB queue covers about 0.74 seconds of SD stalls; stop the session on overflow without silently dropping PCM or allocating unbounded buffers
- Stop/join the SPI producer, drain/join the WAV writer, then close the WAV; count bytes only for successful full transfers and writes

### Acceptance Criteria
- Audio capture with ESP32-H2 produces correct timing
- Recorded WAV duration matches monotonic elapsed capture time within a documented, measured tolerance during a one-hour run
- Zero-filled valid silence is preserved and not reported as confirmed underflow
- Inject SD stalls shorter and longer than queue capacity; verify ordered PCM for short stalls and explicit session failure on overflow
- Measure deadline misses under simultaneous RTSP capture and SD writes on target hardware

---

## Task 5: FFmpeg Video Capture Control

**Status**: Pending

### Deliverables
- [ ] Create `video_capture.h` with API declarations
- [ ] Create `video_capture.c` with implementation
- [ ] Implement `video_start()` with argv-based process spawn
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
       -n -rtsp_transport tcp -i rtsp://localhost:554/live/0 \
       -map 0:v:0 -an \
       -c:v copy -f h264 /sdcard/recordings/2026-10-06_18-30-00_video.h264
```

### Critical Implementation Details
- Use an argv-based launch, never shell interpolation of paths or URLs. Prefer `posix_spawnp()` in the threaded daemon; fork children must perform only async-signal-safe operations until exec
- Use close-on-exec for UART/SPI/WAV/control descriptors and route FFmpeg stderr to a monitored log sink without allowing an undrained pipe to block the child
- Reset the child's signal mask/dispositions so daemon signal handling does not leave SIGTERM blocked or ignored in FFmpeg
- Detect exec failure separately from successful launch; a PID does not prove RTSP is connected or video is being written
- Verify the input is H.264, select only video, and prevent overwrites with `-n`; validate SPS/PPS and a decodable IDR frame in actual output
- Have one owner for child status/reaping (`waitpid`, including interrupted calls); report unexpected exit and avoid zombies or signalling a stale/reaped PID
- Stop with SIGTERM, a bounded monotonic wait, then SIGKILL/reap. Validate connection/read timeout options against the installed FFmpeg version
- Do not assume RTSP reconnection: a failed video track permits audio-only recording, with the failure logged

### Acceptance Criteria
- FFmpeg starts and receives RTSP stream
- `.h264` file is playable with `ffplay`
- Clean shutdown with SIGTERM
- Missing executable, unavailable/stalled RTSP, early child exit, non-H.264 input, existing output, and forced-kill shutdown are tested
- Logs remain observable in daemon mode, including FFmpeg errors

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
| RECORDING | storage error / overrun / pacing loss / last track lost | STOPPING | Log error, stop remaining capture |
| RECORDING | one source fails, other remains | RECORDING | Finalize failed track, log degraded recording |
| STOPPING | finalize done | IDLE | Close WAV, log summary |
| RECORDING | ON | RECORDING | Ignore duplicate start |
| IDLE | OFF | IDLE | Ignore duplicate stop |
| STOPPING | ON / OFF | STOPPING | Ignore; do not queue a new session |

### Critical Implementation Details
- One control loop serializes start/stop and owns session resources. Protect shared state, but never hold its mutex across blocking I/O, joins, or child waits
- Use one timestamp basename for both tracks; if either path exists, choose a shared numeric suffix. Reserve WAV with exclusive creation and never truncate existing recordings
- Serialize startup and honor stop/shutdown events that arrive during it. Start the monotonic duration deadline when startup begins, not after RTSP connects
- Enter RECORDING only with at least one usable track; source-specific startup/runtime failures disable that track and permit the other to continue. Stop if neither remains
- Distinguish source failures from shared storage errors, queue overflow, and sustained pacing loss, which stop both tracks
- Make stop/cleanup idempotent: request audio stop and signal FFmpeg before waiting; stop/join audio production, drain/join its writer, finalize WAV, reap FFmpeg (escalating on timeout), then release session resources and return to IDLE
- Keep nonempty partial files and log per-track outcome, elapsed time, byte counts, and finalization errors. Matching filenames do not imply sample-accurate A/V synchronization

### Acceptance Criteria
- Full start/stop cycle via UART and CLI
- Both files created with correct timestamps
- Max duration auto-stop works
- Duplicate triggers and stop/shutdown during startup or finalization cannot create overlapping sessions, double-close files, or deadlock
- Same-second starts and wall-clock rollback never overwrite files; changing wall clock does not change the timeout
- Audio-only/video-only sessions work, loss of the last track returns to IDLE, and storage failure stops both tracks
- Repeated start/stop cycles leak no descriptors, child processes, threads, or per-session buffers

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

Daemonize before starting any threads or capture processes, resolve relative paths before changing directory, and preserve diagnostics on initialization failure. Enforce a single running instance for the configured capture devices to prevent competing UART/SPI readers. Set explicit, target-validated thread stack sizes; count the queue, stdio buffer, and stacks in the daemon's <4MB target. Measure FFmpeg separately as part of the whole 64MB system budget rather than assuming a 2-3MB footprint.

### Critical Implementation Details
- For the default output, verify `/sdcard` is mounted and writable before creating `/sdcard/recordings`; do not silently record to the root filesystem when the card is missing
- Recheck storage for every session, handle directory/open failures, and stop both tracks on runtime ENOSPC/EIO; free-space checks alone cannot guarantee subsequent writes
- Foreground logs go to stderr; background logs go to syslog and include track failures, queue/deadline diagnostics, child exit status, and finalization results
- Validate on-target SPI ioctl and SD failure behavior: these kernel operations can block, so FFmpeg's shutdown timeout does not guarantee all I/O can be cancelled immediately
- Document that graceful stops patch WAV headers; power loss/SIGKILL recovery and sample-accurate A/V synchronization are not provided by this initial implementation

### Status Log Format
```
[recorder] Recording: 00:05:30, audio: 29.1 MB, video: running, zero_chunks: 0, deadline_misses: 0, overruns: 0
```

### Acceptance Criteria
- Daemon runs in background
- UART triggers recording successfully
- Files saved to `/sdcard/recordings/` with timestamps
- Clean shutdown on signal finalizes files
- Graceful handling of missing hardware
- Missing/unmounted SD, read-only output, full disk, and removal during recording never silently fall back to root storage or report failed finalization as success
- A one-hour target run validates peak daemon/FFmpeg memory, PCM duration, video decodability, startup skew/drift, and cleanup under the existing camera services
- Host tests cover UART parsing, WAV bytes, state transitions, and child lifecycle; hardware-specific timing and memory checks are recorded separately

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
