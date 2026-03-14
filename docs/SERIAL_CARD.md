# Zx50 Serial I/O & Timer Card (CTC + DART)

## 1. Overview
This card provides the Zx50 system with its primary human-machine interface (via two RS-232 serial ports) and precision hardware timing. It combines a **Z80 DART** (Dual Asynchronous Receiver/Transmitter) for serial communications and a **Z80 CTC** (Counter/Timer Circuit) for programmable baud rate generation and system tick interrupts.

## 2. Core Components
* **Z80 DART (Z84C40):** Provides two independent full-duplex asynchronous serial channels (Channel A and Channel B). 
* **Z80 CTC (Z84C30):** Provides four independent 8-bit counter/timer channels. 
* **RS-232 Transceivers:** Two `MAX232` (or `MAX3232` / `SP232`) chips with their associated charge-pump capacitors to convert the DART's 5V TTL logic levels to standard ±10V RS-232 levels.
* **Bus Interface:** `74AHCT245` transceivers for data bus isolation, matching the Zx50 backplane standard.

## 3. The "Classic" Baud Rate Hack
Instead of using a dedicated baud rate generator chip, this board uses the classic Zilog design pattern: **Using the CTC to drive the DART.**
* **The Clock:** A dedicated `7.3728 MHz` crystal oscillator is fed into the CTC's clock inputs. (This specific "magic" frequency divides down perfectly into standard baud rates like 115200, 9600, etc.).
* **The Routing:** * CTC Channel 0 Output $\rightarrow$ DART Channel A `RxC` / `TxC` pins.
  * CTC Channel 1 Output $\rightarrow$ DART Channel B `RxC` / `TxC` pins.
* **Result:** The Z80 can dynamically change the baud rate of either serial port on the fly simply by writing a new divisor to the CTC registers!

## 4. Interrupt Priority & The Daisy Chain
Both the CTC and the DART heavily utilize **Z80 Mode 2 Vectored Interrupts**. Because they sit on the same physical card, we must hardwire their interrupt priority using the Zx50 Backplane's `IEI` (Interrupt Enable In) and `IEO` (Interrupt Enable Out) lines.

* **Routing Path:** `Zx50 Backplane IEI` $\rightarrow$ `CTC IEI` $\rightarrow$ `CTC IEO` $\rightarrow$ `DART IEI` $\rightarrow$ `DART IEO` $\rightarrow$ `Zx50 Backplane IEO`
* **Priority:** In this configuration, the CTC has a higher interrupt priority than the DART. If a timer tick and a serial character arrive at the exact same nanosecond, the CTC gets to vector the CPU first. 

## 5. Addressing Configuration
Both chips map exclusively to the Z80 **I/O Space**.
* **DART:** Requires 4 contiguous ports (driven by `C/~D` and `B/~A` pins).
* **CTC:** Requires 4 contiguous ports (driven by `CS0` and `CS1` pins).
* **Decoder:** A single `74AHCT138` (3-to-8 decoder) will slice the lower 8 bits of the address bus. For example:
  * `0x80 - 0x83`: Mapped to the CTC.
  * `0x84 - 0x87`: Mapped to the DART.

## 6. External Connectors
* **Serial Ports:** Two 2x5 (0.1" pitch) IDC headers that break out `TX`, `RX`, `RTS`, `CTS`, and `GND`. Standard 10-pin IDC to DB9-Male ribbon cables can be used to mount the physical serial ports to the rear chassis of the Zx50 case.