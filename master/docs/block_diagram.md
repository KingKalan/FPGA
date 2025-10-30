# Numen — Block Diagram (v1.0, draft)

> High‑level architecture for the Lattice ECP5 (LFE5U‑45F‑6BG381C) design with **external CPU**. Includes main data/control paths, clock/reset domains, and I/O.

---

## Mermaid (source):

```mermaid
flowchart LR
  subgraph EXT[External Components]
    CPU[Discrete CPU\n(PHI2, RWB, A[23:0], D[7:0],\n/RES, /IRQ, /NMI, RDY)]
    CART[Cartridge\n(ROM / Mapper Contacts)]
    PAD[Controllers\n(Latch/Clock/Data)]
    VIDOUT[Video Encoder/DAC\n(Composite/Component)]
    AUDOUT[Audio DAC/PWM]
  end

  subgraph FPGA[ECP5 LFE5U‑45F]
    CR[Clock/Reset\n(EHXPLLL + POR + Syncs)]

    subgraph BUS[CPU Bus & System Fabric]
      EXTIF[ext_cpu_if\n(PHI2 domain capture, RDY)]
      FAB[bus_fabric\n(addr decode, arbiters)]
      DMA[DMA Engine\n(src/dst/len/ctrl, VBlank aware)]
      SCM[System Ctrl (SCU)\n(IRQs, timers, pads)]
      TPM[TPM / Security / Mapper\n(auth, ROM power‑gate, bank regs)]
      WRAM[WRAM\n(Work RAM)]
      SRAM[SRAM (Battery)\n(Save RAM)]
    end

    subgraph PPU[PPU / Video]
      PREG[PPU Regs]
      VRIF[VRAM IF]
      OAM[OAM]
      CGRAM[CGRAM/Palette]
      PIPE[Render Pipeline\n(scan, sprites, blend)]
    end

    subgraph APU[APU / Audio]
      AREGS[APU Regs]
      V0[Voice 0]
      V1[Voice 1]
      V2[Voice 2]
      V3[Voice 3]
      V4[Voice 4]
      V5[Voice 5]
      V6[Voice 6]
      V7[Voice 7]
      MIX[Mixer + Dither/Limiter]
    end
  end

  %% CPU bus into FPGA
  CPU -- PHI2,RWB,A,D --> EXTIF
  EXTIF --> FAB
  FAB <--> WRAM
  FAB <--> SRAM
  FAB <--> SCM
  FAB <--> TPM
  FAB <--> DMA
  FAB <--> PREG
  FAB <--> AREGS

  %% DMA side paths
  DMA --> VRIF
  DMA --> OAM
  DMA --> CGRAM
  DMA --> WRAM
  DMA --> AREGS

  %% PPU outputs
  VRIF --> PIPE
  CGRAM --> PIPE
  OAM --> PIPE
  PIPE --> VIDOUT

  %% APU outputs
  V0 --> MIX
  V1 --> MIX
  V2 --> MIX
  V3 --> MIX
  V4 --> MIX
  V5 --> MIX
  V6 --> MIX
  V7 --> MIX
  MIX --> AUDOUT

  %% Inputs
  PAD --> SCM
  CART --> TPM

  %% Clocks / resets
  CR -. clk_ppu/clk_apu/clk_tpm .-> PPU
  CR -. clk_ppu/clk_apu/clk_tpm .-> APU
  CR -. clk_tpm .-> BUS
  CR -. resets .-> FPGA
```

---

## ASCII Overview (fallback)

```
           +-------------------+          +---------------------+
           |   Controllers     |          |  Cartridge (ROM)    |
           |  Latch/Clk/Data   |          |  + Bank Contacts    |
           +---------+---------+          +----------+----------+
                     |                               |
                     v                               v
             +-------+--------+               +------+------+
             |   System Ctrl  |<--bus-------> |   TPM/Map   |
             | (IRQs, Timers, |               | Auth/Bank   |
             |  Pad Interface)|               | ROM Gate    |
             +---+---------+--+               +------+------+
                 ^         |                         |
                 |         |                         |
                 |         v                         v
+---------+  PHI2|   +-----+------+   fabric   +-----+------+
|  CPU    |<-----+-->|  ext_cpu_if |<--------->|  bus_fabric |
| (ext.)  | A,D,RWB  +------------+           +--+----------+
| /RES    |  /IRQ/NMI         ^                   |      |
+----+----+                    |                   |      |
     |                         |                   |      |
     |                         |                   |      |
     |                    +----+----+              |      |
     |                    |  WRAM   |<-------------+      |
     |                    +---------+                     |
     |                                                  +-+--+
     |                                                  |SRAM|
     |                                                  +----+
     |
     |   clocks/resets      +---------------------+       video out
     +--------------------->|  PPU (Regs/VRAM/    |-----> DAC/Encoder
                            |  OAM/CGRAM/Pipeline)|
                            +---------------------+

                                     audio out
                            +---------------------+
                            |  APU (Voices/Mixer) |-----> DAC/PWM
                            +---------------------+
```

---

## Domains & Handshakes

* **PHI2 domain:** external CPU bus sampled in `ext_cpu_if`; wait‑states via `RDY` until `bus_fabric` acknowledges.
* **Core domains:** `clk_tpm` (system/bus), `clk_ppu`, `clk_apu` from `EHXPLLL`. CDC FIFOs/handshakes isolate domains.
* **DMA scheduling:** VBlank/HBlank aware for VRAM/OAM/CGRAM; audio buffers refilled by half‑buffer IRQ cadence.

---

## Notes

* This diagram aligns with the `address_map.md` and `audio_pipeline.md` drafts.
* Board‑level connectors (video encoder type, pad ports, cart header) are placeholders until pinout/LPF is finalized.
