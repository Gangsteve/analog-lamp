# Dual-Channel Analog Wireless Voltage Link — TX/RX Conditioning

Two independent ASK/OOK RF links (315 / 433.92 MHz) carry two 0–3 V control voltages wirelessly (2–3 m, indoor). Each voltage is encoded as a tone frequency at the transmitter (V-to-F) and decoded back to DC at the receiver (F-to-V). Fully analog — no MCU. The TX and RX conditioning chains are built and verified in LTspice; the RF ICs (VG4455, WF480RA) are never SPICE-modeled — they are verified by breadboard.

```
LINK 1 (315 MHz)     0-3V → 555 VCO → VG4455 ──RF── WF480RA → squarer → LM331 F→V → LPF → 0-12V
LINK 2 (433.92 MHz)  same chain, second channel
```

---

## 1. TX behavioral explanation (per channel)

```
0-3V IN ── TLV9002 inverting stage ── 2N3906 current source ── C3 ── 555 (pins 2,6,7) ── F_OUT ── VG4455 D_in
```

- **Inverting stage (TLV9002):** R1/Rf/R2/R3 on a 3 V rail set the drive point `VCTRL = 2.08 − 0.10·Vin`, so the op-amp output falls linearly as Vin rises.
- **Current source (2N3906):** emitter pulled to VCC through Re1, base driven by the op-amp. It charges timing cap C3 with a constant current `I = (3 − VCTRL − Vbe)/Re1`, i.e. **linear in Vin** → the cap ramps linearly.
- **555 VCO (astable):** TRIG/THRS sit on C3; the 555 discharges C3 through R5 (fast, ~7 µs). Because the charge is constant-current and the discharge is negligible, `f ≈ I/(C3·ΔV)` — a **linear V-to-F**.
- **Output:** 555 pin 3 is a square wave whose frequency tracks Vin; it drives the VG4455 data input directly (both on the 3 V rail).

## 2. RX behavioral explanation (per channel)

```
ANT → WF480RA DATA out (recovered tone) → TLV9002 squarer → differentiator → LM331 F→V → 2-pole LPF → TLV9002 gain/offset → 0-12V OUT
```

- **Squarer (TLV9002 comparator):** slices the recovered tone at a mid-rail reference (1.71 V), producing clean edges — one per input cycle.
- **Differentiator (2.2 nF / 10 kΩ):** turns each falling edge into a trigger pulse for the LM331 pin 6.
- **LM331 F→V:** each trigger fires a one-shot of width `t = 1.1·Rt·Ct = 74.8 µs`; a switched current `i = 1.90/Rs = 325 µA` (Rs = R6+Rv) is integrated on RL//CL. Transfer: `Vout = i·t·f·RL = 2.42 mV/Hz` → 4.0–6.9 V over 1650–2850 Hz.
- **2-pole Sallen-Key LPF (fc ≈ 192 Hz):** removes the LM331 ripple (~0.5 V on FV → mV at OUT).
- **Gain/offset stage:** `OUT = 4.125·LP − 4.5·VREF`, VREF = 3.68 V → **0 V at 1650 Hz, 12 V at 2850 Hz**. Single 12 V supply, rail-to-rail op-amps.

---

## 3. Bill of Materials (both links)

| Ref | Part | LCSC # | Qty | ~$ ea |
|---|---|---|---|---|
| TX IC (×2) | **Vollgo VG4455** (ASK data-in, 1.8–5 V, 10 kbps) | C20539413 | 2 | 0.14 |
| RX IC (×2) | **WF WF480RA** (superhet, 2–5.5 V, 10 kbps, SOP-8) | C7434470 | 2 | 0.08 |
| TX crystal | 9.84375 MHz (315) / 13.56 MHz (433.92) | — | 2 | — |
| RX crystal | per WF480RA datasheet (9.81563 / 13.52127 MHz) | — | 2 | — |
| VCO | NE555 / LM555 (3 V-capable, e.g. CMOS) | — | 2 | — |
| Current source | 2N3906 (PNP) | — | 2 | — |
| Op-amp | **TLV9002** (rail-to-rail; TX stage + RX squarer/filter/gain) | — | 4 | — |
| F→V | LM331 | — | 2 | — |
| Antenna | 24 cm wire (315) / 17 cm (433.92) | — | 2 | — |

Crystals ±20 ppm, 20 pF load. RX output stage must swing rail-to-rail to 12 V → TLV9002/LMV358 (LM358 tops out ~3.5 V below rail).

---

## 4. Current specs & statistics (verified, LTspice)

### TX — V-to-F (3 V supply)

| Vin (V) | 0.000 | 0.333 | 0.667 | 1.000 | 1.333 | 1.667 | 2.000 | 2.333 | 2.667 | 3.000 |
|---|---|---|---|---|---|---|---|---|---|---|
| f (Hz) | 1635 | 1770 | 1905 | 2041 | 2175 | 2311 | 2447 | 2582 | 2716 | 2852 |

- Fit: `f = 405.8·Vin + 1634.8 Hz`; **max deviation 0.93 Hz (0.08% FS)**.
- Band **1.63–2.85 kHz**, inside the RX demod window.

### RX — F-to-V (12 V supply)

| f (Hz) | 1650 | 1783 | 1917 | 2050 | 2183 | 2317 | 2450 | 2583 | 2717 | 2850 |
|---|---|---|---|---|---|---|---|---|---|---|
| Vout (V) | 0.01 | 1.30 | 2.64 | 3.97 | 5.31 | 6.64 | 7.98 | 9.31 | 10.65 | 11.98 |

- Fit: `Vout = 9.99 mV/Hz·f − 16.506`; **max deviation 0.026 V (0.22% FS)**.
- Raw F→V gain 2.42 mV/Hz, output gain 4.125×; output pole RL·CL = 5 ms, LPF fc ≈ 192 Hz.

### Full chain

- **0–3 V in → 0–12 V out**, combined linearity ≈ **0.3% FS**.

---

## 5. Component values (as-built)

### TX (`GENERATED LTSPICE\rftx\tx.net` / `tx.asc`, 3 V)

| Ref | Value | Note |
|-----|-------|------|
| VSUPP | 3 V | timer must run at 3 V (e.g. CMOS 555 / MY555) |
| R1 | 100 kΩ | inverting stage input |
| Rf | 10 kΩ | feedback (swing ratio) |
| R2 / R3 | 3.3 kΩ / 5.6 kΩ | +input divider |
| Re1 | 20 kΩ | emitter → VCC, sets charge current |
| R5 | 1 kΩ | DIS → timing cap (fast discharge → linearity) |
| Rcv1 | 8.2 kΩ | pin 5 (CV) → GND |
| C3 | 15.5 nF | timing cap (sets absolute band) |
| C1 | 10 nF | pin 5 (CV) bypass |
| R7 | 10 kΩ | F_OUT load |
| Q2 | 2N3906 | PNP current source |
| U1 | TLV9002 | op-amp |
| U3 | NE555 | behavioral model `MY555.sub` |

### RX (`GENERATED LTSPICE\rfrx\RX.net` / `RX.asc`, 12 V)

| Ref | Value | Note |
|-----|-------|------|
| VCC | 12 V | supply |
| R1 / R2 | 90 kΩ / 15 kΩ | squarer REF = 1.71 V |
| C1 / R3 | 2.2 nF / 10 kΩ | input differentiator |
| R4 / R5 | 68 kΩ / 10 kΩ | pin 7 ref = 1.54 V |
| Rt / Ct | 6.8 kΩ / 10 nF | one-shot t = 74.8 µs |
| R6 / Rv | 3.65 kΩ / 2.2 kΩ | Rs = 5.85 kΩ → 325 µA |
| Rl / Cl | 100 kΩ / 50 nF | F→V output (5 ms pole) |
| R7 | 10 kΩ | FOUT pull-up (unused) |
| Rf1 / Rf2 / Cf1 / Cf2 | 150 k / 150 k / 22 n / 10 n | 2-pole LPF fc ≈ 192 Hz |
| Rg1 / Rg2 / Rg3 / Rg4 | 10 k / 30 k / 10 k / 45 k | gain 4.125 diff amp |
| Rva / Rvb | 24.9 kΩ / 11 kΩ | VREF = 3.68 V |
| U2 | LM331 | F→V |
| U1, U3, U4, U5 | TLV9002 | squarer, filter, VREF buffer, gain/offset |

TLV9002 is dual → 2 chips per channel (squarer+filter / VREF buffer+gain), 4 total for both links.