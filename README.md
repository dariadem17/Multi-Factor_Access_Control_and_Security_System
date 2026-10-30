# Multi-Factor Access Control & Security System (MF-ACSS)

A distributed embedded security system developed on an **Arduino Mega 2560 (ATmega2560)** paired with an **ESP32** co-processor for wireless mobile authentication.

The architecture combines three access tiers: contactless RFID cards, 4x4 matrix keypad PIN entry, and remote smartphone unlock over Bluetooth Low Energy (BLE). The Arduino Mega handles local hardware control, driving display multiplexing, status LEDs, and actuators through **direct AVR register manipulation (`DDRA`, `PORTA`)** and **hardware external interrupts (`INTx`)**, while the ESP32 operates as a dedicated wireless bridge over hardware UART.

---

## How It Works (Workflow)

1. **Standby (Idle):** The system waits for an RFID card or a mobile connection. A background 8-LED sweep animation runs on Port A via bit-shifting, and the display indicates an idle status.
2. **Access Methods:**
   * **Local Authentication (RFID + PIN):** The RC522 scans the badge over SPI. If valid, the 16x2 LCD prompts for a 4-digit PIN on the 4x4 keypad (masked with `*`, 15-second timeout).
   * **Smartphone Remote Unlock (BLE):** The user connects via a mobile app to the ESP32. Upon validating the cryptographic token/key, the ESP32 forwards an unlock packet over hardware UART (`Serial1`) to the Arduino Mega.
3. **Access Granted:**
   * The relay actuates the door solenoid strike for 5 seconds.
   * The dual 7-segment display (SSD) runs an active countdown (`05` $\rightarrow$ `00`).
4. **Lockdown & Tamper Alert:**
   * 3 consecutive failed PIN attempts or tripping the magnetic reed switch on the doorframe triggers an immediate alarm.
   * A PWM-driven buzzer sounds, and local authentication freezes until reset.

---

## Architecture & Hardware Control

* **Dual-Controller Bridge (UART):**
  * The ESP32 handles the wireless stack (Bluetooth BLE) and forwards structured serial frames to the Arduino Mega (`TX2/RX2` $\leftrightarrow$ `RX1/TX1`).
  * A simple resistor voltage divider protects the ESP32's 3.3V RX line from the Mega's 5V TX pin.
* **Direct Register Manipulation (`PORTA`, `DDRA`):**
  * Tens and units on the dual 7-segment display share a 7-bit segment bus (`PA0`–`PA6`) and are time-multiplexed using `PA7` for cathode switching. Updates take a single clock cycle using precomputed BCD values (`PORTA = ssdlut[pos] | outvalue`).
* **Debouncing & External Interrupts:**
  * Physical override buttons and door tamper sensors are wired to dedicated hardware interrupt pins (`INT4` on Pin 2 and `INT0` on Pin 21).
  * Contact bounce is filtered inside the ISR by tracking elapsed time with `millis()`.

---

## Pinout & Interfacing

| Subsystem / Peripheral | Arduino Mega Pin | ESP32 Pin | Logic / Function |
| :--- | :--- | :--- | :--- |
| **SSD Segments (a..g)** | Pins 22 – 28 (`PORTA0–6`) | — | Segment bus output |
| **SSD Cathode Select** | Pin 29 (`PORTA7`) | — | Active digit multiplexer |
| **Emergency Button** | Pin 2 (`PE4` / `INT4`) | — | Falling-edge interrupt (`INPUT_PULLUP`) |
| **Door Tamper Switch** | Pin 21 (`PD0` / `INT0`) | — | State change interrupt (`INPUT_PULLUP`) |
| **Relay Strike Signal** | Pin 12 (`PB6`) | — | 12V Solenoid strike control |
| **Piezo Buzzer** | Pin 11 (`PB5` / `OC1A`) | — | PWM audio alarm output |
| **LCD Bus (RS, E, D4-D7)** | Pins 8, 9, 4, 5, 6, 7 | — | 4-bit parallel display bus |
| **RC522 RFID Module** | Pins 50, 51, 52, 53 | — | Hardware SPI (`MISO`, `MOSI`, `SCK`, `SS`) |
| **Inter-Board UART** | Pin 19 (`RX1`), Pin 18 (`TX1`)| GPIO 17 (`TX2`), GPIO 16 (`RX2`)| Serial bridge between Mega and ESP32 |

---

## Bill of Materials (BOM)

* 1x **Arduino Mega 2560** (main logic and direct register I/O)
* 1x **ESP32 NodeMCU / DevKit** (Bluetooth BLE gateway)
* 1x **RC522 RFID module** (13.56 MHz) + MIFARE cards/tags
* 1x **4x4 matrix membrane keypad**
* 1x **16x2 character LCD** (HD44780-compatible)
* 1x **Dual 7-segment display** (Common Cathode)
* 1x **5V optocoupled relay module** + 12V DC solenoid door lock
* 1x **Magnetic reed switch** (perimeter monitoring)
* 1x **Passive piezo buzzer**
* Resistors ($220\,\Omega$, $330\,\Omega$, $1\text{k}\Omega$, $2\text{k}\Omega$ for voltage divider) and a $1\text{N}4007$ flyback diode

---

## Project Structure & Flashing

```text
├── mega_firmware/
│   └── mega_firmware.ino    # Register logic, SSD driver, ISRs, relay controller
├── esp32_gateway/
│   └── esp32_gateway.ino    # BLE peripheral server, serial dispatch to Mega
└── README.md
