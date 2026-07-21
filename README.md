# CAN Bus Basic Demo — Button Press → LED Cycle

A minimal two-board CAN test: pressing the user button on **Board A**
advances an LED cycle on **Board B** (Green → Orange → Red → Blue → repeat).
One board only transmits, the other only receives.

This README covers what's *specific* to this project. For the general CAN
setup (wiring theory, bit-timing math, CubeMX peripheral config, what each
HAL function does) see **[`CAN_Interface_Reference.md`](./CAN_Interface_Reference.md)**
— this doc assumes you've read that and just fills in the exact values and
code used here.

**Hardware:** 2× STM32F407VGT6 Discovery, 2× SN65HVD230 transceiver module.

---

## 1. Roles

| Board | Role | Extra I/O used |
|---|---|---|
| **A** | Transmitter | PA0 (onboard user button, input) |
| **B** | Receiver | PD12–PD15 (onboard LEDs, output) |

CAN wiring, termination, and grounding between the two boards: identical to
**Reference §1** — no changes for this project.

---

## 2. Clock Config Used Here

This project uses the **HSI-only** clock tree (no PLL) — check this in your
`.ioc` under Clock Configuration before trusting the CAN values below:

- SYSCLK = HCLK = PCLK1 = PCLK2 = **16 MHz**

This matters because it drives the CAN1 bit-timing values in §3 — if your
copy of this project uses a different `SystemClock_Config()` (e.g. HSE+PLL),
recompute per **Reference §2** instead of reusing these numbers.

---

## 3. CAN1 Settings Used Here

Per **Reference §2/§3**, for 16 MHz APB1 targeting 500 kbit/s:

| Setting | Value |
|---|---|
| Prescaler | 2 |
| Time Quanta BS1 | 13 |
| Time Quanta BS2 | 2 |
| ReSync Jump Width | 1 |
| Mode | Master, Normal |
| Automatic Retransmission | Enable |
| Everything else | Disable |

`16MHz / (2 × 16) = 500 kbit/s`, sample point 87.5%.

**Pins:** PD0 = CAN1_RX, PD1 = CAN1_TX (both boards) — same remap as
Reference §1/§3, done to avoid the USB OTG conflict on PA11/PA12.

**NVIC:** CAN1 RX0 interrupt enabled on **Board B only** (it's the one
receiving via callback). Harmless to also enable on Board A, not required.

---

## 4. Extra CubeMX GPIO for This Project

Beyond the CAN1 config in the reference doc:

**Board A:**
- PA0 → `GPIO_Input`, no pull (Discovery board already has an external
  pull-down on this pin — button press reads HIGH)

**Board B:**
- PD12, PD13, PD14, PD15 → `GPIO_Output` (Green/Orange/Red/Blue onboard
  LEDs — this is CubeMX's default when the Discovery board template is used,
  just confirm it wasn't changed)

---

## 5. Protocol (deliberately trivial)

| Field | Value |
|---|---|
| Arbitration ID | `0x100` |
| DLC | 1 byte |
| Payload | `0xAA` (fixed — content is meaningless here) |

The receiver doesn't interpret the payload at all — it reacts to "a frame
with ID `0x100` arrived." This keeps the test isolated to "does CAN
transport work," separate from any real payload design. When you move to a
project with actual data, replace this with your real frame format and
build the filter/DLC/payload handling around that instead — see Reference
§4.2–4.4.

---

## 6. Board A — Transmitter Code

Filter setup, peripheral start, and TX header: **accept-all filter is fine
here** (Board A never reads its RX FIFO) — this is exactly Reference §4.1,
plus the header setup from §4.2 with `StdId = 0x100, DLC = 1`.

```c
/* USER CODE BEGIN PV */
CAN_TxHeaderTypeDef TxHeader;
uint8_t TxData[1];
uint32_t TxMailbox;

uint8_t buttonLastState = 0;
uint32_t lastDebounceTime = 0;
#define DEBOUNCE_MS 50
/* USER CODE END PV */
```

```c
/* USER CODE BEGIN 2 */
CAN_FilterTypeDef canFilter;
canFilter.FilterBank = 0;
canFilter.FilterMode = CAN_FILTERMODE_IDMASK;
canFilter.FilterScale = CAN_FILTERSCALE_32BIT;
canFilter.FilterIdHigh = 0x0000;
canFilter.FilterIdLow = 0x0000;
canFilter.FilterMaskIdHigh = 0x0000;
canFilter.FilterMaskIdLow = 0x0000;
canFilter.FilterFIFOAssignment = CAN_FILTER_FIFO0;
canFilter.FilterActivation = ENABLE;
canFilter.SlaveStartFilterBank = 14;

if (HAL_CAN_ConfigFilter(&hcan1, &canFilter) != HAL_OK) { Error_Handler(); }
if (HAL_CAN_Start(&hcan1) != HAL_OK) { Error_Handler(); }

TxHeader.StdId = 0x100;
TxHeader.ExtId = 0;
TxHeader.IDE = CAN_ID_STD;
TxHeader.RTR = CAN_RTR_DATA;
TxHeader.DLC = 1;
TxHeader.TransmitGlobalTime = DISABLE;
/* USER CODE END 2 */
```

The only project-specific logic is *when* to call `HAL_CAN_AddTxMessage()`
(Reference §4.2) — here, on a debounced button press:

```c
/* USER CODE BEGIN WHILE */
while (1)
{
  uint8_t buttonState = HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0);

  if (buttonState == GPIO_PIN_SET && buttonLastState == GPIO_PIN_RESET
      && (HAL_GetTick() - lastDebounceTime) > DEBOUNCE_MS)
  {
    TxData[0] = 0xAA;

    if (HAL_CAN_AddTxMessage(&hcan1, &TxHeader, TxData, &TxMailbox) != HAL_OK)
    {
      Error_Handler();
    }

    lastDebounceTime = HAL_GetTick();
  }

  buttonLastState = buttonState;
  /* USER CODE END WHILE */

  /* USER CODE BEGIN 3 */
}
/* USER CODE END 3 */
```

---

## 7. Board B — Receiver Code

Filter is tightened to `0x100` per Reference §4.4 (rather than the
accept-all used on Board A), since Board B has a specific ID it cares about.
Reception is interrupt-driven per Reference §4.3 Option B.

```c
/* USER CODE BEGIN PV */
CAN_RxHeaderTypeDef RxHeader;
uint8_t RxData[8];

uint8_t ledIndex = 0;
uint16_t ledPins[4] = {GPIO_PIN_12, GPIO_PIN_13, GPIO_PIN_14, GPIO_PIN_15};
/* USER CODE END PV */
```

```c
/* USER CODE BEGIN PFP */
void HAL_CAN_RxFifo0MsgPendingCallback(CAN_HandleTypeDef *hcan);
/* USER CODE END PFP */
```

```c
/* USER CODE BEGIN 2 */
CAN_FilterTypeDef canFilter;
canFilter.FilterBank = 0;
canFilter.FilterMode = CAN_FILTERMODE_IDMASK;
canFilter.FilterScale = CAN_FILTERSCALE_32BIT;
canFilter.FilterIdHigh = (0x100 << 5);
canFilter.FilterIdLow = 0x0000;
canFilter.FilterMaskIdHigh = (0x7FF << 5);
canFilter.FilterMaskIdLow = 0x0000;
canFilter.FilterFIFOAssignment = CAN_FILTER_FIFO0;
canFilter.FilterActivation = ENABLE;
canFilter.SlaveStartFilterBank = 14;

if (HAL_CAN_ConfigFilter(&hcan1, &canFilter) != HAL_OK) { Error_Handler(); }
if (HAL_CAN_Start(&hcan1) != HAL_OK) { Error_Handler(); }

if (HAL_CAN_ActivateNotification(&hcan1, CAN_IT_RX_FIFO0_MSG_PENDING) != HAL_OK)
{
  Error_Handler();
}

// Start with LED 0 (green, PD12) on, rest off
HAL_GPIO_WritePin(GPIOD, ledPins[0], GPIO_PIN_SET);
/* USER CODE END 2 */
```

The application-specific part is the callback body — everything above the
`if (RxHeader.StdId == 0x100)` check is generic per Reference §4.3:

```c
/* USER CODE BEGIN 4 */
void HAL_CAN_RxFifo0MsgPendingCallback(CAN_HandleTypeDef *hcan)
{
  if (HAL_CAN_GetRxMessage(hcan, CAN_RX_FIFO0, &RxHeader, RxData) == HAL_OK)
  {
    if (RxHeader.StdId == 0x100)
    {
      HAL_GPIO_WritePin(GPIOD, ledPins[ledIndex], GPIO_PIN_RESET);
      ledIndex = (ledIndex + 1) % 4;
      HAL_GPIO_WritePin(GPIOD, ledPins[ledIndex], GPIO_PIN_SET);
    }
  }
}
/* USER CODE END 4 */
```

Board B's `while(1)` loop stays empty — this design is fully
interrupt-driven, nothing needs polling.

**Before flashing Board B:** confirm `stm32f4xx_it.c` has a working
`CAN1_RX0_IRQHandler()` calling `HAL_CAN_IRQHandler(&hcan1)` — CubeMX
generates this automatically once the NVIC checkbox in §3 is set, but it's
worth a 10-second check since a missing/empty handler is the most common
reason the callback silently never fires (see Reference §7).

---

## 8. Testing

1. Power **Board B** first — green LED (PD12) should light immediately on
   boot (from the init code).
2. Power **Board A**.
3. Press Board A's blue user button — Board B's LED should advance one step
   per press: green → orange → red → blue → green → ...
4. If nothing happens, work through **Reference §6 (bring-up order)** and
   **§7 (troubleshooting table)** in order rather than guessing — termination
   and NVIC interrupt setup are the two most common misses.

---

## 9. Extending This Project

This transmit/receive pair is the minimum skeleton. To turn it into
something closer to the real sensor/actuator project this demo is meant to
lead into:

- Replace the 1-byte fixed payload with a real multi-field frame (e.g.
  command + distance + counter + MAC) — DLC can go up to 8 bytes.
- Add a second filter bank (`FilterBank = 1`) on Board B if it needs to
  accept more than one arbitration ID.
- If a board needs to both send and receive, combine Board A's TX setup and
  Board B's RX setup in the same project — see Reference §5
  ("bidirectional node").
- Move from a bare `while(1)`/interrupt-callback structure to FreeRTOS tasks
  once acquisition, classification, and transmission need to run as
  separate concerns.
