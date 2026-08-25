# Light Sampler

An embedded Linux application (built for a BeagleBone-class ARM target) that continuously samples ambient light levels through a potentiometer-controlled history buffer, detects "light dips," displays results on a console and a 14-segment display, and exposes a UDP command interface for remote querying and control.

## Overview

The system runs several cooperating threads on top of a shared, resizable circular sample history:

- **Sampler** — reads the light sensor as fast as possible (every 1 ms), maintains a running exponential-smoothed average, and stores samples in a circular history buffer whose size can change at runtime.
- **Buffer Resize** — reads a potentiometer once a second and uses its value to resize the sample history buffer (via `Sampler_setHistorySize`).
- **Buffer Analyzer** — periodically scans the current history for "light dips" (drops below the average beyond a threshold, with hysteresis to avoid double-counting) and extracts every 200th sample for reporting.
- **Terminal Output** — prints a one-second status summary to stdout: samples/sec, POT value, valid sample count, average light level, dip count, and every 200th sample.
- **Display Dips on Seg** — mirrors the current dip count onto a 2-digit 14-segment hardware display.
- **UDP Listener** — accepts commands over UDP (port `12345`) to query counts, history, and length, or to shut the program down remotely.
- **Shutdown** — a small condition-variable-based module any thread can use to unblock `main()` and trigger an orderly shutdown of all modules.

## Project Structure

| File | Purpose |
|---|---|
| `main.c` | Initializes all modules, blocks the main thread until shutdown is requested, then tears everything down in order. |
| `sampler.c` / `sampler.h` | Background light-sampling thread; resizable circular history buffer; average/reading accessors. |
| `bufferResize.c` / `bufferResize.h` | Polls the potentiometer once a second and resizes the sampler's history accordingly. |
| `bufferAnalyzer.c` / `bufferAnalyzer.h` | Light-dip detection logic and "every 200th sample" extraction. |
| `terminalOutput.c` / `terminalOutput.h` | Console status-reporting thread (runs once per second). |
| `displayDipsOnSeg.c` / `displayDipsOnSeg.h` | Drives the 14-segment hardware display with the current dip count. |
| `UDP_Listener.c` / `UDP_Listener.h` | UDP server thread that parses and responds to remote commands. |
| `shutdown.c` / `shutdown.h` | Mutex/condvar helper for blocking and releasing the main thread. |
| `utils.c` / `utils.h` | Shared helper (`sleep_ms`) for millisecond-resolution sleeping via `nanosleep`. |
| `Makefile` | Cross-compiles the project and deploys the binary to the target. |
| `test_tftp.txt` | Sample file used to verify TFTP file transfer to the target board. |
| `HardwareModule/lightSensor.c` / `.h` | Reads the light sensor via the ADC (`in_voltage1_raw`) and converts the raw reading to a voltage. |
| `HardwareModule/potentiometer.c` / `.h` | Reads the potentiometer via the ADC (`in_voltage0_raw`), used to control the sample history size. |
| `HardwareModule/14-segDisplay.c` / `.h` | Drives a 2-digit 14-segment display over I2C (via a GPIO I/O expander) to show the current dip count. |

## Hardware

The application drives a small set of BeagleBone peripherals through the Linux `sysfs`/I2C interfaces:

- **Light sensor** — an analog sensor read through the AM335x ADC at `/sys/bus/iio/devices/iio:device0/in_voltage1_raw`. Raw 12-bit-scale readings (0–4000) are converted to a 0–1.8 V voltage in `getLightSensorReadings()`.
- **Potentiometer** — read through the same ADC subsystem at `in_voltage0_raw`. Its value directly sets the sampler's history buffer size (clamped to a minimum of 1) once per second.
- **14-segment display** — a 2-digit display driven by an I2C GPIO expander (address `0x20` on `/dev/i2c-1`).
  - GPIO pins `61` and `44` (exported via `/sys/class/gpio/export`) act as digit-select lines, multiplexing which of the two digits is currently lit.
  - I2C registers `0x14`/`0x15` (`REG_OUTA`/`REG_OUTB`) drive the segment patterns for each digit via lookup tables in `displayDipsOnSeg.c`.
  - `SegDisplay_Init()` also runs `config-pin P9_17 i2c` and `config-pin P9_18 i2c` to put those pins into I2C mode before opening the bus.
  - The display multiplexes between left and right digits rapidly (5 ms per digit) to show the current 2-digit dip count, capped at `99`.

## Building

This project cross-compiles for an ARM Linux target (e.g., BeagleBone) using:

```
CC = arm-linux-gnueabihf-gcc
CFLAGS = -std=c99 -Wall -g -pthread -D _POSIX_C_SOURCE=200809L -Werror
```

To build and deploy:

```bash
make
```

This compiles `light_sampler`, creates `$(HOME)/cmpt433/public/myApps` if needed, and copies the resulting binary there for deployment to the target board.

To clean build artifacts:

```bash
make clean
```

**Requirements:**
- `arm-linux-gnueabihf-gcc` cross-compiler toolchain installed and on your `PATH`
- Target board reachable for deployment (e.g., via NFS-mounted `$(HOME)/cmpt433/public`)
- The `HardwareModule/` sources (light sensor, potentiometer, 14-segment display) present alongside this code

## Running

Copy or run the deployed `light_sampler` binary on the target board:

```bash
./light_sampler
```

On startup it will:
1. Start the light sampler thread
2. Start the buffer-resize thread (driven by the potentiometer)
3. Start the terminal output thread (1-second status prints)
4. Initialize the 14-segment dip display
5. Start listening for UDP commands on port `12345`

The program blocks until it receives a `stop` command over UDP (see below), at which point it shuts down all modules cleanly.

## Console Output

Once per second, the program prints:

```
#samples taken: <n>
POT value: <n>
#valid samples: <n>
avg light level: <0.000>
#dips: <n>

<200>th sample: <value>
<400>th sample: <value>
...
```

## UDP Command Interface

Send commands as UDP datagrams to port `12345`. Supported commands:

| Command | Description |
|---|---|
| `help` | List available commands |
| `count` | Total number of samples taken |
| `get <N>` | The `N` most recent history values (up to 146 samples per response) |
| `length` | Current history capacity and number of valid samples in it |
| `history` | The full sample history, streamed in UDP-sized chunks |
| `stop` | Shut down the server program |
| *(empty line)* | Repeat the last command |

Example using `netcat`:

```bash
echo "count" | nc -u <board-ip> 12345
```

## Notes on Concurrency

- The sample history buffer is protected by a dedicated mutex/condvar pair (`mutex_HistoryBuffer` / `cond_HistoryBuffer`); readers block until the buffer has been filled at least once.
- History resizing is coordinated separately via `mutex_HistorySizeController`, so a resize request doesn't race with in-progress sampling.
- All background threads disable cancellation while doing work and only become cancellable at safe points, with a short `sleep_ms(1)` cancellation checkpoint before returning — this avoids threads being cancelled mid-critical-section.
- `unlock_main_thread()` (called from the UDP listener on `stop`) is the single coordinated shutdown trigger for the whole program.
