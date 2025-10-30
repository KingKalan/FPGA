# Numen -- Audio Pipeline (v2.0, updated 2025-10-30)

> **Verified against *Numen_Complete_v2.docx*, Sections 7 (APU Overview), 8 (APU Registers), and 9 (Audio Formats NARR4).**
>
> **CHANGE LOG**: Updated from v1.0 to match current NARR4 ADPCM specification. Major changes:
> - Replaced PCM format descriptions with NARR4 4-bit ADPCM
> - Updated accumulator precision from 16-bit to 32-bit
> - Updated resampler from linear/sinc2 to 4-tap cubic Hermite
> - Clarified discrete sample rate support (11025, 16000, 22050 Hz)

---

## 1) High-Level Signal Path

```
ROM/NARR4 -> ADPCM decoder -> 4-tap cubic Hermite resampler -> per-voice gain/pan ->
   -> mix bus (L/R, 32-bit accumulator) -> soft-knee limiter (64-sample lookahead) ->
      -> TPDF dither -> DAC / PWM out (32 kHz)
```

* **Voices:** 8 simultaneous voices (Hexsonic 187-700 APU)
* **Codec:** NARR4 hardware decoder (4-bit ADPCM) providing 4:1 compression with near-transparent quality
* **Mix depth:** 32-bit fixed-point accumulators per channel (prevents overflow, maintains signal fidelity)
* **Nominal output rate:** 32 kHz (exactly 533.3 samples per video frame @ 59.94 Hz)
* **Clocking:** APU domain clock 2.39 MHz derived from PLL; asynchronous to CPU PHI2. Cross-domain via APU registers + DMA queues.

---

## 2) Voice Architecture

Each voice `V[i]` contains:

* **Source:** base address, length, format (NARR4 compressed), loop start/end, loop mode (off/forward/ping-pong).
* **Sample Rate:** Source material recorded at 11025 Hz, 16000 Hz, or 22050 Hz (discrete options).
* **Resampler:** 4-tap cubic Hermite interpolation resamples all input rates to unified 32 kHz output.
* **Gain/Pan:** per-voice linear gain (0.0-2.0 range), pan -1.0..+1.0 (equal-power law).
* **Per-voice control:** Sample address, length, rate, pan, gain.

**Per-voice register stride:** 0x20 (32 bytes) from APU base (0x4000), i.e. `V0 @ 0x4000`, `V1 @ 0x4020`, ... `V7 @ 0x40E0`.

---

## 3) Registers (summary; detailed bitfields in APU section)

**Global (approx. 0x41C0-0x41FF):**

* `APU_VOL_L/R` -- master volume (0x41E0)
* `APU_LIM` -- soft-knee limiter threshold (0x41E1)
* `APU_CFG` -- mixer rate config, default 32 kHz (0x41E2)
* `APU_STAT` -- flags: playing, underrun, clip, voice state, IRQ cause (0x41E3)
* `APU_IRQ_EN / APU_IRQ_STAT` -- buffer half/full, voice done, etc.
* `APU_KEY_ON / APU_KEY_OFF` -- bitmask (voices 0..7)

**Per-voice (offset +0x00..+0x1F):**

* `Vn_BASE` (24-bit), `Vn_LEN` (24-bit), `Vn_LOOP_START/END` (24-bit)
* `Vn_RATE` (sample rate selection: 11025 Hz / 16000 Hz / 22050 Hz)
* `Vn_GAIN`, `Vn_PAN`
* `Vn_CTRL` (enable, format, loop mode, key-on)

> Address anchoring: APU base is **0x4000** in the CPU map; stride and globals align with `address_map.md`.

---

## 4) Formats & Containers (NARR4)

* **NARR4**: Hardware-decoded 4-bit ADPCM audio format providing 4:1 compression with near-transparent quality
* **Container**: Stream container for compressed assets; supports banked ROM layout; includes loop points and per-asset metadata
* **Compression**: 4-bit ADPCM (IMA/DVI ADPCM variant) optimized for hardware decoding
* **Source rates:** Content can be authored at 11025 Hz, 16000 Hz, or 22050 Hz (discrete options)
* **Toolchain**: NABlab / NABber for asset conversion and packaging

**Alignment:** 4-byte aligned addresses are recommended for DMA efficiency.

**Quality:** Near-transparent compression; suitable for music and sound effects. 4:1 reduction vs. uncompressed PCM.

---

## 5) Resampling

* **Algorithm:** 4-tap cubic Hermite interpolation (hardware)
* **Purpose:** All source rates (11025/16000/22050 Hz) are resampled to unified 32 kHz output
* **Quality:** Superior to linear interpolation; reduced aliasing and improved frequency response
* **CPU Cost:** Minimal -- resampling is performed in hardware by the APU

---

## 6) Mixing, Headroom, and Limiting

* **Accumulation:** 32-bit per-channel accumulator prevents overflow and maintains signal fidelity
* **Master volume:** `APU_VOL_L/R` post-mix
* **Soft-knee limiter:** 64-sample lookahead prevents clipping and distortion; transparent gain reduction
* **Dither:** TPDF (Triangular Probability Density Function) dither with noise-shaping applied during bit-depth reduction for optimal signal-to-noise ratio before 16-bit DAC/PWM output

---

## 7) DMA & Buffering

* **Double buffer** per channel: two halves of `M` samples @ 32 kHz. Typical `M=512` (16 ms) or `M=256` (8 ms).
* **DMA descriptors** (in System DMA block @ `0x4400`) pull NARR4 compressed pages into APU stream buffers, ideally scheduled in VBlank to avoid VRAM contention.
* **IRQ:** APU asserts buffer half-empty/full interrupts (maskable) -> CPU refills via DMA or writes.

**Throughput guidance (32 kHz output, 8 voices with NARR4 4:1 compression):**

* Compressed read bandwidth ~ 32 kHz x 0.5 B (4-bit) x 8 ~ **128 KB/s** (much lower than uncompressed PCM)
* ROM/SRAM + DMA can easily handle this within VBlank windows

---

## 8) Timing & Latency

* **Audio-Video Sync:** 32 kHz audio output produces exactly 533.3 samples per video frame (32000 Hz ? 59.94 Hz), maintaining perfect audio-video sync without drift
* **I/O latency:** one buffer half (e.g., 8-16 ms typical)
* **Key-on latency:** <= 1 buffer; use pre-roll or small buffer for SFX
* **IRQ cadence:** at mix rate x buffer_half_size (e.g., 32 kHz x 256 -> 125 Hz)

---

## 9) Programming Model

1. Upload NARR4 assets to ROM; record base addresses.
2. For each voice: write `BASE/LEN/LOOP*`, select rate (11025/16000/22050 Hz); clear `key_on`.
3. Configure `APU_CFG` (confirm 32 kHz output), `APU_VOL`.
4. Set up DMA descriptors to feed streaming voices; enable APU IRQs.
5. On **half-buffer IRQ**: enqueue next compressed chunk (or update voice params for SFX).
6. For notes: write `KEY_ON` bit; optionally set gain and pan.

---

## 10) Test & Bring-Up Checklist

* **Sine sweep** (NARR4-encoded offline) on `V0` -> verify pitch & resampler quality.
* **Compression artifacts:** Verify NARR4 decode quality with various source material.
* **Pan law:** center vs hard-L/R amplitude check.
* **IRQ timing:** verify half-buffer cadence at selected buffer size.
* **Limiter:** drive hot mix to confirm soft-clip action and lookahead behavior.

---

## 11) Known Limits / Implementation Notes (v2.0)

* NARR4 decoding is hardware-accelerated with minimal CPU overhead.
* Cubic Hermite resampling provides high quality; no CPU cost.
* Limiter with 64-sample lookahead prevents clipping without audible pumping.
* All voices share the same 32 kHz output rate; source material resampled in hardware.

---

## 12) Differences from v1.0 Draft

**CRITICAL CHANGES:**

1. **Codec**: Changed from uncompressed PCM (u8/s8/u16/s16) to **NARR4 4-bit ADPCM**
2. **Accumulator**: Updated from 16-bit to **32-bit mixing accumulators**
3. **Resampler**: Changed from linear/sinc2 to **4-tap cubic Hermite** (hardware)
4. **Sample Rates**: Clarified as discrete options (11025/16000/22050 Hz) rather than arbitrary via phase stepping
5. **Compression Ratio**: 4:1 reduction in bandwidth vs. uncompressed PCM
6. **ADSR Envelopes**: *Not mentioned in v2.docx specifications* -- may be HSX macro engine feature or future enhancement

**v1.0 appears to have documented an earlier or alternative design that was replaced with the NARR4-based system.**

---

### Appendix A -- Example Voice Init (pseudo-register writes)

```
APU_KEY_OFF |= (1<<V0)
V0_BASE = 0x12_3400                    // NARR4 compressed asset address
V0_LEN  = 0x0001_F480                  // Length in compressed samples
V0_LOOP_START = 0x0000_2000
V0_LOOP_END   = 0x0001_C800
V0_RATE  = RATE_22050HZ                // Select 22050 Hz source rate
V0_GAIN  = 0.80
V0_PAN   = 0.00                        // Center
APU_CFG  = {mix=32k, limiter=ON, dither=TPDF}
APU_VOL_L= 0.90; APU_VOL_R= 0.90
APU_IRQ_EN |= BUF_HALF
APU_KEY_ON |= (1<<V0)
```

> This updated draft aligns with Numen_Complete_v2.docx Sections 7, 8, and 9. Register bitfields and complete HSX macro engine documentation to be added in future revisions.

---

**Document Status:** ? VERIFIED against Numen_Complete_v2.docx (2025-10-30)
**Verified Sections:** 7 (APU Overview), 8 (APU Registers), 9 (Audio Formats NARR4), plus system overview
**Next Review:** When Section 8 detailed register bitfields are finalized
