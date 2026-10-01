
# how usbcdc works for [this micro thingy]. whats missing in apps board (why its not working)

## How CDC Works, What's In CDC: 
- CDC is the board that shows up on the computer as a virtual serial port (A COM port on windows). That's what is used for the pinging, read, save upper / lower to calibrate the pedal.

## **On the F042 (datasheet, §3.19 and Table 65):**

- It has a full-speed USB _device_ peripheral on **PA11 (D−) and PA12 (D+)**. On small packages those pins can be swapped in for PA9/PA10.
- It has a built-in D+ pull-up and no series resistors are needed, so the USB lines can go straight to the connector.
- USB needs an exact **48 MHz clock**. It can come from an external crystal plus the PLL, or from the internal HSI48 oscillator, which the CRS block keeps accurate by locking onto the computer's USB signal. That's the "crystal-less USB" in the title.
- USB has **1 KB of packet memory**, and the last 256 bytes go to CAN if CAN is enabled. PA11/PA12 are also the CAN_RX/CAN_TX pins, so USB and CAN compete for both pins and memory.
- The USB pins are powered from **VDDIO2, which must be 3.0–3.6 V**.

**On the F446 (what the code is set up for):** it uses the USB_OTG_HS core running at full speed with its internal PHY, on **PB14/PB15**. The 48 MHz clock comes from the PLL: 8 MHz crystal ÷ 4 × 120 ÷ 5 = 48 MHz, which is correct.

**Data flow in the firmware:**

- Incoming: the computer sends bytes, the USB interrupt fires, and `CDC_Receive_HS()` in `usbd_cdc_if.c` is called. It's supposed to hand the data to `main.c`.
- Outgoing: `main.c` calls `CDC_Transmit_HS()`, which sends the bytes to the computer.


## What could be broken/missing in the USB path:
- **Received data never reaches** `main.c` **.** usbd_cdc_if.c's `CDC_Receive_HS` only re-arms reception. It never copies into `UserRxBuffer`, never sets `UserRxLength`, and never sets `DataReceivedFlag`. So the command handler in `main.c` can never run.
- `CDC_SendString` is called 6 times but not defined anywhere.
- `cmd` is never declared 
- ``sensAcceptibleDiff`` (*haha i found this one*) - 
- **There's no line buffering**: Terminals often send one character at a time, or add `\r\n`, so an exact `strcmp(…,"PING")` match would rarely succeed.
- **Replies get dropped silently.** `CDC_Transmit_HS` returns `USBD_BUSY` if a send is still in progress, and nothing retries.
-  **The calibration math is wrong.** `(apps1/4095)*3.3` is integer division, so it always gives 0. It also stores volts, while the safety check compares raw ADC counts. Running `SAVE UPPER` would write 0s, and then the pedal would always read as faulted.
2. **Anyone can recalibrate over USB with no checks.** There's no range check and nothing stops it while the car is live. That's a safety risk.
3. There's a small type mismatch: `UserRxLength` is `extern uint32_t` in `main.c` but `volatile uint32_t` where it's defined.

## Other APPS gaps that may be worth checking:

- **DAC Output**: Constant; doesn't follow the pedal position so the throttle isn't passed through
- **The saved calibration is overwritten on every boot** (`EEWriteLatch = 1`). That also wears out the flash.
- `sens1Range` is computed **before** `EE_Read()`, so it's calculated from zeros.
- There's **no brake input**, which is odd for a board named BSPD (Brake System Plausibility Device).
- There's no 100 ms implausibility timer, no fault latch, no watchdog, and no CAN.
- **PB5 is set up as a TIM3 pin, but TIM3 is never initialized.**
- The ADC sample time is **3 cycles**, which is very short for sensors behind a voltage divider (the README mentions one). Readings could be inaccurate.