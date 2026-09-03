# Sensor Specifications & Geodetic Reference Mapping

## 1. Spaceborne Payload Technical Specifications

| Sensor Payload | Spacecraft / Mission | Modality & Spectral Bands | Spatial Resolution (GSD) | Swath Width / Coverage | Geodetic Control Source |
|---|---|---|---|---|---|
| **OHRC** | Chandrayaan-2 (ISRO) | Panchromatic (0.45 – 0.85 µm) | $0.25 - 0.32 \text{ m/px}$ | $12 \text{ km}$ strip | SPICE CK Kernels |
| **TMC-2** | Chandrayaan-2 (ISRO) | Stereo Panchromatic (3 view angles) | $5.0 \text{ m/px}$ | $20 \text{ km}$ strip | SPICE SPK/CK Kernels |
| **IIRS** | Chandrayaan-2 (ISRO) | Hyperspectral (0.80 – 5.00 µm) | $80.0 \text{ m/px}$ | $40 \text{ km}$ strip | PDS Label Geometry |
| **LRO NAC** | Lunar Reconnaissance Orbiter (NASA) | Panchromatic EDR / CDR | $0.50 - 2.00 \text{ m/px}$ | $5 \text{ km}$ strip | LOLA Laser Altimetry |
| **LRO WAC** | Lunar Reconnaissance Orbiter (NASA) | 7-Band UV/VIS Mosaic (643 nm) | $100.0 \text{ m/px}$ | Global Mosaic | LOLA Geodetic Grid |
| **SELENE TC** | Kaguya (JAXA) | Terrain Camera Stereo | $10.0 \text{ m/px}$ | Global Coverage | SELENE Altimetry |

---

## 2. Automated Source-to-Reference Spatial Selection

```
  Source Product Metadata (.xml / PDS Label)
                     │
                     ▼
  ISIS3 spiceinit ──► Compute Extents (lat_min, lat_max, lon_min, lon_max)
                     │
                     ▼
  Bounding Box Padding (2x to 5x Ephemeris Uncertainty: ~2 km)
                     │
                     ▼
  Reference Query ──► NASA Moon Trek WMTS API / Local LROC WAC Tile Crop
                     │
                     ▼
  GSD Reconciliation ──► Downsample Source / Upsample Reference to Pyramid Level
```

---

## 3. Geodetic Control Hierarchy

1. **Primary Control Network:** LOLA (Lunar Orbiter Laser Altimeter) crossover-corrected laser profiles define absolute lunar radius $R_{\text{moon}} = 1737.4 \text{ km}$.
2. **Reference Basemap:** LROC WAC 643 nm global mosaic (tied to LOLA control grid).
3. **High-Resolution Reference:** LROC NAC CDR strips cropped dynamically per OHRC footprint.
