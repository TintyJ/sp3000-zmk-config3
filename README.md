# SP3000 Scanner Keypad — ZMK Firmware

Custom ZMK firmware for the Ceynetics MICRPAD BLE (CS2025A Rev.A), a Bluetooth/USB-C replacement keyboard for the Fuji Frontier SP3000 film scanner.

**Built for ZMK `main` branch (Zephyr 4.1 / HWMv2)** — December 2025+.

## Hardware Summary

| Component | Part | Notes |
|-----------|------|-------|
| MCU | E73-2G4M08S1C (nRF52840) | BLE 5.0 + USB-C |
| Matrix | 5 rows × 10 columns | row2col diodes (1N4148WS) |
| Switches | 43× Kailh MX hotswap | CPG151101S11-16 |
| Encoder | EC11E09244BS | Rotary + push, 80 steps |
| LEDs | 43× SK6812Mini-E | Via 74AHCT1G125 level shifter |
| Battery | BQ24075 charger + MAX17048 fuel gauge | TPS63020 buck-boost PSU |
| Crystal | 32.768 KHz | On XL1/XL2 (P0.00/P0.01) |

## GPIO Pin Mapping

Extracted from KiCad schematics by matching global label coordinates to E73 module pin positions.

### Matrix

| Signal | Pin | Devicetree | Signal | Pin | Devicetree |
|--------|-----|------------|--------|-----|------------|
| ROW0 | P1.00 | `&gpio1 0` | COL0 | P1.10 | `&gpio1 10` |
| ROW1 | P1.04 | `&gpio1 4` | COL1 | P0.28 | `&gpio0 28` |
| ROW2 | P1.06 | `&gpio1 6` | COL2 | P1.13 | `&gpio1 13` |
| ROW3 | P1.09 | `&gpio1 9` | COL3 | P0.29 | `&gpio0 29` |
| ROW4 | P1.11 | `&gpio1 11` | COL4 | P0.31 | `&gpio0 31` |
| | | | COL5 | P0.30 | `&gpio0 30` |
| | | | COL6 | P0.07 | `&gpio0 7` |
| | | | COL7 | P0.13 | `&gpio0 13` |
| | | | COL8 | P0.24 | `&gpio0 24` |
| | | | COL9 | P0.09 | `&gpio0 9` |

### Peripherals

| Signal | Pin | Notes |
|--------|-----|-------|
| ENC_A | P0.22 | Encoder channel A (high-freq pin ✓) |
| ENC_B | P0.20 | Encoder channel B (high-freq pin ✓) |
| LED_DATA | P1.02 | SPI MOSI → level shifter → SK6812 (⚠ low-freq pin) |
| I2C SDA | P0.02 | MAX17048 fuel gauge (high-freq pin ✓) |
| I2C SCL | P0.03 | MAX17048 fuel gauge (high-freq pin ✓) |

### nRF52840 Pin Frequency Notes

nRF52840 pins are classified as high-frequency (P0.02–P0.03, P0.11–P0.24) or low-frequency (all others). Low-frequency pins should not drive signals above 10 kHz or BLE radio reliability degrades. All matrix pins are fine (<10 kHz scan rate). The LED data pin P1.02 is low-frequency but driven at 4 MHz via SPI — this is a PCB design constraint. If BLE dropouts occur with LEDs active, disable RGB underglow.

## Switch-to-Matrix Map

Verified from KiCad KEY_MATRIX.kicad_sch wiring:

```
         C0    C1    C2    C3    C4    C5    C6    C7    C8    C9
  R0:   SW2   SW7   SW11  SW16  SW21  SW26  SW31  SW36  SW41  ---
  R1:   SW3   SW8   SW12  SW17  SW22  SW27  ---   SW37  ---   SW34
  R2:   SW4   SW9   SW13  SW18  SW23  SW28  SW33  SW38  SW43  SW35
  R3:   SW5   SW1   SW14  SW19  SW24  SW29  ---   SW39  ---   SW42
  R4:   SW6   SW10  SW15  SW20  SW25  SW30  ---   SW40  ---   SW44
```

Empty positions (7 of 50): R0C9, R1C6, R1C8, R3C6, R3C8, R4C6, R4C8

## Zephyr 4.1 / HWMv2 Compliance

This config follows the ZMK Zephyr 4.1 migration requirements (Dec 2025):

- **HWMv2 directory layout**: `boards/ceynetics/` vendor dir (not `boards/arm/`)
- **`board.yml`**: New board metadata file with SoC declaration
- **`Kconfig.sp3000_keypad`**: Renamed from `Kconfig.board`; uses `select` not `depends on`
- **No SOC selection in defconfig**: Removed `CONFIG_SOC_*` and `CONFIG_BOARD_*` lines
- **NFC pins as GPIO via devicetree**: `&uicr { nfct-pins-as-gpios; }` (not Kconfig)
- **Boot retention in SRAM**: Required for `&bootloader` behavior in Zephyr 4.1
- **`CONFIG_WS2812_STRIP` removed**: Auto-enabled by LED strip driver
- **DC/DC mode via devicetree**: Commented out pending PCB verification
- **MAX17048 uses ZMK-namespaced driver**: `zmk,maxim-17048` (not upstream `maxim,max17048`)

## Keymap Status

**Current: PLACEHOLDER keycodes.** Based on SP3000 manual (PP3-B1271E2 §2.3).

## Next Steps

### 1. Capture original keycodes (CRITICAL)

```bash
# On any Linux box with the original keyboard plugged in:
sudo evtest /dev/input/eventX
# Press every key, record the scancodes, update sp3000_keypad.keymap
```

### 2. Verify DC/DC inductor

Check if your PCB has an inductor on the nRF52840 DCC pin. If yes, uncomment `&reg1` in the DTS for better power efficiency.

### 3. Verify SPI dummy pins

P0.05 (SCK) and P0.26 (MISO) are assigned as unused SPI pins. Confirm these aren't routed to anything on your PCB.

### 4. Build and flash

```bash
# GitHub Actions: push repo, UF2 builds automatically
# Local: west build -b sp3000_keypad -- -DZMK_CONFIG=/path/to/config
```

### 5. Pin ZMK version for production

Once working, pin to a release tag in `west.yml` (e.g. `revision: v0.4`) instead of tracking `main`. See: https://zmk.dev/blog/2025/06/20/pinned-zmk

## File Structure

```
config/
├── boards/ceynetics/sp3000_keypad/
│   ├── board.yml                  # HWMv2 board metadata (NEW)
│   ├── sp3000_keypad.dts          # Devicetree (GPIO, peripherals)
│   ├── sp3000_keypad.keymap       # Key bindings (EDIT THIS)
│   ├── sp3000_keypad_defconfig    # Kconfig defaults
│   ├── sp3000_keypad.zmk.yml      # ZMK board metadata
│   ├── Kconfig.sp3000_keypad      # Board Kconfig (renamed for HWMv2)
│   └── board.cmake                # CMake config
├── west.yml                       # West manifest → ZMK main
build.yaml                         # GitHub Actions build config
```
