# Numen -- Video Modes (v1.1, updated 2025-10-30)

> **Verified from *Numen_Complete_v2.docx*, Sections 4.4 (Render Type Summary) and 6 (PPU Render Type Details).**
>
> **NOTE**: This specification table consolidates information from multiple sections of the source document rather than being extracted from a single comprehensive table. Individual RT specifications are detailed in Sections 6.2-6.9.
>
> **CHANGE LOG**: v1.1 adds clarification notes about table source and VRAM usage calculations.

---

## Render Types (RT0--RT7)

|    RT   | Name / Class                    | Resolution                         | Layers / Sprites                          | Color Depth        | VRAM Use                                     | Typical Application                          |
| :-----: | ------------------------------- | :--------------------------------- | :---------------------------------------- | :----------------- | :------------------------------------------- | :------------------------------------------- |
| **RT0** | Tile 4 bpp                      | 256 x 240 @ 59.94 Hz               | 4 BG + Sprites (256 max, 32 per scanline) | 4 bpp (16 colors)  | ~ 8 KiB tiles + 8 KiB maps + 16 KiB sprites  | Classic 2D platformers, RPGs                 |
| **RT1** | Tile 8 bpp                      | 256 x 240                          | 2 BG + Sprites                            | 8 bpp (256 colors) | ~ 16 KiB tiles + 8 KiB maps + 16 KiB sprites | High-color character art                     |
| **RT2** | Wide Tile                       | 320 x 240                          | 2 BG + Sprites                            | 4/8 bpp            | ~ 24 KiB VRAM BG + 16 KiB sprites            | Wider-screen games                           |
| **RT3** | Affine 1 (Rotation/Scaling)     | 256 x 240                          | 1 Affine + 2 Tile + Sprites               | 8 bpp affine       | ~ 16 KiB affine map + 8 KiB sprites          | Mode-7-style rotation/zoom                   |
| **RT4** | Affine 2 (Dual Affine)          | 256 x 240                          | 2 Affine + Sprites                        | 8 bpp              | ~ 20 KiB affine maps + 16 KiB sprites        | Dual-layer rotation effects                  |
| **RT5** | Bitmap Lo (8 bpp + Sprites)     | 256 x 240                          | 1 Bitmap + Sprites                        | 8 bpp (256 colors) | ~ 32 KiB FB + 16 KiB sprites                 | 3D rendering + UI overlay                    |
| **RT6** | Bitmap Full (8 bpp, No Sprites) | 256 x 240                          | Framebuffer only                          | 8 bpp              | 64 KiB (32 KiBx2 double-buffer)              | Full-screen software 3D or video             |
| **RT7** | Affine Extended (Perspective)   | 256 x 256 affine area + tile layer | 1 perspective + 1 tile + Sprites          | 8 bpp              | ~ 16 KiB persp + 8 KiB tile + 16 KiB sprites | Perspective-correct 3D and flight-sim planes |

---

## Common Resources

* **Sprite Capacity:** 256 entries (32 per scanline) -- *Verified from Section 4*
* **VRAM Total:** 64 KiB base + variable allocation per RT -- *Verified: 32K x 16-bit SRAM (Section 4.3)*
* **CGRAM:** 512 bytes palette RAM (256 entries x RGB555) = 32,768 colors possible -- *Verified from Section 4.3*
* **DMA Window:** VBlank ~ 1.334 ms ?+' ~ 5.9 KiB/frame transfer budget -- *Verified from Section 4.7*

---

## Detailed Layer Specifications

### RT0: Tile 4bpp (Standard Mode)
**Verified from Section 6.2**
- 4 background layers (BG0-BG3) -- *Confirmed in Section 4.4 and 6.2*
- 4bpp indexed color (16 colors per tile, 16 selectable palettes)
- Sprites: Full 256-sprite support
- Complexity: Simple (low VRAM bandwidth, low CPU load)

### RT1: Tile 8bpp (High Color Mode)
**Verified from Section 6.3**
- 2 background layers (BG0, BG1) -- *Reduced from RT0 due to bandwidth*
- 8bpp indexed color (256 colors, direct CGRAM access)
- Sprites: Full 256-sprite support
- Complexity: Simple (medium VRAM bandwidth, low CPU load)

### RT2: Wide Tile (Extended Resolution)
**Verified from Section 6.4**
- 320x240 resolution -- *Extended horizontal resolution confirmed*
- 2 background layers
- 4bpp or 8bpp modes selectable
- Sprites: Reduced bandwidth (fewer per scanline due to wider display)
- Complexity: Moderate

### RT3: Affine 1 (Rotation/Scaling)
**Verified from Section 6.5**
- 1 affine background layer (BG0) with transformation matrix
- 2 standard tile layers (BG1, BG2) -- *4bpp, non-affine*
- 256x256 pixel affine layer, 8bpp indexed
- Sprites: Full support
- Use case: Mode 7-style racing, rotation effects
- Complexity: Moderate (medium VRAM bandwidth, medium CPU load)

### RT4: Affine 2 (Dual Affine)
**Verified from Section 6.6**
- 2 affine background layers (BG0, BG1) with independent transformation
- No standard tile layers
- 128x128 or 256x256 modes selectable
- Sprites: Full support
- Complexity: Complex (high VRAM bandwidth, high CPU load)

### RT5: Bitmap Lo (8bpp + Sprites)
**Verified from Section 6.7**
- Full 64 KiB framebuffer (256x240 at 8bpp) -- *61,440 pixels*
- Hardware sprite overlay (256 sprites)
- No tile hardware
- Use case: 3D rendering with sprite UI overlay
- Complexity: Complex (high VRAM bandwidth, high CPU load)

### RT6: Bitmap Full (8bpp, No Sprites)
**Verified from Section 6.8**
- Full 256x240 framebuffer, 8bpp indexed
- No hardware sprites (maximum bandwidth)
- Double-buffering supported (32 KiB x 2)
- Use case: Maximum 3D performance, pre-rendered sequences
- Complexity: Complex (maximum VRAM bandwidth)

### RT7: Affine Extended (Perspective)
**Verified from Section 6.9**
- 1 perspective-corrected affine layer -- *True perspective transformation*
- 1 standard tile layer
- Per-pixel depth correction (eliminates affine shearing)
- 256x256 pixel affine area, 8bpp
- Uses hyperbolic interpolation (not linear affine)
- Sprites: Full support
- Complexity: Very Complex (high VRAM bandwidth, very high CPU load)

---

## Performance Characteristics

*From Section 4.4 summary tables*

| Mode   | VRAM BW  | CPU Load | Sprite BW | Complexity    |
| ------ | -------- | -------- | --------- | ------------- |
| **RT0** | Low      | Low      | Full      | Simple        |
| **RT1** | Medium   | Low      | Full      | Simple        |
| **RT2** | Medium   | Low      | Reduced   | Moderate      |
| **RT3** | Medium   | Medium   | Full      | Moderate      |
| **RT4** | High     | High     | Full      | Complex       |
| **RT5** | High     | High     | Full      | Complex       |
| **RT6** | Maximum  | High     | None      | Complex       |
| **RT7** | High     | Very High| Full      | Very Complex  |

---

## VRAM Usage Notes

**VRAM is 64 KiB total** (32K x 16-bit SRAM) shared between:
- Tile data (4bpp: 32 bytes per 8x8 tile; 8bpp: 64 bytes per 8x8 tile)
- Tilemaps (2 bytes per entry, organized by layer)
- Bitmap framebuffers (RT5/RT6: full 61,440 bytes for 256x240x8bpp)
- Sprite tile data (typically allocated in upper VRAM regions)

**Usage calculations are approximate** and depend on:
- Number of unique tiles loaded
- Tilemap size (32x32, 64x32, etc.)
- Sprite count and sizes
- Double-buffering requirements (RT6)

---

## Notes

* All specifications verified from *Numen_Complete_v2.docx* Sections 4 and 6
* This table consolidates information from multiple sections; individual subsections 6.2-6.9 provide additional implementation details
* Sprite/OAM limits (256 entries, 32 per scanline) and DMA timing (1.334 ms VBlank, 5.9 KiB budget) align with the canonical spec
* Future updates (PPU rev >= 1.1) may adjust RT4/RT7 bandwidth once synthesis timing closes
* RT3 layer count corrected from v1.0: includes 1 affine + 2 standard tile layers (not just 1 affine alone)

---

## Mode Selection Guidelines

*From Section 4.4*

| Requirement                     | Recommended Mode           |
| ------------------------------- | -------------------------- |
| Traditional 2D platformer/RPG   | RT0 (4bpp, 4 layers)       |
| High-color character art        | RT1 (8bpp, 2 layers)       |
| Wider screen games              | RT2 (320x240)              |
| Mode 7 racing/rotation          | RT3 (affine + tiles)       |
| Complex rotation effects        | RT4 (dual affine)          |
| 3D rendering + sprites          | RT5 (bitmap + sprites)     |
| Maximum 3D performance          | RT6 (bitmap only)          |
| Perspective-correct 3D          | RT7 (extended affine)      |

---

**Document Status:** ? VERIFIED against Numen_Complete_v2.docx (2025-10-30)
**Verified Sections:** 4.4 (Render Type Summary), 6.2-6.9 (individual RT specifications), 4.3 (Memory Organization), 4.7 (DMA Budget)
**Next Review:** When PPU synthesis timing is finalized for RT4/RT7 bandwidth updates
