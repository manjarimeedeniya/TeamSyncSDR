# Two-Way Digital Paging System Using Software Defined Radios

**EN2130 – Communication Design Project | Semester 3**  
**University of Moratuwa – Department of Electronic & Telecommunication Engineering**

---

## Project Overview

This project implements a **two-way digital paging system** using Software Defined Radios (SDRs), built in GNU Radio with custom embedded Python blocks. The system enables addressed short message delivery between nodes over a wireless channel, with a complete protocol stack including framing, addressing, CRC-32 error detection, stop-and-wait acknowledgment, and TDMA-based channel access.

The system was first developed and validated as a **GNU Radio simulation** (mid-evaluation), and subsequently demonstrated on **real BladeRF SDR hardware**, where successful over-the-air transmission and reception of addressed messages was confirmed.

---

## Team Members

| Name | GitHub Links |
|------|-------------|
| Jayaweera N.S | https://github.com/NisalJayaweera |
| Manatunga K.D | https://github.com/KaveeDiv23 |
| Meedeniya M.M.H | https://github.com/manjarimeedeniya |
| Ranawaka R.A.G.K. | https://github.com/Gaveesha723 |

---

## Hardware Demo

The system was tested using two **BladeRF SDR** units available in the lab. Over-the-air transmission and reception of addressed text messages was successfully demonstrated. The QPSK modulated signal was transmitted from one BladeRF, received and demodulated by the second BladeRF, with the addressed payload correctly recovered at the receiver.

The acknowledgment (ACK) path was not tested on hardware due to the limitation of having only two BladeRF units available — a full two-way ACK exchange would require both nodes to be configured as simultaneous transmitter and receiver, which was not possible with the available hardware allocation. The ACK mechanism was however fully implemented and validated in the GNU Radio simulation environment.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    APPLICATION LAYER                    │
│         File-to-PDU Sender  │  PDU-to-File Sink        │
└───────────────────┬─────────────────────────────────────┘
                    │ PDU (src_id, dst_id, pkt_num, payload)
┌───────────────────▼─────────────────────────────────────┐
│                    PROTOCOL LAYER                       │
│   CRC-32 Append → Protocol Formatter → PDU Padder      │
│   Addressed Filter → CRC-32 Check → ACK Generator      │
└───────────────────┬─────────────────────────────────────┘
                    │ Tagged stream
┌───────────────────▼─────────────────────────────────────┐
│                   TDMA SCHEDULER                        │
│         Slot-based transmission gating (wall-clock)     │
└───────────────────┬─────────────────────────────────────┘
                    │ Byte stream
┌───────────────────▼─────────────────────────────────────┐
│                  PHYSICAL LAYER (TX)                    │
│   QPSK Modulator → Throttle → BladeRF Sink (SoapySDR)  │
└───────────────────┬─────────────────────────────────────┘
                    │ Over-the-air (BladeRF) / Simulated channel
┌───────────────────▼─────────────────────────────────────┐
│                  PHYSICAL LAYER (RX)                    │
│  PFB Clock Sync → Costas Loop → CMA Equaliser →        │
│  QPSK Decoder → Diff Decoder → Access Code Correlator  │
└─────────────────────────────────────────────────────────┘
```

---

## Features Implemented

### QPSK Digital Modulation

Differential QPSK modulation is used for transmission, with 4 samples per symbol and a Root Raised Cosine (RRC) pulse shaping filter (roll-off factor α = 0.35, 32 filter banks). Differential encoding is enabled on the modulator to resolve phase ambiguity at the receiver without a reference carrier.

### Unique User Addressing

Each node is assigned a unique 1-byte ID (User A = `0x01` through User E = `0x05`). Every packet carries a source ID and destination ID in its 3-byte header. The `addressed_pdu_filter` block on the receiver checks the destination field of every arriving packet and silently discards any packet not intended for its own ID — exactly as a real paging system would behave.

```
Packet structure:
[ src_id (1B) | dst_id (1B) | pkt_num (1B) | payload (up to 29B) ]
```

### CRC-32 Error Detection

Every outgoing PDU has a 32-bit CRC appended using the standard Ethernet CRC-32 polynomial (`0x4C11DB7`), with both input and result bytes reflected and XOR value `0xFFFFFFFF`. On the receiver, `digital_crc_check` validates the CRC and routes packets to an `ok` output on success, or a `fail` output on failure. Corrupted packets are automatically discarded — no ACK is sent for a packet that fails CRC validation.

### Stop-and-Wait ARQ (Acknowledgment Mechanism)

The receiver sends an ACK back to the sender after each successfully received and validated packet. The ACK is a compact 2-byte message:

```
[ 0xAA (marker byte) | pkt_num (1B) ]
```

To improve reliability over a noisy channel, the ACK is transmitted **5 times** in rapid succession. On the sender side, the `file_to_pdu_sender` block waits for the correct ACK within a configurable timeout window (`send_interval`), and retries transmission up to `max_attempts` times before skipping the packet and moving on. This is a complete **Stop-and-Wait ARQ** implementation with per-packet sequence numbers that wrap from 1 to 255.

### TDMA Channel Access

A custom `tdma_scheduler` block gates each node's transmission based on wall-clock time. The TDMA frame is divided into equal-length time slots (`slot_len_ms = 1.5 ms`, `num_slots = 4`). A node's byte stream is passed through only during its assigned slot; the output is zeroed at all other times. This prevents simultaneous transmission collisions between multiple nodes sharing the same channel.

| User | ID     | TDMA Slot |
|------|--------|-----------|
| A    | `0x01` | 0         |
| B    | `0x02` | 1         |
| C    | `0x03` | 2         |
| D    | `0x04` | 3         |
| E    | `0x05` | 4         |

### Packet Framing and Access Code Synchronisation

The `digital_protocol_formatter_async` block prepends a 32-bit access code (`11100001010110101110100010010011`) to every packet before modulation. On the receive side, `digital_correlate_access_code_xx_ts` continuously scans the incoming bit stream for this pattern to detect packet boundaries and tag the stream with `packet_len` for downstream processing.

### PDU Padding and Unpadding

The `pdu_padder` block appends 128 bytes of `0x55` pattern to every PDU before it enters the modulator. The matching `pdu_unpadder` block strips the same number of bytes from received PDUs. This ensures a consistent non-zero-length stream enters the modulator and prevents GNU Radio scheduler stalls caused by very short bursts.

### Garbage Packet Handling

A `pdu_garbage_injector` block sends a single 256-byte random PDU at startup to prime the modulator pipeline and flush buffering artefacts from the GNU Radio runtime. A matching `pdu_garbage_remover` block on the receiver detects and silently drops the first 256-byte PDU it receives, ensuring only clean application data is written to the output file.

### File-Based Message Source and Sink

The `file_to_pdu_sender` block reads any binary or text file, splits it into fixed-size addressed chunks, and transmits each as a sequenced PDU. The `pdu_to_file_sink` block on the receiver writes all successfully received payloads back to disk, enabling straightforward end-to-end verification by comparing the input and output files.

### Receiver Synchronisation Chain

The receive chain implements a full synchronisation pipeline for QPSK:

- **PFB Clock Synchroniser** (`digital_pfb_clock_sync`) with RRC matched filter for timing recovery
- **Costas Loop** (`digital_costas_loop_cc`, 4th order) for carrier phase and frequency tracking
- **CMA Linear Equaliser** (`digital_linear_equalizer`, 15 taps, Constant Modulus Algorithm) for channel equalisation
- **Differential Decoder** (`digital_diff_decoder_bb`) to undo differential encoding from the transmitter
- **Access Code Correlator** (`digital_correlate_access_code_xx_ts`) for packet boundary detection

### Channel Simulation

The flowgraph includes a `channels_channel_model` block for simulation testing, with three parameters exposed on the Qt GUI as real-time sliders:

| Parameter | Range | Default |
|-----------|-------|---------|
| Noise Voltage | 0 – 1 | 0.6 |
| Timing Offset | 0.999 – 1.001 | 1.0005 |
| Frequency Offset | -0.1 – 0.1 | 0.025 |

A 3-tap multipath model (`[1+0j, 0.3+0.1j, 0.1-0.2j]`) simulates a realistic indoor propagation environment. These simulation parameters were used to stress-test the synchronisation chain before the hardware demo.

---

## Hardware Testing (BladeRF)

The flowgraph connects to the BladeRF through a `soapy_bladerf_sink` block at a sample rate of 1 MHz. The TX output is scaled to 80% amplitude (`blocks_multiply_const_vxx`, const = 0.8) before being passed to the BladeRF sink to avoid clipping the DAC.

During the hardware test:
- One BladeRF unit was configured as the transmitter running the TX path of the flowgraph
- The second BladeRF unit was configured as the receiver running the RX synchronisation and decode chain
- Addressed text messages were successfully transmitted over the air and correctly recovered at the receiver
- The ACK path was not tested on hardware due to the two-BladeRF constraint 

---

## Challenges Faced

### Channel Noise and Demodulation Failures

Getting stable demodulation under realistic noise conditions was the most time-consuming part of the project. The PFB clock synchroniser and Costas loop parameters (`phase_bw = 0.0628`, `nfilts = 32`, `sps = 4`) required careful tuning — values that were stable in a low-noise simulation would cause the synchronisation chain to lose lock when noise was increased. We worked through this by iteratively testing with the `channels_channel_model` block and observing the QPSK constellation diagram in the Qt GUI until we found parameters that held lock at realistic noise levels (noise voltage ~0.6).

### TDMA Wall-Clock Synchronisation

The TDMA scheduler uses Python's `time.time()` for slot timing, which works correctly within a single machine running the simulation. However, when extended to two separate physical nodes (two computers each connected to a BladeRF), both machines would need their system clocks synchronised to sub-millisecond precision for the 1.5 ms TDMA slots to align. We did not have an external synchronisation source available in the lab, so the TDMA mechanism was fully validated only in simulation. A proper multi-node deployment would require NTP or GPS-based clock discipline for reliable slot alignment.

### ACK Reliability in Simulation

The stop-and-wait ACK mechanism was difficult to make robust in the simulation environment. Because the ACK travels back through the same noisy simulated channel, it was frequently lost or corrupted, causing the sender to time out and retransmit packets unnecessarily. We addressed this by sending each ACK **5 times** in rapid succession from the receiver, which significantly reduced unnecessary retransmissions. We also tuned the `send_interval` timeout and `max_attempts` parameters to balance responsiveness against excessive retransmission overhead.

---

## Known Limitations

- The ACK mechanism was validated in simulation only; hardware testing was limited by the availability of two BladeRF units.
- The TDMA scheduler relies on wall-clock time (`time.time()`), which requires clock synchronisation between separate physical nodes for correct operation.
- File paths in the sender and sink blocks are hardcoded and must be updated to match the local machine before running.
- The BladeRF TX path requires SoapySDR and the BladeRF plugin to be installed; the simulation channel model can be used without hardware.

---

## Acknowledgements

This project was completed as part of the EN2130 Communication Design module, Semester 3, at the Department of Electronic & Telecommunication Engineering, University of Moratuwa. We thank our module lecturers and lab coordinators for providing BladeRF SDR hardware access and technical guidance throughout the project.
