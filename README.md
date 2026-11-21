# Hardware-Accelerated DAQ Engine

A streaming data-acquisition engine for Xilinx/AMD Zynq SoCs. ADC capture, decimation, FFT and trigger detection run at line rate in the programmable logic (PL), so only decimated samples, spectra or triggered records reach DDR and the host.

> **Status: design phase.** The architecture is defined; no RTL or software has been committed yet. Figures marked *assumed* are working numbers to be confirmed against the chosen board and ADC.

## Goals

- Sustain **1-100 MS/s** acquisition on a Zynq SoC (design point: 8 channels, 16-bit, 50 MS/s each, assumed).
- Process in hardware: **filtering and decimation**, **windowed FFT**, and **triggering** with pre- and post-trigger capture.
- Move reduced data to Linux userspace with scatter-gather DMA and zero-copy buffers.
- Make every gap or overrun visible through sequence numbers and sample-count timestamps.

## Architecture

```
 ADC ──► ADC capture ──► Front end ──► Decimate ──► Window + FFT ──┐
        (LVDS/JESD204)   (offset,gain)  (CIC+FIR)   (magnitude,avg) │
                              │             └── decimated bypass ───┤
                              ▼                                     ▼
                     Trigger + capture buffer ─ records ──────► Packetizer
                     (level/edge/window, pre/post)                  │
                                                                    ▼
   AXI4-Lite control/status ◄───── PS ──────────────────────── AXI DMA (SG)
                                                                    │
                                   ┌────────────────────────────────┘
                                   ▼
        DDR ring buffers ─► driver + ring ─► C++/Python API ─► egress
        (CMA, zero-copy)    (pinned A53)                       (Ethernet/disk/USB)
```

Three output modes share one packetizer: continuous decimated samples, averaged spectra, and triggered records. The PS selects the mode per channel through registers.

### PL data path

| Stage | Implementation |
| --- | --- |
| ADC capture | LVDS deserializer or JESD204 core, per-channel alignment, CDC into AXI4-Stream |
| Front end | DC-offset and gain correction, optional NCO mixer |
| Decimation | CIC followed by polyphase FIR decimator, programmable ratio |
| Spectrum | Window, pipelined streaming FFT, magnitude-squared, optional averaging |
| Trigger engine | Level/edge with hysteresis, window comparator, multi-channel logic, hold-off, re-arm |
| Capture buffer | Circular pre-trigger buffer per channel, post-trigger counter |
| Packetizer | Header (sequence, channel mask, timestamp, trigger source, length) plus payload |

The trigger sees full-rate samples ahead of decimation so it can catch short pulses.

### PS and host path

- AXI DMA (scatter-gather) into reserved CMA memory, exposed to userspace as dma-bufs.
- One descriptor ring per output mode, so spectra never queue behind raw data.
- Lock-free single-producer/single-consumer ring in userspace, DAQ thread pinned to an isolated core.
- C++ library with Python bindings: configure, arm, read records, read statistics (drops, ring fill, trigger count).

## Bandwidth at the design point

Assumed: 8 channels, 16-bit, 50 MS/s each.

| Mode | Rate into DDR | Fits Gigabit Ethernet (~110 MB/s) |
| --- | --- | --- |
| Raw continuous | 800 MB/s | No |
| Decimate by 10 | 80 MB/s | Yes, little margin |
| Averaged spectra (4096-pt, 50% overlap, 64-frame average) | 25 MB/s | Yes |
| Triggered records (4096 samples, 1,000 events/s) | 65.5 MB/s | Yes |

DDR bandwidth is not the limit; the egress link and the userspace consumer are.

## Planned repository layout

```
rtl/        PL blocks (capture, DSP, trigger, packetizer) and testbenches
model/      bit-accurate Python golden model, coefficient generation
sw/driver/  Linux DMA driver and device tree overlay
sw/lib/     C++ library and Python bindings
sw/tools/   capture, throughput and soak-test utilities
docs/       architecture notes
```

This layout is a plan; folders appear as the work lands.

## Roadmap

1. Golden model in NumPy/SciPy in the same fixed-point formats as the PL.
2. Self-checking block testbenches against golden vectors (bit-exact or within 1 LSB).
3. Whole-chain simulation with a synthetic ADC source (tones, noise, known pulses).
4. DMA throughput test with a PL pattern generator at the stress rate (8 ch x 100 MS/s), zero drops for at least one hour.
5. ADC bring-up using test patterns, then real input with an ILA on the capture stage.
6. Signal-level checks: SNR, SFDR, filter pass/stop band, trigger and pre-trigger placement.
7. Soak and fault tests: forced overruns, drop counters, recovery.

## Open questions

- Is 1-100 MS/s per channel or aggregate, and how many channels?
- Which ADC part and link type (LVDS or JESD204)?
- Which Zynq part and board, and which egress medium?
- Required trigger types and trigger-to-record latency?
- FFT length, window and averaging requirements?
- Tool flow: Vivado/Vitis with HDL, or HLS for the DSP blocks?

## License

Not yet chosen.
