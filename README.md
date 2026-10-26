# Multi-Factor Access Control & Security System (MF-ACSS)

A multi-stage physical access control and perimeter security project developed on the **Arduino Mega 2560 (ATmega2560)**.

The system combines two-factor user authentication (RFID token + 4x4 matrix keypad PIN) with perimeter monitoring and emergency overrides. To maintain predictable timing and prevent blocking delays (`delay()`), the dual 7-segment display multiplexing, status LED animations, and time-critical inputs are handled using **direct AVR register manipulation (`DDRA`, `PORTA`)** and **hardware external interrupts (`INTx`)**.

---

## How It Works (Workflow)

1. **Standby (Idle):** The system waits for an RFID card. In the background, an 8-LED sweep animation runs on Port A via bit-shifting, and the display indicates an idle status.
2. **RFID Scan:** The RC522 reader scans the badge over SPI. If the UID matches an authorized entry, the 16x2 LCD screen prompts the user to enter their PIN.
3. **PIN Entry:** The user enters a 4-digit PIN on the 4x4 matrix keypad, masked on-screen with `*`. A 15-second timeout resets the attempt if entry is abandoned.
4. **Access Granted:** If both credentials match:
   * The relay actuates the door solenoid strike for 5 seconds.
   * The dual 7-segment display (SSD) runs a countdown (`05` $\rightarrow$ `00`).
5. **Lockdown & Tamper Detection:**
   * Entering 3 consecutive invalid PINs or tripping the door reed switch triggers an alarm state.
   * A PWM-driven buzzer sounds an alert, and new badge scans are temporarily rejected.

---

## Hardware Implementation & Registers

* **SSD Control via Direct Registers (`PORTA`, `DDRA`):**
  * Tens and units digits share the same segment bus lines and are time-multiplexed using a precomputed BCD Look-Up Table (`ssdlut[]`).
  * Segments ($a..g$) are wired to `PA0`–`PA6`, while digit cathode switching is driven by `PA7`. The display updates in a single clock cycle using bitwise operations (`PORTA = ssdlut[pos] | outvalue`).
* **Debouncing & External Interrupts:**
  * The emergency override button and the magnetic tamper switch are routed to hardware interrupt pins (`INT4` on Pin 2 and `INT0` on Pin 21).
  * Mechanical switch bounce is filtered out inside the ISR by comparing timestamps via `millis()`.
* **LCD Interface:**
  * The 16x2 LCD runs in 4-bit parallel mode to minimize GPIO pin usage.

---

## Pinout & Wiring

| Peripheral | Arduino Mega Pin | AVR Register / Pin | Function |
| :--- | :--- | :--- | :--- |
| **SSD Segments (a..g)** | Pins 22 – 28 | `PORTA0` – `PORTA6` | Segment data output |
| **SSD Cathode Select** | Pin 29 | `PORTA7` | Digit switching |
| **Emergency Override Button** | Pin 2 | `PE4` (`INT4`) | Falling-edge interrupt (`FALLING`) |
| **Tamper Reed Switch (Door)** | Pin 21 | `PD0` (`INT0`) | State change interrupt (`CHANGE`) |
| **Relay Signal (12V Strike)** | Pin 12 | `PB6` | Active-HIGH digital output |
| **Alarm Buzzer** | Pin 11 | `PB5` (`OC1A`) | PWM audio output |
| **LCD Bus (RS, E, D4-D7)** | Pins 8, 9, 4, 5, 6, 7 | Ports H, E, G | 4-bit parallel LCD interface |
| **RC522 RFID Module** | Pins 50, 51, 52, 53 | Port B (`MISO`, `MOSI`, `SCK`, `SS`) | Hardware SPI bus |

---

## Bill of Materials (BOM)

* 1x **Arduino Mega 2560** board (ATmega2560 microcontroller)
* 1x **MFRC522 RFID** reader module (13.56 MHz) + compatible cards/tags
* 1x **4x4 matrix membrane keypad**
* 1x **16x2 character LCD** (HD44780-compatible)
* 1x **Dual 7-segment display** (Common Cathode)
* 1x **5V relay module** (optocoupler-isolated) + 12V DC solenoid door lock
* 1x **Magnetic reed switch** (for door perimeter monitoring)
* 1x **Passive piezo buzzer**
* Current-limiting resistors ($220\,\Omega$ – $330\,\Omega$) and a $1\text{N}4007$ flyback diode for the solenoid relay circuit

---

## Dependencies

The project builds inside the Arduino IDE with the following libraries:
* `SPI.h` (bundled with Arduino core)
* `LiquidCrystal.h` (bundled with Arduino core)
* `MFRC522.h` (by GithubCommunity)
* `Keypad.h` (by Mark Stanley, Alexander Brevig)

---

## Setup & Flashing

1. Open the `.ino` file in the **Arduino IDE**.
2. Go to **Tools > Board** and select **Arduino Mega or Mega 2560**.
3. Under **Tools > Processor**, verify **ATmega2560 (Mega 2560)** is selected.
4. Verify wiring against the pinout table above. Ensure the 12V solenoid has its own external power supply with a shared GND connection to the Arduino.
5. Compile and upload the sketch to the board.
6. Open the **Serial Monitor** at `9600 baud` to inspect real-time debug logs and captured RFID card UIDs.
