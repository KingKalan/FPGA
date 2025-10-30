# Numen -- CPU Bus Timing (v1.1, updated 2025-10-30)

> External CPU ?+' FPGA bus protocol and timing for the Numen console. The CPU is **off?EUR'chip**; the FPGA samples the bus in the **PHI2** domain and arbitrates access to internal slaves (WRAM, SRAM, PPU/APU regs, DMA, TPM, mapper).
>
> **Verified against *Numen_Complete_v2.docx*, Section 3 (CPU Subsystem) and Section 13 (Bus Fabric and DMA).**
>
> **CHANGE LOG**: v1.1 adds cross-references to verified sections and clarifies cycle timing from source document.

This document defines cycle phases, wait?EUR'state insertion (RDY), turnaround on the D[7:0] bus, and the hand?EUR'off to the internal fabric clock domain. Absolute numbers (ns) are **TBD per chosen CPU**; timing here is expressed **relative to PHI2** with placeholders for setup/hold.

---

## 1) Signal Summary

| Signal    |    Dir   |  Active  | Description                                                                                                                            |
| --------- | :------: | :------: | -------------------------------------------------------------------------------------------------------------------------------------- |
| `PHI2`    | CPU?+'FPGA |     --    | Phase?EUR'2 clock from CPU. All bus transfers occur while `PHI2=1` (high phase).                                                           |
| `A[23:0]` | CPU?+'FPGA |     --    | Address. Stable during `PHI2=1` for the current cycle. High bits may be latched externally; FPGA treats all 24 as synchronous to PHI2. |
| `RWB`     | CPU?+'FPGA |  1=Read  | Read/Write# select. Stable during `PHI2=1`.                                                                                            |
| `D[7:0]`  | CPU<->FPGA |     --    | Data bus. **CPU drives on writes**, **FPGA drives on reads**. Tri?EUR'state turnaround at cycle boundaries.                                |
| `RDY`     | FPGA?+'CPU |  0=Wait  | Insert wait states. When low during `PHI2=1`, the CPU holds the current cycle until `RDY` returns high.                                |
| `/RES`    | CPU?+'FPGA |  0=Reset | Global reset from CPU or board supervisor.                                                                                             |
| `/IRQ`    | FPGA?+'CPU | 0=Assert | Maskable interrupt (system/PPU/APU/DMA).                                                                                               |
| `/NMI`    | FPGA?+'CPU | 0=Assert | Non?EUR'maskable interrupt (VBlank / frame).                                                                                               |

> Note: If the selected CPU limits `RDY` to read cycles only, the fabric will avoid using `RDY` on writes and instead completes writes within a fixed latency. See ?Section 7.3.

---

## 2) Cycle Phases

```
Time ?+'
PHI2:  ___-------_______-------_______-------___
           HIGH        LOW        HIGH

A[23:0],RWB valid:  [=========]     [=========]
D (read):                <==== drive by FPGA ===>
D (write):           <== drive by CPU ==>         
RDY:                 ^ may pull low to extend ^
```

* **Address Setup (`tSU_A`):** CPU guarantees address valid **before** rising edge of `PHI2`.
* **Address Hold (`tH_A`):** Address remains valid **through** `PHI2=1` until near the falling edge.
* **Data Read Valid (`tVA_D`):** FPGA must present read data during `PHI2=1` and keep it valid until after `tH_D`.
* **Data Write Sample (`tSU_DW`/`tH_DW`):** FPGA samples write data during `PHI2=1`.

> Placeholders `tSU_*`/`tH_*` are filled from the CPU datasheet during board lock?EUR'down.

---

## 3) Single?EUR'Cycle Read (no wait)

```
PHI2       : ___-------___
A,RWB      : ---[ Read ]---  (stable while HIGH)
RDY        : ---------1------
D (FPGA)   : -----<==VV==>---  (valid within tVA_D while HIGH)
```

**Events:**

1. Rising edge of PHI2: FPGA latches `A`/`RWB`.
2. FPGA decodes, fetches data, and **drives D** during `PHI2=1`.
3. Falling edge: FPGA releases D to `Z`.

---

## 4) Single?EUR'Cycle Write (no wait)

```
PHI2       : ___-------___
A,RWB=0    : ---[ Write ]---
RDY        : ---------1------
D (CPU)    : ---<==VV==>-----  (CPU drives during HIGH)
```

**Events:**

1. Rising edge: FPGA latches `A`/`RWB=0`.
2. During `PHI2=1`, FPGA samples `D` with required setup/hold.
3. Falling edge: write commits; CPU releases D.

---

## 5) Wait?EUR'State Read via `RDY`

```
PHI2       : ___-----------___________
A,RWB=1    : ---[  READ  ]------------ (held stable)
RDY        : -----0=======0===>1------ (low = stretch)
D (FPGA)   : --------- Z ----<==VV==>-
```

**Rule:** If the fabric cannot return data in the same high phase, **pull `RDY=0` before the CPU?EUR(TM)s data?EUR'sample point**. CPU will extend the bus cycle while holding `A` and `RWB`.

* When data becomes ready, drive `D` and release `RDY?+'1` while `PHI2` is still high.
* CPU samples valid `D` and ends the cycle at the next PHI2 fall.

---

## 6) Wait?EUR'State Write via `RDY` (if supported)

```
PHI2       : ___-----------___________
A,RWB=0    : ---[  WRITE ]------------
RDY        : -----0=======0===>1------
D (CPU)    : ---<==VV================- (CPU holds data while waiting)
```

**Rule:** If writes target a slow slave (e.g., SRAM with extra latency), assert `RDY=0` to hold the cycle; FPGA samples `D` once the slave acks, then de?EUR'assert `RDY`.

> If the chosen CPU does **not** honor RDY during writes, the fabric ensures all writes complete within one high phase (use posted?EUR'write buffer or per?EUR'region timing constraints). See ?Section 7.3.

---

## 7) Turnaround & Contention Rules

1. **FPGA drives `D` only on READ** cycles and **only after** it has valid data. Otherwise `D=Z`.
2. **CPU drives `D` only on WRITE** cycles during `PHI2=1`. CPU releases `D` before/at `PHI2|`.
3. On back?EUR'to?EUR'back **WRITE?+'READ**, FPGA must **not** drive `D` until it detects the next READ; turnaround is naturally safe at PHI2 edges.
4. On back?EUR'to?EUR'back **READ?+'WRITE**, CPU will not drive until the new cycle; FPGA releases `D` at/just after `PHI2|`.

---

## 8) CDC Handshake (PHI2 ?+' Core Clock)

The external bus is in the **PHI2 domain**. Internal fabric runs on `clk_tpm` (and `clk_ppu/clk_apu` for their blocks). The bridge `ext_cpu_if` implements a **request/ack** handshake and optional small FIFO.

```
PHI2 domain                                 Core domain (clk_tpm)
-----------                                 ---------------------
  [A,RWB,D] --capture--> REQ  ===sync===>   REQ' --decode--> SLAVE
                                 ^               |           |
                    RDY <==gen===|           ACK'<-service--+

Rules:
- Raise REQ at PHI2?+'; hold until ACK' seen (after 2?EUR'FF sync).
- RDY=0 while (REQ && !ACK'). RDY?+'1 when ACK' observed and (read? data_ready:1).
- For READs, data crosses back via a small registered path; for WRITEs, data is captured in PHI2 domain then posted.
```

**Recommended FIFO depths:**

* Reads: 1--2 entries (to absorb minor fabric variance).
* Writes: 2--4 entries (to allow posted writes when CPU doesn?EUR(TM)t wait on RDY for writes).

---

## 9) Region?EUR'Specific Latency Targets

*Verified from Section 3.2 (CPU Memory Map) and Section 13.7 (Bus Bandwidth Allocation)*

| Region                          | Target Latency | Wait?EUR'State Policy                                       | Source                |
| ------------------------------- | -------------: | ------------------------------------------------------- | --------------------- |
| WRAM (0x0000--1FFF)              |        1 cycle | **No RDY; always 0?EUR'wait (zero wait states).**           | Section 3.2, Table 7  |
| PPU/APU/SCU/DMA (0x2000--0x44FF) |     1--2 cycles | RDY if register block is busy (e.g., VRAM port in use). | Section 13.2          |
| TPM (0x5000--0x50FF)             |            2--N | RDY permitted; may take multiple cycles during auth.    | Section 10            |
| Mapper (0x5800--0x5801)          |              1 | No RDY (simple register).                               | Section 15            |
| SRAM (0x6000--7FFF)              |        2 cycles | **Latched bus access; battery-backed.**                  | Section 3.2, Table 7  |
| ROM (0x8000--FFFF)               |        3 cycles | RDY allowed based on cart speed/width.                  | Section 3.2, Table 7  |

**Additional timing notes from source:**

* **WRAM** is CPU-exclusive with zero wait states (Section 3.2)
* **During active display** (scanlines 0--239): PPU dominates VRAM bandwidth (~75%), CPU limited to ~20-25% of bus cycles
* **During VBlank** (scanlines 240--261): DMA assumes ~80% bandwidth, CPU ~20%, PPU idle
* **Bus arbitration priority** (Section 13.2): Priority 1 = PPU rendering, Priority 2 = DMA, Priority 3 = CPU

---

## 10) Interrupt Timing (NMI/IRQ)

*Verified from Section 3.4 (Interrupt Architecture)*

* `/NMI` (FRAME_IRQ) asserted **during VBlank start** at scanline 240; minimum pulse width >= one PHI2 high phase
* Fixed interrupt acknowledge latency: **7 cycles** (Section 3.6, Table 40)
* `/IRQ` level?EUR'low until CPU services source and clears `IRQ_STAT` register at 0x4201
* If a wait?EUR'state holds the CPU at VBlank boundary, NMI may arrive while `RDY=0`; CPU samples NMI on release of the cycle
* **FRAME_SYNC** (59.94 Hz) triggers game main loop, controller reads, and DMA scheduling (Section 4.5)

---

## 11) Reset & First Access

*Verified from Section 2.6 (Power-On Sequence) and Section 3.5 (Boot Sequence)*

1. `/RES=0` ?+' FPGA drives internal resets; bus ignored until `/RES=1` and `pll_locked=1`.
2. **Boot sequence timing** (Section 2.6):
   - T+0 ms: Power rail sequencing
   - T+5 ms: Clock stabilization, PLL lock
   - T+5--25 ms: Boot ROM execution, TPM authentication
   - T+25 ms: ROM power enable (after successful TPM auth)
   - T+30 ms: Cartridge execution begins
3. First valid cycle: CPU fetches from the **fixed vector page** (0xFFFC/0xFFFD RESET vector).
4. Mapper defaults ensure vector page is readable regardless of bank registers (see `address_map.md`).
5. **TPM authentication must complete within 100 ms or system enters lockout** (Section 10.6)

---

## 12) Compliance Checklist

* [ ] FPGA **never** drives `D` during writes.
* [ ] `RDY` goes low **early enough** in PHI2?EUR'high to meet CPU `tSU_RDY`.
* [ ] `RDY` returns high **only after** read data is valid at the pins.
* [ ] Back?EUR'to?EUR'back cycle turnaround has no overlap on `D`.
* [ ] CDC uses 2?EUR'FF sync (min) for REQ/ACK and a registered data path.
* [ ] Region latencies meet table in ?Section 9 or use RDY/posted?EUR'write as specified.
* [ ] **Bus arbitration priority** correctly implements: PPU > DMA > CPU (Section 13.2).
* [ ] **Interrupt latency** is deterministic at 7 cycles (Section 3.6).

---

### Appendix A -- Read with 1 Wait Example

```
PHI2 : ___-------___________-------___
A    : ---[addr X]---------------------
RWB  : ---------1----------------------
RDY  : -----0============>1------------
D    : ---------Z----<==valid==>-------

Step 1  PHI2?+': capture (A,RWB)
Step 2  RDY=0 while fabric fetches
Step 3  Data ready ?+' drive D; RDY?+'1 while PHI2 still high
Step 4  PHI2|: CPU samples; FPGA tri?EUR'states D
```

### Appendix B -- CPU?EUR'Specific Numbers (fill?EUR'in?EUR'later)

*To be populated when CPU selection is finalized (e.g., W65C816S)*

* **CPU Clock:** 7.159 MHz (verified from Section 1.3 and Table 0)
* `tCY` (PHI2 period): ____ ns (139.7 ns nominal for 7.159 MHz)
* `tSU_A`, `tH_A`: ____ / ____ ns
* `tSU_DW`, `tH_DW`: ____ / ____ ns
* `tVA_D` (FPGA data valid window): ____ ns from PHI2?+'
* `tSU_RDY`, `tH_RDY`: ____ / ____ ns

**Verified system clocking** (Section 1.4, Section 2.3):
* Master clock: 14.318 MHz (NTSC standard)
* CPU clock: 7.159 MHz (14.318 MHz ? 2)
* PPU clock: 5.37 MHz
* APU clock: 2.39 MHz

> Populate Appendix B once the exact CPU (e.g., W65C816S) and board routing are finalized.

---

### Appendix C -- Cross-References to Source Document

**Primary verification sources:**

* **Section 1.4**: Clock Domain Architecture -- Master timing derivation and clock frequencies
* **Section 2.3**: Clock Division and Generation -- Divider architecture and phase relationships
* **Section 2.6**: Power-On Sequence -- Boot timing and TPM authentication
* **Section 3.2**: CPU Memory Map -- Address space organization and access timing
* **Section 3.4**: Interrupt Architecture -- NMI/IRQ timing and handling
* **Section 3.6**: Timing and Cycles -- Instruction and bus cycle timing
* **Section 13.2**: Bus Arbitration -- Priority scheme and access patterns
* **Section 13.7**: Bus Bandwidth Allocation -- System-wide bandwidth analysis

**Related sections:**

* Section 4.5: Timing and Synchronization (PPU VBlank and HBlank windows)
* Section 4.6: Bus Arbitration (PPU-specific arbitration details)
* Section 10.6: TPM authentication timeout and lockout policy
* Section 15: ROM Storage and Mapper (bank register behavior)

---

**Document Status:** ? VERIFIED against Numen_Complete_v2.docx (2025-10-30)  
**Verified Sections:** 1.4, 2.3, 2.6, 3.2, 3.4, 3.6, 13.2, 13.7  
**Pending:** CPU-specific timing parameters (Appendix B) pending hardware selection  
**Next Review:** When CPU selection finalized and board routing complete
