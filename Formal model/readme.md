# ShakeDetect: Formal Model (CSP-M)

This directory contains the formal **Communicating Sequential Processes (CSP)** model for the real-time gyroscope shake detection pipeline. The specification is written in machine-readable CSP-M and is configured for animation and verification using the **ProB** model checker.

While the main repository contains the production C++ source code and hardware-specific drivers (SPI, interrupts), this subdirectory isolates the algorithmic and logical core to mathematically verify its deterministic behavior, lack of deadlocks, and synchronization rules.

## Network Architecture

The system is modeled as a static parallel composition of four synchronized processes interacting via rendezvous channels:

1. **`ENV` (ISR Environment):** Models the hardware interrupt as an external process. It produces an unbounded sequence of triggers (`g_trigger_chan`) to simulate the gyroscope's data-ready signal.
2. **`L3g4200d` (Sensor Reader):** A hardware abstraction process that waits for a trigger, non-deterministically generates a bounded X, Y, Z angular velocity reading (`MAX_VAL = 2`), and passes it along the `msg_chan`.
3. **`ShakeDetect` (Signal Processing):** The core stateful algorithm. It maintains an exponential moving average (EMA) for high-pass filtering and accumulates energy over a sliding window (`WINDOW_SIZE = 3`). It evaluates the averaged energy against hysteresis thresholds (`THRESHOLD_ON = 8`, `THRESHOLD_OFF = 3`) to output a definitive shake state (`1` for true, `0` for false) on the `result_chan`.
4. **`UI` (User Interface):** Consumes the result from `result_chan` and forwards it to an observable `print_` event.

## System Composition

The entire pipeline strictly adheres to CSP rendezvous semantics, composed as a single synchronized network via the `SYSTEM` process:

```csp
SYSTEM =
    (
        (
            ( ENV [| {| g_trigger_chan |} |] L3g4200d )
            [| {| msg_chan |} |]
            ShakeDetect(0, 0, 0, false)
        )
        [| {| result_chan |} |]
        UI
    )
