# Numen — CPU Address Map (v1.0, draft)

> Unified 64 KiB CPU-visible space (memory‑mapped I/O; little‑endian).

## 1) Top‑Level Address Ranges

| Address Range |    Size | Region        | Purpose                                                        |
| ------------- | ------: | ------------- | -------------------------------------------------------------- |
| 0x0000–0x1FFF |   8 KiB | WRAM          | Internal work RAM (zero‑wait); includes zero page and stack.   |
| 0x2000–0x3FFF |   8 KiB | PPU Registers | Video control, VRAM/OAM/CGRAM access (mirrored in this range). |
| 0x4000–0x41FF |   512 B | APU Registers | Hexsonic audio voice, mixer, HSX macro engine.                 |
| 0x4200–0x42FF |   256 B | System Ctrl   | IRQs, timers, watchdog, input (SNES‑style), reset/status.      |
| 0x4400–0x44FF |   256 B | DMA Engine    | Source/Dest/Length/Control, status, chaining.                  |
| 0x5000–0x50FF |   256 B | TPM Security  | Auth handshake, ROM power‑gate, tamper/lockout.                |
| 0x5800–0x5801 |     2 B | ROM Mapper    | BANK0/BANK1 select registers.                                  |
| 0x6000–0x7FFF |   8 KiB | SRAM          | Battery‑backed save RAM.                                       |
| 0x8000–0xBFFF |  16 KiB | ROM BANK0     | Primary switchable program/data window.                        |
| 0xC000–0xFFF9 | ~16 KiB | ROM BANK1     | Secondary switchable/fixed window.                             |
| 0xFFFA–0xFFFF |     6 B | Vectors       | NMI, RESET, IRQ vectors (fixed to last ROM page).              |

---

## 2) Memory‑Mapped I/O Blocks

### 2.1 PPU (0x2000–0x3FFF; mirrored)

* **PPU_CTRL** (0x2000): global enable, render‑type select (RT0–RT7), display on/off.
* **BG / Sprite / Palette Ports**: tilemap & scroll regs; OAM/CGRAM address/data ports with auto‑inc.
* **PPU_STAT** (~0x213F): status (VBlank flag, sprite overflow, etc.).
* **DMA hooks**: VRAM/OAM/CGRAM DMA triggers/config.

### 2.2 APU (0x4000–0x41FF)

* 8 voices, typically **32‑byte stride** starting at 0x4000 (0x4000, 0x4020, … 0x40E0).
* Per‑voice: sample base/length, rate, pan, gain, loop; voice enable.
* Global: **APU_VOL**, limiter/dither config, **APU_STAT**.
* HSX macro engine control/status.

### 2.3 System Control (0x4200–0x42FF)

* **IRQ_EN / IRQ_STAT**; **TMR_LO/HI** (frame timer); **WDT_CTRL**; **SYS_RST**.
* **PAD_LATCH / PAD_DATA0 / PAD_DATA1 / PAD_CFG / PAD_STAT** for controllers, lightgun, multitap.
* **SYS_CFG** exposes TPM_OK, DMA enable, etc.

### 2.4 DMA Engine (0x4400–0x44FF)

* **DMA_SRC, DMA_DST, DMA_LEN, DMA_CTRL, DMA_STAT**.
* Descriptor chaining via next‑ptr; VBlank/HBlank‑aware scheduling.

### 2.5 TPM Security (0x5000–0x50FF)

* **TPM_CMD, TPM_STAT, TPM_LOCK, TPM_FLAG**.
* Auth handshake (HMAC‑SHA256 prod / ECDSA‑P256 dev), ROM power gating, tamper/lockout.

### 2.6 ROM Mapper (0x5800–0x5801)

* **BANK0 @ 0x5800** → maps 16 KiB window at 0x8000–0xBFFF.
* **BANK1 @ 0x5801** → maps 16 KiB window at 0xC000–0xFFF9.

---

## 3) Cartridge / Banking Model

**Cartridge types** (bank index ranges for BANK0/BANK1):

* **ROM2 (1 MiB):** banks 0x00–0x3F (64 banks)
* **ROM4 (2 MiB):** banks 0x00–0x7F (128 banks)
* **ROM8 (4 MiB):** banks 0x00–0xFF (256 banks)

**Vectors are fixed** to the last physical ROM page and are *not* banked, guaranteeing deterministic RESET/IRQ/NMI.

---

## 4) Interrupt Vectors (fixed mapping)

| Address       | Vector | Description                   |
| ------------- | ------ | ----------------------------- |
| 0xFFFA–0xFFFB | NMI    | Frame/VBlank interrupt entry. |
| 0xFFFC–0xFFFD | RESET  | System entry point.           |
| 0xFFFE–0xFFFF | IRQ    | Maskable IRQ/BRK entry.       |

---

## 5) Access Timing (CPU perspective)

| Region | Cycles | Notes                                                |
| ------ | -----: | ---------------------------------------------------- |
| WRAM   |      1 | Zero wait‑state internal.                            |
| SRAM   |      2 | Latched access; battery‑backed.                      |
| VRAM   |      2 | Through PPU DMA/buffers (schedule in VBlank/HBlank). |
| ROM    |      3 | Depends on cart speed grade.                         |

**Bandwidth notes**

* During active scanlines the PPU owns most VRAM/OAM bandwidth; plan updates in **VBlank (~1.33 ms)**; small palette/sprite tweaks in **HBlank (~10.9 µs)**.
* Typical **VBlank DMA budget**: ~5.9 KiB/frame (tiles, OAM, palettes, tilemap slices).

---

## 6) Programming Notes

* Keep common/engine code in **BANK1** and level‑specific code/data in **BANK0**.
* Use **zero page (0x0000–0x00FF)** for hot variables and pseudo‑registers.
* Mapper/SCU/TPM registers are **write‑protected until `TPM_OK=1`**.
* Multi‑byte registers are **little‑endian**; mirrors in the PPU range simplify addressing.

---

### Appendix A — Quick Register Landmarks

* **PPU_CTRL** 0x2000 • **PPU_STAT** ~0x213F • **OAM_ADDR** ~0x2100 • **CGRAM_ADDR** ~0x2120
* **APU base** 0x4000; voices @ +0x00, +0x20, … +0xE0 • **APU_VOL** ~0x41E0
* **SCU** 0x4200.. • **DMA** 0x4400.. • **TPM** 0x5000.. • **BANK0/1** 0x5800/0x5801

> Draft is aligned to current spec; we’ll revise addresses/field names as sections 4/5/7/8/13 mature.
