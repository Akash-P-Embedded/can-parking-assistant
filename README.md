# 🚗 CAN-Based Reverse Parking Assistance System

An interrupt-driven, dual-node reverse parking assistant built on **LPC2129 ARM7 microcontrollers** using the **CAN 2.0B protocol** — implemented entirely at register level with no HAL abstraction.

Developed during advanced embedded systems training at **Vector India, Bengaluru**.

---

## 📌 Project Overview

Most parking sensors use a single microcontroller with a direct sensor-to-buzzer path. This project simulates a **real automotive ECU topology** — two nodes communicating over a CAN bus, just like body electronics in a modern vehicle.

```
┌──────────────────────────┐         CAN Bus (125 kbps)        ┌──────────────────────────┐
│         NODE A            │◄────────────────────────────────►│         NODE B            │
│       Alert Node          │                                   │      Sensor Node          │
│                           │  ──── RTR Frame (EINT0 ISR) ───► │                           │
│  • Button → EINT0 ISR     │                                   │  • HC-SR04 Ultrasonic     │
│  • Sends RTR on trigger   │  ◄──── Distance Data (CAN RX) ─── │  • Timer-based echo cap.  │
│  • Drives LED + Buzzer    │                                   │  • Responds with range()  │
│  • 4-level alert logic    │                                   │  • 250 ms response cycle  │
└──────────────────────────┘                                   └──────────────────────────┘
```

---

## ⚙️ Features

- ✅ CAN 2.0B communication between two LPC2129 nodes at 125 kbps
- ✅ **External interrupt (EINT0)** on Node A triggers RTR frame transmission
- ✅ **VIC** configured for CAN RX and EINT0 on Node A; CAN RX on Node B
- ✅ **Timer0-based echo measurement** on Node B for accurate HC-SR04 distance capture
- ✅ **4-level adaptive alert** — buzzer pattern based on distance range
- ✅ **UART debug output** — distance printed to terminal at 9600 baud on both nodes
- ✅ Entire peripheral config at **register level** — no HAL, no abstraction layer

---

## 🔔 Alert Logic (Node A)

| Distance | Buzzer Pattern | Delay |
|----------|---------------|-------|
| ≤ 99 cm  | Continuous ON 🔴 | — |
| 100–199 cm | Fast beep 🟠 | 100 ms |
| 200–299 cm | Medium beep 🟡 | 250 ms |
| 300–400 cm | Slow beep 🟢 | 500 ms |

---

## 🧰 Hardware

| Component | Details |
|-----------|---------|
| Microcontroller | NXP LPC2129 (ARM7TDMI-S) × 2 |
| Distance Sensor | HC-SR04 — TRIG: P0.8, ECHO: P0.9 (Node B) |
| CAN Transceiver | MCP2551 or equivalent |
| LED | P0.18 (Node A) |
| Buzzer | P0.21 (Node A) |
| Button | EINT0 / P0.14 (Node A) |
| Debug | UART0 @ 9600 baud (both nodes) |

---

## 💻 Software Stack

| Item | Detail |
|------|--------|
| Language | Embedded C |
| IDE | Keil uVision 5 |
| Architecture | ARM7TDMI-S (LPC2129) |
| Protocol | CAN 2.0B |
| Interrupt controller | VIC (Vectored Interrupt Controller) |

---

## 📁 Repository Structure

```
can-parking-assistant/
│
├── NodeA_Alert/
│   ├── main.c                  # Main loop — LED/buzzer alert logic
│   ├── can1_driver.c           # CAN2 init, TX, RX ISR
│   ├── external_interrupt.c    # EINT0 ISR — sends RTR on button press
│   ├── delay.c                 # Timer-based delays (ms, sec, us)
│   ├── uart_driver.c           # UART0 full driver
│   └── header.h                # Typedefs, CAN2 struct, extern declarations
│
├── NodeB_Sensor/
│   ├── main_nodeB.c            # Main loop — measures distance, sends DATA frame
│   ├── can1_driver.c           # CAN2 init, TX, RX ISR (RTR detection)
│   ├── sensor.c                # HC-SR04 trigger + Timer0 echo measurement
│   ├── delay.c                 # Timer-based delays
│   ├── driver_p1.c             # UART0 driver
│   └── header.h                # Shared header
│
└── README.md
```

---

## 📋 Source Code

### Shared — `header.h`

```c
#include <LPC21xx.H>
typedef unsigned int  u32;
typedef unsigned char u8;

typedef struct CAN2_MSG {
    u32 id;
    u32 byteA;
    u32 byteB;
    u8  rtr;
    u8  dlc;
} CAN2;

extern void can2_init(void);
extern void can2_rx(CAN2 *ptr);
extern void can2_tx(CAN2 v);
extern void config_vic_for_can2(void);
extern void send_DF(void);
extern int  range(void);
extern void delay_msec(unsigned int msec);
extern void delay_usec(unsigned int usec);
extern void uart0_init(unsigned int baud);
extern void uart0_tx_string(char *);
extern void uart0_integer(int);
extern void config_vic_for_eint0(void);
extern void config_eint0(void);
```

---

### Node A — `main.c` (Alert Node)

```c
#include <LPC21xx.H>
#include "header.h"

#define LED (1 << 18)
#define BUZ (1 << 21)

CAN2 v2;
u32  flag = 0, flag1 = 0;

main() {
    IODIR0 |= BUZ;
    IODIR0 |= LED;
    IOSET0 |= LED;

    can2_init();
    config_vic_for_can2();
    config_eint0();
    config_vic_for_eint0();
    uart0_init(9600);

    while (1) {
        if (flag == 1) {
            IOCLR0 |= LED;

            if (v2.byteA >= 300 && v2.byteA <= 400 && flag1 == 1) {
                IOSET0 |= BUZ; IOCLR0 |= BUZ;
                flag1 = 0;
                uart0_integer(v2.byteA); uart0_tx_string("cm \r\n");
                delay_msec(500);
            }
            if (v2.byteA >= 200 && v2.byteA <= 299 && flag1 == 1) {
                IOSET0 |= BUZ; IOCLR0 |= BUZ;
                flag1 = 0;
                uart0_integer(v2.byteA); uart0_tx_string("cm \r\n");
                delay_msec(250);
            }
            if (v2.byteA >= 100 && v2.byteA <= 199 && flag1 == 1) {
                IOSET0 |= BUZ; IOCLR0 |= BUZ;
                flag1 = 0;
                uart0_integer(v2.byteA); uart0_tx_string("cm \r\n");
                delay_msec(100);
            }
            if (v2.byteA <= 99 && flag1 == 1) {
                IOSET0 |= BUZ;
                flag1 = 0;
                uart0_integer(v2.byteA); uart0_tx_string("cm \r\n");
            }
        } else {
            IOSET0 |= LED;
        }
    }
}
```

---

### Node A — `can1_driver.c`

```c
#include <LPC21xx.H>
#include "header.h"

extern CAN2 v2;
extern u32  flag1;

void CAN2_Rx_Handler(void) __irq {
    v2.id  = C2RID;
    v2.dlc = (C2RFS >> 16) & 0xFF;
    v2.rtr = (C2RFS >> 30) & 1;

    if (v2.rtr == 1) {
        uart0_tx_string("remote frame \r\n");
    } else {
        flag1    ^= 1;       // toggle to signal new data ready
        v2.byteA  = C2RDA;   // store distance from Node B
    }
    C2CMR       = (1 << 2);  // release RX buffer
    VICVectAddr = 0;
}

void config_vic_for_can2(void) {
    VICIntSelect  = 0;
    VICVectAddr5  = (u32)CAN2_Rx_Handler;
    VICVectCntl5  = 27 | (1 << 5);
    VICIntEnable  = (1 << 27);
    C2IER         = 1;
}

void can2_init(void) {
    VPBDIV   = 1;
    PINSEL1 |= 0x14000;
    C2MOD    = 1;            // reset mode
    C2BTR    = 0x001C001D;   // 125 kbps
    AFMR     = 2;            // bypass acceptance filter
    C2MOD    = 0;            // release reset
}

void can2_tx(CAN2 v) {
    C2TID1 = v.id;
    C2TFI1 = v.dlc << 16;
    if (v.rtr == 0) {
        C2TDA1 = v.byteA;
        C2TDB1 = v.byteB;
    } else {
        C2TFI1 |= (1 << 30);   // set RTR bit
    }
    C2CMR = 1 | (1 << 5);      // select TX buffer 1, start TX
    while (((C2GSR >> 3) & 1) == 0);
}
```

---

### Node A — `external_interrupt.c`

```c
#include <LPC21xx.H>
#include "header.h"

extern u32 flag;

void EINT0_Handler(void) __irq {
    CAN2 v1;
    v1.id  = 0x27;
    v1.dlc = 8;
    v1.rtr = 1;          // Remote Frame — request distance from Node B
    can2_tx(v1);

    flag  ^= 1;
    EXTINT = 1;          // clear EINT0 interrupt flag
    VICVectAddr = 0;
}

void config_vic_for_eint0(void) {
    VICIntSelect  = 0;
    VICVectAddr0  = (unsigned int)EINT0_Handler;
    VICVectCntl0  = 14 | (1 << 5);
    VICIntEnable  = (1 << 14);
}

void config_eint0(void) {
    PINSEL1  |= 1;
    EXTMODE   = 1;   // edge-triggered
    EXTPOLAR  = 0;   // falling edge
}
```

---

### Node B — `main_nodeB.c` (Sensor Node)

```c
#include <LPC21xx.H>
#include "header.h"

CAN2 k2;
u32  flag = 0;

main() {
    u32 distance;

    can2_init();
    config_vic_for_can2();
    uart0_init(9600);

    while (1) {
        if (flag == 1) {     // set by CAN RX ISR when RTR is received
            k2.id  = 0x24;
            k2.rtr = 0;      // DATA frame
            k2.dlc = 1;

            distance = range();
            k2.byteA = distance;

            uart0_integer(distance);
            uart0_tx_string("\r\n");

            can2_tx(k2);     // send distance back to Node A
            delay_msec(250);
        }
    }
}
```

---

### Node B — `can1_driver.c`

```c
#include <LPC21xx.H>
#include "header.h"

extern CAN2 k2;
extern u32  flag;

void CAN2_Rx_Handler(void) __irq {
    k2.id  = C2RID;
    k2.dlc = (C2RFS >> 16) & 0xFF;
    k2.rtr = (C2RFS >> 30) & 1;

    if (k2.rtr == 1) {
        flag ^= 1;           // RTR received — trigger measurement in main loop
    }
    C2CMR       = (1 << 2);
    VICVectAddr = 0;
}

void config_vic_for_can2(void) {
    VICIntSelect  = 0;
    VICVectAddr6  = (u32)CAN2_Rx_Handler;
    VICVectCntl6  = 27 | (1 << 5);
    VICIntEnable  = (1 << 27);
    C2IER         = 1;
}

void can2_init(void) {
    VPBDIV   = 1;
    PINSEL1 |= 0x14000;
    C2MOD    = 1;
    C2BTR    = 0x001C001D;   // 125 kbps
    AFMR     = 2;
    C2MOD    = 0;
}

void can2_tx(CAN2 v) {
    C2TID1 = v.id;
    C2TFI1 = v.dlc << 16;
    if (v.rtr == 0) {
        C2TDA1 = v.byteA;
        C2TDB1 = v.byteB;
    } else {
        C2TFI1 |= (1 << 30);
    }
    C2CMR = 1 | (1 << 5);
    while (((C2GSR >> 3) & 1) == 0);
}
```

---

### Node B — `sensor.c`

```c
#include <LPC21xx.H>
#include "header.h"

#define TRIG (1 << 8)   // P0.8
#define ECHO (1 << 9)   // P0.9

void send_DF(void) {
    IODIR0 |= TRIG;
    IOSET0  = TRIG;
    delay_usec(10);      // 10 µs trigger pulse
    IOCLR0  = TRIG;
}

int range(void) {
    u32 dis;
    send_DF();

    T0TC = 0;
    T0PC = 0;

    while (((IOPIN0 >> 9) & 1) == 0);  // wait for ECHO to go HIGH
    T0TCR = 1;                           // start Timer0
    while (((IOPIN0 >> 9) & 1) == 1);  // wait for ECHO to go LOW
    T0TCR = 0;                           // stop Timer0

    dis = T0TC;

    // distance (cm) = echo_time_us / 58  (speed of sound round trip)
    if (dis < 38000)
        dis = dis / 59;
    else
        dis = 0;    // out of range (> ~650 cm)

    return dis;
}
```

---

## 🔄 Full System Flow

```
USER PRESSES BUTTON (Node A, P0.14)
        │
        ▼
  EINT0 ISR fires (falling edge)
        │
  Node A sends RTR Frame (ID: 0x27) ──────────────────► Node B CAN2 RX ISR
                                                                │
                                                          flag ^= 1
                                                                │
                                                     Main loop detects flag==1
                                                                │
                                                         range() called
                                                                │
                                               send_DF() → 10µs pulse on P0.8
                                                                │
                                               Timer0 counts echo width on P0.9
                                                                │
                                               distance = T0TC / 59  (cm)
                                                                │
                                         Node B sends DATA Frame (ID: 0x24, byteA=distance)
                                                                │
  Node A CAN2 RX ISR ◄────────────────────────────────────────────
        │
  v2.byteA = distance, flag1 ^= 1
        │
  Main loop → alert based on range
        │
  ┌─────────────────────────────┐
  │ ≤99cm   → buzzer continuous │
  │ 100–199 → beep 100ms delay  │
  │ 200–299 → beep 250ms delay  │
  │ 300–400 → beep 500ms delay  │
  └─────────────────────────────┘
```

---

## 🧠 What This Project Demonstrates

| Concept | Implementation |
|---------|---------------|
| CAN 2.0B | RTR + DATA frames, register-level config (C2BTR, C2TFI, C2CMR), 125 kbps |
| VIC | Priority-based vectored interrupt assignment for CAN RX and EINT0 |
| ISR design | Short ISRs with flag-based handoff — processing done in main loop |
| Timer-based sensing | T0TC polling for HC-SR04 echo pulse width measurement |
| Multi-node system | Two MCUs communicating over CAN — mirrors real automotive ECU topology |
| Bare-metal C | No HAL, no RTOS — direct register access throughout |
| UART diagnostics | Real-time distance streaming at 9600 baud for debug and validation |

---

## 🚀 How to Build & Run

1. Open `NodeA_Alert/NodeA.uvprojx` in Keil uVision → Build → Flash to first LPC2129
2. Open `NodeB_Sensor/NodeB.uvprojx` → Build → Flash to second LPC2129
3. Connect both boards via MCP2551 CAN transceiver
4. HC-SR04: TRIG → P0.8, ECHO → P0.9 on Node B
5. Button → EINT0 (P0.14) on Node A
6. Power on → press button → observe LED/buzzer + UART output

**UART debug:** Connect P0.0/P0.1 to USB-serial adapter, open terminal at **9600 baud** — distance in cm printed on every measurement cycle.

---

## 👤 Author

**Akash P** — Embedded Software Engineer
📎 [LinkedIn](https://www.linkedin.com/in/akash-p-1294a12b3/)
🔗 [GitHub Profile](https://github.com/akash-embedded)

---

