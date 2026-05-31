# 🎛️ My Melody MotionBox
**A handheld gesture-controlled synthesizer for children's STEM education**

> Tilt it, shake it, hear it. Every movement produces a distinct pitch — fast gestures go high, slow tilts go low. No buttons, no screens, no instructions needed.

Built for McGill ECSE 444 (Microprocessors) · Team of 4

---

## What it does

The MotionBox reads 3D angular velocity from an onboard gyroscope and maps it to real-time sine-wave audio output. Two independent channels (X-axis and Y-axis) are routed to separate speakers so the device produces true stereo feedback — each spatial direction has its own voice.

| Input | Output |
|---|---|
| Tilt left/right (X-axis) | Pitch on left speaker |
| Tilt forward/back (Y-axis) | Pitch on right speaker |
| Fast movement | Higher frequency |
| Slow movement | Lower frequency |
| Still / resting | No audio output |

---

## Hardware

| Component | Part | Interface |
|---|---|---|
| Microcontroller | STM32L4S5VIT (IoT01 Discovery) | — |
| Gyroscope / IMU | LSM6DSL (6-axis) | I2C |
| Audio output | 8 Ω speakers ×2 | DAC + DMA |
| Visual debug | Serial terminal | UART1 (115200 baud) |

---

## Software architecture

```
┌────────────────────────────────────────────┐
│                FreeRTOS (CMSIS-OS v2)       │
│                                            │
│  ┌─────────────┐   shared float gyro[3]   │
│  │ReadSensorTask│──────────────────────┐  │
│  │  (I2C poll) │                       │  │
│  └─────────────┘                       ▼  │
│                          ┌──────────────────┐ │
│                          │  GenerateXTask   │ │
│                          │  X-axis → DAC_1  │ │  → Speaker L
│                          │  (CMSIS sine +   │ │
│                          │   DMA transfer)  │ │
│                          └──────────────────┘ │
│                          ┌──────────────────┐ │
│                          │ GenerateYWaveTask│ │
│                          │  Y-axis → DAC_2  │ │  → Speaker R
│                          └──────────────────┘ │
│                          ┌──────────────────┐ │
│                          │   PrintTask      │ │
│                          │   UART feedback  │ │  → Serial terminal
│                          └──────────────────┘ │
└────────────────────────────────────────────┘
```

### Four RTOS threads

| Thread | Role |
|---|---|
| `ReadSensorTask` | Polls LSM6DSL over I2C, writes X/Y/Z angular velocity to shared `gyro[3]` |
| `GenerateXTask` | Reads gyro X, computes sample count, generates sine wave, streams to `DAC_CHANNEL_1` via DMA |
| `GenerateYWaveTask` | Same as above for Y-axis → `DAC_CHANNEL_2` |
| `PrintTask` | Monitors gyro state, sends human-readable direction messages over UART |

### Frequency modulation formula

```c
// Applied in both GenerateXTask and GenerateYWaveTask
if (abs(gyro[axis]) > 10000) {
    sine_modulation = 1 + (abs(gyro[axis]) - 10000) / 15000.0f;
    samples = BASE_SAMPLES / sine_modulation;  // fewer samples = higher frequency
}
```

Faster motion → fewer samples per cycle → higher perceived pitch.

### Sine wave generation (CMSIS-DSP)

```c
void generate_sine_wave(uint32_t *sine_wave_final, int sample_count) {
    for (uint32_t i = 0; i < sample_count; i++) {
        float32_t angle = (2 * PI * i) / sample_count;
        float32_t sine_value = arm_sin_f32(angle);
        // Scale to 12-bit DAC range [0, DAC_SCALE_FACTOR]
        sine_wave_final[i] = (uint32_t)((sine_value + 1) * DAC_SCALE_FACTOR / 2);
    }
}
```

- Uses `arm_sin_f32` from CMSIS-DSP — single-cycle hardware-accelerated on Cortex-M4
- Dynamically allocates buffer per cycle (`malloc` / `free`)
- DMA transfer to DAC — CPU is free to sample the next gyro reading while audio plays out

---

## Getting started

### Requirements

- STM32L4S5VIT IoT01 Discovery kit
- STM32CubeIDE (≥ 1.13)
- Two 8 Ω speakers wired to `DAC_OUT1` and `DAC_OUT2`
- USB-Serial adapter for UART debug output (optional)

### Build & flash

1. Clone the repo
   ```bash
   git clone https://github.com/sofiavelasquezsierra/melody-motionbox.git
   ```

2. Open in STM32CubeIDE: **File → Open Projects from File System** → select the repo root

3. Build: **Project → Build All** (or `Ctrl+B`)

4. Flash: connect the board via USB → **Run → Debug** (F11) → Resume (F8)

5. Optional UART monitor: open a serial terminal at **115200 baud, 8N1** on the board's virtual COM port

---

## UART output example

```
Great job kid! You are moving in the X direction!
Great job kid! You are moving in the Y direction!
Great job kid! You are moving in the X AND Y directions!
```

---

## Known limitations & future work

| Limitation | Proposed fix |
|---|---|
| `malloc/free` per audio cycle (fragmentation risk) | Replace with fixed-size ring buffer |
| Visual feedback on external terminal only | Add WS2812 LED ring for onboard color feedback |
| Gyroscope only — no translation detection | Fuse with LSM6DSL accelerometer for full 6-DOF gestures |
| Preset deadband (±10000 dps) | Expose as calibration parameter, auto-tune at startup |
| Mono perceived pitch (same wave both axes) | Add harmonic overtones for timbral variation |

---

## Team

| Name | McGill ID |
|---|---|
| Sofia Velasquez Sierra | 260967314 |
| Cyril El Feghali | 261010517 |
| Lynn Haddad | 261005938 |
| Walid Aissa | 261039319 |

---

## License

Academic project — McGill University ECSE 444, Fall 2023. Code shared for portfolio purposes.
