---
name: rust-embedded
description: Use when developing embedded, no_std, or bare-metal Rust for microcontrollers (ARM, RISC-V, ESP32, STM32, nRF). Covers HAL, PAC, probe-rs, defmt logging, panic-halt, alloc, static mut, volatile access, DMA, interrupt handler, real-time constraints, no_std setup, heapless collections, interrupt-safe state with Mutex RefCell, peripheral ownership, RTIC, Embassy, and cortex-m patterns.
---

# Embedded Development

## Domain Constraints

| Domain Rule | Design Constraint | Rust Implication |
|-------------|-------------------|------------------|
| No heap | Stack allocation | heapless, no Box/Vec |
| No std | Core only | #![no_std] |
| Real-time | Predictable timing | No dynamic alloc |
| Resource limited | Minimal memory | Static buffers |
| Hardware safety | Safe peripheral access | HAL + ownership |
| Interrupt safe | No blocking in ISR | Atomic, critical sections |

## Critical Rules

- **Cannot use heap** (no allocator) — use `heapless::Vec<T, N>` and arrays for deterministic memory.
- **Shared state must be interrupt-safe** — ISR can preempt at any time. Use `Mutex<RefCell<T>>` + critical section.
- **Peripherals must have clear ownership** — HAL takes ownership via singletons to prevent conflicting access.

## Layer Stack

| Layer | Examples | Purpose |
|-------|----------|---------|
| PAC | stm32f4, esp32c3 | Register access |
| HAL | stm32f4xx-hal | Hardware abstraction |
| Framework | RTIC, Embassy | Concurrency |
| Traits | embedded-hal | Portable drivers |

## Framework Comparison

| Framework | Style | Best For |
|-----------|-------|----------|
| RTIC | Priority-based | Interrupt-driven apps |
| Embassy | Async | Complex state machines |
| Bare metal | Manual | Simple apps |

## Key Crates

| Purpose | Crate |
|---------|-------|
| Runtime (ARM) | cortex-m-rt |
| Panic handler | panic-halt, panic-probe |
| Collections | heapless |
| HAL traits | embedded-hal |
| Logging | defmt |
| Flash/debug | probe-run |

## Static Peripheral Pattern

```rust
#![no_std]
#![no_main]

use cortex_m::interrupt::{self, Mutex};
use core::cell::RefCell;

static LED: Mutex<RefCell<Option<Led>>> = Mutex::new(RefCell::new(None));

#[entry]
fn main() -> ! {
    let dp = pac::Peripherals::take().unwrap();
    let led = Led::new(dp.GPIOA);

    interrupt::free(|cs| {
        LED.borrow(cs).replace(Some(led));
    });

    loop {
        interrupt::free(|cs| {
            if let Some(led) = LED.borrow(cs).borrow_mut().as_mut() {
                led.toggle();
            }
        });
    }
}
```

## Common Mistakes

| Mistake | Domain Violation | Fix |
|---------|-----------------|-----|
| Using Vec | Heap allocation | heapless::Vec |
| No critical section | Race with ISR | Mutex + interrupt::free |
| Blocking in ISR | Missed interrupts | Defer to main loop |
| Unsafe peripheral | Hardware conflict | HAL ownership |
