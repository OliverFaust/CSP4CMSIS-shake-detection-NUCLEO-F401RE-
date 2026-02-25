
# CSP4CMSIS Shake Detection  
### NUCLEO-F401RE + L3G4200D Gyroscope

This project demonstrates interrupt-driven shake detection on the STM32 NUCLEO-F401RE using the L3G4200D gyroscope.

It uses:

- STM32CubeIDE
- FreeRTOS (CMSIS-RTOS)
- CSP4CMSIS (Communicating Sequential Processes)
- Interrupt-driven SPI acquisition
- Real-time signal processing

The system detects when the sensor is shaken and prints a message to the serial terminal.

---

# 1. Hardware Used

- STM32 NUCLEO-F401RE
- Digilent PMOD Gyro (L3G4200D)
- USB connection for power + serial output

Gyroscope reference:

https://digilent.com/reference/pmod/pmodgyro/start

The PMOD Gyro uses the L3G4200D 3-axis digital gyroscope.

---

# 2. Pin Connections

This project uses **SPI3** on the NUCLEO-F401RE.

| Gyro Pin | STM32 Pin | Description |
|----------|-----------|------------|
| VCC      | 3.3V      | Power |
| GND      | GND       | Ground |
| SCK      | PC10      | SPI3 SCK |
| MISO     | PC11      | SPI3 MISO |
| MOSI     | PC12      | SPI3 MOSI |
| CS       | PB12      | Chip Select (software controlled) |
| INT2     | PA1       | Data Ready Interrupt (EXTI) |

Important:
- SPI is configured in master mode.
- INT1 is configured as rising-edge external interrupt.
- Chip Select (CS) is controlled in software.

---

# 3. System Architecture

This project uses a **CSP (Communicating Sequential Processes)** architecture.

The processing chain is:
```text
Interrupt (INT1) -> L3g4200d Process (Sensor Reader) -> ShakeDetect Process (Signal Processing) -> UI Process (printf output)
```

Each block runs as an independent CSP process.

---

# 4. Interrupt Handling

The gyro asserts INT1 when new data is ready.

The interrupt handler:

```cpp
extern "C" void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
```

does NOT perform SPI communication.

It only sends a trigger event into a CSP channel:
This keeps the interrupt short and safe.

# 5. L3g4200d Process (Sensor Layer)

This process:
1. Waits for interrupt trigger
1. Reads X, Y, Z angular velocity using SPI
1. Sends the data into the next CSP channel
```cpp
trigger_reader >> t;
L3G4200D_ReadDPS(&gyro, &msg.x, &msg.y, &msg.z);
out << msg;
```
This ensures:
- No polling
- No blocking in ISR
- Clean separation of hardware and processing

# 6. Shake Detection Algorithm

Shake detection is based on short-term angular energy.

The algorithm works as follows:

## Step 1 – Compute Magnitude

We calculate total angular velocity:
```text
mag = sqrt(x² + y² + z²)
```
This gives total rotational motion.

## Step 2 – Remove Slow Drift

We maintain a running mean:
```text  
mean += alpha * (mag - mean)
hp = mag - mean
```
This acts like a high-pass filter.

Slow movement is removed.
Rapid movement remains.

## Step 3 – Energy Calculation

We accumulate squared high-pass values:
```text
energy += hp²
```
After a short time window (about 100 ms):
```text
avg_energy = energy / window_size
```
## Step 4 – Hysteresis Detection

Two thresholds are used:
- threshold_on
- threshold_off

If energy rises above threshold_on → SHAKE detected
If energy falls below threshold_off → Shake ended

This prevents rapid toggling.

# 7. UI Layer

The UI process prints:
```text
>>> SHAKE DETECTED! <<<
```
and
```text
Shake ended.
```
It only prints when the shake state changes.

# 8. Why CSP?

CSP provides:
- Clear process separation
- Deterministic communication
- No shared global data
- Zero heap (static network mode)

This is good practice for real-time embedded systems.

# 9. Learning Outcomes
This project demonstrates:
- External interrupt handling
- SPI communication
- Interrupt-to-task synchronization
- Real-time signal processing
- State machine design
- CSP-based architecture
- Hysteresis thresholding

# 10. How To Build

1. Open project in STM32CubeIDE
1. Build
1. Flash to NUCLEO-F401RE
1. Open serial terminal (115200 baud)
1. Shake the board

# 11. Example Output
```text
--- Launching CSP Static Network (Zero-Heap) ---
>>> SHAKE DETECTED! <<<
Shake ended.
>>> SHAKE DETECTED! <<<
```

# 12. Possible Extensions
- Multi-level shake intensity
- CSV streaming for plotting
- Fixed-point implementation
- LED indicator instead of printf
- Machine learning classification

# Author
Oliver Faust

