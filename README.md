# App 3 scaffold — Interrupts & bottom-half

Scaffold level: **~70% complete**.

## What's given

- Complete `IRAM_ATTR` ISR with debounce, scope pulse on GPIO 19
- Two bottom-half paths: binary semaphore AND direct task notification
- Live latency measurement (µs) printed on every press
- Wokwi diagram with button + indicator LED
- An **optional background load** (App 2's four tasks) behind one `#define`, so "measure latency under load" is a flag flip rather than a copy-paste

## Run modes (`WITH_LOAD`)

App 3 reports the same per-press latency fields in both modes; only the contention on Core 1 changes. The switch is one line at the top of `main.c`:

```c
#define WITH_LOAD 0   /* 0 = idle baseline, 1 = App 2's 4 tasks on Core 1 */
```

- `WITH_LOAD = 0` — **idle**. Only the two bottom-half tasks live on Core 1. This is your baseline `latency-max`.
- `WITH_LOAD = 1` — **loaded**. App 2's four periodic tasks (10/20/50/100 ms) run on Core 1 at the rate-monotonic ladder (15/10/5/2). They are a fixed load fixture — deterministic, peripheral-free compute (xorshift, FIR, CRC-32, forced-worst-case insertion sort) carried over from App 2 so the load is reproducible and Wokwi-safe.

**Priority geometry you must account for:** your bottom-half tasks sit at priority **12**. Load Task A is **15**, so it outranks them; load Tasks B/C/D (10/5/2) do not. Under load, Task A can therefore delay a wake while B/C/D cannot preempt your bottom half. That asymmetry is the result you explain in the analysis.

## What you do

1. **Theme rename** — `YOURTHEME` and customize the log messages
2. **Run >= 50 presses, idle** (`WITH_LOAD 0`). Record `latency-max` for both paths.
3. **Flip to `WITH_LOAD 1`**, rebuild, and run >= 50 presses again. Re-record both paths. Confirm the four load tasks are live (their heartbeat counters climb).
4. **Induce a failure** — pick ONE and document the symptom:
   - Remove `portYIELD_FROM_ISR(higher_woken)` → notification is delivered, but the task doesn't run until the next tick
   - Remove `IRAM_ATTR` → first-press latency on cold cache spikes 10-100x
   - Replace `xSemaphoreGiveFromISR` with `xSemaphoreGive` → undefined behavior; system may crash
5. **Defend in README** (see prompts below)

## Capturing latency with Wokwi's logic analyzer

1. In Wokwi, click the "+" near the chip, add a "Logic Analyzer".
2. Connect channel 0 to GPIO 18 (button), channel 1 to GPIO 19 (ISR pulse).
3. Run, click the button N times.
4. Click the logic-analyzer to download a VCD file.
5. Open in PulseView / sigrok or Wokwi's built-in viewer.
6. Measure: time from button-low edge to GPIO-19 rising edge = total interrupt response time (HW latency + your debounce gate + the ISR prologue).
7. Time from GPIO-19 falling edge to the next `[sem]` or `[notif]` log line = bottom-half wake latency. This is the more interesting number, and it's the one that moves when you flip `WITH_LOAD`.

## Engineering analysis (README, graded)

1. **What's in your ISR? What's NOT?** List every line. Defend each (or remove it).
- int64_t now = esp_timer_get_time(): Reads the high-resolution hardware timer.
- if (now - last_edge_us < DEBOUNCE_US) return: Implements a software time-gate to 
reject contact bouncing. Filters out noise using conditional math without blocking delay loops.
- last_edge_us = now: Updates the state tracking variable. The compiler is forced to read or 
write it directly to memory.
- gpio_set_level(ISR_PULSE_GPIO, 1): Directly toggles the hardware register of GPIO 19.
- isr_entry_time_us = now: Logs the top half arrival time into a structure so the 
bottom half task can read it.
- presses_observed++: Increments the counter atomically.
- BaseType_t higher_woken = pdFALSE: Allocates a stack flag to track task priority.
- xSemaphoreGiveFromISR(btn_sem, &higher_woken): This line releases the binary 
semaphore token from the interrupt context to unblock the waiting task.
- vTaskNotifyGiveFromISR(task_notif_handle, &higher_woken): Signals the bottom-half safety thread.
- gpio_set_level(ISR_PULSE_GPIO, 0): Clears the pulse pin, signaling the end of the top half execution.
- portYIELD_FROM_ISR(higher_woken): Forces the scheduler to 
check the priority queue when exiting the interrupt vector.
- NOT in the ISR: there are no print or ESP_LOGI statements since they are both considered undefined behavior.
2. **Binary semaphore vs direct task notification** — quote your measured latency numbers. Which is faster? Why?
Direct Task Notification: Maximum Latency = 3,699 us
Binary Semaphore Path: Maximum Latency = 5,573 us
Running idly, the direct task notification path is faster.
3. **Latency under load** — quote idle (`WITH_LOAD 0`) vs loaded (`WITH_LOAD 1`) numbers. By what factor does latency increase? Use the priority geometry above (Task A at 15 outranks your bottom half at 12) to explain *which* load task is responsible for the worst-case increase, and why B/C/D are not.
Direct Task Notification: Maximum Latency = 19,809 us
Binary Semaphore Path: Maximum Latency = 10,384 us
The Semaphore path latency increased by a factor of 1.86.
The Notification path latency exploded by a factor of 5.36.
The background load for the bottom half tasks consists of four tasks. Because Task A has higher priority than the other tasks, it acts as an execution barrier. The extreme 19,809 us spike on the notification path happens due to the notification task waking up first, starting its slow serial print, 
and getting halfway through transmitting data over UART.
4. **Induced failure** — name the rule you broke, the predicted symptom, the observed symptom, and how they match (or don't).
Rule Broken: commenting out portYIELD_FROM_ISR(higher_woken).
Prediction: The notification signal will be placed into the target task's control block, but the CPU will return context to
whatever background task was running before the interrupt.
Result: The Notification latencies went from a stable baseline to an oscillating window stretching from 1,000 us to over 19,000 us.

## Common pitfalls

- **Calling `printf` inside the ISR.** `printf` takes a UART mutex. Mutex from ISR = undefined behavior. The scaffold puts logging in the BOTTOM-HALF tasks for a reason.
- **Forgetting `IRAM_ATTR`.** The first interrupt after a long quiescent period has to load the ISR from flash. That's ~10s of µs of cache fill on top of your nominal latency. With `IRAM_ATTR`, the ISR is in always-on internal RAM.
- **Debounce too short.** A clean push-button bounces for 1–10 ms typically. Wokwi's simulated button is clean, but if you wire a real button, drop `DEBOUNCE_US` to something like 10000 µs.
- **Editing the load-task bodies.** Under `WITH_LOAD 1` the four tasks are a fixture, not the assignment. You're timing your ISR path, so leave their bodies alone; tune only the `*_ITERS`/`*_N`/`*_LEN` knobs if you want a heavier or lighter load.
- **Both bottom-half tasks racing on `latency_max_*`.** This is fine for the scaffold (32-bit reads are atomic, and the max-update is benign-racy). In production you'd use atomics or a mutex — that's App 6's lesson.

## Setup in Wokwi

Same shape as App 1. In a fresh Wokwi ESP-IDF project (`https://wokwi.com/projects/new/esp32-s3`):

1. Replace `diagram.json`, `wokwi.toml`, and `main/CMakeLists.txt` with this folder's versions. (App 3 has no `sdkconfig.defaults` &mdash; uses IDF defaults.)
2. Place this folder's `main.c` at `main/main.c` (delete Wokwi's `main/src/`), **or** leave `main/src/main.c` and edit `main/CMakeLists.txt` to use `SRCS "src/main.c"` + `INCLUDE_DIRS "src"`.
3. Confirm `wokwi.toml`'s `firmware` / `elf` paths match `app3_interrupts` (the `project(...)` name in `CMakeLists.txt`).
4. Click &#9654;.

**No web page in App 3.** All output is on the serial monitor; visual feedback is the yellow ISR-pulse LED on GPIO 19. Turning on the background load (`WITH_LOAD 1`) needs no extra components and no Wi-Fi &mdash; the four tasks are pure compute on Core 1. The Wokwi logic-analyzer instructions above are how you capture the timing numbers.

### Build locally with ESP-IDF instead

```bash
. $HOME/esp/esp-idf/export.sh
idf.py set-target esp32s3
idf.py build
idf.py -p /dev/ttyUSB0 flash monitor
```

To build the loaded configuration from the command line without editing the file, override the flag at configure time:

```bash
idf.py build -DWITH_LOAD=1     # or set it in main.c
```

## Assets

Here is a link which leads to a PDF containing the latency measurement table and the images of the wavescope:
https://github.com/topspin05/A3-images/blob/main/A3_tables_images.pdf

Here is a link to the concurrency diagram:
https://raw.githubusercontent.com/topspin05/A3-images/refs/heads/main/A3_Concurrency_Diagram.jfif
## Honor code

AI was only utilized lightly to give me a better understanding of the code and what it does.
I also used it to help with the github repository and pages.
