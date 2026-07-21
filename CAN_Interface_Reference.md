# STM32F407 + SN65HVD230 — CAN Bus Reference

A reusable guide for getting two STM32F407 (Discovery) boards talking over CAN
using SN65HVD230 transceivers. This is generic on purpose — it teaches the
setup and the functions you need, not one specific project. Once this works,
you decide what data to actually send and what to do with it.

---

## 1. Hardware Wiring

The SN65HVD230 is a 3.3V-native transceiver — matches the STM32 directly, no
level shifter needed.

**Module pins:**

| Pin | Meaning |
|---|---|
| 3.3V | power |
| GND | ground |
| CAN Rx (R) | output from transceiver → wire to STM32 **RX** pin |
| CAN Tx (D) | input to transceiver ← wire to STM32 **TX** pin |
| CAN H | bus line, high |
| CAN L | bus line, low |

> The transceiver's "Rx" outputs data the MCU reads, so it connects to the
> MCU's RX pin. Its "Tx" is an input the MCU drives. Naming is from the
> transceiver's point of view, not the MCU's — easy to wire backwards.

**STM32 pins used (CAN1, remapped):**

| STM32 pin | Function |
|---|---|
| PD0 | CAN1_RX |
| PD1 | CAN1_TX |

(CAN1's default pins PA11/PA12 collide with USB OTG on the Discovery board —
PD0/PD1 avoid that entirely.)

**Per board:**
```
STM32 PD1 (TX) ──► D/Tx  (transceiver)
STM32 PD0 (RX) ◄── R/Rx  (transceiver)
STM32 3.3V     ──► VCC
STM32 GND      ──► GND
```
Do this identically on both boards.

**Between the two boards (the actual bus):**
```
Board A CAN H ── Board B CAN H
Board A CAN L ── Board B CAN L
```
Never cross H and L.

**Termination:** CAN needs 120Ω resistors at *each end* of the bus. For a
2-node bus that's exactly two — one across H/L at each transceiver. Some
breakout boards have this built in or jumper-selectable — check yours.

**Ground:** run an explicit GND-to-GND wire between the boards in addition
to H/L. Don't rely on a shared USB host to provide this silently.

---

## 2. CAN Bit Timing (do this before CubeMX)

bxCAN (CAN1) clocks off **APB1**. Baud rate:

```
BaudRate = APB1_CLK / (Prescaler × (1 + BS1 + BS2))
```

**Check your project's actual APB1 frequency first** (CubeMX → Clock
Configuration tab) — it depends on your `SystemClock_Config()`, e.g. HSI-only
gives 16 MHz APB1, while HSE+PLL setups commonly give 42 MHz APB1. Don't
assume — different projects/templates can differ.

Both boards must resolve to the **same baud rate** — easiest is identical
CubeMX CAN1 settings on both.

**Example: 16 MHz APB1, target 500 kbit/s:**

| Setting | Value |
|---|---|
| Prescaler | 2 |
| BS1 | 13 TQ |
| BS2 | 2 TQ |

Check: `16MHz / (2 × (1+13+2)) = 500 kbit/s`, sample point `(1+13)/16 = 87.5%`
(standard automotive convention, 75–87.5%).

If your APB1 is different (e.g. 42 MHz), recompute — don't reuse these
numbers blindly. Pick BS1/BS2 to keep the sample point in the 75–87.5% range.

---

## 3. CubeMX Configuration

Apply identically on both boards:

**Pinout view:**
- PD0 → `CAN1_RX`
- PD1 → `CAN1_TX`

**Connectivity → CAN1 → Mode:**
- Master Mode: checked

**Connectivity → CAN1 → Parameter Settings:**

| Setting | Value |
|---|---|
| Prescaler | (from §2) |
| Time Quanta BS1 | (from §2) |
| Time Quanta BS2 | (from §2) |
| ReSynchronization Jump Width | 1 |
| Automatic Retransmission | Enable |
| Everything else (Time-Triggered, Auto Bus-Off, Auto Wake-Up, RX FIFO Locked, TX FIFO Priority) | Disable |

**NVIC (only needed on boards that will receive via interrupt):**
- Enable **CAN1 RX0 interrupt**

This is a separate checkbox from the peripheral config itself — easy to
forget, and without it your receive callback (§5) will never fire.

Generate code once both `.ioc` files are set.

---

## 4. Core Concepts You Need in Code

Everything below plugs into the `USER CODE BEGIN/END` blocks CubeMX
preserves. Never edit outside those markers — CubeMX will wipe it on
regeneration.

### 4.1 Starting the peripheral

Every node, transmitter or receiver, needs a filter configured and the
peripheral started before it can send or receive anything:

```c
CAN_FilterTypeDef canFilter;
canFilter.FilterBank = 0;
canFilter.FilterMode = CAN_FILTERMODE_IDMASK;
canFilter.FilterScale = CAN_FILTERSCALE_32BIT;
canFilter.FilterIdHigh = 0x0000;
canFilter.FilterIdLow = 0x0000;
canFilter.FilterMaskIdHigh = 0x0000;   // 0x0000 mask = accept everything
canFilter.FilterMaskIdLow = 0x0000;
canFilter.FilterFIFOAssignment = CAN_FILTER_FIFO0;
canFilter.FilterActivation = ENABLE;
canFilter.SlaveStartFilterBank = 14;

HAL_CAN_ConfigFilter(&hcan1, &canFilter);
HAL_CAN_Start(&hcan1);
```

A filter is **mandatory** even if you want to accept every message — an
all-zero mask (as above) means "don't care about any ID bit," i.e. accept
everything. This is the right starting point while bringing up a new bus;
tighten it later (§4.4) once you know it works.

### 4.2 Sending a frame — `HAL_CAN_AddTxMessage()`

```c
CAN_TxHeaderTypeDef TxHeader;
uint8_t TxData[8];
uint32_t TxMailbox;

TxHeader.StdId = 0x100;           // pick any 11-bit ID for standard frames
TxHeader.ExtId = 0;
TxHeader.IDE = CAN_ID_STD;        // CAN_ID_EXT if you need 29-bit IDs
TxHeader.RTR = CAN_RTR_DATA;
TxHeader.DLC = 1;                 // payload length in bytes, 0–8
TxHeader.TransmitGlobalTime = DISABLE;

// ... whenever you actually want to send:
TxData[0] = 0xAA;   // put your real data here
HAL_CAN_AddTxMessage(&hcan1, &TxHeader, TxData, &TxMailbox);
```

Call `HAL_CAN_AddTxMessage()` any time your application decides it has
something to send — a button press, a sensor reading crossing a threshold, a
periodic timer tick, whatever your logic needs. Header fields don't need to
be reset each time unless they change (e.g. same ID/DLC every send — just set
once at init).

### 4.3 Receiving a frame — two ways

**Option A — Polling** (simple, check periodically in your main loop):

```c
if (HAL_CAN_GetRxMessage(&hcan1, CAN_RX_FIFO0, &RxHeader, RxData) == HAL_OK)
{
    // RxHeader.StdId  = arbitration ID of the received frame
    // RxHeader.DLC    = number of valid bytes in RxData
    // RxData[0..DLC-1] = payload
}
```
This returns `HAL_ERROR` immediately if no frame is waiting — safe to call
every loop iteration, but you'll miss frames if your loop is doing something
slow elsewhere.

**Option B — Interrupt-driven** (recommended once polling gets limiting):

```c
// One-time, after HAL_CAN_Start():
HAL_CAN_ActivateNotification(&hcan1, CAN_IT_RX_FIFO0_MSG_PENDING);
```

```c
// Re-implement this weak callback anywhere in your code:
void HAL_CAN_RxFifo0MsgPendingCallback(CAN_HandleTypeDef *hcan)
{
    CAN_RxHeaderTypeDef RxHeader;
    uint8_t RxData[8];

    if (HAL_CAN_GetRxMessage(hcan, CAN_RX_FIFO0, &RxHeader, RxData) == HAL_OK)
    {
        // handle the frame here — this runs in interrupt context,
        // so keep it short: set a flag / copy data out, do heavy
        // work in your main loop instead.
    }
}
```

This only fires if **all three** are true:
1. `HAL_CAN_ActivateNotification()` was called for `CAN_IT_RX_FIFO0_MSG_PENDING`
2. NVIC CAN1 RX0 interrupt is enabled in CubeMX (§3)
3. `stm32f4xx_it.c` has a working IRQ handler:
```c
void CAN1_RX0_IRQHandler(void)
{
    HAL_CAN_IRQHandler(&hcan1);
}
```
CubeMX generates this automatically once step 2 is done — just confirm it's
there and calls `HAL_CAN_IRQHandler()`.

### 4.4 Filtering by ID (once you know what you're listening for)

Once your application has a specific arbitration ID it cares about, tighten
the filter from "accept everything" to "accept only this ID" — this offloads
the check to hardware instead of software:

```c
canFilter.FilterIdHigh     = (0x100 << 5);   // your ID, shifted into bits[31:21]
canFilter.FilterMaskIdHigh = (0x7FF << 5);   // exact-match mask, all 11 bits
```
(the `<< 5` is a fixed quirk of how bxCAN's 32-bit filter registers are laid
out for standard IDs — not something you need to derive yourself, just use
this pattern.)

You can add multiple filter banks (`FilterBank = 0, 1, 2, ...`) if a node
needs to accept several distinct IDs.

---

## 5. Putting a Node Together

A **transmit-only node** needs:
- CAN1 init (CubeMX-generated)
- Filter config + `HAL_CAN_Start()` (accept-all is fine, it never reads RX)
- A `TxHeader` set up once
- Calls to `HAL_CAN_AddTxMessage()` whenever it has something to send

A **receive-only node** needs:
- CAN1 init (CubeMX-generated)
- Filter config (tightened to the ID(s) it cares about) + `HAL_CAN_Start()`
- Either a polling loop calling `HAL_CAN_GetRxMessage()`, or
  `HAL_CAN_ActivateNotification()` + a `HAL_CAN_RxFifo0MsgPendingCallback()`
- NVIC RX0 interrupt enabled in CubeMX if using the callback approach

A **bidirectional node** is just both of the above combined in the same
project — nothing about them conflicts.

---

## 6. Bring-Up Order (debug one thing at a time)

1. **Loopback mode** on a single board (`CAN_MODE_LOOPBACK` in CubeMX) —
   validates clock/peripheral init without touching the physical bus at all.
2. **Two boards, accept-all filter, one direction** — validates wiring and
   bit timing. Send something trivial (a counter byte) and confirm it
   arrives unchanged.
3. **Tighten the filter** to your real ID(s) once basic transport works.
4. **Add your real application logic** on top — at this point CAN is a
   solved problem and you're just deciding what data means what.

---

## 7. Troubleshooting

| Symptom | Likely cause |
|---|---|
| Works in loopback, nothing received from the other board | Missing/wrong termination, H/L swapped, transceiver unpowered |
| Bus works briefly then goes Bus-Off | Bit-timing mismatch between boards, or only one termination resistor present |
| Intermittent frame loss | Missing termination (reflections), or no shared ground between boards |
| Callback never fires | NVIC RX0 interrupt not enabled, or `CAN1_RX0_IRQHandler` missing/not calling `HAL_CAN_IRQHandler()` |
| Nothing at all, no bus errors either | Filter misconfigured — no message is entering the RX FIFO |
| CAN2-based node won't init | CAN2 is a slave peripheral — CAN1's clock/filter banks must be initialized even if you never use CAN1 directly |

---

## 8. Recap — the minimum you need to remember

- Wire: STM32 TX→transceiver D, STM32 RX←transceiver R, H↔H, L↔L, 120Ω at
  both ends, shared GND.
- CubeMX: remap CAN1 to PD0/PD1, set Prescaler/BS1/BS2 for your actual APB1
  clock, enable NVIC RX0 if receiving by interrupt.
- Code: `HAL_CAN_ConfigFilter()` + `HAL_CAN_Start()` on every node →
  `HAL_CAN_AddTxMessage()` to send → `HAL_CAN_GetRxMessage()` (polling) or
  `HAL_CAN_RxFifo0MsgPendingCallback()` (interrupt) to receive.

From here, what you send and what you do with what you receive is entirely
up to your application.
