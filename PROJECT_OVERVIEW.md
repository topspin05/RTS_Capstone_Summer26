# Project Overview

**E-Stop Interrupt Response — Industrial Real-Time Systems**

## What it is

An emergency-stop interrupt handler for an industrial ride-control system, built on FreeRTOS and an ESP32. A button press on GPIO18 fires a hardware interrupt; the ISR timestamps the event and wakes two independent bottom-half tasks, one via a binary semaphore, one via a direct task notification, so both signaling paths can be measured and compared under real conditions.

## What it demonstrates

- **Top-half / bottom-half interrupt pattern** — a minimal, `IRAM_ATTR` ISR does only timestamping and signaling; all logging and processing happens in scheduled tasks.
- **Two signaling primitives compared head-to-head** — binary semaphore vs. direct task notification, measured for worst-case wake-up latency both idle and under real Core-1 contention.
- **Dual-core scheduling behavior** — the ISR runs on Core 0, the response tasks run on Core 1.

## Key results

| Path | Idle Max (µs) | Loaded Max (µs) | Ratio |
|---|---|---|---|
| Binary Semaphore | 5,573 | 10,384 | 1.86× |
| Direct Task Notification | 3,699 | 19,809 | 5.36× |

Notification is faster in isolation, but its worst case grows more under load — because it tends to win the race to run first, exposing its logging call to preemption by the one background task with a higher priority than the response tasks.

