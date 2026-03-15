# Zx50 Serial I/O & Timer Card (CTC + DART)

## 1. Overview
This card provides the Zx50 system with its primary human-machine interface (via two serial ports) and precision hardware timing. It combines a **Z80 DART** (Dual Asynchronous Receiver/Transmitter) for serial communications and a **Z80 CTC** (Counter/Timer Circuit) for programmable baud rate generation and system tick interrupts.

## 2. Core Chip Manifest
1. **Z80 DART (Z84C40):** The dual serial controller.
2. **Z80 CTC (Z84C30):** The 4-channel timer (provides baud rate clocks to the DART and system ticks).
3. **74AHCT245:** Octal bus transceiver (Firewall for the `D0-D7` data bus).
4. **74AHCT138:** 3-to-8 line decoder (Address slicing for the Z80 I/O space).
5. **MAX232 (or MAX3232):** RS-232 level shifter charge-pump (Dedicated to Serial Port 1).
6. **7.3728 MHz Oscillator:** A standard 4-pin active crystal oscillator can (feeds the CTC's clock inputs).
7. **FTDI DB9-USB-D5:** Standalone USB-to-TTL module (Physical connector and level translator for Serial Port 0).
8. *(Optional Glue Logic):* `74AHCT32` or `74AHCT04` to qualify `~IORQ` and `~M1` for the address decoder.

## 3. The Output Stages (The Split Design)
* **Serial 0 (USB via FTDI):**
  * **Source:** Z80 DART Channel A.
  * **Connector:** FTDI DB9-USB-D5 module.
  * **Voltages:** 5V TTL logic. The DART's `TXA`, `RXA`, `~RTSA~`, and `~CTSA~` pins connect *directly* to the module's footprint.
* **Serial 1 (Classic RS-232):**
  * **Source:** Z80 DART Channel B.
  * **Connector:** Standard Right-Angle DB9 Female (PCB Mount).
  * **Voltages:** True RS-232 (±10V). The DART's `TXB`, `RXB`, `~RTSB~`, and `~CTSB~` pins route through the `MAX232` level-shifting IC before hitting the DB9.

## 4. Hardware Activity LEDs
To provide immediate visual feedback during data transmission, the board features hardware-driven activity LEDs:
* **Implementation:** High-efficiency LEDs are connected directly to the `TXA` and `TXB` output pins of the DART, pulled to ground through a 1KΩ or 2.2KΩ current-limiting resistor.
* **Result:** The LEDs will naturally flicker exactly when bytes are being shifted out of the UART, requiring zero software overhead or modem control line initialization.

## 5. The Baud Rate Engine (The CTC Hack)
Instead of using a dedicated baud rate generator chip, this board uses the classic Zilog design pattern: **Using the CTC to drive the DART.**
* **The Clock:** The `7.3728 MHz` crystal oscillator feeds the CTC's clock inputs (`CLK/TRG0` and `CLK/TRG1`). This specific frequency divides down perfectly into standard baud rates (115200, 9600, etc.).
* **The Routing:** * CTC Channel 0 Output (`ZC/TO0`) $\rightarrow$ DART Channel A `RxCA` / `TxCA` pins.
  * CTC Channel 1 Output (`ZC/TO1`) $\rightarrow$ DART Channel B `RxCB` / `TxCB` pins.
* **Result:** The Z80 can dynamically change the baud rate of either serial port on the fly by writing a new divisor to the respective CTC channel.

## 6. Interrupt Priority & The Daisy Chain
Both the CTC and the DART utilize **Z80 Mode 2 Vectored Interrupts**. Because they sit on the same physical card, their interrupt priority is hardwired using the Zx50 Backplane's `IEI` and `IEO` lines.
* **Routing Path:** `Backplane IEI` $\rightarrow$ `CTC IEI` | `CTC IEO` $\rightarrow$ `DART IEI` | `DART IEO` $\rightarrow$ `Backplane IEO`.
* **Priority:** The CTC has a higher interrupt priority than the DART. 

## 7. Addressing Configuration
Both chips map exclusively to the Z80 **I/O Space**.
* **DART:** Requires 4 contiguous ports (driven by `C/~D` and `B/~A` pins).
* **CTC:** Requires 4 contiguous ports (driven by `CS0` and `CS1` pins).
* **Decoder (74AHCT138):** Slices the lower 8 bits of the address bus. Assuming a base address of `0x80`:
  * `0x80 - 0x83`: Mapped to the CTC `~CE`.
  * `0x84 - 0x87`: Mapped to the DART `~CE`.