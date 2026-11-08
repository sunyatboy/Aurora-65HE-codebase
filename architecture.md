Main path (40 keys scanned by the G4):

DRV5055 sensor -> ADG708 mux -> G4 ADC -> SPI (x1 or x2) -> H7 core -> USB HID (8 kHz) -> host

Local path (28 keys scanned by the H7 itself):

DRV5055 sensor -> ADG706 mux -> H7 ADC -> H7 core

Side paths from the H7 core:

H7 core -> WS2812B-B chain (68 LEDs) via SN74LV1T25DBV level shifter
H7 core -> ER-TFT2.79-2 over SPI
IP2368 -> I2C -> H7 (fuel / charge status)


Block table

| Hall sensor (x68) | DRV5055 (variant ?) on 3.3VA | Magnetic field from the key magnet | An analog voltage (ratiometric to 3.3VA) |
| Mux (40 keys) | 5x ADG708 (8:1), one per G4 ADC | Select signals from the G4 | One key's voltage to that ADC |
| Mux (28 keys) | 2x ADG706 (16:1), one per H7 ADC | Select signals from the H7 | One key's voltage to that ADC |
| G4 ADC | STM32G474 (5 ADCs) | A voltage, referenced to 3.3VADC | A number (12-bit, 0-4095) per key |
| G4 scan + pack | STM32G474 | ADC numbers for 40 keys | Packed frames to the SPI link (format ?, filtering/oversampling ?) |
| SPI link G4 -> H7 | G4 slave / H7 master | Packed frames | Frames to the H7 core (clock ?, 1 or 2 lanes ?, DMA ?) |
| H7 ADC | STM32H7R3V8 (2 ADCs) | A voltage from ADG706 | A number per key (resolution ?) |
| Calibration / travel | STM32H7R3V8 | Raw numbers for all 68 keys | Travel per key, in mm or normalized (method ?) |
| Actuation FSM | H7 | The travel for each key | "key pressed / released" events |
| Classifier | H7 | ? | ? |
| Predictor (n-gram) | H7, model in XIP flash (about 60 MB) | The word / key sequence typed so far (context length ?) | The top 3 suggestions, or next-key probabilities (output ?) |
| Actuation policy | H7 | ? (predictor output + key state ?) | ? (per-key actuation / rapid-trigger thresholds ?) |
| Correction | H7 | ? | ? |
| USB HID | H7 (USB HS) | Press / release events | HID reports to the host at 8 kHz |
| UART link | H7 <-> nRF52840 | HID reports or events (format ?) | Bytes to the nRF (baud ?, flow control ?) |
| BLE HID | nRF52840 (Raytac MDBT50Q) | UART bytes | BLE HID (HOGP) reports to the host; 2.4 GHz mode ? |
| LED driver | H7 | Key state, effects, battery state | WS2812B-B data stream (timer/DMA method ?) |
| Display | H7 -> ER-TFT2.79-2 | UI frames (driver IC ?, resolution ?)  | Pixels over SPI |
| Power monitor | IP2368 -> H7 over I2C | Battery / charge state | Fuel level to the UI and BLE |

Parts and roles

| STM32G474 | Fast analog scanning of 40 keys | STM32Cube; crystal ? |
| STM32H7R3V8 | Compute, USB HID, LEDs, TFT, predictor | 8 MHz + 32.768 kHz crystals; owns the 64 MB XSPI flash |
| nRF52840 (Raytac MDBT50Q) | Bluetooth, maybe 2.4 GHz | Framework ? (Zephyr / nRF Connect SDK is the likely default) |
| DRV5055 x68 | Hall sensing | Variant ? (A1-A4 differ in sensitivity / range) |
| ADG708 x5, ADG706 x2 | Analog muxes | Used up exactly: 5x8 = 40 and 2x16 = 32 slots for 28 keys |
| WS2812B-B x68 | Per-key LEDs, no underglow | 5 V data via SN74LV1T25DBV from 3.3 V |
| ER-TFT2.79-2 | Status display | 12-pin; interface and driver IC ? |
| IP2368 | USB-C PD sink at 20 V, charger | Reports fuel to the H7 over I2C |
| BQ77915 | 3S protection and cell balancing | 3S1P LiPo |
| TVS Diode Array | USB ESD protection | |
| SWD headers (x3) | 3.3V, SWDIO, SWCLK, GND on every chip | No nRST pin in this order (see open items) |

Power tree

USB-C (20 V PD) -> IP2368 -> 3S LiPo (BQ77915) -> [ideal diode / power path ?] -> VSYS
VSYS -> 5.2 V rail (LEDs)            : Buck-Boost
5.2 V -> 3.3VD (MCUs, logic)         : LDO
5.2 V -> 3.3VA (hall sensors)        : High-PSRR LDO
3.3VA -> 1 uF feedthrough cap -> 100 nF feedthrough cap -> 3.3VADC (ADC reference)

Budget check: 68 LEDs at full white can theoretically pull about 4 A, above the 3 A buck limit. LED brightness needs a firmware current cap. The hall sensors add roughly 0.4-0.7 A on 3.3VA (check the DRV5055 supply current in its datasheet), and the 3.3VA LDO drops 1.9 V across itself, which is about 1 W worst case, so check thermals.

Open items

1. Mux settling: ADG708/ADG706 settling time against the ADC sample time at 256 kHz needs calculating.
2. Flash budget: a 60 MB model in a 64 MB flash leaves about 4 MB for code, TFT assets and config.
3. SWD header has no nRST: add one, or accept connect-under-reset limits.
4. G4 Crystal?
5. Classifier, actuation policy and correction: inputs, outputs and algorithms all ?
6. Framework and RTOS on the H7 and nRF: ? (STM32Cube is the only one decided).
