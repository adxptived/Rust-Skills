# Writing no_std Rust: Core Systems Architecture

In microcontrollers, bare-metal hardware, and OS kernel development, Rust runs without the standard library.

## 1. Disabling the Standard Library

Remove std linkage by placing `#![no_std]` at the root of `main.rs` or `lib.rs`.

```rust
#![no_std]
#![no_main] // microcontroller setups execute directly without standard main entry points

use core::panic::PanicInfo;

// Define Panic handler since standard abort is missing
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {
        // Halt processor execution
    }
}
```

## 2. Using Core and Alloc Crates

Without `std`, you can still access type boundaries in the `core` library. If you have heap memory configured via an allocator, import collections from `alloc`.

```rust
#![no_std]

extern crate alloc;

use alloc::vec::Vec;
use alloc::string::String;

pub fn process_data() {
    // Dynamic collections allowed once alloc global_allocator is set up
    let mut buffer = Vec::new();
    buffer.push(42);
}
```

## 3. What You Lose Without std

| Feature | `core` alternative | Notes |
|---------|-------------------|-------|
| `Vec`, `String`, `Box` | `alloc::*` | Needs global allocator |
| `HashMap` | `alloc::collections::BTreeMap` | No hash-based map in `alloc` |
| `File`, `TcpStream` | Peripheral/HAL crate | Platform-specific |
| `std::thread` | None / RTOS binding | Cooperative or RTOS tasks |
| `std::sync::Mutex` | `critical_section` | Disable interrupts briefly |
| `Instant`, `Duration` | `core::time::Duration` | No wall clock; use timer peripheral |
| `println!` | `defmt!` / custom UART | defmt preferred for efficiency |
| `Box<dyn Trait>` | Works in `alloc` | Requires `extern crate alloc` |
| `#[global_allocator]` | Same attribute | Must define allocator |

## 4. Global Allocator Setup

```rust
use embedded_alloc::Heap;

#[global_allocator]
static HEAP: Heap = Heap::empty();

fn init_heap() {
    const HEAP_SIZE: usize = 1024 * 4; // 4 KB
    static mut HEAP_MEM: [u8; HEAP_SIZE] = [0; HEAP_SIZE];
    unsafe { HEAP.init(HEAP_MEM.as_mut_ptr() as usize, HEAP_SIZE) }
}

// Call init_heap() before any allocation
```

Without a global allocator, you cannot use `Vec`, `String`, or `Box`. Everything must be fixed-size stack-allocated.

## 5. Microcontroller Memory Layouts

Crates like `cortex-m-rt` map processor memory sections using custom linkers (`memory.x`).

```rust
use cortex_m_rt::entry;

#[entry]
fn main() -> ! {
    // Core boot execution code
    loop {}
}
```

Typical `memory.x` sections:

```linker
MEMORY {
  FLASH : ORIGIN = 0x08000000, LENGTH = 512K
  RAM   : ORIGIN = 0x20000000, LENGTH = 64K
}

_stack_start = ORIGIN(RAM) + LENGTH(RAM);
```

## 6. Critical Sections and Mutexes

```rust
use core::cell::RefCell;
use critical_section::Mutex;

static SHARED: Mutex<RefCell<Option<u32>>> = Mutex::new(RefCell::new(None));

fn write_value(val: u32) {
    critical_section::with(|cs| {
        *SHARED.borrow(cs).borrow_mut() = Some(val);
    });
}

fn read_value() -> Option<u32> {
    critical_section::with(|cs| {
        *SHARED.borrow(cs).borrow()
    })
}
```

## 7. Interrupt Handling

```rust
use cortex_m::interrupt::free as interrupt_free;

// Data shared between main loop and interrupt handler
static INTERRUPT_FLAG: Mutex<RefCell<bool>> = Mutex::new(RefCell::new(false));

fn main() -> ! {
    loop {
        interrupt_free(|cs| {
            if *INTERRUPT_FLAG.borrow(cs).borrow() {
                // Handle interrupt-triggered event
                *INTERRUPT_FLAG.borrow(cs).borrow_mut() = false;
            }
        });
        // WFI (Wait For Interrupt) to save power
        cortex_m::asm::wfi();
    }
}
```

## 8. Hardware Abstraction Layers (HAL Crates)

| Vendor | HAL crate | PAC crate |
|--------|-----------|-----------|
| STM32 | `stm32f4xx-hal`, `stm32g0xx-hal` | `stm32f4`, `stm32g0` (via `stm32-rs`) |
| Nordic | `nrf-hal`, `nrf52840-hal` | `nrf52840-pac` |
| ESP32 | `esp-hal` (esp-rs) | `esp32c3` |
| Raspberry Pi | `rp2040-hal` | `rp2040-pac` |
| AVR | `avr-hal` | `avr-device` |

## 9. Testing no_std Code

```rust
// Use #[cfg(test)] with std for host-side tests
#[cfg(test)]
extern crate std;

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_algorithm() {
        // Here, std is available because we're on the host
        let result = my_calc(5);
        assert_eq!(result, 25);
    }
}
```

For integration tests on real hardware: `probe-rs`, `defmt-test`, and `drogue-device/test`.

## 10. Common no_std Mistakes

- Using `std::` paths out of habit — use `core::` or `alloc::`.
- Forgetting the `#[panic_handler]` — compiler will error with cryptic message.
- Unbounded recursion on stack — no guard pages on embedded, stack overflow = silent corruption.
- Large stack allocations — use `static mut` for big buffers.
- Calling `alloc` without a global allocator — runtime panic.
- Assuming `usize` is 64-bit — on Cortex-M it is 32-bit.
