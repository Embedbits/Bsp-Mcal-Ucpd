# UCPD Peripheral Driver

This module provides an abstraction layer for the **USB Type-C / USB Power Delivery interface** (`UCPD`) of STM32 MCUs: the analog front
end of the two configuration channel lines (`CC1`, `CC2`) with the Rp / Rd resistors, the **connection detection and the cable
orientation**, the dead-battery support, and the **physical layer of the USB Power Delivery** - the BMC coding, the 4b5b coding, the CRC, the ordered
sets and the byte-level transmit and receive interface for the DMA. It is a low-level MCAL module: it owns the UCPD hardware,
the application and the other modules never touch the registers.

The public interface (`Ucpd_Port.h`, `Ucpd_Types.h`) is **identical for all supported families**. The implementation
for a family lives in its own branch (`Dev/<Family>` for development, `Releases/<Family>` for the released versions), so
a module or an application written against the interface does not change when the MCU family changes.

This README is the landing page and the **specification of the interface** (master). The family branches contain their
own README with everything that is specific to the family: the instances of the device, their settings and the status of
the implementation.

> **Status: design.** No implementation exists yet, the interface below is a proposal. Stories and branches follow.

---

## Intent and purpose

- A device with a USB-C connector has to answer two questions before anything else works: **is a partner connected, and as what?**
  (the voltage on `CC1` / `CC2` against the thresholds tells a sink from a source, the cable orientation and the current the source
  offers) and **may I draw or provide the power?** (the Power Delivery negotiation of the voltage and the current over the CC wire). The module
  owns the first question completely and the transport of the second one.
- The device can be a **sink** (it presents Rd and takes the power: a battery powered gadget, a display), a **source** (it presents
  Rp, offers the current and the power: a charger, a hub, a dock) or a **dual-role** device that toggles between them. The module configures the role,
  the Rp value that advertises the source current (default USB, 1.5 A, 3 A) and reports the attach / detach and the resulting state per CC line.
- The Power Delivery needs a stack of the state machines (the protocol layer, the policy engine, the device policy manager - the specification
  is large). They are a **middleware** that sits on the top of this module; the module delivers the messages as bytes with the ordered set
  and the header, and takes the messages to send, with the CRC, the retry timing at the physical level and the hard reset signalling.
- The **dead battery** support lets a device with a flat battery be powered by a Type-C source: the CC pins present the Rd by
  hardware right after the power-up, before any software runs. When the software starts, it takes over the control of the resistors
  (or switches the dead-battery behaviour off if the interface is not used, so that the pull-downs do not remain on the pins).
- The module hides the differences between the families: the number of the CC channels, the Rp / Rd control, the dead-battery bits
  (in the UCPD or in the PWR / SBS block), the clock requirements and the FRS support differ, the interface stays the same.

What the module is **not**: it does not implement the Power Delivery protocol (the message state machines, the contract
negotiation, the structured vendor messages), it does not switch the VBUS (a power switch controlled by the application) and it does not
measure the VBUS voltage (the Adc module).

---

## Concept

| Item | Meaning |
|---|---|
| **Role** (`ucpd_Role_t`) | `UCPD_ROLE_SINK`, `UCPD_ROLE_SOURCE`, `UCPD_ROLE_DRP` (the module toggles Rp / Rd with the programmed periods). |
| **Rp value** (`ucpd_RpValue_t`) | `UCPD_RP_DEFAULT`, `UCPD_RP_1_5A`, `UCPD_RP_3_0A` - the current a source advertises. |
| **CC state** (`ucpd_CcState_t`) | Per line `UCPD_CC_STATE_OPEN`, `_RA`, `_RD`, `_RP_DEFAULT`, `_RP_1_5A`, `_RP_3_0A` - what the partner presents, debounced by the hardware. |
| **Orientation** | The CC line that carries the communication (the other one is `VCONN`); the module reports it and routes the PD receiver / transmitter to it. |
| **PD message** (`ucpd_PdMessage_t`) | The ordered set (`SOP`, `SOP'`, `SOP''`, hard reset, cable reset), the header and the data objects as bytes, the length. |
| **Timing** | The PD clock prescalers (the hardware clock of the BMC) and the inter-frame gap, the transition window; derived by the module from the kernel clock. |

Rules:

1. `Ucpd_Init` configures the role, the Rp, the PD clock and the interrupt and starts the CC line detection. The dead-battery pull-downs are released
   at the end of `Ucpd_Init` (the module now controls Rd), or, when the interface is **not** used, by `Ucpd_Set_DeadBatteryInactive` from the start-up
   code of the platform, so that the pull-downs never stay on the CC pins by accident.
2. `Ucpd_Get_CcState` returns the detected state of both lines and the attach / detach event is reported by the `ConnectionCallback` with the orientation.
3. The reception of the PD messages is started by `Ucpd_Set_PdRxStart( buffer, size )` (the receiver is enabled for the selected ordered sets, the mask is
   programmable); the `PdRxCallback` delivers a message (the ordered set, the length, the CRC status). The transmission is a request
   `Ucpd_Set_PdTxStart( message )`; the `PdTxCallback` reports the end (sent, discarded by a received message, the retry timeout).
4. The **hard reset** and the **cable reset** are sent by `Ucpd_Set_HardReset` / `Ucpd_Set_CableReset`; the received ones are reported by the
   callback. The fast role swap (FRS) detection is enabled by `Ucpd_Set_FrsActive`.
5. The transmit and the receive data move by the DMA (or the interrupt); the same request and the same callbacks are used in both modes.
6. All parameters are checked before a register is written; a role change or a Rp change is accepted only when the interface is not in a PD
   transaction.

---

## Supported families

Every family gets its own story, its own tasks and its own branches `Dev/<Family>` and `Releases/<Family>` cut from this
master.

| Family | Branch | Interface | Status |
|---|---|---|---|
| STM32H5 | `Dev/STM32H5` | `UCPD1` on all lines except STM32H503: USB Type-C 2.4, USB PD 2.0 / 3.2 (PPS), Rp (all values) and Rd, dead-battery Rd, BMC and 4b5b, CRC, ordered sets, FRS Rx, operation in Stop mode, DMA | planned (first) |
| STM32U5 | `Dev/STM32U5` | `UCPD1` | planned |
| STM32G4, STM32L4 | `Dev/STM32G4`, `Dev/STM32L4` | `UCPD1` on the lines that have it | planned |
| STM32H7 | `Dev/STM32H7` | device dependent | planned |
| STM32U3 | `Dev/STM32U3` | device dependent | planned |
| STM32F4, STM32F7 | `Dev/STM32F4`, `Dev/STM32F7` | no UCPD on the device - the module is a stub that returns `UCPD_REQUEST_ERROR` | planned |

A function without a meaning in a family stays in the interface, returns `UCPD_REQUEST_ERROR` and suppresses the unused
parameters (for example the fast role swap on a device without it, the second CC channel of a single-channel device).

---

## Public interface in short (proposal)

| Group | Functions |
|---|---|
| Module management | `Ucpd_Get_ModuleVersion`, `Ucpd_Init`, `Ucpd_Deinit`, `Ucpd_Task`, `Ucpd_Get_DefaultConfig` |
| CC lines | `Ucpd_Set_Role` / `Get`, `Ucpd_Set_RpValue` / `Get`, `Ucpd_Get_CcState`, `Ucpd_Get_Orientation`, `Ucpd_Set_DeadBatteryInactive` |
| Power Delivery | `Ucpd_Set_PdRxStart`, `Ucpd_Set_PdRxStop`, `Ucpd_Set_PdTxStart`, `Ucpd_Get_PdTxState`, `Ucpd_Set_RxOrderedSetMask`, `Ucpd_Set_HardReset`, `Ucpd_Set_CableReset` |
| Fast role swap | `Ucpd_Set_FrsActive`, `Ucpd_Set_FrsInactive`, `Ucpd_Set_FrsSignal` |
| Interrupts | `Ucpd_Set_IrqPriority` / `Get`, the callbacks are a part of the configuration |

```c
ucpd_ModuleVersion_t Ucpd_Get_ModuleVersion        ( void );

ucpd_RequestState_t  Ucpd_Init                     ( const ucpd_Config_t * const ucpdConfig );
ucpd_RequestState_t  Ucpd_Deinit                   ( void );
void                 Ucpd_Task                     ( void );
ucpd_RequestState_t  Ucpd_Get_DefaultConfig        ( ucpd_Config_t * const ucpdConfig );

ucpd_RequestState_t  Ucpd_Set_Role                 ( ucpd_Role_t role );
ucpd_RequestState_t  Ucpd_Set_RpValue              ( ucpd_RpValue_t rpValue );
ucpd_RequestState_t  Ucpd_Get_CcState              ( ucpd_CcState_t * const cc1State, ucpd_CcState_t * const cc2State );
ucpd_RequestState_t  Ucpd_Get_Orientation          ( ucpd_Orientation_t * const orientation );
ucpd_RequestState_t  Ucpd_Set_DeadBatteryInactive  ( void );

ucpd_RequestState_t  Ucpd_Set_PdRxStart            ( ucpd_Data_t * const buffer, ucpd_DataCnt_t bufferSize );
ucpd_RequestState_t  Ucpd_Set_PdTxStart            ( const ucpd_PdMessage_t * const message );
ucpd_RequestState_t  Ucpd_Set_HardReset            ( void );
ucpd_RequestState_t  Ucpd_Set_FrsActive            ( void );
```

Every function returns `UCPD_REQUEST_OK` / `UCPD_REQUEST_ERROR` and checks its parameters and the state of the module
before it touches a register.

| Type | Meaning |
|---|---|
| `ucpd_Config_t` | Role, Rp value, the DRP periods, the PD clock, the received ordered set mask, transfer mode (`UCPD_XFER_MODE_ISR` / `DMA`), DMA channels, callbacks, interrupt priority |
| `ucpd_PdMessage_t` | The ordered set, the header and the data bytes, the length |
| `ucpd_Orientation_t` | `UCPD_ORIENTATION_NONE`, `UCPD_ORIENTATION_CC1`, `UCPD_ORIENTATION_CC2` |
| Callbacks | `ConnectionCallback( ccState, orientation )`, `PdRxCallback( message, crcStatus )`, `PdTxCallback( result )`, `HardResetCallback`, `FrsCallback`, `ErrorCallback` |

---

## Usage

### A sink that detects the source and its current

```c
#include "Ucpd_Port.h"

static void App_Connection( ucpd_CcState_t ccState, ucpd_Orientation_t orientation )
{
    /* ccState tells the advertised current (default USB / 1.5 A / 3 A); start the PD stack on the orientation */
}

ucpd_Config_t ucpdConfig;

(void)Ucpd_Get_DefaultConfig( &ucpdConfig );
ucpdConfig.Role               = UCPD_ROLE_SINK;
ucpdConfig.ConnectionCallback = App_Connection;
ucpdConfig.XferMode           = UCPD_XFER_MODE_DMA;

(void)Ucpd_Init( &ucpdConfig );                 /* releases the dead-battery pull-downs, the module controls Rd now */
```

### A product that does not use Type-C

```c
(void)Ucpd_Set_DeadBatteryInactive();             /* called by the start-up code: the CC pull-downs must not stay on */
```

---

## Behaviour and limits

- **Dead battery.** After the power-up the CC pull-downs of the devices that have the dead-battery support are **on** until the software switches them
  off; a board without Type-C must switch them off, otherwise the CC pins keep a 5.1 kohm pull-down.
- **Timing.** The PD messages have hard time limits (the response within tens of milliseconds, the retry timeout); the middleware
  timers are the application's, the module keeps the physical timing (the bit clock, the inter-frame gap, the transition window).
- **Stop mode.** The detection of the attach / detach works in the Stop mode on the families that support it; the PD communication does not.
- **Safety.** The module cannot know the VBUS; a source implementation drives the power switch itself and must follow the
  discharge and the soft-start rules of the specification.
- **Not part of the first version:** the Power Delivery protocol and policy, the VCONN source switching, the
  audio accessory and the debug accessory modes, the extended messages beyond the transport, the Type-C authentication.

---

## Dependencies

- **Rcc** - peripheral clock, the kernel clock of the PD
- **Nvic** - interrupt
- **Gpdma** - DMA transfers of the PD data
- **Pwr** - the dead-battery bits of the families that keep them in the power controller
- **RAL (Register Abstraction Layer)** - used internally to access the registers

All mandatory RCC, NVIC and DMA configurations are handled internally.

---

## Testing

- **Unit tests** (host, Unity + CMock + RegMem): the CC state decoding for all the Rp / Rd combinations, the orientation, the role and the Rp
  change rules, the dead-battery sequence, the PD clock dividers, the message transmit / receive flow (CRC error, discard, retry timeout), the
  hard reset, the FRS path, the callbacks.
- **Integration tests** (target, probe-rs): the attach to a source and a sink with the known Rp / Rd values, the cable flip, a PD
  message exchange with a PD analyser as the partner, the hard reset.

---

## Interface conventions

The module follows the common rules of the MCAL layer, so that it behaves like the other modules:

- **Lists instead of free members.** Everything that the hardware restricts (the pins, the kernel clock sources, the DMA
  channels, the triggers and the other selectable sources) is a `typedef enum` list with exactly the items that are valid
  for the device line; a pin is an item `UCPD_<SIGNAL>_PIN_<INSTANCE>_P<port><pin>` and there is an `*_UNUSED` item. A choice
  that is not in the list cannot be written.
- **Own peripheral only.** The module calls only the registers / LL functions of its own peripheral. The clocks, the
  interrupts, the pins, the DMA and the EXTI lines go through the API of the Rcc, Nvic, Gpio, Gpdma and Exti modules; a
  missing function is added to that module, it is not bypassed.
- **Init verifies itself.** `Ucpd_Init` checks the result at the end; when it is not as required, the module is
  deinitialized and an error is returned. The waiting for the hardware is a loop on the register state with a timeout,
  never a blind delay.
- **Transfer modes.** Where the module moves data, the POLL (served by `Ucpd_Task`), ISR and DMA modes have the same
  configuration structure, the same request and the same callbacks (error, half transfer, transfer complete).
- **Flags and states are enums of the module** (`ucpd_FunctionState_t`, `ucpd_FlagState_t`), the limits and the constants are
  named, there are no magic numbers in the interface.
- **Unsupported functions stay.** A function without a meaning in a family keeps its signature, suppresses the unused
  parameters and returns `UCPD_REQUEST_ERROR`.
- **Documentation.** Every function and type has a Doxygen description, the main header has the `\defgroup`.

---

## Repository layout and branches

The master holds this README and the license. The family branches (`Dev/<Family>`, `Releases/<Family>`) hold:

| Path | Content |
|---|---|
| `README.md` | README of the family: the instances of the device, specifics, errata, status |
| `Ucpd_Port.h`, `Ucpd_Types.h` | Public interface (identical in all families) |
| `Ucpd*.c`, `Ucpd.h` | Implementation of the family |
| `CMakeLists.txt` | Library `Ucpd_Lib` of the EmBi platform |
| `Tests/UnitTests` | Unit tests (Unity + CMock + RegMem) |
| `Tests/IntegrationTests` | Integration tests on the target (probe-rs) |

A change of the public interface is made in all families.

---

## CMake integration

```cmake
target_link_libraries( User_Lib PRIVATE Ucpd_Lib )
```

1. Link `Ucpd_Lib` to your CMake library.
2. Include `Ucpd_Port.h` in your project.
3. Configure the module as needed for your hardware.

---

## License

This project is licensed under the **Creative Commons Attribution-NonCommercial 4.0 International (CC BY-NC 4.0)**.

You are free to use, modify, and share this work for **non-commercial purposes**, provided appropriate credit is given.

See [LICENSE.md](LICENSE.md) for full terms or visit [creativecommons.org/licenses/by-nc/4.0](https://creativecommons.org/licenses/by-nc/4.0/).

---

## Authors

- **Mr.Nobody** - [embedbits.com](https://embedbits.com)

Contributions are welcome! Please open a pull request.
