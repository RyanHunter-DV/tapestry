# HDMI Source (TX) IP Controller — Architectural Brief

## 1. Purpose & Scope

This document defines the architecture for a typical **HDMI Source (Transmitter) IP controller** targeting FPGA/ASIC integration. The IP accepts digital video and audio streams from the host system, encodes them into an HDMI-compliant TMDS stream, and outputs serialized high-speed data to the physical layer for transmission to an HDMI sink (TV/monitor).

**Reference implementations consulted:**
- `hdl-util/hdmi` — Open-source SystemVerilog HDMI 1.4b TX for FPGA 
- `magic-gates/hvtx` — Compact HDMI TX core (video-only) for Arora FPGA 
- Intel HDMI RX components documentation — I²C/DDC/SCDC architecture reference 
- Xilinx HDMI 1.4/2.0 TX Subsystem — AXI-based video/audio interfaces, HDCP cascade topology 

---

## 2. Top-Level Block Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          HDMI SOURCE (TX) IP CONTROLLER                              │
│                                                                                      │
│  ┌──────────────┐    ┌────────────────────────────────────────────────────────────┐  │
│  │  HOST / SoC  │    │                    HDMI TX CORE                            │  │
│  │  INTERFACE   │    │                                                            │  │
│  │              │    │  ┌──────────────┐    ┌──────────────────────────────────┐  │  │
│  │ Video AXI4-S ├───►│  │ Video Timing │    │        TMDS ENCODER              │  │  │
│  │  (pixel data)│    │  │  Generator   ├───►│  ┌──────────┐  ┌──────────────┐  │  │  │
│  │              │    │  │ (HSYNC/VSYNC │    │  │ 8b/10b   │  │  TERC4       │  │  │  │
│  │ Audio AXI4-S ├───►│  │  /DE Gen)    │    │  │ Encoder  │  │  Encoder     │  │  │  │
│  │  (PCM/ACR)   │    │  └──────────────┘    │  │ (Video)  │  │ (Data Island)│  │  │  │
│  │              │    │                      │  └────┬─────┘  └──────┬───────┘  │  │  │
│  │ Control      │    │  ┌──────────────┐    │       │               │          │  │  │
│  │ AXI4-Lite ├───►│  │  Data Island │    │       └───────┬───────┘          │  │  │
│  │  (registers) │    │  │   Packet     ├───►│              │                  │  │  │
│  └──────────────┘    │  │   Builder    │    │       ┌──────▼──────┐          │  │  │
│                      │  │ (Audio/Info- │    │       │  TMDS MUX   │          │  │  │
│  ┌──────────────┐    │  │  Frame/ECC)  │    │       │ (3 channels)│          │  │  │
│  │  DDC/EDID    │    │  └──────────────┘    │       └──────┬──────┘          │  │  │
│  │  CONTROLLER  │    │                      │              │                  │  │  │
│  │              │    │  ┌──────────────┐    │       ┌──────▼──────┐          │  │  │
│  │ I²C Master ├───►│  │  HDCP Engine │    │       │ SERIALIZER  │          │  │  │
│  │  (EDID read) │    │  │ (optional)   ├───►│       │   (OSERDES) │          │  │  │
│  │              │    │  └──────────────┘    │       └──────┬──────┘          │  │  │
│  │ HPD Monitor  │    │                      │              │                  │  │  │
│  └──────────────┘    │  ┌──────────────┐    │              │                  │  │  │
│                      │  │  CEC         │    │              │                  │  │  │
│                      │  │  Controller  │    │              │                  │  │  │
│                      │  │  (optional)  │    │              │                  │  │  │
│                      │  └──────────────┘    │              │                  │  │  │
│                      │                      │              │                  │  │  │
│                      └──────────────────────┴──────────────┼──────────────────┘  │
│                                                            │                      │
│  ┌─────────────────────────────────────────────────────────┼──────────────────┐  │
│  │                    PHYSICAL LAYER INTERFACE              │                  │  │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │                  │  │
│  │  │ TX PHY  │ │ TX PHY  │ │ TX PHY  │ │ TX PHY  │◄──────┘                  │  │
│  │  │ Ch0 (B) │ │ Ch1 (G) │ │ Ch2 (R) │ │ Ch3(Clk)│                          │  │
│  │  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘                          │  │
│  └───────┼───────────┼───────────┼───────────┼───────────────────────────────┘  │
└──────────┼───────────┼───────────┼───────────┼──────────────────────────────────┘
           ▼           ▼           ▼           ▼
     ┌─────────────────────────────────────────────────┐
     │           HDMI CONNECTOR (Type-A)                │
     │  Pins 1-3    Pins 4-6    Pins 7-9    Pins 10-12 │
     │  (TMDS D2)   (TMDS D1)   (TMDS D0)   (TMDS CLK) │
     │                                                  │
     │  Pin 13: CEC    Pin 15: SCL    Pin 16: SDA      │
     │  Pin 18: +5V    Pin 19: HPD                     │
     └─────────────────────────────────────────────────┘
```

---

## 3. Major Unit Components

### 3.1 Video Timing Generator (VTG)

**Function:** Generates all video timing signals (HSYNC, VSYNC, Data Enable, and blanking intervals) based on the configured video format .

**Key Behavior:**
- Accepts pixel data from the AXI4-Stream slave interface
- Generates DE (Data Enable) to distinguish active video from blanking periods
- Outputs timing parameters to the TMDS Encoder for determining Video Data Period vs. Data Island Period vs. Control Period
- Supports standard video ID codes (VIC) defined by CEA-861 

**Typical Parameters (720p60 example):**
| Parameter | Value |
|-----------|-------|
| Active Width | 1280 |
| Active Height | 720 |
| H Front Porch | 110 |
| H Sync Width | 40 |
| H Back Porch | 220 |
| V Front Porch | 5 |
| V Sync Width | 5 |
| Pixel Clock | 74.25 MHz |

---

### 3.2 TMDS Encoder

**Function:** Converts 8-bit video and auxiliary data into 10-bit TMDS characters using transition-minimized differential signaling .

**Key Sub-Blocks:**

| Sub-Block | Encoding | Purpose |
|-----------|----------|---------|
| **8b/10b Encoder** | 8-bit → 10-bit | Active video pixels (RGB) |
| **TERC4 Encoder** | 4-bit → 10-bit | Data Island packets (audio, InfoFrames) |
| **Control Encoder** | 2-bit → 10-bit | HSYNC/VSYNC during blanking |

**Behavior:**
- During **Video Data Period**: 8b/10b encoding for RGB pixel data
- During **Data Island Period**: TERC4 encoding for packet payloads
- During **Control Period**: Control characters carrying HSYNC/VSYNC state 
- Uses running disparity to maintain DC balance and minimize transitions 

#### video data encoding
- [x] TMDS video encoding
1. count_ones of input_data > 4 or (is 4 and input_data\[0\] is 0), then use xnor, else use xor
```
q_m[0] = D[0]
q_m[1] = q_m[0] xnor/xor D[1]
q_m[2] = q_m[1] xnor/xor D[2]
...
q_m[8] is 1 for XOR operation; is 0 for XNOR operation
```
2. if accumulated_disparity is 0 or count of 1 and 0 are the same in q_m
	accumulated disparity is balanced.
```
q_out[7:0] = q_m[8] ? q_m[7:0] : ~q_m[7:0]
q_out[8] = q_m[8]; q_out[9] = ~q_m[8];
```
and if `q_m[8]` is 0, count_disparity = previous_count_disparity + (current zeros of `q_m[7:0]` - current ones)
if `q_m[8]` is 1, count_disparity = previous_count_disparity + (current ones of `q_m[7:0]` - current zeros)

3. (previous ones > zeros and current ones > zeros) or (previous zeros > ones and current zeros > ones)
```
q_out[7:0] = ~q_m[7:0]
q_out[8] = q_out[8]
q_out[9] = 1
count_disparity = previous_count_disparity + 2*q_m[8] + current ones - current zeros
```

else ...
9 is 0, bit 8 is bit 8, bit 7:0 is bit 7:0
count is previous count - 2*(~bit 8) + ones - zeros

#### 



---

### 3.3 Data Island Packet Builder

**Function:** Constructs HDMI auxiliary packets (Audio Sample, Audio InfoFrame, AVI InfoFrame, etc.) and inserts them into Data Island periods .

**Packet Structure (per HDMI spec):**
```
┌─────────────────────────────────────────────────────────────┐
│  PACKET HEADER (4 bytes)                                     │
│  ┌──────────────┬──────────────┬──────────────┬───────────┐ │
│  │  Byte 0      │  Byte 1      │  Byte 2      │  ECC      │ │
│  │ (Packet Type)│ (Version)    │ (Length)     │ (BCH)     │ │
│  └──────────────┴──────────────┴──────────────┴───────────┘ │
├─────────────────────────────────────────────────────────────┤
│  PACKET BODY (28 bytes)                                      │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Subpacket 0 (7 bytes) + ECC (1 byte)                │   │
│  │  Subpacket 1 (7 bytes) + ECC (1 byte)                │   │
│  │  Subpacket 2 (7 bytes) + ECC (1 byte)                │   │
│  │  Subpacket 3 (7 bytes) + ECC (1 byte)                │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

**Packet Types Handled:**
- **Audio Sample Packet**: Carries L-PCM or compressed audio samples 
- **Audio InfoFrame**: Describes audio stream parameters (sample rate, channels, bit depth)
- **AVI InfoFrame**: Describes video format (RGB/YCbCr, aspect ratio, color depth) 
- **Audio Clock Regeneration (ACR)**: Enables sink to reconstruct audio clock
- **Generic/Null packets**: Fill remaining Data Island space

---

### 3.4 Serializer (OSERDES)

**Function:** Converts parallel 10-bit TMDS characters into high-speed serial bitstreams for each lane .

**Key Characteristics:**
- **Serialization Ratio**: 10:1 (10-bit parallel → 1-bit serial)
- **Clock Domain**: Runs at 5× pixel clock (DDR) or 10× pixel clock (SDR)
- **FPGA Implementation**: Uses OSERDES/SelectIO primitives; ASIC uses custom SERDES
- **Clock Channel**: Dedicated lane serializes the pixel clock itself (TMDS clock)

**Timing Example (720p60):**
| Signal | Frequency |
|--------|-----------|
| Pixel Clock | 74.25 MHz |
| TMDS Bit Clock | 742.5 MHz (10× pixel) |
| Serial Clock (DDR) | 371.25 MHz |

---

### 3.5 DDC / EDID Controller

**Function:** Reads the sink's Extended Display Identification Data (EDID) via the I²C-based Display Data Channel to discover supported video/audio formats .

**Key Behavior:**
- Acts as **I²C Master** on SCL (Pin 15) and SDA (Pin 16)
- I²C slave address: **0xA0/0xA1** for EDID read
- Triggers on **HPD** (Hot Plug Detect) assertion 
- Parses EDID to determine:
  - Supported resolutions and refresh rates
  - Preferred timing (native resolution)
  - Audio format capabilities
  - Color depth and HDR support
- Optional: SCDC (Status and Data Channel) at 0xA8/0xA9 for HDMI 2.0+ scrambling control

---

### 3.6 HDCP Engine (Optional)

**Function:** Implements HDCP 1.4 or 2.2/2.3 content protection for encrypted transmission .

**Key Components:**
- **Authentication and Key Exchange (AKE)**: RSA-based handshake with sink
- **AES-128 Encryption**: Encrypts video/audio data path
- **Key Management**: Secure storage/loading of device keys and SRM
- **State Machine**: Must be polled regularly (≤20ms) to meet HDCP deadlines 

**Cascade Topology (HDCP 1.4 + 2.2):**
The HDCP block sits between the packet builder and TMDS encoder. Unencrypted AV data exits the core, gets encrypted by the active HDCP block, and re-enters for transmission .

---

### 3.7 CEC Controller (Optional)

**Function:** Implements the Consumer Electronics Control single-wire bus for device interoperability (e.g., one-touch play, remote control passthrough).

**Key Behavior:**
- Bidirectional open-drain bus with 27kΩ pull-up to 3.3V
- Bit-level timing: 1.5ms per bit, with arbitration and retransmission
- Used for system-level commands, not video/audio data 

---

## 4. Data Flow Summary

| Domain | Source | Path | Destination |
|--------|--------|------|-------------|
| **Video** | SoC AXI4-Stream | VTG → 8b/10b Encoder → TMDS MUX | Serializer → PHY |
| **Audio** | SoC AXI4-Stream | Packet Builder → TERC4 Encoder → TMDS MUX | Serializer → PHY |
| **InfoFrames** | SoC AXI4-Lite (registers) | Packet Builder → TERC4 Encoder → TMDS MUX | Serializer → PHY |
| **Control** | HPD/EDID | DDC Controller (I²C) | SoC software |
| **CEC** | SoC AXI4-Lite | CEC Controller | Pin 13 |

---

## 5. Interface Summary

| Interface | Type | Description |
|-----------|------|-------------|
| **Video Input** | AXI4-Stream Slave | Pixel data (RGB888 or YCbCr) |
| **Audio Input** | AXI4-Stream Slave | 32-bit PCM audio samples |
| **Control** | AXI4-Lite Slave | Register access for configuration |
| **DDC** | I²C Master | EDID/SCDC communication |
| **HPD** | GPIO Input | Hot Plug Detect |
| **CEC** | GPIO Bidirectional | Consumer Electronics Control |
| **TMDS Output** | Differential Pairs | 4 lanes (3 data + 1 clock) |

---

## 6. Design References from Open-Source Code

| Feature | hdl-util/hdmi | magic-gates/hvtx |
|---------|---------------|------------------|
| **Language** | SystemVerilog | SystemVerilog |
| **License** | MIT/Apache-2.0  | — |
| **Audio Support** | Yes (L-PCM, 2-8ch)  | No (video-only)  |
| **Data Island** | Yes (Audio, AVI, ACR)  | No |
| **Modularity** | Single core + demo | 4 modules (cursor, sync, mod, ser)  |
| **Target** | Altera, Xilinx, Gowin | Arora FPGA |
| **LUT Usage** | — | <200 LUTs, <200 FFs  |

The `magic-gates/hvtx` design is particularly instructive for understanding the **minimal viable HDMI TX pipeline**, while `hdl-util/hdmi` provides a complete reference including audio and InfoFrame packet generation .