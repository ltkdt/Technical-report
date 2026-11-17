# CSI-Bench: WiFi CSI Sensing and Hardware Brief

Compiled from the paper *CSI-Bench: A Large-Scale In-the-Wild Dataset for Multi-task WiFi Sensing* (Zhu, Hu, Gao, Wang, Wang, Liu — Origin Research, NeurIPS 2025 Datasets and Benchmarks Track).

## Question

What is WiFi CSI, how is it obtained, what sensing tasks can it support, and what hardware can be used to collect it — as presented in the CSI-Bench paper.

## Executive Summary

1. **WiFi CSI** (Channel State Information) captures fine-grained temporal and spatial variations of the WiFi wireless channel. When a person moves within the propagation field, their body reflects and scatters the signals, imprinting detectable patterns onto CSI. Commodity WiFi devices can therefore infer human-centric phenomena passively, contactlessly, and without cameras or wearables, making it attractive for privacy-preserving health monitoring.
2. **How it is obtained in CSI-Bench**: IoT client devices periodically transmit CSI "sounding" packets to routers (100 Hz for general sensing, 30 Hz for breathing). Routers, running firmware/drivers with CSI extraction enabled (provided in collaboration with chipset vendors), extract, buffer, and upload CSI to cloud storage (AWS S3). Streams are synchronized in software using per-device system clocks and packet-level timestamps.
3. **CSI format varies by chipset**: e.g., NXP 88W8997 gives 2×2 MIMO with 58 subcarriers at 40 MHz/5 GHz; ESP32-S3 gives 1×1 with 64 subcarriers at 20 MHz/2.4 GHz; Qualcomm IPQ4019/4018 give 1×2 MIMO with 128 subcarriers at 40 MHz and 256 at 80 MHz (5 GHz); Broadcom BCM4345 gives a 1×4 antenna setup with only 14/28 subcarriers at 20/40 MHz due to proprietary subcarrier grouping.
4. **Tasks supported**: fall detection, breathing monitoring, room-level localization, motion source recognition (human vs pet vs robot vs fan), human activity recognition, user identification, and proximity estimation — the last three co-labeled in a multi-task dataset.
5. **Hardware used**: 16 commercial device types spanning Qualcomm, Broadcom, Espressif, MediaTek, and NXP chipsets, from smart plugs and smart speakers to routers; 802.11 n/ac/ax, 2.4 GHz and 5 GHz bands, 20/40/80 MHz bandwidths, 1×1 to 2×2 MIMO and 1×4 antennas.
6. **Scale**: 461+ hours of CSI from 35 users, 26 environments, 16 device configurations, 231.6k samples across 7 tasks.
7. **Key caveat**: models generalize poorly out-of-distribution (cross-device, cross-environment, cross-user), motivating research on domain-adaptive, calibration-free learning.

## Methodology

- Primary source: the CSI-Bench paper itself (uploaded to this project's library), read in full including Appendix A (dataset description and hardware Table 7).
- Findings below are drawn directly from the paper's Sections 1–5 (introduction, dataset collection, CSI extraction/format, benchmark tasks) and Appendix A.3 (devices and hardware diversity), plus task-specific appendices A.5–A.9.
- Limitation: no independent cross-checking against external sources was performed; this brief reflects the paper's own claims.

## Findings

### 1. What is WiFi CSI and how is it obtained?

**Concept.** WiFi signals between IoT devices (smart speakers, smart plugs, routers, mesh hubs) carry rich information about the surrounding environment through their propagation paths. CSI is a fine-grained physical-layer descriptor of the WiFi channel: for each subcarrier frequency f at time t, the channel response H(f, t) is measured, capturing how human motion perturbs the multipath field. Gross motor events such as falls and subtle physiological signals such as breathing all leave characteristic imprints on the CSI's amplitude and phase over time and subcarriers.

**Why it is attractive.** Compared to vision-, audio-, or wearable-based systems, WiFi sensing is:
- Contactless and passive — no cameras, microphones, or user-worn devices;
- Privacy-preserving — no imagery of the user;
- Continuous and non-intrusive — operates on infrastructure that is already deployed;
- Cost-effective — reuses existing home WiFi devices.

**How CSI-Bench obtains it.**
1. IoT client devices periodically transmit CSI packets to routers at two sounding rates: **100 Hz** for general sensing tasks and **30 Hz** for breathing detection (slower breathing dynamics).
2. The routers run vendor-provided firmware and drivers with CSI extraction enabled, plus proprietary CSI capture utilities; the authors built custom tools for Linux and FreeRTOS routers that send commands from the application layer to the WLAN kernel module to collect and buffer CSI from all registered devices into unified binary files.
3. Files are periodically uploaded to cloud storage (AWS S3), with filenames timestamped by the router's local clock; the router coordinates batch requests so distributed devices record within the same time window, and streams are later aligned in software using per-device system clocks and embedded packet-level timestamps.
4. Labels come from a lightweight Google-Spreadsheet annotation tool (users tap Start/End buttons to log activities and room occupancy), time-aligned camera recordings, and sensor logs for non-human motion.
5. Preprocessing: because phase is corrupted by timing/frequency synchronization offsets and varies unreliably across platforms, CSI-Bench uses **amplitude-only CSI** |H(f, t)|, normalized per sample, with subcarrier standardization (zero-padding/clipping to a fixed subcarrier count). Data is segmented into 5-second samples (10 seconds for breathing).

### 2. What tasks can WiFi CSI sensing do?

CSI-Bench supports **seven classification tasks** across single-task (specialist) and multi-task (co-labeled) datasets:

**Single-task specialist datasets**
- **Fall detection** (2 classes: fall vs non-fall) — 6.7k samples, 17 users, 6 environments, 2 device types; annotated with synchronized video. Healthcare/safety use.
- **Breathing detection** (binary, during sleep, 30 Hz) — 100k samples, 3 users, 3 environments, 6 device types; passive overnight recording for vital-sign/wellness monitoring.
- **Motion source recognition** (4 classes: human, pet, robot, fan) — 60.9k samples, 35 users, 10 environments; useful to reduce false alarms in home-care/security systems.
- **Room-level localization** (6 classes: which room the user is in) — 7.1k samples, 8 users, 6 environments, 8 device types; single- and multi-user occupancy.

**Multi-task joint dataset (co-labeled)**
- **Human activity recognition** (5 classes: walking, running, jumping, seated breathing, hand waving) — 41.5k samples.
- **User identification** (6 users) — 20.3k samples.
- **Proximity recognition** (4 distance classes: 0.5, 1.5, 2.5, 3.5 m) — 20.3k samples.

All tasks use accuracy and weighted F1 as metrics, with 70/15/15 splits, Easy/Medium/Hard difficulty tiers, and out-of-distribution (cross-user, cross-environment, cross-device) evaluation. The paper's related-work section also notes the broader WiFi sensing literature covers gesture detection, gesture writing, indoor tracking, and vital-sign monitoring.

### 3. Hardware specifications for obtaining CSI data

**Overall requirements.** Devices are commercial, off-the-shelf WiFi edge devices whose chipset vendors enabled CSI extraction in firmware/drivers. Key parameters: chipset vendor/model, antenna configuration (MIMO), channel bandwidth, frequency band, subcarrier count, and sampling rate.

- **Standards:** IEEE 802.11 n/ac/ax
- **Bands:** 2.4 GHz and 5 GHz
- **Bandwidths:** 20, 40, and 80 MHz
- **Antennas:** 1×1 up to 2×2 MIMO, and 1×4 setups
- **Sounding rates:** 100 Hz (general tasks) and 30 Hz (breathing)
- **Chipset vendors represented:** Qualcomm, Broadcom, Espressif, MediaTek, NXP

**Device-by-device specifications (paper's Table 7):**

| Device | Chipset vendor | Chipset model | Antenna | Bandwidth | Band | Avg RSSI |
| --- | --- | --- | --- | --- | --- | --- |
| Amazon Plug | MediaTek | MT7697N | 1×1 | 20 MHz | 2.4G | -49.95 dBm |
| Govee Plug | Espressif | ESP8266/ESP8285 | 1×1 | 20 MHz | 2.4G | -50.26 dBm |
| Wyze Plug | Espressif | ESP8266/ESP8285 | 1×1 | 20 MHz | 2.4G | -55.57 dBm |
| Eightree Plug | Espressif | ESP8266/ESP8285 | 1×1 | 20 MHz | 2.4G | -54.01 dBm |
| Echo Plus | MediaTek | MT8516 | 1×1 | 20/40/80 MHz | 2.4G & 5G | -42.24 dBm |
| Google Nest | Qualcomm | IPQ4019 | 1×1 | 20/40 MHz | 2.4G & 5G | -49.41 dBm |
| Apple HomePod | - | - | 1×1 | 20/40 MHz | 2.4G & 5G | -52.56 dBm |
| Echo Spot | MediaTek | MT6625L | 1×1 | 20/40 MHz | 2.4G & 5G | -51.20 dBm |
| Echo Show 8 | MediaTek | MT8183 | 1×1 | 20/40/80 MHz | 2.4G & 5G | -38.14 dBm |
| Echo Dot 2nd gen | MediaTek | MT6625LN | 1×1 | 20/40 MHz | 2.4G & 5G | -76.83 dBm |
| Echo Dot 3rd gen | MediaTek | MT7658CSN | 1×1 | 20/40/80 MHz | 2.4G & 5G | -57.75 dBm |
| Hex Home | Qualcomm | - | 1×2 | 20/40 MHz | 5G | - |
| HealthPod | NXP | 88W899 | 2×2 | 20/40/80 MHz | 5G | - |
| ESP32 | Espressif | S3 | 1×1 | 20/40 MHz | 2.4G | -62.86 dBm |
| Google Nest Hub | Broadcom | BCM4345 | 1×1 | 20/40/80 MHz | 2.4G & 5G | -60.39 dBm |
| Lyra | Qualcomm | - | 2×2 | 20/40/80 MHz | 2.4G & 5G | - |

(RSSI values are empirical averages from one-hour static indoor sessions; "-" = not reported in the paper.)

**CSI format differences across chipsets:**
- **NXP 88W8997**: 2×2 MIMO, 58 subcarriers, 40 MHz on 5 GHz; used for fall detection and motion source recognition at 100 Hz (each TX-RX pair forms 4 spatial links). The paper also cites a related "HealthPod" entry (2×2, 20/40/80 MHz, 5G).
- **ESP32-S3**: 1×1, 64 subcarriers, 20 MHz on 2.4 GHz; a cheaper, lower-quality CSI source used in the "hard" fall-detection split.
- **Qualcomm IPQ4019/IPQ4018**: 1×2 MIMO, 128 subcarriers at 40 MHz and 256 subcarriers at 80 MHz on 5 GHz.
- **Broadcom BCM4345**: 1×4 antenna configuration, only 14/28 subcarriers at 20/40 MHz due to proprietary subcarrier grouping.

**Contrast with classic research hardware.** Earlier datasets almost universally used the Intel 5300 NIC (with Atheros CSI Tool in MM-Fi), which cannot record CSI continuously; data had to be captured in fragmented, manually triggered, pre-scripted sessions. CSI-Bench instead relies on commercial edge devices that support continuous recording under natural daily activity, which is what enables the 461-hour, in-the-wild scale.

## Source Notes

| Source | Credibility | Last updated |
| --- | --- | --- |
| [CSI Bench.pdf](doclib://01a1e253-6e75-7060-9506-17dd41d626cb/f6f05eb0-1421-4f4b-b429-2cb3f715fd08) (paper under review, NeurIPS 2025 arXiv:2505.21866v2) | 5/5 | Nov 2025 |

- All findings come from the uploaded paper itself; no external sources were needed or used.
- Caveats: RSSI for four devices (Hex Home, HealthPod, Lyra) and two chipset models are not disclosed ("-"). The Apple HomePod's chipset is not identified. The HealthPod's NXP model is written as "88W899" in Table 7 while the text elsewhere references "88W8997"; the 88W8997 is likely the intended part.

## Open Questions

- Exact chipset details for Apple HomePod, Hex Home, and Lyra routers are not disclosed in the paper.
- Whether CSI-Bench's amplitude-only choice can be revisited: the authors note phase instability across platforms as the reason, and calibrated-phase or angle-of-arrival methods remain unexplored on this dataset.
- The benchmark covers classification only; regression tasks (e.g., continuous breathing-rate estimation) and long-term activity tracking are future extensions.

## Recommendations / Next Steps

- If you plan your own CSI collection, prioritize devices with vendor-enabled CSI extraction (Qualcomm IPQ-series, NXP 88W8997, ESP32/ESP32-S3, Broadcom BCM4345 families) and match bandwidth/band to the task: 5 GHz / 40–80 MHz / 100 Hz for motion tasks; 30 Hz for breathing.
- Use the CSI-Bench dataset and code (linked in the paper) as a baseline before collecting new data, and evaluate against its cross-device/environment/user OOD splits.
- For model development, transformer backbones with lightweight task adapters (LoRA) were the strongest and most parameter-efficient baselines in the paper.