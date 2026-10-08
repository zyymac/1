# SC500 MIPI Camera → ISP → HDMI 720P FPGA Project

This project is originally developed on **Anlogic PH1P35MDG324** FPGA (Tang Dynasty IDE v6.2).
It captures video from an **SC500 MIPI CSI-2 camera** (2-lane, RAW10), runs hardware ISP
(BLC → AWB → Demosaic → RGB888), and outputs **HDMI 720p60** via TMDS.

## System Block Diagram

```
SC500 Camera (MIPI CSI-2, 2-lane)
        │  MIPI D-PHY RX (4 data lanes + clk lane)
        ▼
  csi_unpacket_2lane / raw10_unpacket_2lane   ← CSI-2 packet decode, RAW10 unpack
        │  AXI4-Stream video (40-bit)
        ▼
  isp_top:
    ├── BLC (black level correction)
    ├── AWB (auto white balance, uses FIFO + divider IP)
    └── demosaic_4x_2_0 (Bayer → RGB via bilinear interpolation)
        │  RGB888 video stream
        ▼
  video_out / hdmi_mixer / hdmi_tx  ← VTC timing + TMDS encoder
        │
        ▼
  HDMI TX (TMDS 4 data + 1 clock pair) → Monitor
```

Auxiliary:
- `uics500_cfg/` — I2C master + SC500 register init table (720p60 mode)
- `mc_to_user_interface.v` — DDR2 memory controller user interface (frame buffer)
- `pll_hdmi.v` — PLL wrapper for pixel clock generation

## Directory Structure

```
├── rtl/                          # Verilog / SystemVerilog source
│   ├── design_top_wrapper.v      # ★ TOP LEVEL — start here
│   ├── video_in.v / video_out.v
│   ├── hdmi_tx.v / hdmi_mixer.v
│   ├── isp/                      # ISP pipeline (pure logic, portable)
│   │   ├── isp_top.v
│   │   ├── BLC/BLC.v
│   │   ├── awb/awb.v
│   │   ├── demosaic_4x_2_0/     # Bayer→RGB demosaic
│   │   ├── data128_96/, data96_128/
│   │   └── csi_unpacket_2lane.v, raw10_unpacket_2lane.v
│   ├── mipi_dphy_rx/             # ⚠ Anlogic PHY wrappers (vendor-bound)
│   ├── uics500_cfg/              # SC500 camera I2C config
│   ├── vtc/uivtc.v               # video timing generator
│   └── ...
├── constraints/
│   ├── pin.adc                   # Anlogic pin assignment
│   └── camera_to_dsi_display.sdc # clock constraints
└── project/
    └── camera_to_dsi_display.al  # Anlogic TD project file (file list)
```

## Ports (design_top_wrapper)

| Port | Dir | Description |
|---|---|---|
| `I_sys_clk` | in | 33 MHz oscillator |
| `I_rst_n` | in | global reset, active low |
| `O_cam_scl`, `IO_cam_sda` | out/inout | camera I2C config |
| `O_cam_24m`, `O_cam_rst` | out | camera 24 MHz clock, reset |
| `IO_rx_clk_pad_p/n` | inout | MIPI RX clock lane |
| `IO_rx_data_pad_p/n[3:0]` | inout | MIPI RX data lanes |
| `O_tmds_ch0/1/2_p`, `O_tmds_clk_p` | out | HDMI TMDS output |
| `ddr_*` | inout/out | DDR2 frame buffer |

## What to Replace When Porting to Another FPGA

The following modules are **Anlogic PH1P-vendor-specific** and must be re-generated
using the target FPGA vendor's tools:

1. **MIPI D-PHY RX** — `mipi_dphy_rx/ph1p_mipiio_rx_wrapper.v`,
   `mipi_dphy_rx/mipi_dphy_rx_ph1p_mipiio_wrapper.sv`
   (encrypted `*.enc.v`/`*.enc.sv` PHY binaries are NOT included)
2. **DDR2 Controller** — `ph1p35_ddr/` (not included; vendor IP)
3. **PLL** — `pll_hdmi.v` wraps Anlogic PLL IP (not included)
4. **HDMI TX PHY** — `hdmi_phy_warpper.v`, `lane_lvds_10_1.v`
   (TMDS serializer; encrypted core `hdmi_1_4b_transmitter_core_wrapper.enc.v` not included)
5. **FIFO / BRAM / Divider IP** — generated via Anlogic IpTool (not included)

The **pure logic** modules under `rtl/isp/`, `rtl/uics500_cfg/`, `rtl/vtc/`,
`rtl/video_in.v`, `rtl/video_out.v`, `rtl/hdmi_tx.v`, `rtl/hdmi_mixer.v`
are vendor-independent and can be reused directly.
