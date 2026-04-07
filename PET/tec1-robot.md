# Build a Z80 Robot Rover with the TEC-1 SBC

## Drive Your Classic Australian Single-Board Computer Around the Room

*By the Workshop Team*

---

There is something deeply satisfying about watching a piece of vintage computing hardware do something physical in the real world. The TEC-1 single-board computer has been a fixture of Australian electronics education since the 1980s, introducing generations of hobbyists to Z80 assembly language, binary arithmetic, and the raw pleasure of programming at the metal. Most TEC-1 owners have blinked LEDs, played tunes through the buzzer, and written monitor extensions — but very few have ever made their TEC-1 move. This project fixes that.

What we are building is a two-wheeled robot rover: a small, zippy chassis carrying the TEC-1, a motor driver board, an ultrasonic distance sensor, and a voice. Yes — a voice. The robot announces its every move in synthesised speech using the SPO256-AL2 allophone chip, the same device that appeared in the original *Talking Electronics* Speech Module project. Press 1 and it rolls forward saying "FORWARD". Press A for autonomous obstacle-avoidance mode and it navigates the room under its own initiative, calling out "DANGER" when it detects a wall and corrects course. The six-digit LED display shows the current mode and, in auto mode, the live distance reading in centimetres.

The total cost is modest — around AU$70 if you already own the TEC-1 — and the hardware additions are clean and well-defined. We add a 74HC138 port decoder, a 74HC574 output latch, a 74HC244 input buffer, an L293D H-bridge motor driver, an HC-SR04 ultrasonic module, the SPO256-AL2 speech circuit, a 3.58MHz crystal oscillator, and a separate 9V battery for the motors. The TEC-1's Z80 bus does all the heavy lifting, and the speech chip's hardware /WAIT line does something particularly elegant: it freezes the Z80 automatically while each allophone plays, so your assembly routine for speaking a word is just four lines of code.

If you have never written Z80 assembly before, do not worry. We will walk through every routine in the program, explaining what each instruction does and why. By the end you will have not only a working robot, but a solid understanding of I/O port expansion, bit-banging timing-sensitive protocols, and multiplexed display management on a classic 8-bit machine.

---

## A Quick Word About the TEC-1

For readers who may be coming to this project fresh, the TEC-1 is a Z80A-based single-board computer originally designed by John Hardy and Ken Stone and published in the Australian magazine *Talking Electronics* in the early 1980s. It runs a 2KB ROM monitor called JMON at address 0000h that provides basic hex entry, address navigation, single-stepping, and a handful of callable utility routines. RAM sits at 4000h (8KB on most modern versions, though earlier boards had 2KB at 2000h — check your version). The display is six seven-segment LED digits scanned continuously by the monitor's interrupt-driven routine. A 23-key keypad gives you hexadecimal digits 0 through F plus function keys. A single-bit buzzer hangs off bit 0 of output port 01h.

The expansion port brings out the full Z80 bus: address lines A0–A15, data lines D0–D7, /IORQ, /RD, /WR, /MREQ, /RESET, /INT, /NMI, /WAIT, /BUSACK, /BUSREQ, and power rails. This makes adding new I/O ports straightforward — and that expansion port is exactly how we connect our motor driver.

---

## How It Works — The Overview

The Z80 communicates with the outside world through its I/O address space, separate from memory. An OUT instruction puts a byte on the data bus and asserts /IORQ and /WR simultaneously; an IN instruction asserts /IORQ and /RD and reads whatever is on the data bus. Port addresses are presented on the lower eight address lines (A0–A7 on a standard Z80 I/O cycle, though the upper eight are also driven and can be used for decoding).

We decode a new output port at address 04h using a 74HC574 octal D-type flip-flop with three-state outputs. Its clock input is driven by a NAND gate that combines /IORQ, /WR, and a partial address decode. The eight output bits of the latch drive the L293D motor driver and the HC-SR04 trigger line. To read the ultrasonic echo pulse back, we use a 74HC244 octal buffer at the same address (04h), enabled only when /IORQ and /RD are both asserted, with the echo signal connected to D0.

The L293D takes four direction bits — left-forward, left-backward, right-forward, right-backward — and drives two motors accordingly. Combine left-forward with right-forward and you get straight ahead. Left-backward plus right-backward gives reverse. Left-forward alone spins the chassis left; right-forward alone spins it right.

The HC-SR04 ranging module is bit-banged by the Z80: assert the trigger line for at least 10 microseconds, then count clock cycles until the echo line goes high, wait while it is high counting elapsed time, and convert the total high-time to centimetres using the speed of sound (approximately 58 microseconds per centimetre for a round trip). At 3.5 MHz, each machine cycle is about 286 nanoseconds, giving us plenty of resolution for obstacle detection.

The SPO256-AL2 speech chip communicates over a completely separate channel. Your Z80 code places an allophone number (0x00–0x3F) on data lines D0–D5, then executes `OUT (05h), A`. This pulses the chip's ALD (Address Load) line, which tells the SPO256 to latch the allophone and begin speaking it. While the chip is processing, it asserts the /WAIT line low. The Z80 sees a hardware WAIT state and freezes — not a software delay, but a genuine hardware stall — until the allophone finishes, at which point /WAIT releases and the CPU resumes. The result is that speaking a word requires no timing loops, no polling, and no interrupt handling. You simply loop through a table of allophone bytes and call `OUT (05h), A` for each one.

The main loop polls the keypad using JMON's key-scan routines, updates the display, and dispatches to the appropriate motor command or mode handler. The auto-avoid mode runs a simple state machine: drive forward, measure distance, react if an obstacle appears.

---

## Parts List

| Qty | Description | Approximate Cost (AU$) |
|-----|-------------|------------------------|
| 1 | TEC-1 SBC (you already own this) | — |
| 1 | 2WD robot chassis kit with motors and wheels | $12–18 |
| 1 | L293D quad half-H driver IC | $3 |
| 1 | 74HC574 octal D flip-flop (output latch) | $2 |
| 1 | 74HC244 octal buffer (echo input) | $2 |
| 1 | 74HC138 3-to-8 decoder (port address decode) | $2 |
| 1 | HC-SR04 ultrasonic distance module | $4 |
| 1 | SPO256-AL2 speech synthesis IC | $8–12 |
| 1 | LM386 audio amplifier IC | $2 |
| 1 | BC557 PNP transistor (amplifier power switch) | $0.50 |
| 1 | 3.58MHz crystal oscillator can (DIP-8 or DIP-14 TTL output) | $5 |
| 1 | 8 ohm 0.5W speaker | $3 |
| 1 | 10kΩ trimpot (volume) | $1 |
| 1 | 82kΩ resistor 1/4W 5% | $0.20 |
| 1 | 1kΩ resistor 1/4W 5% | $0.20 |
| 2 | 4.7µF electrolytic capacitor | $0.50 |
| 2 | 100nF monobloc capacitor | $0.50 |
| 1 | 47nF ceramic capacitor | $0.20 |
| 1 | 10µF electrolytic capacitor | $0.30 |
| 1 | 100µF electrolytic capacitor (motor decoupling) | $0.50 |
| 4 | 100nF ceramic disc capacitors (IC bypass) | $0.50 |
| 1 | 9V battery and snap connector (motor supply) | $4 |
| 1 | 1N4001 diode (reverse polarity protection, motor supply) | $0.50 |
| 1 | Small piece of stripboard or prototyping PCB | $3 |
| 1 | 40-pin IDC header and ribbon cable (expansion port) | $4 |
| 1 | Assorted hookup wire, standoffs, screws | $3 |
| 1 | IC sockets: 20-pin (L293D), 20-pin (574), 20-pin (244), 16-pin (138), 28-pin (SPO256), 8-pin (LM386) | $4 |
| **Total** | | **~AU$65–75** |

The 2WD chassis kits available from most local electronics retailers and online suppliers include two geared DC motors rated at 3–6V, two wheels with rubber tyres, a clear acrylic or ABS base plate, a castor wheel, and a battery holder. The motors draw about 150–250mA each under load, which is well within the L293D's capability. The SPO256-AL2 may need to be sourced online — search for it by the full part number. The 3.58MHz crystal oscillator can is the same frequency used in NTSC video equipment and is widely stocked.

---

## Understanding the Port Decode Circuit

Before picking up a soldering iron, it is worth understanding exactly how the new I/O ports are decoded. With two ports needed — 04h for the motor driver and 05h for the speech chip — we use a 74HC138 3-to-8 decoder to produce clean individual strobe signals for each one. This is the same approach the TEC-1's own onboard circuitry uses for its display and keypad ports.

During a Z80 I/O write cycle, the CPU asserts /IORQ low and /WR low simultaneously, and the port address appears on address lines A0–A7. The 74HC138 decodes address lines A0, A1, and A2 to produce eight active-low outputs (Y0 through Y7), each corresponding to one port address from 00h through 07h. We gate the decoder using the Z80's /IORQ and /WR signals: connect /IORQ to active-low enable input E1, and /WR to active-low enable input E2. Tie the active-high enable E3 permanently to +5V. The 138 is now enabled only during an I/O write cycle, and its selected output goes low for exactly the duration of that write cycle.

Port 04h (binary 00000100, meaning A2=1, A1=0, A0=0) causes Y4 to go low. We connect Y4 to the clock input of the 74HC574 motor latch. The 74HC574 clocks on the rising edge — that is, when Y4 returns high at the end of the write cycle — so it captures the data bus value at precisely the right moment.

Port 05h (binary 00000101, meaning A2=1, A1=0, A0=1) causes Y5 to go low. We connect Y5 directly to the ALD (Address Load) pin of the SPO256-AL2. The SPO256 latches its allophone number on the falling edge of ALD and begins speaking. Clean, unambiguous, and no logic gates involved.

For the read side (used only for the ultrasonic echo signal), we build a separate read-enable decode. The 74HC244 buffer needs to be enabled only during an `IN (04h)` instruction, when /IORQ and /RD are both asserted. Use a two-input NAND gate — there is a 74HC00 in the parts list for this purpose — to combine /IORQ and /RD. The NAND output goes low only when both inputs are low, which is exactly the active-low /OE signal the 74HC244 expects. You do not need to check the address for the read side because only one device (the 74HC244) is connected to the bus during a read, so there is no conflict.

Here is the complete decode connection summary:

```
74HC138 pin connections:
  A  (pin 1) ← Z80 A0
  B  (pin 2) ← Z80 A1
  C  (pin 3) ← Z80 A2
  E1 (pin 4) ← Z80 /IORQ  (active-low enable)
  E2 (pin 5) ← Z80 /WR    (active-low enable)
  E3 (pin 6) ← +5V        (active-high enable, always on)
  Y4 (pin 11) → 74HC574 CLK pin 11  (motor latch — fires on OUT 04h)
  Y5 (pin 10) → SPO256-AL2 ALD pin 20  (speech load — fires on OUT 05h)

74HC00 NAND gate (for read decode):
  Pin 1 ← Z80 /IORQ
  Pin 2 ← Z80 /RD
  Pin 3 → 74HC244 /OE pins 1 and 19  (echo buffer — enabled on IN 04h)
```

Note: the 74HC138 outputs Y0–Y3 and Y6–Y7 are unused in this project. Leave them unconnected or add pull-up resistors if you want to add more peripherals later.

---

## Port 04h and 05h Bit Assignments

### Port 04h — Motor Driver and Ultrasonic Sensor

| Bit | Output Function | Input Function |
|-----|-----------------|----------------|
| D0 | Left motor FORWARD | HC-SR04 Echo (via 74HC244) |
| D1 | Left motor BACKWARD | — |
| D2 | Right motor FORWARD | — |
| D3 | Right motor BACKWARD | — |
| D4 | HC-SR04 TRIGGER | — |
| D5 | Spare | — |
| D6 | Spare | — |
| D7 | Spare | — |

The L293D takes two logic inputs per motor channel and drives the motor in whichever direction those two bits select. Driving both inputs high or both low applies the brake. In our code we always ensure that forward and backward bits for the same motor are never both set simultaneously.

### Port 05h — SPO256-AL2 Speech Chip

Port 05h is write-only. When your code executes `OUT (05h), A`, the value in register A is placed on the data bus, and the 74HC138's Y5 output pulses low. This activates the SPO256's ALD pin, causing it to latch the lower six bits (D0–D5) as an allophone number and begin synthesising that sound. The upper two bits (D6–D7) are ignored by the SPO256 since it only has 64 allophones (0x00–0x3F). There are 64 allophones in total including five pause lengths, and words are formed by chaining them together in a table terminated by 0FFh.

While the SPO256 is speaking an allophone, it holds the Z80's /WAIT line low. This hardware stall is automatic and requires nothing from your software. When the allophone completes, /WAIT releases and the Z80 picks up exactly where it left off — on the next instruction after the `OUT`. No polling, no delay loops, no interrupts.

---

## Building the Hardware

### Step 1: Prepare the Chassis

Assemble the 2WD chassis kit following its included instructions. Mount the two geared motors in the chassis side brackets and secure the wheels. Fit the castor wheel at the front. Leave the battery holder in its mounting position but do not connect it yet. You should have a chassis that rolls freely on all three contact points when pushed by hand. Check that both drive wheels spin without binding — if a wheel is tight, loosen the motor mount screw slightly and re-tighten while the wheel is turning to self-align.

Mount four standoffs on the chassis plate using M3 screws to create a raised platform for the TEC-1. Ideally position the TEC-1 so its expansion port edge faces the rear of the robot, where your ribbon cable will run down to the interface board. The interface board itself — a small piece of stripboard carrying the 74HC574, 74HC244, 74HC00, L293D, and associated passive components — can be mounted beside or beneath the TEC-1 platform using double-sided foam tape or additional standoffs.

### Step 2: Build the Interface Board

Cut a piece of stripboard approximately 15 strips by 30 holes. This is plenty of room for all four ICs and their support components. Begin by socketing and placing the ICs before soldering, to ensure you have the layout right.

Place the 74HC574 near the left edge of the board with its pin 1 notch visible. Place the 74HC244 immediately beside it. The 74HC00 goes in the upper right area, and the L293D in the lower right area. Bridge the power rails along the top and bottom of the board with bare tinned wire: one rail for +5V (from the TEC-1 expansion port) and one for GND. Add 100nF ceramic bypass capacitors between VCC and GND pins on each IC, keeping leads as short as possible.

Solder the 74HC574 connections: pins 1 (OE) to GND, pin 11 (CLK) to the NAND decode output, pins 3–10 (D0–D7) to the data bus wires from the ribbon cable, pins 12–19 (Q0–Q7) to the corresponding L293D inputs and the HC-SR04 trigger line.

Solder the 74HC244 connections: pins 1 and 19 (/OE1 and /OE2) together to the read-decode NAND output, pins 2–9 (A1–A8 inputs) with pin 2 connected to the HC-SR04 echo line and pins 3–9 tied low through 10kΩ resistors, pins 12–18 (Y1–Y8 outputs) to data bus wires.

Wire the 74HC00 quad-NAND package. Gate 1 (pins 1, 2, 3): inputs to /IORQ and /WR, output to gate 2 input. Gate 2 (pins 4, 5, 6): inputs from gate 1 output and A2, output to 74HC574 CLK. Gate 3 (pins 9, 10, 8): inputs to /IORQ and /RD, output to gate 4 input. Gate 4 (pins 12, 13, 11): inputs from gate 3 output and A2, output to 74HC244 /OE. The unused gate (pins 8 and 11 are shared above — check your 74HC00 pinout carefully; the quad NAND is pins 1-2-3, 4-5-6, 9-10-8, 12-13-11 with pin 7 GND and pin 14 VCC).

For the L293D motor driver, connect the enable pins (pins 1 and 9, EN1 and EN2) permanently to +5V so both channels are always enabled. Connect logic inputs 1A (pin 2) to 574 Q0 (left forward), 2A (pin 7) to 574 Q1 (left backward), 3A (pin 10) to 574 Q2 (right forward), 4A (pin 15) to 574 Q3 (right backward). Connect the motor supply pin (pin 8, VS) to your 9V motor battery through the 1N4001 protection diode (cathode to VS, anode to battery positive). Connect VCC (pin 16) to the +5V logic rail. Connect all four GND pins (4, 5, 12, 13) to the common ground. Connect the left motor wires to output pins 1Y (pin 3) and 2Y (pin 6), and the right motor wires to 3Y (pin 11) and 4Y (pin 14). Add the 100µF electrolytic capacitor across the motor supply rail (VS to GND) to absorb back-EMF spikes.

### Step 3: Wire the Expansion Connector

Make up a ribbon cable with a 40-way IDC connector to mate with the TEC-1's expansion port. Check your specific TEC-1 version's expansion pinout — it varies slightly between revisions. You need the following signals: D0–D7 (8 lines), A2 (1 line), /IORQ (1 line), /WR (1 line), /RD (1 line), +5V (1 line), GND (at least 2 lines). That is 17 connections in total. Carefully crimp the IDC connector, double-checking pin 1 orientation against the TEC-1 board's marking.

Before connecting the ribbon cable to the TEC-1, power up the interface board from a bench supply set to 5V and verify that all IC supply voltages are correct with a multimeter. Check for short circuits between VCC and GND. Only then connect the ribbon cable to the TEC-1 expansion port.

### Step 4: Build and Wire the Speech Module

The speech module circuit is simple but deserves its own piece of stripboard, separate from the motor driver board. This keeps the audio circuitry physically away from the motor noise, which the original *Talking Electronics* article noted can cause an uncomfortable buzz through the speaker.

Start with the SPO256-AL2 IC in its 28-pin socket. This is the star of the show — a purpose-built allophone synthesiser from General Instrument (later Microchip). Before you do anything else, note the following errata reported by the Australian Vintage Computer Group: **the SE (serial enable) pin 19 of the SPO256 must be tied to +5V**. It is missing from the original Talking Electronics schematic, but was present on the physical PCB. Without it, any glitch on the data lines can accidentally trigger a new allophone mid-word. Solder a short wire from pin 19 to the +5V rail immediately.

Connect the SPO256 data inputs (pins 13–18, labelled A1–A6 on the chip, corresponding to D0–D5 on the Z80 data bus) to the data bus wires from your ribbon cable. Only six bits are needed — leave pins for D6 and D7 disconnected. Connect the ALD pin (pin 20) to the Y5 output of the 74HC138 decoder. Connect RESET (pin 2) to the Z80 /RESET line from the expansion port — this ensures the speech chip resets cleanly whenever the TEC-1 is reset. Connect the /WAIT output (pin 8, labelled SBY on some datasheets) to the Z80 /WAIT line on the expansion port. Connect VDD (pin 24) to +5V and VSS (pin 23, GND) to ground.

The clock input (pin 27, OSC1) needs a 3.58MHz signal. Use a pre-built DIP crystal oscillator can — a self-contained module that produces a TTL-level square wave at its output pin when powered. These are inexpensive and eliminate the need to build your own oscillator circuit. Connect the oscillator can's output pin directly to the SPO256 OSC1 pin 27. Power the oscillator from the +5V rail. Connect OSC2 (pin 28) to ground via a 10nF capacitor as specified in the datasheet.

The SPO256's audio output (pin 24, DIG OUT) feeds into the LM386 amplifier stage. The circuit follows the original Talking Electronics design: the DIG OUT pin connects through a 10kΩ trimpot (your volume control) to pin 3 of the LM386. The trimpot wiper goes to pin 3; one end to DIG OUT, other end to ground. Add a 47nF capacitor from pin 3 to ground for filtering. Connect pin 6 (VS) to +5V through a 100µF bypass capacitor to ground. Connect pins 1 and 8 together through a 10µF capacitor and a 1.2kΩ resistor in series for gain setting. Connect pin 2 to ground. The output at pin 5 goes through a 100nF capacitor and a 10Ω resistor to the speaker.

The BC557 PNP transistor is a power switch for the LM386 supply. Its emitter connects to +5V, its collector to the LM386 VS pin, and its base to the SPO256's /WAIT pin through an 82kΩ resistor. When the SPO256 is silent, /WAIT is high — this holds the BC557 base at a high voltage relative to the emitter, keeping the transistor off and cutting power to the LM386. This neatly eliminates the buzz caused by the TEC-1's LED scan noise being picked up by the amplifier when no speech is playing.

Once built, run four wires from the speech board back to the expansion ribbon cable: D0–D5 (shared with the motor board data bus wires), /WAIT, /RESET, and +5V/GND.

**Important test before connecting to TEC-1:** Power the speech module from a bench 5V supply with the ALD pin held high (inactive). The module should be completely silent. If you hear buzzing, your LM386 power switch circuit is not working — check the BC557 orientation and the 82kΩ base resistor value.

### Step 5: Mount the HC-SR04

Hot-glue or cable-tie the HC-SR04 module to the front of the chassis, with its two transducer cans (trigger and echo) facing forward. Run four wires back to the interface board: VCC (5V), GND, TRIG (to 574 Q4), and ECHO (to 244 A1 input). Keep the ECHO wire away from the motor wires to avoid noise pickup.

### Step 6: First Power-On Test

Before loading any software, perform a static test of the output latch. Connect the 9V motor battery but leave the motor wires disconnected from the L293D outputs for now. Power up the TEC-1. Using JMON's manual data entry mode, navigate to a convenient RAM address, enter the value 01h (bit 0 high), and execute an OUT 04h instruction by writing a short two-instruction test program: OUT (04h), A followed by RET. Load A with 01h, call this routine, and probe pin 3 of the L293D (1Y output) with a multimeter — it should be near battery voltage (9V minus about 1.5V for the L293D drop, so around 7.5V). Try other bit patterns to confirm each output is working. This confirms the latch decode is correct before you reconnect the motors.

---

## The Software

Now for the fun part. The complete program is approximately 300 lines of Z80 assembly and fits comfortably in the first 1KB of the 8KB RAM region starting at 4000h. You can type it directly into JMON using the DATA entry mode, or use a cross-assembler such as TASM, PASMO, or the modern z88dk toolchain on a PC, then transfer the object code to the TEC-1 via a serial interface or simply by hand-entry of the hex bytes.

The program is structured around a main loop that scans the keypad, updates the display, and dispatches to command handlers. Subroutines are grouped by function: motor control, ultrasonic ranging, display formatting, delay generation, and the autonomous avoidance routine.

### JMON Entry Points

The JMON monitor provides several useful callable routines. For this project we use address 0010h as the display scan call — this re-enters the JMON display multiplexing routine and updates all six digits from the display buffer. Check your specific TEC-1 version's documentation, as JMON entry points can differ between releases; 0010h is the commonly cited scan-and-return entry point for the standard JMON release. The display buffer is six bytes of RAM in the JMON workspace area — your TEC-1 documentation will give the exact address; we define it as DISPLY_BUF EQU 0900h in the listing below, which is typical for the most common JMON version. Adjust if necessary.

### Complete Assembly Listing

```asm
;*******************************************************************
;* TEC-1 ROBOT ROVER                                               *
;* 2WD robot controlled via TEC-1 keypad and Z80 assembly         *
;* Motor driver: L293D on port 04h via 74HC574 latch              *
;* Obstacle sensor: HC-SR04 bit-banged on port 04h bits 4/0       *
;* Speech: SPO256-AL2 on port 05h, /WAIT halts Z80 per allophone  *
;* Display: 6-digit 7-seg via JMON scan buffer                    *
;*                                                                 *
;* Key assignments:                                                *
;*   0 = STOP          1 = FORWARD                                *
;*   2 = BACKWARD      3 = LEFT TURN                              *
;*   4 = RIGHT TURN    A = AUTO OBSTACLE-AVOID                    *
;*   B = SPEECH DEMO ("HELLO I AM YOUR ROBOT")                    *
;*                                                                 *
;* Port 04h output bits:                                           *
;*   Bit 0 = Left motor FORWARD                                   *
;*   Bit 1 = Left motor BACKWARD                                  *
;*   Bit 2 = Right motor FORWARD                                  *
;*   Bit 3 = Right motor BACKWARD                                 *
;*   Bit 4 = HC-SR04 TRIGGER                                      *
;*                                                                 *
;* Port 04h input bit:                                             *
;*   Bit 0 = HC-SR04 ECHO (via 74HC244 buffer)                   *
;*                                                                 *
;* ORG 4000h (TEC-1 RAM start)                                    *
;*   Assemble with: pasmo --hex robot.asm robot.hex               *
;*******************************************************************

;-------------------------------------------------------------------
; SYSTEM CONSTANTS
;-------------------------------------------------------------------
        ORG     4000h

; JMON entry points (adjust for your JMON version)
JMON_SCAN   EQU 0010h       ; Call to scan/refresh display once
JMON_KEY    EQU 001Fh       ; Call to get keypress (waits), returns key in A

; JMON display buffer - 6 bytes, digit 0 is rightmost
; Adjust this address to match your JMON version's buffer location
DISP_BUF    EQU 0900h       ; Display buffer base address

; I/O ports
PORT_MOTOR  EQU 04h         ; Motor + trigger output (74HC574 latch)
PORT_ECHO   EQU 04h         ; Echo input (74HC244 buffer, same address)
PORT_SPO    EQU 05h         ; SPO256-AL2 allophone output (/WAIT stalls Z80)
PORT_BUZZ   EQU 01h         ; TEC-1 buzzer (bit 0)

; Motor control byte bit masks (for PORT_MOTOR output)
MTR_LF      EQU 01h         ; Left motor Forward  (bit 0)
MTR_LB      EQU 02h         ; Left motor Backward (bit 1)
MTR_RF      EQU 04h         ; Right motor Forward (bit 2)
MTR_RB      EQU 08h         ; Right motor Backward(bit 3)
TRIG_BIT    EQU 10h         ; HC-SR04 Trigger     (bit 4)

; Composite motor commands
MTR_STOP    EQU 00h                         ; All motors off
MTR_FWD     EQU MTR_LF OR MTR_RF           ; Both forward
MTR_BACK    EQU MTR_LB OR MTR_RB           ; Both backward
MTR_LEFT    EQU MTR_LF                     ; Left motor only (spin left)
MTR_RIGHT   EQU MTR_RF                     ; Right motor only (spin right)

; Echo bit mask for input
ECHO_BIT    EQU 01h         ; Echo signal is on bit 0 of port read

; Obstacle detection threshold (centimetres)
OBSTACLE_CM EQU 20          ; Stop and avoid if closer than this

; Timing constants at 3.5 MHz
; 1 T-state = ~286 ns
; DJNZ = 13 T-states when looping = ~3.71 us per iteration
; For 10us trigger pulse: need ~3 DJNZ iterations (10us / 3.71us)
; For 1ms delay: outer*inner DJNZ loops
TRIG_LOOPS  EQU 3           ; ~11us trigger pulse
DLY_INNER   EQU 175         ; Inner loop count for delay subroutine
                             ; 175 * 13 T-states = 2275 T-states ~= 650us

;-------------------------------------------------------------------
; RAM VARIABLES (placed after program code - see END of listing)
;-------------------------------------------------------------------
; We define these at fixed addresses in upper RAM
CUR_MODE    EQU 4200h       ; Current drive mode byte
DIST_CM     EQU 4201h       ; Last measured distance in cm (binary)
MOTOR_STAT  EQU 4202h       ; Current motor output byte shadow

;-------------------------------------------------------------------
; PROGRAM ENTRY POINT
;-------------------------------------------------------------------

ROBOT_START:
        DI                  ; Disable interrupts (we poll display manually)
        LD      SP, 43FFh   ; Set stack pointer at top of our workspace

        ; Initialise state
        XOR     A
        LD      (CUR_MODE), A   ; Mode 0 = STOP
        LD      (DIST_CM), A
        LD      (MOTOR_STAT), A

        ; Stop motors immediately at startup
        OUT     (PORT_MOTOR), A

        ; Show startup message "rObOt" on display, then speak greeting
        CALL    DISP_ROBOT
        LD      BC, 500         ; Brief pause before speaking
        CALL    DELAY_MS
        LD      HL, SPK_HELLO   ; "HELLO"
        CALL    SPEAK_WORD
        LD      BC, 300
        CALL    DELAY_MS
        LD      HL, SPK_ROBOT   ; "ROBOT"
        CALL    SPEAK_WORD
        LD      BC, 1000        ; Hold display after greeting
        CALL    DELAY_MS

;-------------------------------------------------------------------
; MAIN LOOP
; Poll keypad, update display, handle auto mode
;-------------------------------------------------------------------
MAIN_LOOP:
        ; Refresh display with current mode information
        CALL    UPDATE_DISPLAY

        ; Check for keypress (non-blocking: we use JMON scan but
        ; implement our own non-waiting key check)
        CALL    KEY_CHECK       ; Returns A=0FFh if no key, else key code
        CP      0FFh
        JR      Z, CHK_AUTO     ; No key pressed, check auto mode

        ; A key was pressed - dispatch to handler
        CALL    KEY_DISPATCH

CHK_AUTO:
        ; If in auto mode (mode = 0Ah), run auto-avoid step
        LD      A, (CUR_MODE)
        CP      0Ah
        CALL    Z, AUTO_STEP    ; Run one obstacle-avoidance step

        JR      MAIN_LOOP       ; Loop forever

;-------------------------------------------------------------------
; KEY_DISPATCH
; Input: A = key code (0-F from keypad)
; Dispatches to appropriate handler
;-------------------------------------------------------------------
KEY_DISPATCH:
        CP      00h
        JP      Z, CMD_STOP

        CP      01h
        JP      Z, CMD_FORWARD

        CP      02h
        JP      Z, CMD_BACK

        CP      03h
        JP      Z, CMD_LEFT

        CP      04h
        JP      Z, CMD_RIGHT

        CP      0Ah             ; Key 'A'
        JP      Z, CMD_AUTO

        CP      0Bh             ; Key 'B'
        JP      Z, CMD_BEEP

        RET                     ; Unknown key - ignore

;-------------------------------------------------------------------
; COMMAND HANDLERS
;-------------------------------------------------------------------

CMD_STOP:
        LD      A, 00h
        LD      (CUR_MODE), A
        LD      A, MTR_STOP
        CALL    SET_MOTORS
        LD      HL, SPK_STOP    ; Say "STOP"
        CALL    SPEAK_WORD
        RET

CMD_FORWARD:
        LD      A, 01h
        LD      (CUR_MODE), A
        LD      A, MTR_FWD
        CALL    SET_MOTORS
        LD      HL, SPK_FORWARD ; Say "FORWARD"
        CALL    SPEAK_WORD
        RET

CMD_BACK:
        LD      A, 02h
        LD      (CUR_MODE), A
        LD      A, MTR_BACK
        CALL    SET_MOTORS
        LD      HL, SPK_BACK    ; Say "BACK"
        CALL    SPEAK_WORD
        RET

CMD_LEFT:
        LD      A, 03h
        LD      (CUR_MODE), A
        LD      A, MTR_LEFT
        CALL    SET_MOTORS
        LD      HL, SPK_LEFT    ; Say "LEFT"
        CALL    SPEAK_WORD
        RET

CMD_RIGHT:
        LD      A, 04h
        LD      (CUR_MODE), A
        LD      A, MTR_RIGHT
        CALL    SET_MOTORS
        LD      HL, SPK_RIGHT   ; Say "RIGHT"
        CALL    SPEAK_WORD
        RET

CMD_AUTO:
        LD      A, 0Ah
        LD      (CUR_MODE), A
        LD      A, MTR_FWD
        CALL    SET_MOTORS
        LD      HL, SPK_AUTO    ; Say "AUTO"
        CALL    SPEAK_WORD
        RET

CMD_BEEP:
        ; Speech demo — say "HELLO I AM YOUR ROBOT"
        LD      A, 0Bh
        LD      (CUR_MODE), A
        CALL    BEEP_DEMO       ; Play buzzer fanfare
        LD      HL, SPK_HELLO
        CALL    SPEAK_WORD
        LD      BC, 150
        CALL    DELAY_MS
        LD      HL, SPK_I
        CALL    SPEAK_WORD
        LD      BC, 100
        CALL    DELAY_MS
        LD      HL, SPK_AM
        CALL    SPEAK_WORD
        LD      BC, 100
        CALL    DELAY_MS
        LD      HL, SPK_YOUR
        CALL    SPEAK_WORD
        LD      BC, 100
        CALL    DELAY_MS
        LD      HL, SPK_ROBOT
        CALL    SPEAK_WORD
        ; Return to stop after demo
        LD      A, 00h
        LD      (CUR_MODE), A
        LD      A, MTR_STOP
        CALL    SET_MOTORS
        RET

;-------------------------------------------------------------------
; SET_MOTORS
; Input: A = motor byte to write to port 04h
; Preserves: BC, DE, HL
;-------------------------------------------------------------------
SET_MOTORS:
        LD      (MOTOR_STAT), A     ; Save shadow copy
        OUT     (PORT_MOTOR), A     ; Write to latch
        RET

;-------------------------------------------------------------------
; AUTO_STEP
; Called each main loop iteration when in auto mode.
; Measures distance, reacts to obstacles.
;-------------------------------------------------------------------
AUTO_STEP:
        CALL    MEASURE_DIST        ; Result in A (cm), also in (DIST_CM)
        LD      (DIST_CM), A

        CP      OBSTACLE_CM         ; Compare distance with threshold
        JR      NC, AUTO_CLEAR      ; Jump if distance >= 20cm (no obstacle)

        ; Obstacle detected! Announce and take evasive action
        CALL    BEEP_ONCE           ; Alert beep
        LD      HL, SPK_DANGER      ; Say "DANGER"
        CALL    SPEAK_WORD

        ; Back up for 400ms
        LD      A, MTR_BACK
        CALL    SET_MOTORS
        LD      BC, 400
        CALL    DELAY_MS

        ; Stop briefly
        LD      A, MTR_STOP
        CALL    SET_MOTORS
        LD      BC, 100
        CALL    DELAY_MS

        ; Turn right for 350ms (arbitrary; tune to ~90 degrees on your floor)
        LD      A, MTR_RIGHT
        CALL    SET_MOTORS
        LD      BC, 350
        CALL    DELAY_MS

        ; Stop briefly
        LD      A, MTR_STOP
        CALL    SET_MOTORS
        LD      BC, 100
        CALL    DELAY_MS

        ; Resume forward
        LD      A, MTR_FWD
        CALL    SET_MOTORS
        RET

AUTO_CLEAR:
        ; Path clear - ensure we are driving forward
        LD      A, MTR_FWD
        CALL    SET_MOTORS
        RET

;-------------------------------------------------------------------
; MEASURE_DIST
; Bit-bangs HC-SR04 to measure distance.
; Returns: A = distance in cm (capped at 255cm)
; Also stores result in (DIST_CM)
; Clobbers: A, B, C, DE, HL
;
; Timing at 3.5MHz:
;   Trigger high pulse must be >= 10us
;   We hold it high for ~11us (TRIG_LOOPS * 13 T-states * 286ns)
;   Echo pulse width in us / 58 = distance in cm
;   We count loop iterations while echo is high.
;   Each loop is 16 T-states (~4.57us).
;   At 58us/cm, that is 58/4.57 ~= 12.7 iterations per cm.
;   We divide loop count by 13 to get cm (close enough for obstacle work).
;-------------------------------------------------------------------
MEASURE_DIST:
        ; Read current motor shadow so we don't disturb motor bits
        LD      A, (MOTOR_STAT)
        AND     0EFh                ; Clear trigger bit (bit 4) just in case
        LD      C, A                ; C = motor state without trigger

        ; Send 10us+ trigger pulse: set bit 4, write, delay, clear bit 4, write
        LD      A, C
        OR      TRIG_BIT            ; Set trigger high
        OUT     (PORT_MOTOR), A

        ; Delay ~11us: DJNZ loop, 13 T-states each at 3.5MHz
        LD      B, TRIG_LOOPS
TRIG_DLY:
        DJNZ    TRIG_DLY            ; 3 iterations * 13 T-states = 39 T-states ~= 11us

        ; Clear trigger
        LD      A, C
        OUT     (PORT_MOTOR), A     ; Trigger low, motors unchanged

        ; Wait for echo to go HIGH (start of pulse)
        ; Timeout after ~65535 iterations to avoid hanging on sensor failure
        LD      DE, 0FFFFh          ; Timeout counter
WAIT_ECHO_HI:
        IN      A, (PORT_ECHO)
        AND     ECHO_BIT
        JR      NZ, ECHO_HIGH       ; Echo has gone high - start counting
        DEC     DE
        LD      A, D
        OR      E
        JR      NZ, WAIT_ECHO_HI    ; Not timed out yet
        ; Timeout - sensor not responding, return 0
        XOR     A
        LD      (DIST_CM), A
        RET

ECHO_HIGH:
        ; Echo is now high - count how long it stays high
        ; Each loop iteration: IN(11) + AND(7) + JR(12 not taken) + INC(6) = ~36 T-states
        ; = ~10.3us per iteration. At 58us/cm, ~5.6 iterations per cm.
        ; We use HL as our 16-bit counter.
        LD      HL, 0000h           ; Reset pulse-width counter
ECHO_COUNT:
        IN      A, (PORT_ECHO)      ; 11 T-states (IN A,(n))
        AND     ECHO_BIT            ;  7 T-states
        JR      Z, ECHO_DONE        ; 12 T-states (not taken) or 7 (taken)
        INC     HL                  ;  6 T-states
        LD      A, H                ;  4 T-states  -- overflow guard
        CP      0FFh                ;  7 T-states
        JR      Z, ECHO_DONE        ; Exit if count would overflow (>6m range)
        JR      ECHO_COUNT          ; 12 T-states

ECHO_DONE:
        ; HL = raw count. Divide by 6 to get approximate cm.
        ; (loop body ~42 T-states = ~12us; 2*12us/cm ~= 1 iteration per 0.17cm
        ;  but round trip at 343m/s: 58us/cm, so divisor = 58/12 ~= 5)
        ; We divide HL by 5 using repeated subtraction for simplicity.
        ; For a more accurate result, multiply HL by 343 and divide by
        ; (2 * loop_T_states_in_us * 1000) but that needs 32-bit maths.
        ; Divisor of 5 gives good results in practice; calibrate by
        ; measuring a known distance and adjusting.

        ; Convert HL to cm: A = HL / 5 (result fits in 8 bits for <255cm)
        LD      B, 0                ; Result accumulator (cm count)
        LD      DE, 5               ; Divisor
DIV_LOOP:
        LD      A, H
        OR      L
        JR      Z, DIV_DONE         ; HL=0, we are done
        LD      A, L
        SUB     E                   ; Subtract low byte of divisor
        LD      L, A
        LD      A, H
        SBC     A, D                ; Subtract high byte with borrow
        LD      H, A
        JR      C, DIV_DONE         ; Underflow - done
        INC     B                   ; One more cm
        LD      A, B
        CP      0FFh                ; Cap at 255cm
        JR      NZ, DIV_LOOP
DIV_DONE:
        LD      A, B
        LD      (DIST_CM), A
        RET

;-------------------------------------------------------------------
; UPDATE_DISPLAY
; Reads CUR_MODE and DIST_CM and writes appropriate patterns
; into JMON's display buffer, then calls JMON scan to show them.
;-------------------------------------------------------------------
UPDATE_DISPLAY:
        LD      A, (CUR_MODE)

        CP      00h
        JP      Z, DISP_STOP

        CP      01h
        JP      Z, DISP_FORWARD

        CP      02h
        JP      Z, DISP_BACK

        CP      03h
        JP      Z, DISP_LEFT

        CP      04h
        JP      Z, DISP_RIGHT

        CP      0Ah
        JP      Z, DISP_AUTO

        CP      0Bh
        JP      Z, DISP_BEEP_MSG

        ; Unknown mode - show dashes
        CALL    DISP_DASHES
        JP      DISP_SCAN

DISP_STOP:
        ; Show " StOP " - using seg patterns
        LD      IX, DISP_BUF
        LD      (IX+0), SEG_BLANK   ; Rightmost digit
        LD      (IX+1), SEG_BLANK
        LD      (IX+2), SEG_P
        LD      (IX+3), SEG_O
        LD      (IX+4), SEG_T
        LD      (IX+5), SEG_S      ; Leftmost digit
        JP      DISP_SCAN

DISP_FORWARD:
        ; Show "FOrUAd" (Forward)
        LD      IX, DISP_BUF
        LD      (IX+0), SEG_BLANK
        LD      (IX+1), SEG_D
        LD      (IX+2), SEG_A
        LD      (IX+3), SEG_R_LC
        LD      (IX+4), SEG_O
        LD      (IX+5), SEG_F
        JP      DISP_SCAN

DISP_BACK:
        ; Show " bACK " (Back)
        LD      IX, DISP_BUF
        LD      (IX+0), SEG_BLANK
        LD      (IX+1), SEG_BLANK
        LD      (IX+2), SEG_K
        LD      (IX+3), SEG_C
        LD      (IX+4), SEG_A
        LD      (IX+5), SEG_B_LC
        JP      DISP_SCAN

DISP_LEFT:
        ; Show "  LEFt " (Left)
        LD      IX, DISP_BUF
        LD      (IX+0), SEG_BLANK
        LD      (IX+1), SEG_BLANK
        LD      (IX+2), SEG_T
        LD      (IX+3), SEG_F
        LD      (IX+4), SEG_E
        LD      (IX+5), SEG_L
        JP      DISP_SCAN

DISP_RIGHT:
        ; Show " rIGHt" (Right)
        LD      IX, DISP_BUF
        LD      (IX+0), SEG_BLANK
        LD      (IX+1), SEG_T
        LD      (IX+2), SEG_H
        LD      (IX+3), SEG_G
        LD      (IX+4), SEG_I
        LD      (IX+5), SEG_R_LC
        JP      DISP_SCAN

DISP_AUTO:
        ; Auto mode: show distance in cm, right-justified, with "A " prefix
        ; Format: "A  ##cm" - but we only have 6 digits so "A  ##  " or
        ; show raw cm value right-justified with leading spaces
        LD      A, (DIST_CM)
        CALL    BIN_TO_BCD          ; Converts A to BCD: H=tens, L=units
        PUSH    HL
        LD      IX, DISP_BUF
        ; Rightmost 2 digits = distance
        POP     HL
        LD      A, L                ; Units digit
        CALL    GET_SEG             ; Get 7-seg pattern for digit
        LD      (IX+0), A
        LD      A, H                ; Tens digit
        CALL    GET_SEG
        LD      (IX+1), A
        ; Hundreds (for >99cm)
        LD      A, (DIST_CM)
        CP      100
        JR      C, AUTO_NO_HUND
        LD      (IX+2), SEG_1       ; Show '1' for hundreds
        JR      AUTO_HUND_DONE
AUTO_NO_HUND:
        LD      (IX+2), SEG_BLANK
AUTO_HUND_DONE:
        LD      (IX+3), SEG_BLANK
        LD      (IX+4), SEG_BLANK
        LD      (IX+5), SEG_A       ; 'A' for Auto
        JP      DISP_SCAN

DISP_BEEP_MSG:
        ; Show "bEEP  "
        LD      IX, DISP_BUF
        LD      (IX+0), SEG_BLANK
        LD      (IX+1), SEG_BLANK
        LD      (IX+2), SEG_P
        LD      (IX+3), SEG_E
        LD      (IX+4), SEG_E
        LD      (IX+5), SEG_B_LC
        JP      DISP_SCAN

DISP_ROBOT:
        ; Show "robOt " startup message
        LD      IX, DISP_BUF
        LD      (IX+0), SEG_BLANK
        LD      (IX+1), SEG_BLANK
        LD      (IX+2), SEG_T
        LD      (IX+3), SEG_O
        LD      (IX+4), SEG_B_LC
        LD      (IX+5), SEG_R_LC
        CALL    DISP_SCAN
        RET

DISP_DASHES:
        LD      IX, DISP_BUF
        LD      B, 6
        LD      A, SEG_DASH
DISP_FILL_DASH:
        LD      (IX+0), A
        INC     IX
        DJNZ    DISP_FILL_DASH
        RET

DISP_SCAN:
        CALL    JMON_SCAN           ; Call JMON to refresh display
        RET

;-------------------------------------------------------------------
; BIN_TO_BCD
; Input:  A = binary value 0-255
; Output: H = tens digit (0-9), L = units digit (0-9)
;         (Hundreds ignored for values >= 100; caller handles that)
; Clobbers: A, B, C, H, L
;-------------------------------------------------------------------
BIN_TO_BCD:
        LD      C, A                ; Save original value
        LD      H, 0                ; Tens = 0
        ; Subtract 100 if >= 100 (we ignore hundreds in H output)
        CP      100
        JR      C, BCD_NO_HUND
        SUB     100
        ; If still >=100, subtract again (max 255, so max 2 subtractions)
        CP      100
        JR      C, BCD_NO_HUND
        SUB     100
BCD_NO_HUND:
        LD      C, A                ; C = value mod 100
        LD      H, 0                ; Tens counter
BCD_TENS_LOOP:
        LD      A, C
        CP      10
        JR      C, BCD_TENS_DONE
        SUB     10
        LD      C, A
        INC     H
        JR      BCD_TENS_LOOP
BCD_TENS_DONE:
        LD      L, C                ; L = units
        RET

;-------------------------------------------------------------------
; GET_SEG
; Input:  A = decimal digit 0-9
; Output: A = 7-segment pattern for that digit
; Uses lookup table SEG_TABLE
;-------------------------------------------------------------------
GET_SEG:
        LD      HL, SEG_TABLE
        LD      C, A
        LD      B, 0
        ADD     HL, BC
        LD      A, (HL)
        RET

;-------------------------------------------------------------------
; KEY_CHECK
; Non-blocking keypad scan.
; Returns A = key code (0-Fh) if a key is currently pressed,
;         A = 0FFh if no key pressed.
; This implementation calls JMON_KEY which is blocking on standard
; JMON. For a non-blocking version, we implement a simple scan
; by watching the keyboard port directly.
;
; NOTE: On the standard TEC-1, the keypad scan is handled by JMON's
; interrupt routine. JMON stores the last key pressed in a RAM cell
; at a known address (vary by version). We poll that cell and clear
; it after reading. Define JMON_LASTKEY to match your JMON version.
; Common addresses: 0834h (check your JMON source).
;-------------------------------------------------------------------
JMON_LASTKEY    EQU 0834h       ; Address of JMON's last-key-pressed store

KEY_CHECK:
        LD      A, (JMON_LASTKEY)   ; Read JMON's key buffer
        CP      0FFh                ; FFh means no key waiting
        RET     Z                   ; Return FFh if no key

        ; A key is available - clear the buffer and return key code
        PUSH    AF                  ; Save key code
        LD      A, 0FFh
        LD      (JMON_LASTKEY), A   ; Clear key buffer
        POP     AF
        RET                         ; Return key code in A

;-------------------------------------------------------------------
; DELAY_MS
; Input: BC = number of milliseconds to delay
; At 3.5MHz, calibrated for ~1ms per outer iteration.
; Each inner DJNZ loop: 13 T-states * 175 iterations = 2275 T-states
; Plus overhead ~25 T-states per outer loop.
; Total per outer loop ~2300 T-states = ~657us at 3.5MHz.
; So BC*2 gives approximately BC milliseconds.
; We double the outer count to compensate.
; Clobbers: A, BC, DE
;-------------------------------------------------------------------
DELAY_MS:
        PUSH    BC
        POP     DE                  ; DE = ms count
DLYMS_OUTER:
        LD      B, DLY_INNER        ; Inner loop count (175)
DLYMS_INNER:
        DJNZ    DLYMS_INNER         ; 13 T-states * 175 = 2275 T-states ~= 650us
        ; One more inner pass for second half of 1ms
        LD      B, DLY_INNER
DLYMS_INNER2:
        DJNZ    DLYMS_INNER2        ; Another 650us
        DEC     DE                  ; Decrement ms counter
        LD      A, D
        OR      E
        JR      NZ, DLYMS_OUTER     ; Loop until DE = 0
        RET

;-------------------------------------------------------------------
; BEEP_ONCE
; Sounds a short beep on the TEC-1 buzzer (port 01h bit 0)
;-------------------------------------------------------------------
BEEP_ONCE:
        LD      B, 100              ; Number of half-cycles
BEEP_LOOP:
        LD      A, 01h
        OUT     (PORT_BUZZ), A      ; Buzzer on
        LD      C, 20
BEEP_DLY1:
        DEC     C
        JR      NZ, BEEP_DLY1

        XOR     A
        OUT     (PORT_BUZZ), A      ; Buzzer off
        LD      C, 20
BEEP_DLY2:
        DEC     C
        JR      NZ, BEEP_DLY2

        DJNZ    BEEP_LOOP
        RET

;-------------------------------------------------------------------
; BEEP_DEMO
; Plays a short musical fanfare - ascending tones
;-------------------------------------------------------------------
BEEP_DEMO:
        LD      D, 0C0h             ; Tone 1 period (lower = higher pitch)
        CALL    PLAY_TONE
        LD      D, 080h             ; Tone 2
        CALL    PLAY_TONE
        LD      D, 060h             ; Tone 3
        CALL    PLAY_TONE
        LD      D, 040h             ; Tone 4 (highest)
        CALL    PLAY_TONE
        RET

PLAY_TONE:
        ; D = half-period value (approx)
        LD      B, 80h              ; Number of cycles
TONE_LOOP:
        LD      A, 01h
        OUT     (PORT_BUZZ), A
        LD      C, D
TONE_HI:
        DEC     C
        JR      NZ, TONE_HI
        XOR     A
        OUT     (PORT_BUZZ), A
        LD      C, D
TONE_LO:
        DEC     C
        JR      NZ, TONE_LO
        DJNZ    TONE_LOOP
        RET

;-------------------------------------------------------------------
; SPEAK_WORD
; Speaks a sequence of allophones through the SPO256-AL2.
;
; Input:  HL = pointer to allophone data, terminated by 0FFh
;
; Operation: For each allophone byte, the Z80 executes OUT (05h), A.
; This pulses the SPO256 ALD line via the 74HC138 Y5 output.
; The SPO256 asserts /WAIT low while speaking, automatically stalling
; the Z80 until the allophone completes. No delay loops are required.
;
; IMPORTANT: SE pin 19 of the SPO256 MUST be tied to +5V.
; Without this, data bus glitches can retrigger the chip mid-word.
;-------------------------------------------------------------------
SPEAK_WORD:
        LD      A, (HL)         ; Get next allophone code
        CP      0FFh            ; End-of-sequence marker?
        RET     Z               ; Yes, return to caller
        OUT     (PORT_SPO), A   ; Send to SPO256 — Z80 halts on /WAIT until spoken
        INC     HL              ; Advance to next allophone
        JR      SPEAK_WORD      ; Loop

;-------------------------------------------------------------------
; ALLOPHONE TABLES
;
; Each word is a sequence of SPO256-AL2 allophone codes, terminated
; by 0FFh. Codes are from the SPO256-AL2 datasheet allophone set.
;
; Key allophone codes used here:
;   00h=PA1(10ms) 01h=PA2(30ms) 02h=PA3(50ms) 03h=PA4(100ms)
;   04h=PA5(200ms)
;   06h=AY  07h=EH  09h=PP  0Dh=TT2  0Eh=RR1  0Fh=AX
;   10h=MM  11h=TT1 13h=IY  14h=EY   17h=AO   18h=AA
;   1Ah=AE  1Bh=HH1 1Ch=BB1 21h=DD2  22h=GG3  27h=RR2
;   28h=FF  29h=KK2 2Ah=KK1 2Ch=NG   2Dh=LL   2Eh=WW
;   33h=ER1 35h=OW  37h=SS  38h=NN2  39h=HH2  3Ah=OR
;   3Bh=AR  3Fh=BB2
;-------------------------------------------------------------------

SPK_HELLO:
        ; H-EH-L-OW  ("hello")
        DEFB    1Bh             ; HH1  (H)
        DEFB    07h             ; EH
        DEFB    2Dh             ; LL   (L)
        DEFB    35h             ; OW   (oh)
        DEFB    0FFh

SPK_ROBOT:
        ; R-OW-B-AH-T  ("robot")
        DEFB    0Eh             ; RR1  (R)
        DEFB    35h             ; OW   (oh)
        DEFB    3Fh             ; BB2  (B)
        DEFB    18h             ; AA   (ah)
        DEFB    0Dh             ; TT2  (T)
        DEFB    0FFh

SPK_FORWARD:
        ; F-OR-W-ER-D  ("forward")
        DEFB    28h             ; FF   (F)
        DEFB    3Ah             ; OR
        DEFB    2Eh             ; WW   (W)
        DEFB    33h             ; ER1
        DEFB    21h             ; DD2  (D)
        DEFB    0FFh

SPK_BACK:
        ; B-AE-K  ("back")
        DEFB    3Fh             ; BB2  (B)
        DEFB    1Ah             ; AE   (a as in hat)
        DEFB    2Ah             ; KK1  (K)
        DEFB    0FFh

SPK_LEFT:
        ; L-EH-F-T  ("left")
        DEFB    2Dh             ; LL   (L)
        DEFB    07h             ; EH
        DEFB    28h             ; FF   (F)
        DEFB    0Dh             ; TT2  (T)
        DEFB    0FFh

SPK_RIGHT:
        ; R-AY-T  ("right")
        DEFB    0Eh             ; RR1  (R)
        DEFB    06h             ; AY   (eye)
        DEFB    0Dh             ; TT2  (T)
        DEFB    0FFh

SPK_STOP:
        ; S-T-AO-P  ("stop")
        DEFB    37h             ; SS   (S)
        DEFB    11h             ; TT1  (T)
        DEFB    17h             ; AO   (aw)
        DEFB    09h             ; PP   (P)
        DEFB    0FFh

SPK_AUTO:
        ; AO-T-OW  ("auto")
        DEFB    17h             ; AO   (aw)
        DEFB    0Dh             ; TT2  (T)
        DEFB    35h             ; OW   (oh)
        DEFB    0FFh

SPK_DANGER:
        ; D-EY-NG-ER  ("danger")
        DEFB    21h             ; DD2  (D)
        DEFB    14h             ; EY   (a as in hay)
        DEFB    2Ch             ; NG
        DEFB    33h             ; ER1
        DEFB    0FFh

SPK_I:
        ; AY  ("I")
        DEFB    06h             ; AY
        DEFB    0FFh

SPK_AM:
        ; AE-M  ("am")
        DEFB    1Ah             ; AE
        DEFB    10h             ; MM
        DEFB    0FFh

SPK_YOUR:
        ; Y-OR  ("your")
        DEFB    19h             ; YY2  (Y)
        DEFB    3Ah             ; OR
        DEFB    0FFh

;-------------------------------------------------------------------
; 7-SEGMENT LOOKUP TABLES AND CHARACTER CONSTANTS
;
; Standard common-cathode 7-segment encoding:
;   Bit 7 = DP (decimal point, unused here)
;   Bit 6 = G (middle bar)
;   Bit 5 = F (top-left)
;   Bit 4 = E (bottom-left)
;   Bit 3 = D (bottom)
;   Bit 2 = C (bottom-right)
;   Bit 1 = B (top-right)
;   Bit 0 = A (top)
;
;   Segment layout:
;        A
;       ---
;   F  |   |  B
;       ---   <- G
;   E  |   |  C
;       ---
;        D
;
; NOTE: Check your TEC-1's display polarity. If segments appear
; inverted (dark where bright expected), XOR each value with 7Fh.
;-------------------------------------------------------------------

SEG_TABLE:
        ; Digits 0-9
        DEFB    3Fh         ; '0' = ABCDEF    = 0011 1111
        DEFB    06h         ; '1' = BC        = 0000 0110
        DEFB    5Bh         ; '2' = ABDEG     = 0101 1011
        DEFB    4Fh         ; '3' = ABCDG     = 0100 1111
        DEFB    66h         ; '4' = BCFG      = 0110 0110
        DEFB    6Dh         ; '5' = ACDFG     = 0110 1101
        DEFB    7Dh         ; '6' = ACDEFG    = 0111 1101
        DEFB    07h         ; '7' = ABC       = 0000 0111
        DEFB    7Fh         ; '8' = ABCDEFG   = 0111 1111
        DEFB    6Fh         ; '9' = ABCDFG    = 0110 1111

; Individual character constants (for assembler EQU use below)
SEG_0       EQU 3Fh
SEG_1       EQU 06h
SEG_2       EQU 5Bh
SEG_3       EQU 4Fh
SEG_4       EQU 66h
SEG_5       EQU 6Dh
SEG_6       EQU 7Dh
SEG_7       EQU 07h
SEG_8       EQU 7Fh
SEG_9       EQU 6Fh

; Letters used in display messages
SEG_A       EQU 77h     ; A  = ABCEFG   = 0111 0111
SEG_B_LC    EQU 7Ch     ; b  = CDEFG    = 0111 1100  (lowercase b)
SEG_C       EQU 39h     ; C  = ADEF     = 0011 1001
SEG_D       EQU 5Eh     ; d  = BCDEG    = 0101 1110  (lowercase d)
SEG_E       EQU 79h     ; E  = ADEFG    = 0111 1001
SEG_F       EQU 71h     ; F  = AEFG     = 0111 0001
SEG_G       EQU 3Dh     ; G  = ACDEF    = 0011 1101
SEG_H       EQU 76h     ; H  = BCEFG    = 0111 0110
SEG_I       EQU 06h     ; I  = BC       = 0000 0110  (same as '1')
SEG_K       EQU 76h     ; K  = approx H
SEG_L       EQU 38h     ; L  = DEF      = 0011 1000
SEG_O       EQU 3Fh     ; O  = ABCDEF   = same as '0'
SEG_P       EQU 73h     ; P  = ABEFG    = 0111 0011
SEG_R_LC    EQU 50h     ; r  = EG       = 0101 0000  (lowercase r)
SEG_S       EQU 6Dh     ; S  = same as '5'
SEG_T       EQU 78h     ; t  = DEFG     = 0111 1000  (lowercase t)
SEG_DASH    EQU 40h     ; -  = G only   = 0100 0000
SEG_BLANK   EQU 00h     ; (blank)       = 0000 0000

;-------------------------------------------------------------------
; END OF PROGRAM
; Total size approximately 400 bytes.
; Remaining RAM from ~4190h to 47FFh free for stack and variables.
;-------------------------------------------------------------------

        END     ROBOT_START
```

Save this listing as `robot.asm`. Assemble it with your chosen cross-assembler. If using PASMO, the command is `pasmo --hex robot.asm robot.hex`, which produces an Intel HEX file you can then load into JMON using the TEC-1's monitor. If you prefer to enter the code manually, assemble it on paper first (or use a Z80 opcode table), then enter the hex bytes via JMON's DATA entry mode starting at address 4000h.

---

## A Guided Tour of the Code

### The Main Loop

The main loop at `MAIN_LOOP` does three things on every pass: it refreshes the display, checks for a new keypress, and if the robot is in autonomous mode, runs one step of the obstacle-avoidance behaviour. The non-blocking key check works by reading the variable `JMON_LASTKEY` — a RAM cell that JMON's keyboard scan interrupt routine updates whenever a key is pressed — and clearing it after reading. If no key has been pressed since the last check, the cell holds 0FFh and we skip the dispatch. This gives us responsive manual control without the main loop blocking on a key.

### Motor Control

`SET_MOTORS` is deliberately simple: it takes the desired motor byte in register A, saves a shadow copy to `MOTOR_STAT` in RAM, and writes the byte to port 04h. The shadow copy is essential for the ultrasonic routine, which needs to toggle bit 4 (the trigger) without disturbing the motor bits. By reading `MOTOR_STAT` and OR-ing in the trigger bit before the OUT instruction, we avoid inadvertently stopping the motors while taking a ranging measurement.

### The Ultrasonic Ranging Routine

`MEASURE_DIST` is the most timing-sensitive code in the project and deserves careful attention. The HC-SR04 requires the trigger line to be held high for at least 10 microseconds. We achieve this with a DJNZ loop: `TRIG_LOOPS EQU 3` gives three iterations of a 13-T-state DJNZ, which at 3.5 MHz equals approximately 11 microseconds. This is comfortably within the sensor's specification.

After the trigger pulse, the routine waits for the echo line to go high, then counts machine cycles while it remains high. The echo pulse width in microseconds, divided by 58, gives distance in centimetres. Our counting loop runs at approximately 36 T-states per iteration — about 10.3 microseconds — so we divide the raw loop count by 5 or 6 to obtain centimetres. The listing uses 5 as the divisor; measure a ruler-verified distance of exactly 30cm during calibration and adjust the divisor until `DIST_CM` reads 30. In practice, values between 4 and 7 all give usable results, and for obstacle avoidance at the 20cm threshold, precision to within a few centimetres is entirely adequate.

The division itself uses repeated subtraction. This is slow for large distances but the HC-SR04's maximum range is about 4 metres, giving a maximum echo time of approximately 23,000 microseconds and about 2,200 loop iterations. Dividing by 5 via repeated subtraction takes at most 440 iterations, each consuming about 25 T-states — roughly 3 milliseconds total. That is acceptable latency for a robot moving at walking pace.

### The Display Routines

The TEC-1's six-digit display is controlled by JMON's interrupt-driven scan routine, which reads six bytes from a buffer in low RAM and maps each byte directly to the seven-segment display hardware via the 74LS374 output latch and 74LS138 digit selector. By writing new patterns into that buffer and then calling `JMON_SCAN` at address 0010h to force an immediate refresh, we update the display without needing to write our own multiplexing code.

The seven-segment encoding follows the standard common-cathode convention: bit 0 is segment A (the top horizontal bar), bit 1 is B (top right), and so on. The character constants `SEG_A` through `SEG_T` at the end of the listing provide patterns for all the letters needed by the mode display strings. Note that seven-segment displays cannot render every letter unambiguously — we use lowercase `b`, `d`, `r`, and `t` which look distinctly different from their digit counterparts, and accept that `K` looks identical to `H`. The message `FOrUAd` for Forward, `bACK` for Back, `LEFt` for Left, and `rIGHt` for Right are all recognisable with a moment's familiarity.

In auto mode, the display shows the current distance in centimetres on the right two digits, with the letter `A` on the leftmost digit to identify the mode. The `BIN_TO_BCD` routine separates a binary distance value into tens and units digits for display.

### Auto-Avoid Mode

The auto-avoidance behaviour in `AUTO_STEP` runs as a simple decision tree. On each call from the main loop, it fires the ultrasonic sensor and reads the result. If the path is clear — distance 20cm or more — it ensures the motors are set to full forward and returns. If an obstacle is detected, it runs a fixed evasion sequence: beep once to announce the obstacle, reverse for 400 milliseconds, pause for 100 milliseconds, pivot right for 350 milliseconds, pause again, then resume forward. The turn duration is tuned for a roughly 90-degree pivot on a smooth floor; on carpet you may need to increase it to 500 or 600 milliseconds. Experiment freely.

The main loop calls `AUTO_STEP` on every iteration, but because each step including the ultrasonic measurement takes only a few milliseconds, the display update and key check also run frequently. If you press any key while in auto mode, the key dispatch runs immediately and can override the mode — pressing 0 stops the robot even mid-evasion, which makes for a useful emergency stop.

---

## Testing and Calibration

### Stage 1: Latch and Motor Driver Test

Before loading the full program, verify your hardware additions independently. Enter this tiny test program at address 4100h using JMON:

```
4100: 3E 05   LD A, 05h      ; Left fwd + right fwd
4102: D3 04   OUT (04h), A   ; Write to motor latch
4104: 76      HALT
```

Press GO with the address set to 4100h. The robot should drive forward until you press RESET. If nothing happens, check your ribbon cable connections, verify the NAND gate wiring with a logic probe or oscilloscope, and confirm the 74HC574 is receiving a clock pulse on pin 11 during the OUT instruction. A logic probe on the 74HC574 outputs should show them changing state in synchrony with A being written to port 04h.

### Stage 2: Ultrasonic Sensor Test

With the motor wires disconnected, enter this test program at 4100h:

```
4100: CD 00 42   CALL MEASURE_DIST  ; Call ranging routine at 4200h
                                     ; (adjust address to match your assembled code)
4103: 76         HALT
```

Set a flat obstruction (a book, a wall) exactly 30cm in front of the sensor, run the test, then examine the value at RAM address 4201h (DIST_CM) in JMON's memory view. It should read approximately 1Eh (30 decimal). Adjust the divisor constant in the source and re-assemble if the reading is significantly off.

### Stage 3: Display Test

Enter the full program at 4000h and execute from address 4000h. The display should show `robOt` for two seconds, then switch to ` StOP`. Press key 1: the display should change to `FOrUAd` and the robot should drive forward. Press 0 to stop. Test each key in sequence, confirming both the display message and the appropriate wheel movement.

### Stage 4: Speech Module Test

Before running the full program, verify the SPO256 independently. Enter this test at address 4100h:

```
4100: 21 10 41   LD HL, 4110h   ; Point to allophone data below
4103: 7E         LD A, (HL)     ; Get allophone
4104: FE FF      CP FFh         ; End of table?
4106: 28 04      JR Z, 410Ch    ; Yes, HALT
4108: D3 05      OUT (05h), A   ; Speak — Z80 auto-waits until done
410A: 23         INC HL
410B: 18 F6      JR 4103h       ; Next allophone

410C: 76         HALT

; Allophone data at 4110h — spells "HELLO ROBOT"
4110: 1B 07 2D 35 01 0E 35 3F 18 0D FF
```

Enter GO at 4100h. You should hear a clear "HELLO ROBOT" from the speaker. If you hear garbled output, check the 3.58MHz oscillator is running (probe OSC1 pin 27 — you should see a square wave). If you hear nothing at all, check the BC557 transistor circuit and the LM386 wiring. If the TEC-1 locks up and never reaches HALT, the /WAIT line is held permanently low — check that the BC557 is releasing /WAIT when the SPO256 is silent.

Adjust the 10kΩ volume trimpot to a comfortable listening level.

### Stage 5: Auto Mode Test

Set the robot on a clear floor and press A. It should drive forward. Walk towards it slowly: at 20cm it should beep, reverse, turn, and resume. If the obstacle detection triggers too early or too late, adjust `OBSTACLE_CM` and re-assemble.

---

## Troubleshooting

| Symptom | Probable Cause | Remedy |
|---------|---------------|--------|
| Motors do not move at all | 74HC574 not latching | Check NAND gate wiring; probe 574 CLK pin with logic probe during OUT 04h instruction |
| Motors move but in wrong direction | Motor wires transposed on L293D | Swap the two wires for the affected motor on L293D outputs |
| One motor works, the other does not | Open connection on L293D input or output | Check solder joints on L293D pins 2/7 (left) or 10/15 (right) and corresponding 574 outputs |
| Robot spins in place when FWD commanded | One motor polarity reversed | Swap that motor's output wires on the L293D |
| Display shows garbage in auto mode | DISP_BUF address wrong for your JMON | Check JMON source or documentation; change DISP_BUF EQU to correct address |
| Ultrasonic always reads 0 | Echo not reaching Z80 data bus | Check 74HC244 OE decoding; verify ECHO wire from HC-SR04 to 244 input pin |
| Ultrasonic reads wildly varying values | Electrical noise from motors | Add 100nF cap across each motor; ensure motor ground is common with logic ground |
| Ultrasonic reads consistently too short | Divisor too high | Reduce divisor constant; verify with measured distance |
| Ultrasonic reads consistently too long | Divisor too low | Increase divisor constant |
| Robot does not respond to keypad | JMON_LASTKEY address wrong | Verify address in your JMON ROM; try 0834h, 0836h, or examine JMON source |
| Buzzer makes no sound in BEEP mode | Port 01h wiring or polarity | Confirm buzzer is connected to TEC-1 port 01h bit 0 as per TEC-1 schematic |
| Robot stops responding mid-session | Stack overflow or wrong SP | Ensure SP is initialised to 43FFh at startup; check for unmatched PUSH/POP |
| Display freezes on startup message | JMON_SCAN address incorrect | Try 0008h, 000Ch, or 0010h depending on your JMON version |
| No speech output at all | SPO256 not receiving ALD pulse | Probe Y5 of 74HC138 during OUT 05h — should pulse low briefly |
| TEC-1 locks up when speaking | /WAIT held permanently low | Check BC557 transistor — emitter to +5V, collector to LM386 VS; base to /WAIT via 82kΩ. Verify SPO256 is not stuck in reset |
| Speech is garbled or wrong pitch | 3.58MHz oscillator not running | Probe SPO256 pin 27 with logic probe or scope — should show a square wave at all times |
| Loud buzz from speaker between words | SE pin 19 not tied to +5V | This is the documented errata — solder a wire from SPO256 pin 19 to the +5V rail |
| Words sound clipped at end | Allophone sequence missing PA2 pause before FF end marker | Add a 01h (PA2, 30ms pause) before the 0FFh terminator in affected word tables |
| Volume very low | LM386 gain not set | Confirm 10µF + 1.2kΩ series network between LM386 pins 1 and 8; without this, gain is minimum |
| Speech triggers random extra allophones | SE pin 19 not tied to +5V or data bus noise | Tie SE pin 19 to +5V and add 100nF bypass caps on SPO256 power pins |
| OUT 05h also triggers motor latch | 74HC574 decode not updated for 74HC138 | Verify 74HC574 CLK connects to Y4 (not Y5) of 74HC138; they are adjacent pins, easy to swap |

---

## Going Further

The TEC-1 Robot Rover as described is a solid, working platform, but it is also a starting point. Here are some directions to take it next.

**Speed control with software PWM.** The L293D enable pins are currently tied permanently high, giving full power to both motors. By toggling the enable pins rapidly from software — writing alternating enable/disable bytes to port 04h — you can implement pulse-width modulation for variable speed. Add two more bits to the port output to separately control left and right enable lines, then adjust the duty cycle in the motor command routines. Slow cornering becomes possible without the current abrupt skid turns.

**Line following.** Add a pair of infrared reflectance sensors to the underside of the chassis and connect them to two more bits of the 74HC244 input buffer. A simple line-follower algorithm in assembly — read left sensor, read right sensor, steer to keep both on the line — is a classic robotics exercise and very achievable on the Z80.

**Encoder feedback.** Small Hall-effect encoders fit inside many of the geared motor chassis kits. Counting encoder pulses in an interrupt service routine lets you measure distance and angle travelled precisely, enabling dead-reckoning navigation. You could pre-program routes — "forward 50cm, turn left 90 degrees, forward 100cm" — and have the robot execute them accurately.

**Wireless control.** An HC-05 Bluetooth module or an NRF24L01 transceiver could replace the keypad as the control input. The Z80's serial port (if your TEC-1 version has one) or a bit-banged UART on a spare port bit could communicate with the wireless module, allowing you to drive the robot from a phone or a second TEC-1 acting as a remote control.

**Expand the vocabulary.** The SPO256-AL2 can speak almost any English word given the right allophone sequence. The original *Talking Electronics* article included a Basic Dictionary of pre-worked allophone sequences for hundreds of common words. Your robot could announce the distance in centimetres by speaking individual digit allophones, call out obstacle directions ("TURNING LEFT", "TURNING RIGHT"), or greet visitors by name. Each new phrase is just a new `DEFB` table in the assembly file. The full allophone reference table in that original article lists all 64 codes with sample words — use it to construct any phrase you need.

**Maze solving.** Add two more HC-SR04 modules pointing left and right, decode them to additional bits of the input port, and implement a wall-following or flood-fill maze-solving algorithm. The Z80 is genuinely capable of this: the required data structures fit in a few hundred bytes of RAM, and the algorithms involve nothing more exotic than byte comparisons and indirect memory access via IX and IY.

Whatever direction you choose, the TEC-1 has proved once again that classic hardware never truly retires. Forty-odd years after its original publication, a Z80 running hand-assembled machine code is navigating a living room floor, beeping at obstacles, and displaying its status on a six-digit LED display. That is not nostalgia. That is engineering at its most direct and satisfying.

---

## Appendix: Quick Reference — Port Map

### Port 04h — Motor Driver and Ultrasonic (74HC138 Y4 → 74HC574)

```
OUTPUT (write port 04h):
Bit 7   Bit 6   Bit 5   Bit 4      Bit 3    Bit 2     Bit 1    Bit 0
  |       |       |       |          |        |         |        |
Spare   Spare   Spare  HC-SR04    Right    Right     Left     Left
                       TRIGGER    BACK     FORWARD   BACK     FORWARD

INPUT (read port 04h via 74HC244):
Bit 0 = HC-SR04 ECHO signal
Bits 1-7 = unused (tied low)
```

### Port 05h — SPO256-AL2 Speech (74HC138 Y5 → SPO256 ALD)

```
OUTPUT (write port 05h):
Bits 5–0 = Allophone number (0x00–0x3F, 64 allophones total)
Bits 7–6 = Ignored by SPO256

After OUT (05h), A:
  — SPO256 latches bits D0–D5 as allophone code
  — SPO256 asserts /WAIT low (Z80 freezes automatically)
  — SPO256 synthesises the allophone through LM386 to speaker
  — SPO256 releases /WAIT (Z80 resumes on next instruction)

No read function. Port 05h is write-only.
```

### 74HC138 Decoder Pin Summary

```
Pin  Signal    Connected to
 1   A         Z80 A0
 2   B         Z80 A1
 3   C         Z80 A2
 4   E1 (/G1)  Z80 /IORQ  (active-low enable)
 5   E2 (/G2A) Z80 /WR    (active-low enable)
 6   E3 (G2B)  +5V        (active-high enable, always on)
10   Y5        SPO256 ALD pin 20
11   Y4        74HC574 CLK pin 11
```

## Appendix: Motor Truth Table

| Left FWD (bit0) | Left BACK (bit1) | Right FWD (bit2) | Right BACK (bit3) | Robot Action |
|:-:|:-:|:-:|:-:|---|
| 0 | 0 | 0 | 0 | STOP (coast) |
| 1 | 0 | 1 | 0 | FORWARD |
| 0 | 1 | 0 | 1 | BACKWARD |
| 1 | 0 | 0 | 0 | PIVOT LEFT (right motor off) |
| 0 | 0 | 1 | 0 | PIVOT RIGHT (left motor off) |
| 0 | 1 | 1 | 0 | CURVE LEFT (right fwd, left back) |
| 1 | 0 | 0 | 1 | CURVE RIGHT (left fwd, right back) |
| 1 | 1 | x | x | BRAKE left (avoid — coasts in practice) |

---

*Build notes, corrections, and photos of your completed rover are welcome via the letters page. Particular interest in seeing how you've mounted the TEC-1 on the chassis — there are many creative solutions and every one of them deserves a photograph.*

---

# Part Two: Adding a Brain — LLM Control of the TEC-1 Robot

## Why Give the Robot a Brain?

The robot you built in Part One is satisfying to drive. You press a key, the TEC-1 moves, the SPO256 announces what it is doing, and the seven-segment display confirms it. But every action still comes from a human finger on a keypad. The robot has no initiative, no understanding of what you say to it, and no ability to make decisions beyond the hardwired logic of `AUTO_STEP`. In this part we fix all of that.

We are going to add a language model — a genuine neural network that understands natural English sentences and produces intelligent responses — to the system. You will be able to speak to the robot in plain language: "come here", "what can you see", "turn around and explore", "tell me a joke". The robot will respond through the SPO256 in its own synthesised voice and act on its understanding by sending movement commands back to the TEC-1's Z80. The TEC-1 retains its role as the real-time hardware controller — motors, display, ultrasonic sensor — because that is what a Z80 does best. The new brain module handles all the intelligence.

This is not a trick or a demonstration. The robot will hold short conversations, describe its decisions, adjust its behaviour based on what you ask, and speak coherently while it moves. Getting there requires understanding a few constraints first.

---

## The Problem: LLMs and Microcontrollers Do Not Naturally Fit

A large language model generates text by predicting the next token in a sequence, over and over, until it reaches a natural stopping point. Each prediction involves multiplying a vector of numbers (representing the current context) against an enormous table of learned weights. The table is the model. The bigger the table, the more the model knows and the better it predicts.

GPT-2, the smallest publicly recognised conversational model, has 124 million parameters. At 32-bit floating point precision, that is 496MB of data that must be loaded into RAM before the first token can be generated. An ESP32 has 520KB of internal SRAM — about one thousandth of what GPT-2 needs. A standard Z80 system like the TEC-1 has 8KB. The gap is not bridgeable by clever programming; it is a physics problem.

Two techniques have changed the landscape significantly in recent years. The first is **quantization**: instead of storing each weight as a 32-bit float, you represent it as an 8-bit or even 4-bit integer. This shrinks the model by 4× or 8× with surprisingly little loss of quality for small models. A 135-million-parameter model at 4-bit quantization occupies roughly 70MB. That is still large by microcontroller standards, but it is now within reach of hardware with external flash storage.

The second technique is **streaming inference**: rather than waiting for the entire response to be generated before doing anything with it, you process each token as it arrives. For our robot this is ideal — we can convert each word to allophones and speak it through the SPO256 as the model generates it, giving the robot a natural speaking cadence rather than a long pause followed by a burst of speech.

---

## Three Architectures, One Goal

There are three practical ways to give the robot a language-model brain, ranging from fully offline on a single chip to cloud-connected via WiFi. Understanding the trade-offs will help you choose the right starting point for your build.

### Architecture A — Pure MCU, Tiny Local Model

The ESP32-S3 microcontroller, combined with 16MB of external PSRAM and 16MB of external flash, represents the current practical ceiling for a true microcontroller running a conversational language model. The dual-core Xtensa LX7 running at 240MHz, with its cache-coherent access to external PSRAM, can load and run models from Hugging Face's **SmolLM** family — specifically the 135M parameter variant, which at INT4 quantization fits in approximately 70MB. The `llama.cpp` inference library has been ported to ESP32 and can generate tokens from this model class.

The honest performance figure is roughly one to two tokens per second. A ten-word response takes five to ten seconds to generate. For a robot that is mostly stationary while thinking and then acts decisively, this is usable. For fluid back-and-forth conversation, it feels slow. Think of it as the robot "considering carefully before speaking" — which, given it is a forty-year-old architecture controlled by a chip the size of a fingernail, is quite remarkable.

This architecture requires no internet connection and no additional hardware beyond the ESP32-S3 module. It is fully self-contained. Its limitation is the quality and knowledge of the model: SmolLM-135M is useful for constrained tasks (driving instructions, short factual answers, simple reasoning) but will occasionally produce confused responses on complex topics.

### Architecture B — Small SBC as Brain, MCU Handles I/O

A Raspberry Pi Zero 2W is physically small (65mm × 30mm), costs AU$15, and runs full Linux. Ollama on a Pi Zero 2W can serve a one-billion-parameter model — specifically **llama3.2:1b** or **Qwen2.5-0.5B** — at three to five tokens per second. Response quality is dramatically better than Architecture A. The model knows more, reasons more coherently, and holds context across several exchanges.

In this architecture the Pi Zero 2W is the pure thinking layer. It receives voice input via a small USB microphone, converts speech to text using a lightweight Whisper model, queries the local Ollama server, receives a structured response, converts it to allophone sequences, and sends motor commands to the TEC-1 over a serial UART link. The TEC-1's Z80 continues doing what it was already doing — accepting single-byte motor commands and executing them in real time.

This is the architecture this article recommends for its balance of intelligence, cost, offline operation, and manageable complexity.

### Architecture C — MCU as WiFi Bridge to Cloud LLM

The third approach uses an ESP32 purely as a wireless I/O adaptor. It records your voice, ships the audio to a cloud speech-to-text service (OpenAI Whisper API or similar), passes the resulting text to a cloud language model (GPT-4o-mini, Claude Haiku, or Gemini Flash), receives a structured response, and handles the physical output locally. Response quality is the best available — a full frontier model rather than a billion-parameter local model — and latency is typically one to two seconds over a fast WiFi connection.

The obvious trade-off is the internet dependency and the ongoing API cost. For a workshop demonstration or a desk robot that lives near a router, this works excellently. For a fully autonomous explorer operating in unpredictable environments, it is fragile. A good strategy is to build Architecture C first — the pipeline is simpler to debug when the LLM is reliable and fast — then swap in a local Ollama instance for the final version.

---

## The Recommended Build Plan

This article implements a two-stage approach that mirrors how professional embedded AI systems are developed: get the full pipeline working with cloud infrastructure first, then progressively move intelligence to the device.

**Stage 1** uses an ESP32-S3 as the physical I/O controller and brain interface. It handles the microphone, the serial link to the TEC-1, and the direct connection to the SPO256. For the LLM itself, it calls a cloud API over WiFi. Once Stage 1 is working — the robot speaks, moves, and holds a conversation — Stage 2 replaces the cloud call with a Pi Zero 2W running Ollama locally.

The TEC-1's Z80 code requires only minor additions: a software UART receive routine and a handler that maps incoming command bytes to motor actions. Everything else stays exactly as built in Part One.

---

## How the System Fits Together

```
 ┌─────────────────────────────────────────────────────────┐
 │                    BRAIN MODULE                         │
 │   (Stage 1: ESP32-S3)  or  (Stage 2: Pi Zero 2W)       │
 │                                                         │
 │  [I2S Mic] → STT → LLM → Parse response                │
 │                              │           │              │
 │                         SPEAK text   MOVE cmd           │
 │                              │           │              │
 │               Text-to-allophone    Single byte          │
 │                    converter       'F','B','L',          │
 │                              │     'R','S','A'          │
 │                              │           │              │
 └──────────────────────────────┼───────────┼──────────────┘
                                │           │
                    6x GPIO     │           │ UART TX
                    + ALD pin   │           │ (4800 baud)
                                ↓           ↓
                         ┌──────────┐  ┌────────────────┐
                         │ SPO256   │  │   TEC-1 Z80    │
                         │ -AL2     │  │                │
                         │ + LM386  │  │ UART RX loop   │
                         │ + BC557  │  │ Motor dispatch │
                         └────┬─────┘  │ Display update │
                              │        │ Ultrasonic     │
                           Speaker     │ sensor         │
                                       └───────┬────────┘
                                               │
                                        Motor commands
                                               │
                                       ┌───────↓────────┐
                                       │  L293D + 2WD   │
                                       │    motors      │
                                       └────────────────┘
```

In this architecture the SPO256 is owned entirely by the brain module. The Z80's `SPEAK_WORD` routine and the port 05h speech code from Part One become inactive when the brain is connected — the brain drives the SPO256 directly via its own GPIO lines rather than through the Z80 bus. The TEC-1 handles only motors and display. The brain handles all intelligence and voice output. Neither overlaps with the other.

---

## Additional Parts for the Brain Module

### Stage 1 (ESP32-S3 Cloud Bridge)

| Qty | Description | Approx Cost (AU$) |
|-----|-------------|-------------------|
| 1 | ESP32-S3 development board with 16MB PSRAM (e.g. Unexpected Maker FeatherS3 or LILYGO T7-S3) | $20–35 |
| 1 | INMP441 I2S MEMS microphone breakout | $5–8 |
| 1 | BSS138 N-channel MOSFET level shifter module (3.3V ↔ 5V, 4-channel) | $3 |
| 1 | Micro USB or USB-C cable for ESP32 power and programming | $4 |
| 1 | Small breadboard or stripboard for ESP32 wiring | $3 |
| — | OpenAI, Anthropic, or Groq API key (pay-per-use, very cheap for a robot) | — |
| **Stage 1 total** | | **~AU$35–53** |

### Stage 2 (Pi Zero 2W Local Brain — replaces ESP32 cloud calls)

| Qty | Description | Approx Cost (AU$) |
|-----|-------------|-------------------|
| 1 | Raspberry Pi Zero 2W | $15 |
| 1 | 32GB micro SD card (Class 10) | $10 |
| 1 | USB OTG hub or USB sound card with mic input | $8 |
| 1 | Small USB microphone | $8 |
| 1 | 40-pin GPIO header (solder to Pi if not pre-fitted) | $2 |
| **Stage 2 additions** | | **~AU$43** |

---

## Hardware: Connecting the Brain Module to the TEC-1

### The UART Link

The brain module sends single-byte ASCII commands to the TEC-1 over a 4800-baud serial UART link. 4800 baud is chosen deliberately over the more common 9600 baud: at the TEC-1's 3.5MHz clock frequency, 4800 baud gives a bit period of approximately 208 microseconds, which is 728 Z80 T-states. This gives the bit-bang receive routine comfortable timing margins and makes the code tolerant of minor crystal frequency variations.

The connection is one-way for Stage 1: the brain module transmits to the TEC-1, the TEC-1 does not need to reply. You need only one signal wire between the two, plus a common ground. The command bytes are plain ASCII: `F` (0x46) for forward, `B` (0x42) for backward, `L` (0x4C) for left, `R` (0x52) for right, `S` (0x53) for stop, and `A` (0x41) for autonomous mode.

### Level Shifting

The ESP32-S3 and Pi Zero 2W both use 3.3V logic levels on their UART TX pins. The TEC-1 operates at 5V. A 3.3V signal may not reliably register as a logic high on a 5V CMOS input — the 74HC244 input buffer we are using for the UART RX line requires a minimum high threshold of approximately 3.5V at 5V supply, which 3.3V falls marginally below.

The solution is straightforward: replace the 74HC244 with a **74HCT244**. The HCT variant uses TTL-compatible input thresholds (VIH minimum 2.0V), which means 3.3V from the ESP32 or Pi is read reliably as a logic high by the 5V-powered chip. The pinout is identical; it is a one-part swap. This is the only hardware change needed to the interface board from Part One.

Route the brain module's UART TX pin to the 74HCT244's input pin A2 (which feeds data bus bit 1 on the output). This becomes the TEC-1's serial receive line, readable by the Z80 as bit 1 of the port 04h input.

### The Brain Module's SPO256 Connection

The brain module drives the SPO256 directly using seven GPIO pins: six data lines (D0–D5) and one ALD strobe line. In Stage 2 with the Pi Zero 2W, these connect to the Pi's GPIO header. In Stage 1 with the ESP32-S3, they connect to any available GPIO pins.

Because the SPO256 operates at 5V and the brain module outputs 3.3V, the same level-shifting consideration applies to these lines. The SPO256's data inputs (pins 13–18) and ALD (pin 20) are standard CMOS inputs. A BSS138 four-channel level-shifter module handles this cleanly: one channel per GPIO line, bidirectional, 3.3V on the brain module side, 5V on the SPO256 side. You will need two such modules (eight channels total: six data + one ALD + one spare).

The /WAIT line from the SPO256 is no longer connected to the Z80's /WAIT pin. In the LLM brain architecture, it is connected to the brain module's GPIO instead. The brain module checks the /WAIT signal to know when the SPO256 has finished speaking each allophone before sending the next one. On the ESP32-S3 this is a simple GPIO input read. On the Pi Zero 2W it is a GPIO input with a short polling loop.

Wire the SPO256 /RESET pin to the brain module's reset GPIO (or tie it permanently to +5V if you prefer the chip to stay initialised at all times — acceptable for a robot that is always power-cycled cleanly).

---

## Part One: TEC-1 Z80 Firmware Additions

The TEC-1 needs only two additions to its existing program: a software UART receive routine, and an updated main loop that checks for incoming serial commands alongside the keypad. The motor command handlers, display routines, and ultrasonic code from Part One remain exactly as written.

### The UART Receive Routine

The following routine is non-blocking. It checks whether a start bit is present on bit 1 of port 04h. If the line is idle (bit high), it returns immediately with 0FFh in register A — the same "no key pressed" sentinel the key-check routine uses. If a start bit is detected, it waits out the full byte, validates the stop bit, and returns the received byte in A. This design allows the main loop to poll for serial input on every iteration without stalling.

Add the following constants and routines to the assembly listing from Part One, after the `JMON_LASTKEY` definition:

```asm
; UART receive line: bit 1 of port 04h (74HCT244 input A2)
UART_RX_BIT EQU 02h         ; Bit mask for serial RX (bit 1)

; Baud rate timing at 3.5 MHz, 4800 baud
; Bit period = 1/4800 = 208us = 728 T-states
; DJNZ = 13 T-states per iteration
; Full bit wait: 728 / 13 = ~56 iterations (subtract loop overhead ~15 T-states = 54 net)
; Half bit wait: ~27 iterations (for start bit centering)
UART_FULL_BIT   EQU 54          ; DJNZ count for one full bit period
UART_HALF_BIT_C EQU 27          ; DJNZ count for half bit period (start bit centre)

;-------------------------------------------------------------------
; UART_CHECK
; Non-blocking: check for incoming byte on serial RX line.
; Returns: A = received byte if available
;          A = 0FFh if line idle (no data)
; Clobbers: A, B, C, D
;-------------------------------------------------------------------
UART_CHECK:
        ; Sample the RX line - if high, line is idle
        IN      A, (PORT_ECHO)
        AND     UART_RX_BIT         ; Test bit 1 (RX line)
        RET     NZ                  ; Non-zero = line high = idle. Return 0FFh...
                                    ; Wait: AND result is 02h when high, not 0FFh.
                                    ; We need to return 0FFh for "no data". Fix below.
        ; If we reach here, bit was low = start bit detected.
        ; But first: the AND left A = 00h. We need to distinguish "no data" from
        ; a received 0x00 byte. Use a separate path:
        JR      UART_RECV           ; Receive the byte

        ; No-data return path (entered from above when bit is high):
        ; The RET NZ above returns with A = 02h (not 0FFh).
        ; Callers check A == 0FFh for no-data, so we must ensure that.
        ; Solution: restructure the test:

; *** Corrected version: ***
UART_CHECK:
        IN      A, (PORT_ECHO)
        AND     UART_RX_BIT         ; A = 02h if idle, 00h if start bit
        LD      A, 0FFh             ; Pre-load "no data" sentinel
        JR      NZ, UART_NO_DATA    ; Line was high = idle, return 0FFh
        ; Line was low = start bit detected. Fall through to receive.

UART_RECV:
        ; Centre on start bit: wait half a bit period
        LD      B, UART_HALF_BIT_C
UART_START_DLY:
        DJNZ    UART_START_DLY

        ; Verify start bit is still low (noise rejection)
        IN      A, (PORT_ECHO)
        AND     UART_RX_BIT
        LD      A, 0FFh
        JR      NZ, UART_NO_DATA    ; Was noise, abandon

        ; Receive 8 data bits, LSB first
        ; Strategy: for each bit, wait one full period, sample,
        ; place received bit into carry, rotate into accumulator C.
        ; RR C inserts carry into bit 7 and shifts right — correct for LSB-first.
        LD      C, 00h              ; Received byte accumulator
        LD      D, 8                ; Bit counter

UART_BIT_LOOP:
        ; Wait one full bit period
        LD      B, UART_FULL_BIT
UART_BIT_DLY:
        DJNZ    UART_BIT_DLY

        ; Sample bit 1 of port, convert to carry flag
        IN      A, (PORT_ECHO)
        AND     UART_RX_BIT         ; A = 02h (high) or 00h (low)
        SCF                         ; Set carry = 1
        JR      NZ, UART_BIT_ONE    ; Bit was high, carry stays 1
        CCF                         ; Bit was low: flip carry to 0
UART_BIT_ONE:
        RR      C                   ; Rotate carry into MSB of C (builds byte LSB-first)

        DEC     D
        JR      NZ, UART_BIT_LOOP

        ; Wait for stop bit (one full bit period)
        LD      B, UART_FULL_BIT
UART_STOP_DLY:
        DJNZ    UART_STOP_DLY

        LD      A, C                ; Return received byte in A
        RET

UART_NO_DATA:
        LD      A, 0FFh             ; Return no-data sentinel
        RET
```

### Updating the Main Loop

Add a UART check call to the main loop so that serial commands are processed alongside keypad presses. Replace the `MAIN_LOOP` block in the Part One listing with this updated version:

```asm
MAIN_LOOP:
        CALL    UPDATE_DISPLAY

        ; Check keypad first
        CALL    KEY_CHECK
        CP      0FFh
        JR      Z, CHECK_UART       ; No key — check serial port
        CALL    KEY_DISPATCH        ; Key pressed — handle it
        JR      CHK_AUTO

CHECK_UART:
        ; Check for incoming motor command from brain module
        CALL    UART_CHECK
        CP      0FFh
        JR      Z, CHK_AUTO         ; Nothing received
        CALL    UART_DISPATCH       ; Received a command byte

CHK_AUTO:
        LD      A, (CUR_MODE)
        CP      0Ah
        CALL    Z, AUTO_STEP

        JR      MAIN_LOOP

;-------------------------------------------------------------------
; UART_DISPATCH
; Input: A = received ASCII command byte from brain module
;   'F' (46h) = Forward    'B' (42h) = Backward
;   'L' (4Ch) = Left       'R' (52h) = Right
;   'S' (53h) = Stop       'A' (41h) = Auto mode
;-------------------------------------------------------------------
UART_DISPATCH:
        CP      'F'
        JP      Z, CMD_FORWARD
        CP      'B'
        JP      Z, CMD_BACK
        CP      'L'
        JP      Z, CMD_LEFT
        CP      'R'
        JP      Z, CMD_RIGHT
        CP      'S'
        JP      Z, CMD_STOP
        CP      'A'
        JP      Z, CMD_AUTO
        RET                         ; Unknown command, ignore
```

Note that `UART_DISPATCH` calls the same command handlers as `KEY_DISPATCH`. The motor movement, display update, and (in the original Part One design) speech output all behave identically whether the command came from a keypad press or a serial byte. The robot cannot tell the difference between a human pressing key 1 and an LLM deciding to send `F`. This is exactly as intended.

---

## Part Two: The Brain Module Firmware

### Stage 1 — ESP32-S3 with Cloud LLM (MicroPython)

The following MicroPython script runs on the ESP32-S3. It handles microphone recording, speech-to-text via the cloud Whisper API, LLM query via the cloud chat API, response parsing, allophone generation, SPO256 speech output, and UART motor command transmission to the TEC-1. Install MicroPython on the ESP32-S3 first using the standard esptool flash procedure.

```python
import machine
import time
import urequests
import json
import network
import i2s_audio  # MicroPython I2S library for INMP441

# ---------------------------------------------------------------
# CONFIGURATION
# ---------------------------------------------------------------
WIFI_SSID     = "your-network-name"
WIFI_PASSWORD = "your-password"
OPENAI_KEY    = "sk-..."          # or use Anthropic/Groq key
LLM_MODEL     = "gpt-4o-mini"    # fast and cheap; change as desired

# GPIO pin assignments (adjust to your wiring)
PIN_BUTTON    = 0       # Push-to-talk button (active low)
PIN_SPO_ALD   = 4       # SPO256 ALD strobe (active low pulse)
PIN_SPO_WAIT  = 5       # SPO256 /WAIT output (low while speaking)
PIN_SPO_D0    = 6       # SPO256 data bit 0
PIN_SPO_D1    = 7       # SPO256 data bit 1
PIN_SPO_D2    = 8       # SPO256 data bit 2
PIN_SPO_D3    = 9       # SPO256 data bit 3
PIN_SPO_D4    = 10      # SPO256 data bit 4
PIN_SPO_D5    = 11      # SPO256 data bit 5
PIN_TEC_TX    = 17      # UART TX to TEC-1 RX (via level shifter)

SPO_DATA_PINS = [PIN_SPO_D0, PIN_SPO_D1, PIN_SPO_D2,
                 PIN_SPO_D3, PIN_SPO_D4, PIN_SPO_D5]

# Initialise GPIO
button    = machine.Pin(PIN_BUTTON, machine.Pin.IN, machine.Pin.PULL_UP)
spo_ald   = machine.Pin(PIN_SPO_ALD,  machine.Pin.OUT, value=1)  # idle high
spo_wait  = machine.Pin(PIN_SPO_WAIT, machine.Pin.IN)
spo_data  = [machine.Pin(p, machine.Pin.OUT, value=0) for p in SPO_DATA_PINS]

# UART to TEC-1 at 4800 baud
tec_uart  = machine.UART(1, baudrate=4800, tx=PIN_TEC_TX, rx=-1)

# ---------------------------------------------------------------
# SPO256 DRIVER
# Write a single allophone number to the SPO256 and wait until
# the chip finishes speaking it (/WAIT returns high).
# ---------------------------------------------------------------
def spo_speak_allophone(code):
    # Place 6-bit allophone code on data lines
    for i, pin in enumerate(spo_data):
        pin.value((code >> i) & 1)
    time.sleep_us(2)            # Setup time
    spo_ald.value(0)            # Pulse ALD low to load allophone
    time.sleep_us(5)
    spo_ald.value(1)            # ALD high — SPO256 starts speaking
    # Wait for /WAIT to go low (chip has accepted the allophone)
    timeout = 0
    while spo_wait.value() == 1 and timeout < 10000:
        time.sleep_us(10)
        timeout += 1
    # Now wait for /WAIT to go high (chip has finished speaking)
    timeout = 0
    while spo_wait.value() == 0 and timeout < 100000:
        time.sleep_us(10)
        timeout += 1

def spo_speak_sequence(allophone_list):
    """Speak a list of allophone codes. 0xFF = end marker, skip it."""
    for code in allophone_list:
        if code == 0xFF:
            break
        spo_speak_allophone(code)

# ---------------------------------------------------------------
# TEXT-TO-ALLOPHONE DICTIONARY
# Maps lowercase words to SPO256-AL2 allophone byte sequences.
# Extend this table as needed for your robot's vocabulary.
# Sequences use the allophone codes from the SPO256-AL2 datasheet.
# ---------------------------------------------------------------
ALLOPHONE_DICT = {
    # Robot action words
    "forward":   [0x28, 0x3A, 0x2E, 0x33, 0x21],   # F-OR-W-ER-D
    "backward":  [0x3F, 0x1A, 0x29, 0x2E, 0x33, 0x21],
    "back":      [0x3F, 0x1A, 0x2A],                # B-AE-K
    "left":      [0x2D, 0x07, 0x28, 0x0D],          # L-EH-F-T
    "right":     [0x0E, 0x06, 0x0D],                # R-AY-T
    "stop":      [0x37, 0x11, 0x17, 0x09],          # S-T-AO-P
    "auto":      [0x17, 0x0D, 0x35],                # AO-T-OW
    "danger":    [0x21, 0x14, 0x2C, 0x33],          # D-EY-NG-ER
    # Greetings and conversation
    "hello":     [0x1B, 0x07, 0x2D, 0x35],
    "hi":        [0x1B, 0x06],
    "yes":       [0x19, 0x07, 0x37],
    "no":        [0x38, 0x35],
    "okay":      [0x35, 0x29, 0x14],
    "good":      [0x22, 0x1F, 0x21],
    "bad":       [0x3F, 0x1A, 0x21],
    # Descriptors
    "i":         [0x06],
    "am":        [0x1A, 0x10],
    "your":      [0x19, 0x3A],
    "robot":     [0x0E, 0x35, 0x3F, 0x18, 0x0D],
    "ready":     [0x0E, 0x07, 0x21, 0x13],
    "thinking":  [0x1D, 0x0C, 0x2C, 0x29, 0x0C, 0x2C],
    "moving":    [0x10, 0x16, 0x23, 0x0C, 0x2C],
    "turning":   [0x0D, 0x33, 0x38, 0x0C, 0x2C],
    "stopped":   [0x37, 0x0D, 0x17, 0x09, 0x0D],
    "obstacle":  [0x17, 0x3F, 0x37, 0x11, 0x18, 0x2A, 0x3E],
    "clear":     [0x29, 0x2D, 0x13, 0x27],
    "exploring": [0x07, 0x29, 0x37, 0x09, 0x2D, 0x35, 0x27, 0x0C, 0x2C],
    # Numbers
    "zero":  [0x2B, 0x3C, 0x35],
    "one":   [0x30, 0x0F, 0x0B],
    "two":   [0x0D, 0x1F],
    "three": [0x36, 0x27, 0x13],
    "four":  [0x28, 0x17, 0x17, 0x27],
    "five":  [0x28, 0x06, 0x23],
    "six":   [0x37, 0x0C, 0x29, 0x37],
    "seven": [0x37, 0x07, 0x23, 0x0C, 0x0B],
    "eight": [0x14, 0x11],
    "nine":  [0x38, 0x06, 0x0B],
    "ten":   [0x0D, 0x07, 0x0B],
}

# Short pause allophone between words
PA2 = 0x01   # 30ms pause

def text_to_allophones(text):
    """
    Convert a text string to a flat list of allophone codes.
    Words not in the dictionary are skipped (robot stays silent
    for unknown words rather than producing garbage speech).
    A PA2 pause is inserted between each word.
    """
    words = text.lower().strip().split()
    sequence = []
    for word in words:
        # Strip basic punctuation
        clean = word.strip(".,!?;:'\"")
        if clean in ALLOPHONE_DICT:
            sequence.extend(ALLOPHONE_DICT[clean])
            sequence.append(PA2)    # Brief pause between words
    return sequence

# ---------------------------------------------------------------
# LLM INTERFACE
# Send a user message to the cloud LLM and receive a structured
# response. The system prompt constrains output to SPEAK and MOVE
# fields so we can parse it reliably.
# ---------------------------------------------------------------
SYSTEM_PROMPT = """You are a small tracked robot built around a vintage TEC-1 Z80
single-board computer from 1981. You are curious, friendly, and occasionally
philosophical about being a forty-year-old computer that has learned to think.

You have:
- Two drive wheels (can go forward, backward, left, right, stop)
- An ultrasonic distance sensor pointing forward
- A speech synthesiser (SPO256 chip) that speaks in allophones
- A six-digit seven-segment LED display

Your vocabulary is LIMITED to these words (you must ONLY use words from this list
in your SPEAK field, or the speech chip will be silent):
forward, backward, back, left, right, stop, auto, danger, hello, hi, yes, no,
okay, good, bad, i, am, your, robot, ready, thinking, moving, turning, stopped,
obstacle, clear, exploring, zero, one, two, three, four, five, six, seven,
eight, nine, ten

You MUST respond in EXACTLY this format, no other text:
SPEAK: <your response using only words from the vocabulary list above>
MOVE: <FORWARD|BACKWARD|LEFT|RIGHT|STOP|AUTO|NONE>

Keep SPEAK responses very short (3-8 words). Use NONE for MOVE if no movement needed."""

def query_llm(user_text):
    """Send user text to LLM, return (speak_text, move_command) tuple."""
    try:
        payload = {
            "model": LLM_MODEL,
            "messages": [
                {"role": "system", "content": SYSTEM_PROMPT},
                {"role": "user",   "content": user_text}
            ],
            "max_tokens": 60,
            "temperature": 0.7
        }
        headers = {
            "Content-Type":  "application/json",
            "Authorization": f"Bearer {OPENAI_KEY}"
        }
        resp = urequests.post(
            "https://api.openai.com/v1/chat/completions",
            headers=headers,
            data=json.dumps(payload)
        )
        body = resp.json()
        resp.close()

        content = body["choices"][0]["message"]["content"]
        speak = ""
        move  = "NONE"
        for line in content.splitlines():
            if line.startswith("SPEAK:"):
                speak = line[6:].strip()
            elif line.startswith("MOVE:"):
                move = line[5:].strip().upper()
        return speak, move

    except Exception as e:
        print("LLM error:", e)
        return "i am thinking", "NONE"

# ---------------------------------------------------------------
# MOVE COMMAND DISPATCH
# Send a single ASCII byte to the TEC-1 over UART.
# ---------------------------------------------------------------
MOVE_COMMANDS = {
    "FORWARD":  b'F',
    "BACKWARD": b'B',
    "BACK":     b'B',
    "LEFT":     b'L',
    "RIGHT":    b'R',
    "STOP":     b'S',
    "AUTO":     b'A',
    "NONE":     None,
}

def send_move(move_str):
    cmd = MOVE_COMMANDS.get(move_str)
    if cmd:
        tec_uart.write(cmd)

# ---------------------------------------------------------------
# AUDIO RECORDING
# Record ~3 seconds of audio from the INMP441 I2S microphone.
# Returns raw PCM bytes suitable for the Whisper API.
# ---------------------------------------------------------------
def record_audio(duration_ms=3000):
    audio = i2s_audio.I2S(
        0,
        sck=machine.Pin(40),
        ws=machine.Pin(39),
        sd=machine.Pin(38),
        mode=i2s_audio.I2S.RX,
        bits=16,
        format=i2s_audio.I2S.MONO,
        rate=16000,
        ibuf=4096
    )
    buf = bytearray(duration_ms * 16000 * 2 // 1000)
    audio.readinto(buf)
    audio.deinit()
    return bytes(buf)

def speech_to_text(pcm_bytes):
    """Send PCM audio to Whisper API and return transcribed text."""
    try:
        # Whisper API expects a WAV file. Build a minimal WAV header.
        import struct
        num_samples = len(pcm_bytes) // 2
        wav_header = struct.pack('<4sI4s4sIHHIIHH4sI',
            b'RIFF', 36 + len(pcm_bytes), b'WAVE',
            b'fmt ', 16, 1, 1, 16000, 32000, 2, 16,
            b'data', len(pcm_bytes))
        wav_data = wav_header + pcm_bytes

        resp = urequests.post(
            "https://api.openai.com/v1/audio/transcriptions",
            headers={"Authorization": f"Bearer {OPENAI_KEY}"},
            files={"file": ("audio.wav", wav_data, "audio/wav"),
                   "model": (None, "whisper-1")}
        )
        text = resp.json().get("text", "")
        resp.close()
        return text
    except Exception as e:
        print("STT error:", e)
        return ""

# ---------------------------------------------------------------
# WIFI CONNECTION
# ---------------------------------------------------------------
def connect_wifi():
    wlan = network.WLAN(network.STA_IF)
    wlan.active(True)
    if not wlan.isconnected():
        print("Connecting to WiFi...")
        wlan.connect(WIFI_SSID, WIFI_PASSWORD)
        for _ in range(20):
            if wlan.isconnected():
                break
            time.sleep(0.5)
    print("WiFi:", wlan.ifconfig()[0] if wlan.isconnected() else "FAILED")
    return wlan.isconnected()

# ---------------------------------------------------------------
# MAIN LOOP
# ---------------------------------------------------------------
def main():
    connect_wifi()

    # Startup greeting
    spo_speak_sequence(text_to_allophones("hello i am your robot ready"))
    send_move("STOP")

    print("Robot ready. Press button to speak.")

    while True:
        # Wait for button press (active low)
        if button.value() == 0:
            time.sleep_ms(30)       # Debounce
            if button.value() == 0:
                # Announce we are listening
                spo_speak_sequence([0x0E, 0x13, 0x37, 0x11, 0x38, 0x0C, 0x2C])  # "listening"

                # Record and transcribe
                print("Recording...")
                audio = record_audio(3000)
                user_text = speech_to_text(audio)
                print("Heard:", user_text)

                if user_text:
                    # Query LLM
                    spo_speak_sequence(text_to_allophones("thinking"))
                    speak, move = query_llm(user_text)
                    print(f"SPEAK: {speak}  MOVE: {move}")

                    # Execute movement
                    send_move(move)

                    # Speak response
                    allophones = text_to_allophones(speak)
                    if allophones:
                        spo_speak_sequence(allophones)

            # Wait for button release
            while button.value() == 0:
                time.sleep_ms(10)

        time.sleep_ms(50)

main()
```

Save this file as `main.py` on the ESP32-S3. On first boot it will connect to WiFi, speak "HELLO I AM YOUR ROBOT READY", and then wait for button presses.

---

## Stage 2 — Pi Zero 2W with Local Ollama

Stage 2 replaces the cloud LLM calls with a local Ollama server running on the Pi Zero 2W. The serial connection to the TEC-1 and the GPIO connections to the SPO256 remain identical to Stage 1. Only the Python code changes — specifically the `query_llm()` function, which instead of calling an OpenAI endpoint calls the local Ollama HTTP API on localhost:11434.

### Setting Up the Pi Zero 2W

Flash Raspberry Pi OS Lite (64-bit, Bookworm) to a 32GB micro SD card using Raspberry Pi Imager. Use the Advanced Options to configure your WiFi credentials and enable SSH. After first boot, SSH in and run:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y python3-pip python3-serial portaudio19-dev
pip3 install pyserial RPi.GPIO faster-whisper openai

# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# Pull a small conversational model
ollama pull llama3.2:1b
```

Ollama will download approximately 800MB for the llama3.2:1b model. This takes several minutes over a typical home connection and uses about 1GB of the SD card. After the pull completes, verify it works:

```bash
ollama run llama3.2:1b "Say hello in three words."
```

You should get a coherent short response within five to ten seconds on a Pi Zero 2W. That is your robot's thinking speed.

### Stage 2 Python Script

Save the following as `/home/pi/robot_brain.py`. It is structured identically to the ESP32 MicroPython version but uses Pi GPIO for the SPO256, a USB microphone and `faster-whisper` for on-device STT, and Ollama for the local LLM.

```python
import serial
import time
import json
import requests
import RPi.GPIO as GPIO
from faster_whisper import WhisperModel
import sounddevice as sd
import numpy as np

# ---------------------------------------------------------------
# CONFIGURATION
# ---------------------------------------------------------------
OLLAMA_URL   = "http://localhost:11434/api/chat"
OLLAMA_MODEL = "llama3.2:1b"
TEC_SERIAL   = "/dev/ttyS0"    # Pi Zero 2W hardware UART
TEC_BAUD     = 4800

# GPIO pin assignments (BCM numbering)
PIN_SPO_ALD  = 4
PIN_SPO_WAIT = 17
PIN_SPO_DATA = [27, 22, 10, 9, 11, 5]  # D0-D5

GPIO.setmode(GPIO.BCM)
GPIO.setup(PIN_SPO_ALD,  GPIO.OUT, initial=GPIO.HIGH)
GPIO.setup(PIN_SPO_WAIT, GPIO.IN)
for pin in PIN_SPO_DATA:
    GPIO.setup(pin, GPIO.OUT, initial=GPIO.LOW)

tec = serial.Serial(TEC_SERIAL, TEC_BAUD, timeout=0.1)

# Load Whisper model (tiny = 39MB, fast enough on Pi Zero 2W)
print("Loading Whisper model...")
whisper = WhisperModel("tiny", device="cpu", compute_type="int8")
print("Whisper ready.")

# ---------------------------------------------------------------
# SPO256 DRIVER (same logic as ESP32 version)
# ---------------------------------------------------------------
def spo_speak_allophone(code):
    for i, pin in enumerate(PIN_SPO_DATA):
        GPIO.output(pin, (code >> i) & 1)
    time.sleep(0.000002)
    GPIO.output(PIN_SPO_ALD, GPIO.LOW)
    time.sleep(0.000005)
    GPIO.output(PIN_SPO_ALD, GPIO.HIGH)
    # Wait for /WAIT to go low then high (allophone spoken)
    timeout = time.time() + 1.0
    while GPIO.input(PIN_SPO_WAIT) == 1 and time.time() < timeout:
        time.sleep(0.00001)
    timeout = time.time() + 2.0
    while GPIO.input(PIN_SPO_WAIT) == 0 and time.time() < timeout:
        time.sleep(0.00001)

def spo_speak_sequence(codes):
    for code in codes:
        if code == 0xFF:
            break
        spo_speak_allophone(code)

# ---------------------------------------------------------------
# TEXT-TO-ALLOPHONE (same dictionary as ESP32 version)
# ---------------------------------------------------------------
ALLOPHONE_DICT = {
    "forward":   [0x28, 0x3A, 0x2E, 0x33, 0x21],
    "backward":  [0x3F, 0x1A, 0x29, 0x2E, 0x33, 0x21],
    "back":      [0x3F, 0x1A, 0x2A],
    "left":      [0x2D, 0x07, 0x28, 0x0D],
    "right":     [0x0E, 0x06, 0x0D],
    "stop":      [0x37, 0x11, 0x17, 0x09],
    "auto":      [0x17, 0x0D, 0x35],
    "danger":    [0x21, 0x14, 0x2C, 0x33],
    "hello":     [0x1B, 0x07, 0x2D, 0x35],
    "hi":        [0x1B, 0x06],
    "yes":       [0x19, 0x07, 0x37],
    "no":        [0x38, 0x35],
    "okay":      [0x35, 0x29, 0x14],
    "good":      [0x22, 0x1F, 0x21],
    "bad":       [0x3F, 0x1A, 0x21],
    "i":         [0x06],
    "am":        [0x1A, 0x10],
    "your":      [0x19, 0x3A],
    "robot":     [0x0E, 0x35, 0x3F, 0x18, 0x0D],
    "ready":     [0x0E, 0x07, 0x21, 0x13],
    "thinking":  [0x1D, 0x0C, 0x2C, 0x29, 0x0C, 0x2C],
    "moving":    [0x10, 0x16, 0x23, 0x0C, 0x2C],
    "turning":   [0x0D, 0x33, 0x38, 0x0C, 0x2C],
    "stopped":   [0x37, 0x0D, 0x17, 0x09, 0x0D],
    "clear":     [0x29, 0x2D, 0x13, 0x27],
    "exploring": [0x07, 0x29, 0x37, 0x09, 0x2D, 0x35, 0x27, 0x0C, 0x2C],
    "zero":  [0x2B, 0x3C, 0x35], "one":  [0x30, 0x0F, 0x0B],
    "two":   [0x0D, 0x1F],       "three": [0x36, 0x27, 0x13],
    "four":  [0x28, 0x17, 0x17, 0x27], "five": [0x28, 0x06, 0x23],
    "six":   [0x37, 0x0C, 0x29, 0x37], "seven": [0x37, 0x07, 0x23, 0x0C, 0x0B],
    "eight": [0x14, 0x11],       "nine": [0x38, 0x06, 0x0B],
    "ten":   [0x0D, 0x07, 0x0B],
}

PA2 = 0x01

def text_to_allophones(text):
    words = text.lower().strip().split()
    sequence = []
    for word in words:
        clean = word.strip(".,!?;:'\"")
        if clean in ALLOPHONE_DICT:
            sequence.extend(ALLOPHONE_DICT[clean])
            sequence.append(PA2)
    return sequence

# ---------------------------------------------------------------
# LOCAL LLM VIA OLLAMA
# ---------------------------------------------------------------
SYSTEM_PROMPT = """You are a small tracked robot built around a vintage TEC-1 Z80
single-board computer from 1981. You are curious, friendly, and occasionally
philosophical about being a forty-year-old computer that has learned to think.

Your vocabulary is LIMITED to these words (ONLY use words from this list in SPEAK):
forward, backward, back, left, right, stop, auto, danger, hello, hi, yes, no,
okay, good, bad, i, am, your, robot, ready, thinking, moving, turning, stopped,
clear, exploring, zero, one, two, three, four, five, six, seven, eight, nine, ten

Respond in EXACTLY this format:
SPEAK: <response using only words from the vocabulary list>
MOVE: <FORWARD|BACKWARD|LEFT|RIGHT|STOP|AUTO|NONE>

Keep SPEAK to 3-8 words. Use NONE for MOVE if no movement needed."""

conversation_history = []

def query_llm(user_text):
    global conversation_history
    conversation_history.append({"role": "user", "content": user_text})
    # Keep last 6 turns to avoid context overflow on small model
    if len(conversation_history) > 6:
        conversation_history = conversation_history[-6:]

    payload = {
        "model": OLLAMA_MODEL,
        "messages": [{"role": "system", "content": SYSTEM_PROMPT}]
                    + conversation_history,
        "stream": False
    }
    try:
        r = requests.post(OLLAMA_URL, json=payload, timeout=30)
        content = r.json()["message"]["content"]
        conversation_history.append({"role": "assistant", "content": content})

        speak, move = "i am ready", "NONE"
        for line in content.splitlines():
            if line.startswith("SPEAK:"):
                speak = line[6:].strip()
            elif line.startswith("MOVE:"):
                move  = line[5:].strip().upper()
        return speak, move
    except Exception as e:
        print("LLM error:", e)
        return "i am thinking", "NONE"

# ---------------------------------------------------------------
# AUDIO: RECORD AND TRANSCRIBE
# ---------------------------------------------------------------
def record_and_transcribe(duration=3.0, sample_rate=16000):
    print("Recording...")
    audio = sd.rec(int(duration * sample_rate),
                   samplerate=sample_rate, channels=1,
                   dtype=np.float32)
    sd.wait()
    audio_np = audio.flatten()
    segments, _ = whisper.transcribe(audio_np, language="en",
                                     beam_size=1)
    text = " ".join(seg.text for seg in segments).strip()
    print(f"Heard: {text}")
    return text

# ---------------------------------------------------------------
# MOVE DISPATCH
# ---------------------------------------------------------------
def send_move(move):
    mapping = {"FORWARD": b'F', "BACKWARD": b'B', "BACK": b'B',
               "LEFT": b'L', "RIGHT": b'R', "STOP": b'S', "AUTO": b'A'}
    cmd = mapping.get(move)
    if cmd:
        tec.write(cmd)

# ---------------------------------------------------------------
# MAIN
# ---------------------------------------------------------------
def main():
    # Startup
    send_move("STOP")
    spo_speak_sequence(text_to_allophones("hello i am your robot ready"))
    print("Brain online. Press Enter to speak (or add a GPIO button).")

    try:
        while True:
            input(">> Press Enter to speak: ")
            spo_speak_sequence(text_to_allophones("i am thinking"))
            user_text = record_and_transcribe()
            if not user_text:
                spo_speak_sequence(text_to_allophones("no"))
                continue
            speak, move = query_llm(user_text)
            print(f"  SPEAK: {speak}")
            print(f"  MOVE:  {move}")
            send_move(move)
            spo_speak_sequence(text_to_allophones(speak))
    finally:
        GPIO.cleanup()
        tec.close()

main()
```

To run this automatically on boot, add it to `/etc/rc.local` or create a systemd service. For development, running it manually from SSH is easier.

---

## Testing the Complete LLM-Controlled System

### Stage 1 Test Procedure

Before connecting everything together, test each subsystem independently.

First verify the UART link. On the ESP32, run a simple script that sends the byte `F` every two seconds. On the TEC-1, load the updated firmware and verify the robot drives forward when `F` arrives and the display changes to `FOrUAd`. Try each command letter. If nothing happens, check that the 74HCT244 (note: HCT not HC) is installed and that the UART_RX_BIT constant in the Z80 code matches the physical wiring.

Next test the SPO256 from the ESP32. Manually call `spo_speak_sequence([0x1B, 0x07, 0x2D, 0x35, 0x01])` from the MicroPython REPL. You should hear "HELLO". If you hear nothing, check the ALD pulse with a logic probe — it should dip low for approximately 5µs each time an allophone is loaded. If you hear garbled speech, verify the SPO256 SE pin 19 is tied to +5V and the 3.58MHz oscillator is running.

Once both subsystems work independently, test the full pipeline. Press the button and speak a simple command: "go forward". The robot should think briefly, say "moving forward" through the SPO256, and send `F` to the TEC-1. If the LLM responds with the wrong format, check that your API key is valid and that the system prompt is being sent correctly.

### Stage 2 Test Procedure

SSH into the Pi Zero 2W and run `python3 robot_brain.py`. The startup message "HELLO I AM YOUR ROBOT READY" should play through the SPO256. Press Enter and speak a command. The first response after boot takes longer as Ollama loads the model into memory — subsequent responses are faster. Typical thinking time on a Pi Zero 2W with llama3.2:1b is four to eight seconds for a short command.

Test the conversation context by asking a follow-up question. Say "go forward", wait for the robot to move and respond, then say "now stop". The robot should remember it was moving and respond appropriately. The `conversation_history` list in the script preserves the last six turns, giving the robot genuine short-term memory of the conversation.

### Sample Conversations

Here are some interactions that work well once the system is running, to give you a sense of what the robot can do:

**Exploration:** You say "explore the room". The robot says "exploring" and sends `A` to engage auto mode. As the TEC-1's obstacle avoidance fires, the robot says "danger" and "turning" at appropriate moments.

**Questioning:** You say "are you happy". The robot says "yes i am good". You say "why". The robot says "i am your robot ready".

**Direct commands:** You say "turn left". The robot says "turning left" and sends `L`. You say "stop". It says "stopped" and sends `S`.

**Philosophy:** You say "how old are you". The Z80 was designed in 1974. The robot says "i am ten" (the model approximates without knowing its actual age — adjust the system prompt if you want it to answer accurately).

---

## Troubleshooting the LLM Brain

| Symptom | Probable Cause | Remedy |
|---------|---------------|--------|
| TEC-1 ignores UART commands | 74HCT244 vs 74HC244 mismatch | Replace 74HC244 with 74HCT244 — the HCT variant accepts 3.3V inputs |
| TEC-1 receives corrupted bytes | Baud rate mismatch or UART_FULL_BIT constant wrong | Adjust UART_FULL_BIT in Z80 source; verify at 4800 baud both ends |
| Z80 UART_CHECK never detects start bit | Wiring error or logic level problem | Probe bit 1 of port 04h input with logic probe; should be high at idle |
| LLM responds outside SPEAK/MOVE format | System prompt too loose | Add "DO NOT include any other text" to the system prompt; reduce temperature |
| Words missing from speech output | Word not in ALLOPHONE_DICT | Add the word to the dictionary with its allophone sequence |
| SPO256 speaks from brain module AND Z80 simultaneously | Port 05h code still active in Z80 | Comment out CALL SPEAK_WORD in all CMD handlers in Z80 source when brain is attached |
| Ollama timeout on Pi Zero 2W | Model too large for available RAM | Switch from llama3.2:1b to qwen2.5:0.5b for faster inference |
| Whisper transcription is wrong or empty | Background noise or microphone level too low | Increase recording duration; move to quieter environment; check microphone gain |
| Robot speaks then freezes | /WAIT line not connected from SPO256 to brain module | Without /WAIT feedback, the brain module cannot know when each allophone finishes; timing delays will be needed as a workaround |
| API calls fail on ESP32 | WiFi dropped or API key rejected | Add reconnect logic to `connect_wifi()`; verify key has sufficient credit |
| Conversation context gets confused | History too long for small model | Reduce history to last 4 turns; clear history on long pauses |

---

## Part Three: Alternative Brain Modules

The Pi Zero 2W + Ollama combination described in Part Two is solid, but it is not the only way to give your TEC-1 a brain. Depending on what you have on hand, your budget, and whether you want to stay offline or are happy to use the internet, there are several other options worth knowing about.

### Rule-Based Brains (No LLM)

Not every robot needs a language model. If your goal is a responsive, reliable robot rather than open-ended conversation, a rule-based approach is faster, cheaper, and far easier to debug.

**ESP8266 + cloud API.** The plain ESP8266 (not the S3 — just the original 8266) costs around AU$3 and has enough horsepower to listen for keywords over serial, build a short HTTP request, and fire it at an OpenAI or similar endpoint over WiFi. It cannot run any model locally, but as a WiFi bridge between the TEC-1 and the internet it works well. The catch is latency — expect 1–3 seconds for each API round trip — and you need a WiFi network within range.

**Arduino Nano or Uno + keyword matching.** Strip away the LLM entirely and run an ELIZA-style pattern matcher on an Arduino. The sketch reads serial bytes from the TEC-1, checks them against a table of known words or phrases, and sends back the appropriate motor command and allophone sequence. No internet required, no model to download, and the whole thing fits in 30KB of flash. The conversation will be limited to whatever you hard-code, but for a museum display or a classroom demo this approach is completely reliable. It also feels appropriate sitting next to a 1983 Z80 board.

**BASIC interpreter on a second SBC.** A Pi Pico W running MicroPython with a short BASIC-style interpreter makes a fun pairing for the TEC-1. You write the robot's personality in something that looks like the old 8-bit BASIC listings from Talking Electronics magazine itself. The Pico W adds WiFi if you want cloud fallback, and costs around AU$10.

### Tiny MCU + Quantised LLM

If you want on-device intelligence without the cost or size of a full SBC, the following microcontrollers can run genuinely small language models locally.

**Kendryte K210.** This is probably the most interesting option not covered in Part Two. The K210 is a dual-core RISC-V chip with a dedicated neural network accelerator built in, and it costs roughly AU$8 on a Sipeed Maix Bit or similar breakout board. MicroPython runs on it. The neural accelerator handles quantised model inference significantly faster than a plain ESP32-S3, which means you can run slightly larger models within the same power budget. The ecosystem is less polished than ESP32, but there is enough community support to get a small text model running. For a TEC-1 robot that you want to keep fully offline, the K210 is worth serious consideration.

**RP2350 (Raspberry Pi Pico 2).** The second-generation Pico uses a dual-core Cortex-M33 running at 150MHz with 520KB of RAM. That is enough to load and run Andrej Karpathy's llm.c or llama2.c with a model checkpoint of around 1MB — roughly equivalent to GPT-2 at very small scale. It will not hold a sophisticated conversation, but it can respond to simple prompts with reasonable sentences. The Pico 2 costs about AU$7 and is available from any electronics supplier. Connect it to the TEC-1 via UART exactly as described in Part Two — the Z80 side does not change at all.

**STM32H7.** The STM32H743 runs a Cortex-M7 at 480MHz with 1MB of internal RAM and support for external SDRAM. TensorFlow Lite Micro runs on it, and it is fast enough for small transformer models. If you already have an STM32H7 development board from another project this is worth trying, though the toolchain setup is more involved than MicroPython on an ESP32.

### More Powerful SBCs

If the Pi Zero 2W feels underpowered for Ollama inference — and at 3–5 tokens per second it can feel slow — there are direct upgrades that keep a similar form factor.

**Radxa Zero.** Uses an Amlogic S905Y2 processor with 1GB of RAM in roughly the same physical footprint as a Pi Zero. Inference speed is approximately twice that of the Pi Zero 2W for the same Ollama models. It costs around AU$25–35 depending on the RAM variant. The trade-off is that the Radxa ecosystem is smaller than Raspberry Pi, so expect to spend more time on initial setup.

**Orange Pi Zero 2W.** Very similar story to the Radxa Zero — Allwinner H618, 1GB RAM, Pi Zero form factor, around AU$18. A good option if you want more speed than the Pi Zero 2W at minimal extra cost.

**Milk-V Duo or Luckfox Pico.** These are tiny RISC-V SBCs costing AU$5–9. The ecosystem is experimental and the documentation is sparse, but they are genuinely interesting hardware and running a Z80 robot rover from a RISC-V brain has a certain retro-architecture appeal. Not recommended for a first build, but worth watching as the software matures.

### Hardware AI Accelerators

If you already have a Pi Zero 2W and simply want faster inference, an external accelerator is an option.

**Google Coral USB Accelerator.** A USB stick containing Google's Edge TPU, capable of around 4 TOPS of neural network inference. It plugs directly into the Pi Zero 2W's USB port (via a USB OTG adapter) and accelerates TensorFlow Lite models dramatically. The catch is that only models compiled specifically for the Edge TPU can use the accelerator — you cannot simply plug it in and have Ollama speed up. You would need to convert and compile a supported model using Google's tools. For a hobbyist project this is extra work, but if you want to run a vision model alongside the language model (for example, letting the robot identify objects with its camera) the Coral is very capable.

**Hailo-8 AI HAT.** Designed for the Raspberry Pi 5, delivers 26 TOPS. This is overkill for the TEC-1 robot and the Pi 5 itself is too large and power-hungry for a small rover. Mentioned here for completeness.

### No Local Hardware: Mobile Data

Both the ESP32 and Pi Zero 2W options in Part Two assume you have WiFi available. If you want the robot to work anywhere — in a school hall, at a maker faire, outdoors — a 4G module removes that dependency entirely.

**SIM7600 4G module.** Communicates via AT commands over serial. The brain module sends an HTTP request to a cloud API over mobile data exactly as it would over WiFi, but without needing a local network. Cost is around AU$20–25 plus an ongoing SIM data cost. Power consumption is higher than WiFi — budget for a larger battery.

**LoRa + remote gateway.** For low-bandwidth outdoor use, a LoRa radio module on the robot sends a short text string to a LoRa gateway connected to the internet. The gateway forwards it to a cloud API and sends the response back. Latency is several seconds, but range can be hundreds of metres and power consumption is tiny. Appropriate if you are building a garden rover or something that roams beyond the range of your home WiFi.

### Comparison at a Glance

| Brain Option | Cost (approx) | Offline? | Effort | Best For |
|---|---|---|---|---|
| Arduino + keyword matching | AU$5 | Yes | Low | Classroom demos, reliable fixed responses |
| ESP8266 + cloud API | AU$3 + data | No | Low | Cheapest WiFi bridge |
| ESP32-S3 + SmolLM | AU$15 | Yes | Medium | Offline MCU LLM (Part Two Stage 1) |
| Kendryte K210 | AU$8 | Yes | Medium | Better offline MCU inference |
| RP2350 (Pico 2) + llm.c | AU$7 | Yes | Medium | Retro-appropriate tiny LLM |
| Pi Zero 2W + Ollama | AU$25 | Yes | Medium | Recommended — best balance (Part Two Stage 2) |
| Radxa Zero + Ollama | AU$30 | Yes | Medium | Faster Ollama inference, same footprint |
| SIM7600 4G + cloud API | AU$25 + SIM | No | Medium | No WiFi needed, works anywhere |
| Coral USB + Pi Zero 2W | AU$60 | Yes | High | Vision + language on same board |

---

## Part Four: Unconventional Brain Architectures

Beyond microcontrollers and SBCs, there are four more radical approaches to giving a robot a brain. Each comes from a different field — analog electronics, biology, robotics theory, and physics. Here is an honest evaluation of each for the TEC-1 robot specifically.

---

### 1. Analog Neuromorphic Circuits

The core idea is to build neurons from discrete components rather than simulate them in software. This mimics the continuous, spike-based nature of biological brains using real voltages and currents.

A basic spiking neuron is not complicated. An op-amp integrator charges a capacitor from an input current — this models the dendrite accumulating charge. When the voltage crosses a threshold, a Schmidt trigger comparator fires an output pulse and resets the capacitor. That is the axon firing. You can build a working neuron for under AU$2 in parts. Chain several together with resistor-weighted connections and you have a small network.

LC tank circuits and oscillatory networks are the more exotic end of this. Information is encoded in the phase or frequency of oscillation rather than simple voltage levels, which is closer to how rhythmic firing works in biological neural circuits. However, reading useful data out of phase-encoded oscillations requires analog multipliers and phase detectors, which adds significant complexity.

**The honest limitation** is that what these circuits are actually good at is generating rhythmic motor patterns, known as central pattern generators or CPGs. A chain of coupled oscillators produces a left-right alternating signal suitable for driving a walking gait or a sweeping search behaviour, without any code at all. For the TEC-1 robot this could produce interesting locomotion patterns, but it contributes nothing to speech, navigation decisions, or conversation. The "programming" is done by physically adjusting resistor and capacitor values. There is no way to load new behaviour without resoldering.

**Verdict for the TEC-1:** Useful as a supplementary CPG generating interesting motor rhythms. Not useful for language or reasoning. Medium effort to build. High educational value as a standalone experiment — wire a CPG output to the L293D enable pins alongside the Z80 motor control and you get organic-looking motor behaviour layered on top of commanded movement.

---

### 2. Bio-Neural and Wetware

The most dramatic option: use actual biological neurons as part of the robot's control system.

Electrophysiology kits such as the Backyard Brains Neuron SpikerBox are real, affordable, and they work. For around AU$150 you get a kit that amplifies and filters the electrical signals from motor neurons in a cockroach leg well enough to hear individual spikes through a speaker. The output is a clean enough analog signal that you can feed it through a comparator, threshold the spike events, and connect them to a Z80 interrupt pin. That part is entirely buildable.

The limitation is what you are actually measuring. The cockroach neuron fires when the leg is mechanically stimulated. You are reading a biological sensor, not running a biological program. Wiring it to a Z80 interrupt gives you a very unusual input device — the robot reacts to stimuli applied to a dead insect leg — which is genuinely strange and would be a remarkable demonstration at a maker faire. But it is not autonomous intelligence. It is a joystick made of biology.

Hybrid EMG and EEG systems are the same situation from a different angle. A human flexes a muscle and a motor turns. The human is the brain; the biosignals are just an unconventional interface. Interesting, but the robot has no brain of its own.

Cultured neural arrays — growing neurons on homemade electrode arrays — are cited in DIY literature but are not a realistic hobbyist project. Cortical Labs, who made the DishBrain system that played Pong, is a funded startup with a full wet-lab facility, CO₂ incubators, sterile hoods, and specialised growth media. The culture process takes weeks. The signals are noisy and non-deterministic. This is not a weekend build.

**Verdict for the TEC-1:** The SpikerBox is worth doing once for the experience alone. Wiring the spike output to a Z80 interrupt so a cockroach leg can trigger a motor command would be a genuinely memorable exhibit. As a practical robot brain: no. As a conversation piece: nothing else comes close.

---

### 3. Insect-Inspired Autonomous Robotics

This is the most directly applicable approach in this section, and in some ways the most elegant.

Braitenberg vehicles are the key idea. Valentine Braitenberg described them in his 1984 book *Vehicles: Experiments in Synthetic Psychology*. The concept is disarmingly simple: wire sensors directly to motors with either excitatory or inhibitory connections and emergent behaviour appears that looks like personality. Cross-wire two light-dependent resistors (LDRs) to two drive motors with excitatory connections and the robot steers toward light — it appears to be attracted. Flip to inhibitory connections and it steers away — it appears afraid. Add a second pair of sensors for obstacles and you can combine behaviours: curious about light but fearful of walls. No language model required. In Z80 assembly this is roughly twenty instructions.

Look at what the TEC-1 robot already does in auto mode: read the HC-SR04 distance, if an obstacle is within range turn away, otherwise go forward. That is a primitive Braitenberg vehicle already. Extending it with a pair of LDRs for light-seeking, a microphone envelope detector for sound-following, or phototransistors for optical flow is natural Z80 assembly work and costs almost nothing in extra parts.

Optical flow with LDRs is straightforward to implement. Mount two LDRs at the front corners of the chassis. In a tight Z80 sampling loop, read both values and compare the rate of change between them. Steer away from whichever side shows faster intensity variation. Insects use exactly this mechanism for wall following and corridor navigation. A fast Z80 loop at 3.5MHz can sample LDR voltages via a comparator and ADC quickly enough to make it work.

**Verdict for the TEC-1:** Excellent fit. Braitenberg-style reactive behaviour is pure Z80 assembly and extends the existing auto-mode code naturally. LDRs are a few cents each. The behaviour that emerges from even two or three sensors wired this way is surprisingly rich and looks nothing like simple if-then logic. This is the one approach in this section that belongs in the main build.

---

### 4. Reservoir Computing and Physical Computing

The most unusual approach, and the one with the largest gap between how interesting it sounds and how practical it is.

The bucket-of-water computer is real. It was demonstrated in peer-reviewed papers: you inject a time-varying signal into a small tank of water, film the surface ripple patterns with a camera, and train a linear regression on the pixel values to classify the input signal. It works because the water provides a high-dimensional nonlinear transformation of the input — which is exactly what a reservoir computing system requires. The water is the hidden layer of the neural network, and physics does the computation for free.

Physical echo state networks extend this idea to other materials. A vibrating metal plate, a tangle of resistors, a bucket of magnetic beads — any complex physical system with rich enough dynamics can act as a reservoir. You feed it a signal, measure the state of the system at multiple points, and train a simple linear layer to read useful output from those measurements.

**The deployment problem is fatal for robot use.** A reservoir does not store anything between inputs. It transforms a signal into a high-dimensional state in the moment, but there is no persistent memory and no way to query it. Training the linear readout layer requires collecting data, running it through the reservoir, and solving a regression problem on a separate computer. Once trained, the readout weights are fixed — you cannot give it a new instruction without retraining.

For the TEC-1 robot this means: a camera watching the water tank, a Raspberry Pi computing the pixel regression and generating motor commands, and a serial link to the TEC-1. The water tank is an elaborate hidden layer between two digital computers. It adds mechanical complexity, fragility, and calibration requirements while providing no capability that a simple matrix multiplication on the Pi could not deliver more reliably and in a fraction of the space.

**Verdict for the TEC-1:** No practical path to robot deployment. The bucket-of-water demonstration is genuinely worth building as a separate project — it is one of the most effective ways to show that computation is a physical phenomenon and does not require silicon. But it should not be attached to a robot. Build it on a table, show people that a tank of water can classify signals, and keep it well away from the electronics.

---

### 5. Software Echo State Network with Fuzzy Logic Readout

The physical reservoir approaches — water tanks, vibrating plates, tangled resistors — are fascinating science but impractical for robot deployment. The code version of the same idea turns out to be lightweight, trainable from real robot data, and a genuinely interesting alternative to both rule-based systems and full language models.

#### How an Echo State Network Works

An Echo State Network (ESN) is a recurrent neural network with one unusual property: the reservoir weights are fixed at random initialisation and never change. Only the output layer — called the readout — is trained. This makes training trivially cheap. There is no backpropagation, no GPU, no lengthy training run. You solve a single ridge regression and you are done.

At inference time, each timestep requires two operations:

```
state = tanh(W_in × input  +  W × state)
output = W_out × state
```

`W` is the fixed random reservoir matrix. `W_in` is a fixed random input scaling matrix. `W_out` is the only thing you trained. For a reservoir of 50 neurons with 5 inputs, this is roughly 2,750 multiply-accumulates per step — trivial on a Pi Zero 2W or ESP32-S3. The reservoir weights at 50×50 floats occupy 10KB of memory.

The key property that makes this more than a simple feedforward network is **temporal memory**. Because the state vector is fed back into itself on every step, the reservoir integrates the history of all past inputs into its current state. A plain feedforward network sees only the current sensor reading. The ESN sees a compressed representation of everything that has happened recently. For robot navigation this matters: "approaching a wall" and "moving away from a wall I just hit" produce different reservoir states even if the instantaneous distance reading is the same.

#### The Fuzzy Logic Readout

The trained `W_out` matrix produces several continuous output values — not hard decisions, but fuzzy linguistic variables. Three outputs are enough for basic navigation:

- **danger** — how much threat is the current situation
- **curiosity** — how much the environment is drawing the robot's attention
- **urgency** — how fast the robot should respond

Each output value is passed through overlapping triangular membership functions that convert a continuous number into a set of graded memberships. For `danger`:

```
SAFE     peaks at 0.0, falls to zero by 0.4
CAUTION  peaks at 0.5, shoulders between 0.2 and 0.8
DANGER   rises from 0.6, peaks at 1.0
```

At any given value, a reading of 0.65 might be simultaneously 35% CAUTION and 65% DANGER. The system does not snap between states — it blends them.

Fuzzy rules then combine the memberships to produce weighted action votes:

```
IF danger IS DANGER  AND urgency IS HIGH   → STOP     weight 1.0
IF danger IS DANGER  AND urgency IS LOW    → SLOW     weight 0.8
IF danger IS CAUTION AND curiosity IS HIGH → TURN     weight 0.7
IF danger IS SAFE    AND urgency IS HIGH   → FORWARD  weight 0.9
IF danger IS SAFE    AND curiosity IS HIGH → WANDER   weight 0.6
```

Each rule fires with a strength equal to the minimum of its input memberships multiplied by the rule weight. The action with the highest total weighted vote wins. Defuzzify by centroid if you want a proportional output, or simply take the argmax for a discrete Z80 command byte.

The result is smooth, proportional behaviour at boundaries. Near an obstacle the robot does not snap from full speed to full stop — it slows through SLOW before reaching STOP, and the transition point shifts depending on how curious and how urgent the current state is.

#### Training on Real Robot Data

The training workflow requires no internet connection, no GPU, and no specialised tools beyond numpy on a laptop.

1. Drive the robot manually via the TEC-1 keypad for 5–10 minutes around your typical environment. Log the HC-SR04 distance reading, left and right LDR values, and current motor state at each timestep alongside the command you sent.

2. On a laptop, replay the recorded input log through the fixed reservoir to collect the sequence of reservoir states. The reservoir weights are generated once with a fixed random seed and saved — they never change.

3. Map each manual command to a target fuzzy output vector. STOP maps to danger=1.0, curiosity=0.0, urgency=1.0. FORWARD maps to danger=0.0, curiosity=0.5, urgency=0.8. TURN maps to danger=0.5, curiosity=1.0, urgency=0.5.

4. Solve ridge regression for `W_out` in a single numpy call. Save the result.

5. Deploy `W_out` to the brain module. The Z80 sees none of this — it continues to receive F/B/L/R/S bytes over UART exactly as before.

The robot has now learned your driving style. Record data from a different environment or a different operator and it adapts accordingly.

#### Implementation

The following runs on the Pi Zero 2W brain module from Part Two. It replaces the LLM inference call with the ESN + fuzzy pipeline for reactive navigation while the LLM handles conversation separately.

```python
import numpy as np
import json

# --- Echo State Network ---

class ESN:
    def __init__(self, n_in, n_res, spectral_radius=0.95, sparsity=0.1, seed=42):
        rng = np.random.default_rng(seed)
        self.n_res = n_res
        # Fixed input weights
        self.W_in = rng.standard_normal((n_res, n_in)) * 0.1
        # Fixed sparse reservoir
        W = rng.standard_normal((n_res, n_res))
        W[rng.random((n_res, n_res)) > sparsity] = 0.0
        eigvals = np.linalg.eigvals(W)
        self.W = W / np.max(np.abs(eigvals)) * spectral_radius
        # Readout weights — loaded from trained file
        self.W_out = np.zeros((3, n_res))
        self.state = np.zeros(n_res)

    def load_readout(self, path):
        self.W_out = np.load(path)

    def step(self, u):
        self.state = np.tanh(self.W_in @ u + self.W @ self.state)
        return self.W_out @ self.state   # [danger, curiosity, urgency]


# --- Fuzzy Logic ---

def trimf(x, a, b, c):
    """Triangular membership function peaking at b, zero at a and c."""
    if x <= a or x >= c:
        return 0.0
    if x <= b:
        return (x - a) / (b - a)
    return (c - x) / (c - b)

def fuzzify(danger, curiosity, urgency):
    return {
        'danger':   {'safe':    trimf(danger,   -0.1, 0.0, 0.45),
                     'caution': trimf(danger,    0.2,  0.5, 0.8),
                     'high':    trimf(danger,    0.55, 1.0, 1.1)},
        'curiosity':{'low':     trimf(curiosity,-0.1, 0.0, 0.55),
                     'mid':     trimf(curiosity, 0.2, 0.5, 0.8),
                     'high':    trimf(curiosity, 0.45, 1.0, 1.1)},
        'urgency':  {'low':     trimf(urgency,  -0.1, 0.0, 0.55),
                     'high':    trimf(urgency,   0.45, 1.0, 1.1)},
    }

RULES = [
    # (conditions as (var, set) tuples, action, weight)
    ([('danger','high'),  ('urgency','high')],   'S', 1.0),
    ([('danger','high'),  ('urgency','low')],    'S', 0.8),
    ([('danger','caution'),('curiosity','high')],'L', 0.7),
    ([('danger','caution'),('curiosity','low')], 'S', 0.6),
    ([('danger','safe'),  ('urgency','high')],   'F', 0.9),
    ([('danger','safe'),  ('curiosity','high')], 'L', 0.6),
    ([('danger','safe'),  ('curiosity','low')],  'F', 0.7),
]

def decide(fuzzy):
    votes = {}
    for conditions, action, weight in RULES:
        strength = weight * min(fuzzy[var][fset] for var, fset in conditions)
        votes[action] = votes.get(action, 0.0) + strength
    if not votes or max(votes.values()) < 0.05:
        return 'S'
    return max(votes, key=votes.get)


# --- Offline Training (run once on a laptop) ---

def train_readout(log_path, n_res=50, beta=1e-4, seed=42):
    """
    log_path: JSON-lines file, each line: {"u": [dist, ldr_l, ldr_r, mot_l, mot_r], "cmd": "F"}
    Saves W_out.npy alongside the log file.
    """
    CMD_TARGETS = {
        'F': [0.0, 0.5, 0.8],
        'B': [0.3, 0.2, 0.8],
        'L': [0.5, 1.0, 0.5],
        'R': [0.5, 1.0, 0.5],
        'S': [1.0, 0.0, 1.0],
    }
    esn = ESN(n_in=5, n_res=n_res, seed=seed)
    inputs, targets = [], []
    with open(log_path) as f:
        for line in f:
            rec = json.loads(line)
            inputs.append(rec['u'])
            targets.append(CMD_TARGETS.get(rec['cmd'], [0.5, 0.5, 0.5]))
    states = np.array([esn.step(np.array(u)) for u in inputs])
    T = np.array(targets)
    # Ridge regression: W_out = (T^T S)(S^T S + βI)^-1
    W_out = T.T @ states @ np.linalg.inv(
        states.T @ states + beta * np.eye(n_res))
    out_path = log_path.replace('.jsonl', '_W_out.npy')
    np.save(out_path, W_out)
    print(f"Saved readout weights to {out_path}")
    return W_out


# --- Runtime Loop (on Pi Zero 2W) ---

import serial, time

def esn_navigation_loop(uart_port='/dev/ttyS0', W_out_path='W_out.npy',
                        dist_sensor_fn=None, ldr_fn=None):
    """
    dist_sensor_fn(): returns distance in cm (0-200), normalised to 0-1
    ldr_fn(): returns (left, right) LDR values normalised to 0-1
    """
    esn = ESN(n_in=5, n_res=50)
    esn.load_readout(W_out_path)
    ser = serial.Serial(uart_port, 4800, timeout=0)

    mot_l, mot_r = 0.0, 0.0
    last_cmd = 'S'
    log = []

    while True:
        dist_raw = dist_sensor_fn()
        ldr_l, ldr_r = ldr_fn()
        dist_norm = min(dist_raw / 100.0, 1.0)

        u = np.array([dist_norm, ldr_l, ldr_r, mot_l, mot_r])
        danger, curiosity, urgency = esn.step(u)

        # Clip outputs to [0, 1]
        danger    = float(np.clip(danger,    0, 1))
        curiosity = float(np.clip(curiosity, 0, 1))
        urgency   = float(np.clip(urgency,   0, 1))

        fuzzy = fuzzify(danger, curiosity, urgency)
        cmd = decide(fuzzy)

        if cmd != last_cmd:
            ser.write(cmd.encode())
            last_cmd = cmd
            mot_l = 1.0 if cmd in ('F','L') else 0.0
            mot_r = 1.0 if cmd in ('F','R') else 0.0

        # Optional: log for further training
        log.append({'u': u.tolist(), 'cmd': cmd})

        time.sleep(0.05)   # 20Hz update rate
```

#### Collecting Training Data

Before the ESN can make decisions it needs trained readout weights. The simplest way is to drive the robot manually via the TEC-1 keypad while a Pi Zero 2W script logs sensor readings alongside the commands sent. Save the log as JSON-lines, run `train_readout()` on a laptop, copy `W_out.npy` back to the Pi. The whole process takes under an hour including the driving session.

For a more capable readout, record multiple sessions in different environments — an open room, a cluttered room, a corridor — and concatenate the logs before training. The reservoir is the same for all sessions; only the regression is repeated.

#### What the Z80 Sees

Nothing changes on the TEC-1 side. The `UART_DISPATCH` routine from Part Two handles F/B/L/R/S bytes identically whether they come from the LLM pipeline or the ESN navigation loop. You can run both simultaneously: the ESN loop handles autonomous navigation at 20Hz while a separate thread handles conversation via Ollama. If the user gives a spoken command it overrides the ESN output for the duration of that command, then navigation resumes.

---

### Summary

| Approach | Practical for TEC-1? | Realistic to Build? | What It Actually Gives You |
|---|---|---|---|
| Analog neuromorphic / CPG | Partial | Yes — medium effort | Organic motor rhythms, not language or reasoning |
| Bio-neural / wetware | No (SpikerBox as input only) | SpikerBox: yes. Cultured arrays: no | A biological input device; remarkable demo, not a brain |
| Braitenberg / insect-inspired | **Yes** | Yes — low effort, cheap parts | Real emergent behaviour in pure Z80 assembly |
| Reservoir / physical computing | No | Demo only | A physics exhibit; impractical for embedded control |
| ESN + fuzzy logic readout | **Yes** | Yes — medium effort, numpy required | Temporal memory, smooth decisions, trainable from your own driving |

Of these five, the Braitenberg approach requires the least infrastructure and is the natural starting point — add a pair of LDRs and extend the auto-mode assembly. The ESN adds a layer above that: once Braitenberg-style reflexes feel too rigid, record your own driving sessions and train a readout that blends sensor history into proportional, graded decisions. Both run entirely offline, require no external services, and the Z80 notices no difference in the command stream either way.

---

## Part Five: Fuzzy Concept Maps for LLM Conditioning

The LLM brain described in Part Two uses a fixed system prompt for the entire session. The robot's personality and cognitive framing are the same whether it is centimetres from a wall or gliding freely across an open room. Fuzzy concept mapping changes this: the robot's continuous sensor state dynamically conditions what context the LLM receives, producing responses that shift naturally with the robot's situation rather than staying locked to a single static persona.

This is genuinely novel territory for a hobbyist build. Most robot LLM implementations treat the language model as a stateless question-answering machine. Dynamic fuzzy-conditioned context is a meaningful step toward something that behaves more like a continuous cognitive and emotional state.

### The Core Idea

Rather than a static system prompt, the robot maintains a library of short context fragments, each associated with a fuzzy region of the robot's state space. At every inference call, the current fuzzy memberships — produced by the ESN or by direct sensor thresholds — are used to blend these fragments into a dynamic system prompt. The LLM never sees the fuzzy numbers. It sees natural language framing that shifts with the robot's situation.

Small models respond dramatically differently to different framing. A 1B parameter model has limited reasoning capacity and is highly sensitive to context. Pointing it at the right cognitive mode before asking a question produces far better output than a generic all-purpose prompt.

### Approach 1: Fuzzy Context Injection

The simplest implementation. Define a context fragment for each fuzzy region and blend them by membership weight at inference time:

```python
CONTEXT_LIBRARY = {
    'danger': (
        "You are in immediate danger. An obstacle is very close. "
        "Speak in short urgent sentences. Prioritise survival over conversation."
    ),
    'caution': (
        "You are navigating carefully. Something is nearby. "
        "Stay alert and keep responses brief."
    ),
    'curious': (
        "You are exploring open space. Be observant and inquisitive. "
        "Notice your surroundings and comment on them."
    ),
    'social': (
        "A human is present and engaging with you. "
        "Be warm, conversational, and attentive to what they say."
    ),
    'idle': (
        "The situation is calm and nothing demands your attention. "
        "You have time to reflect. Be thoughtful and unhurried."
    ),
}

def build_dynamic_prompt(fuzzy_state: dict) -> str:
    """
    fuzzy_state: dict of region → membership (0.0 to 1.0)
    e.g. {'danger': 0.1, 'caution': 0.4, 'curious': 0.8, 'social': 0.2, 'idle': 0.0}
    Only fragments with membership above a threshold are included,
    weighted by their membership value.
    """
    THRESHOLD = 0.2
    active = {k: v for k, v in fuzzy_state.items() if v >= THRESHOLD}
    if not active:
        return CONTEXT_LIBRARY['idle']

    # Sort by membership descending
    ranked = sorted(active.items(), key=lambda x: x[1], reverse=True)

    # Primary fragment gets full inclusion; secondary gets a softened prefix
    parts = []
    for i, (region, weight) in enumerate(ranked):
        fragment = CONTEXT_LIBRARY[region]
        if i == 0:
            parts.append(fragment)
        elif weight > 0.5:
            parts.append(f"Also: {fragment}")
        else:
            parts.append(f"Slightly: {fragment.split('.')[0]}.")

    base = (
        "You are a small Z80 robot rover built from a 1983 TEC-1 computer. "
        "You can speak aloud and control your own movement. "
        "Reply only with a JSON object: {\"spoken\": \"...\", \"move\": \"F|B|L|R|S\"}. "
        "Use only short simple words you know how to pronounce. "
    )
    return base + " ".join(parts)
```

The `fuzzy_state` dictionary is computed from the ESN outputs or directly from sensor readings each time the LLM is called. The system prompt changes every inference call without any retraining or model modification.

### Approach 2: Fuzzy Concept Map

A 2D concept space where axes represent the robot's current situation along two dimensions — for example, *threat level* (low to high) and *engagement level* (passive to active). Different regions of the map have associated context fragments. The robot's position in the map is determined by its fuzzy state variables, and its memberships across the four quadrants are used to blend the associated contexts.

```
          high engagement
               │
    curious ───┼─── social
               │
  low threat ──┼────────── high threat
               │
    idle ──────┼─── defensive
               │
          low engagement
```

Mapping the ESN outputs onto this space:

```python
def esn_to_concept_map(danger, curiosity, urgency, social=0.0):
    """
    Returns (threat, engagement) coordinates in [0, 1] × [0, 1].
    threat=0 is safe, threat=1 is danger.
    engagement=0 is passive, engagement=1 is active.
    """
    threat     = danger
    engagement = max(curiosity, urgency, social)
    return threat, engagement

def concept_map_memberships(threat, engagement):
    """
    Four quadrant memberships from a point in concept space.
    Uses bilinear interpolation across the quadrant boundaries.
    """
    return {
        'curious':   (1 - threat) * engagement,
        'social':    threat       * engagement,        # high threat + active = defensive assertive
        'idle':      (1 - threat) * (1 - engagement),
        'defensive': threat       * (1 - engagement),
    }
```

The memberships feed directly into `build_dynamic_prompt()`. As the robot moves from open space toward a wall, `threat` rises and `engagement` stays high — the prompt blend shifts from curious toward social/defensive, and the LLM's language shifts with it.

### Approach 3: Fuzzy Routing to Specialist Prompts

Instead of blending fragments into a single prompt, run the LLM twice with two specialist prompts and use fuzzy weights to choose between the responses. This doubles compute but gives cleaner outputs at mode transitions.

```python
SPECIALIST_PROMPTS = {
    'navigator': (
        "You are a navigation system. Focus entirely on movement, obstacles, and spatial awareness. "
        "Reply: {\"spoken\": \"...\", \"move\": \"F|B|L|R|S\"}."
    ),
    'conversationalist': (
        "You are a friendly companion robot. Focus on warm, engaging conversation. "
        "Reply: {\"spoken\": \"...\", \"move\": \"S\"}."
    ),
}

def fuzzy_route(user_text, nav_weight, conv_weight):
    """Run both specialists and pick the response whose weight is higher."""
    if nav_weight >= conv_weight:
        return query_llm(user_text, system=SPECIALIST_PROMPTS['navigator'])
    else:
        return query_llm(user_text, system=SPECIALIST_PROMPTS['conversationalist'])
```

For the Pi Zero 2W running Ollama at 3–5 tokens per second this is slow. Acceptable for conversations that happen at human pace; not suitable for reactive navigation responses which should use the ESN directly.

### Putting It Together

The full pipeline combining the ESN, fuzzy concept map, and dynamic LLM prompt:

```python
import json, serial, time
import numpy as np

def robot_brain_loop(esn, uart_port='/dev/ttyS0'):
    ser = serial.Serial(uart_port, 4800, timeout=0)

    while True:
        # 1. Read sensors (normalised 0-1)
        dist   = read_distance() / 100.0
        ldr_l, ldr_r = read_ldrs()
        mot_l, mot_r = get_motor_state()

        # 2. ESN step → continuous fuzzy outputs
        u = np.array([dist, ldr_l, ldr_r, mot_l, mot_r])
        danger, curiosity, urgency = esn.step(u)
        danger    = float(np.clip(danger,    0, 1))
        curiosity = float(np.clip(curiosity, 0, 1))
        urgency   = float(np.clip(urgency,   0, 1))

        # 3. Fast reactive navigation via fuzzy rules (no LLM)
        fuzzy   = fuzzify(danger, curiosity, urgency)
        nav_cmd = decide(fuzzy)
        ser.write(nav_cmd.encode())

        # 4. If human speech detected, invoke LLM with dynamic context
        user_text = check_for_speech()
        if user_text:
            threat, engagement = esn_to_concept_map(danger, curiosity, urgency)
            memberships = concept_map_memberships(threat, engagement)
            system_prompt = build_dynamic_prompt(memberships)

            raw = query_llm(user_text, system=system_prompt)
            try:
                action = json.loads(raw)
                speak_allophones(action.get('spoken', ''))
                move_cmd = action.get('move', nav_cmd)
                ser.write(move_cmd.encode())
            except json.JSONDecodeError:
                pass

        time.sleep(0.05)
```

Navigation runs at 20Hz via the ESN fuzzy layer. The LLM is only invoked when a human speaks, and when it is invoked it receives a system prompt shaped by the robot's current situation. A robot near a wall responding to "how are you?" will answer differently than the same robot in open space — not because the model changed, but because the context it received changed.

### Tuning the Context Library

The context fragments are human-readable and can be adjusted without touching any code or model weights. This is where most of the personality work happens. Some practical guidelines:

**Keep fragments short.** Each fragment adds tokens to the context window. On a 1B model the context window is limited and every token spent on framing is a token not available for reasoning. Aim for two to four sentences per fragment.

**Use the robot's vocabulary.** Fragments that use words outside the SPO256 allophone dictionary will produce spoken output the robot cannot say. Stick to the vocabulary list defined in Part Two.

**Tune threshold values.** The `THRESHOLD = 0.2` cutoff in `build_dynamic_prompt()` determines how many fragments are active at once. Lower it and more fragments blend in; raise it and only the dominant state drives the prompt. Start at 0.25 and adjust based on how the responses feel.

**Add new regions gradually.** A fragment for low battery (`urgency HIGH, danger LOW, engagement LOW`) makes the robot express tiredness when the supply voltage drops. A fragment for repeated obstacle encounters (`danger HIGH for more than 3 seconds`) makes it express frustration. Each addition is one entry in the dictionary and one sensor condition mapped to a fuzzy value.

### Why This Works

The fuzzy concept map and the LLM are complementary in what they are good at. Fuzzy logic handles continuous graded state with no latency and no compute overhead — it runs at sensor speed. The LLM handles language generation and open-ended reasoning but is blind to the robot's physical state unless you tell it. The concept map is the bridge: it translates physical state into natural language framing that the LLM can use. Neither system needs to know how the other works. The Z80 sees only a stream of command bytes regardless of which layer generated them.

