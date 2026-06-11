# Hardware

Reference datasheet: [NVIDIA Jetson Orin NX Series Data Sheet](https://developer.download.nvidia.com/assets/embedded/secure/jetson/orin_nx/docs/Jetson_Orin_NX_DS-10712-001_v0.5.pdf?t=eyJscyI6InJlZiIsImxzZCI6IlJFRi1zdGF0aWNzLnRlYW1zLmNkbi5vZmZpY2UubmV0LyJ9&__token__=exp=1779966212~hmac=fa55451088e5027972fe78b56acd632856f3078e7f20a17fd67aa4157173c9d2)

This page focuses on the Jetson Orin NX 16GB hardware details that matter when the board is used for local LLM inference, voice ASR pipelines, PostgreSQL, NGINX, and network-attached edge workloads.

## Overview

| Area | Jetson Orin NX SUPER 16GB |
| --- | --- |
| CPU | 8-core Arm Cortex-A78AE, up to 2.0 GHz |
| GPU | NVIDIA Ampere GPU, 1024 CUDA cores, 32 Tensor Cores, up to 1173 MHz |
| Dedicated AI accelerators | 2x NVDLA, up to 614 MHz, up to 20 sparse INT8 TOPS each |
| Peak AI throughput | Up to 100 sparse INT8 TOPS, 50 dense INT8 TOPS |
| Memory | 16 GB LPDDR5, 128-bit |
| Memory bandwidth | Theoretical peak 102 GB/s |
| Storage on module | Boot flow uses QSPI; application storage is typically external |
| External storage | NVMe supported over PCIe x2 or x4 |
| Ethernet | 10/100/1000 GbE MAC on module |
| USB | USB type A x 4 |
| PCIe | 3x1 or 1x2 + 1x1, plus 1x4 Gen4 |
| Power modes | 10 W, 15 W, 25 W |
| Input voltage | 5 V to 20 V |

## Compute

### CPU

- 8 Arm Cortex-A78AE cores
- Up to 2.0 GHz


### GPU

- Ampere architecture
- 1024 CUDA cores
- 32 Tensor Cores
- Up to 1173 MHz
- Supports TF32, BF16, FP16, and INT8 inference-oriented execution paths
- Supports structured sparsity, which NVIDIA states can provide up to 2x higher inference performance for sparse models


### NVDLA

- 2x NVDLA engines
- Up to 614 MHz
- Up to 20 sparse INT8 TOPS each


## Memory and storage

### RAM

- 16 GB LPDDR5
- 128-bit bus
- Up to 3200 MHz memory frequency
- 102 GB/s theoretical peak bandwidth


### Storage

- External NVMe support over PCIe x2 or x4.
- QSPI as the secondary boot device.

## Networking

### Ethernet

- On-module 10/100/1000 Gigabit Ethernet MAC
- IEEE 802.3ab support

Important practical point: the module exposes the MAC, but the final Ethernet port behavior still depends on the carrier board design, PHY, magnetics, and thermal/power budget.

- 1 GbE can become the bottleneck if the box also streams raw media, synchronizes large model artifacts, or serves many clients concurrently.


### USB and PCIe

- USB 3.2 Gen2 support up to 10 Gbps
- Up to 3x USB 3.2 and 3x USB 2.0
- USB0 supports recovery/device mode; other ports are host-only
- PCIe Gen4 on all lanes/controllers
- Effective layout: 3x1 or 1x2 + 1x1, plus 1x4


## Media

### Camera input

- 8 lanes MIPI CSI-2
- D-PHY 2.1
- Aggregate bandwidth up to 20 Gbps
- Two 4-lane or four 2-lane configurations
- Support for virtual channels and data type interleaving

### Audio

- Dedicated programmable audio processor
- Arm Cortex-A9 with NEON
- PDM in/out
- HDA path to HDMI/DP
- AHUB supports I2S, DMIC, DSPK, mixing, SRC/ASRC, and low-power audio processing


## Power and thermal

- Supported input voltage: 5 V to 20 V
- Nominal power modes: 10 W, 15 W, 25 W
- SoC slowdown temperature: 99 C
- Junction operating range in datasheet: -25 C to 105 C

