# DSL_WiggleBits

---

## Overview
This project explores hardware-based random number generation using FPGA and external ADC noise sources.

The key focus is a **comparative study of two entropy conditioning pipelines**, both feeding a shared pseudo-random generator composed of an **MCG (Multiplicative Congruential Generator) followed by an LCG (Linear Congruential Generator)**.

The system evaluates how preprocessing affects entropy quality before deterministic expansion.

---

## Quick Setup

### 1. Clone Repository
```bash
git clone <repo link>
cd DSL_WiggleBits
```

---

### 2. Import Design Modules
Open the FPGA project (Vivado or equivalent) and import all RTL sources:
- ADC driver modules
- signal processors (Pipeline 1 & 2)
- RNG core (MCG + LCG)
- UART + display drivers

Ensure all files are included in synthesis.

---

### 3. Set Top Module (Critical Step)

Select ONE top module depending on test mode:

#### ADC Debug Mode
```text
adc_top_module
```
- ADC data visualization only
- Used to verify ADC channels and entropy source quality

---

#### Full Pipeline Mode
```text
adc_top_module_transform
```
- Full entropy conditioning pipeline
- Outputs final RNG values via UART
- Includes: XOR → conditioning → MCG → LCG

---

## PC / COM Port Setup

### 4. Install Dependencies
```bash
pip install pyqtgraph pyserial numpy
```

---

### 5. Select Plotter Script

#### If using `adc_top_module`
```text
graphplotter_2ch.py
```
- Visualizes raw ADC channels (CH0 + CH1)
- Used for entropy inspection and debugging

---

#### If using `adc_top_module_transform`
```text
graphplotter_1ch.py
```
- Visualizes processed 12-bit transformed signal
- Displays entropy-conditioned output

---

### 6. Configure COM Port
Edit inside the script:
```python
SERIAL_PORT = 'COM3'
```

- Windows → COM3 / COM4
- macOS → /dev/tty.usbserial-XXXX
- Linux → /dev/ttyUSB0

---

### 7. Run Visualizer
```bash
python py_script/graphplotter_1ch.py
```

or
```bash
python py_script/graphplotter_2ch.py
```


## System Architecture (Global)

Both pipelines share the same backend:

```text
Entropy Pipeline → MCG → LCG → RNG Output (UART / Display)
```

---

## ADC Sampling Front-End

- ADC: MCP3202 (SPI interface)
- Two channels sampled alternately
- Controlled by:
  - clk_adc_spi (SPI clock)
  - clk_adc_tri (sampling trigger)

Captured data:
- r_adc_data0 → Channel 0
- r_adc_data1 → Channel 1

---

## Entropy Processing Pipelines

---

### Pipeline 1: Direct XOR Path (signal_processor_adc)

```text
ADC CH0 + CH1 → 12-bit XOR → MCG → LCG
```

- Uses full 12-bit XOR:
```verilog
r_adc_data0 ^ r_adc_data1
```

- Baseline entropy reference
- High throughput, minimal processing

---

### Pipeline 2: Conditioned Entropy Path (signal_processor)

```text
ADC CH0 + CH1 → LSB XOR → Markov → Von Neumann → MCG → LCG
```

Processing steps:

1. LSB XOR:
```verilog
r_adc_data0[0] ^ r_adc_data1[0]
```

2. 1-bit Markov conditioning  
3. Von Neumann whitening (bias removal)

- Lower throughput
- Higher statistical conditioning quality

---

## PRNG Backend (Shared)

### MCG Stage
- Multiplicative recurrence generator
- Expands entropy state

### LCG Stage
- Linear recurrence refinement
- Final output diffusion

```text
MCG → LCG → rng_num
```

Output:
- 32-bit random number stream

---

## Display System
- 7-segment display shows 16-bit segments
- Button toggles upper/lower halves

---

## UART Transmission
```text
[0xAA][Byte3][Byte2][Byte1][Byte0][0xFF]
```

- clk_uart_clk → baud timing
- clk_uart_tri → FSM control

---

## Clock Domains

| Clock | Frequency | Purpose |
|------|----------|--------|
| clk_adc_tri | 1 kHz | ADC sampling |
| clk_adc_spi | 1 MHz | ADC SPI |
| clk_uart_tri | 1.5 kHz | UART FSM |
| clk_uart_clk | 115200 | UART baud |
| clk_seg_clk | 1 kHz | Display |

---

## Reset Behavior
- Active low reset: rstn = ~btn[1]

---

## Data Flow Summary

### Pipeline 1
```text
ADC → 12-bit XOR → MCG → LCG → Output
```

### Pipeline 2
```text
ADC → LSB XOR → Markov → Von Neumann → MCG → LCG → Output
```

---

## Known Issues

- Multiple-driver risk on entropy signals in earlier revisions
- No clock domain crossing protection (ADC ↔ UART)
- UART has no backpressure handling

---

## Suggested Improvements

- Add FIFO between ADC and PRNG stages
- Add statistical validation (NIST tests / entropy metrics)
- Formalize MCG/LCG parameters
- Add explicit pipeline selection parameter
- Synchronize cross-domain signals properly

---

## File Structure

- adc_top_module.v → ADC debug pipeline
- adc_top_module_transform.v → full RNG pipeline
- signal_processor_adc.v → Pipeline 1
- signal_processor.v → Pipeline 2
- rng_algo_processor.v → MCG + LCG core
- drv_uart_tx.v → UART
- drv_mcp3202.v → ADC driver
- drv_7segment.v → display

---

## Bottom Line

This project is a **comparative entropy conditioning system**, not a single RNG design:

- Pipeline 1: raw ADC XOR baseline
- Pipeline 2: conditioned entropy stream (Markov + Von Neumann)
- MCG + LCG: shared deterministic amplification backend

The core objective is to evaluate how preprocessing affects entropy quality before PRNG expansion.
