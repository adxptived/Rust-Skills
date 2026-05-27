---
name: rust-embedded
description: |
  Embedded Rust — no_std, bare-metal, microcontrollers. Use this skill when:
  - User writes #![no_std] or #![no_main] code
  - User targets ARM Cortex-M, RISC-V, ESP32, STM32, nRF, RP2040, or similar MCUs
  - User works with embedded-hal, RTIC, Embassy, or cortex-m crates
  - User asks about GPIO, SPI, I2C, UART, DMA, timers, or interrupts
  - User asks about linker scripts, memory layout, or .cargo/config.toml targets
  - User deals with volatile memory, register-level access, or PAC/HAL
  - User works with Embassy async executor on embedded targets
  - User asks about no_std alternatives for alloc, formatting, or collections
  Trigger for any embedded/no_std Rust question.
---



## Quick Navigation

- [references/no_std.md](references/no_std.md)
- [references/drivers_hal.md](references/drivers_hal.md)

# Embedded Rust

> On embedded targets, safety guarantees matter even more — no OS to catch your mistakes.

## The Embedded Rust Stack

```
Application code
        │
   HAL (Hardware Abstraction Layer)
        │    e.g. stm32f4xx-hal, nrf52840-hal, esp-hal
   PAC (Peripheral Access Crate)
        │    e.g. stm32f4, nrf52840
   cortex-m / riscv crates (CPU support)
        │
   Hardware
```

## Minimal no_std Binary

```rust
#![no_std]
#![no_main]

use cortex_m_rt::entry;
use panic_halt as _; // Defines panic handler

#[entry]
fn main() -> ! {
    // Infinite loop — embedded programs don't return
    loop {}
}
```

```toml
# Cargo.toml
[package]
name = "my-firmware"
edition = "2021"

[dependencies]
cortex-m = "0.7"
cortex-m-rt = "0.7"
panic-halt = "0.2"
# HAL for your specific chip:
stm32f4xx-hal = { version = "0.21", features = ["stm32f411"] }

[profile.release]
opt-level = "s"   # Optimize for size
lto = true
codegen-units = 1
debug = true      # Keep symbols for debugging
```

```toml
# .cargo/config.toml
[build]
target = "thumbv7em-none-eabihf"  # Cortex-M4F

[target.thumbv7em-none-eabihf]
rustflags = ["-C", "link-arg=-Tlink.x"]
runner = "probe-rs run --chip STM32F411CEUx"
```

## GPIO Example (embedded-hal)

```rust
#![no_std]
#![no_main]

use cortex_m_rt::entry;
use panic_halt as _;
use stm32f4xx_hal::{pac, prelude::*};

#[entry]
fn main() -> ! {
    let dp = pac::Peripherals::take().unwrap();

    // Configure clocks
    let rcc = dp.RCC.constrain();
    let clocks = rcc.cfgr.freeze();

    // Configure GPIO
    let gpioc = dp.GPIOC.split();
    let mut led = gpioc.pc13.into_push_pull_output();
    let button = gpioa.pa0.into_pull_down_input();

    loop {
        if button.is_high() {
            led.set_high();
        } else {
            led.set_low();
        }
    }
}
```

## Embassy: Async Embedded

Embassy is the modern async framework for embedded Rust. No RTOS needed.

```toml
[dependencies]
embassy-executor = { version = "0.7", features = ["arch-cortex-m", "executor-thread"] }
embassy-stm32 = { version = "0.2", features = ["stm32f411ce", "time-driver-tim2"] }
embassy-time = "0.4"
```

```rust
#![no_std]
#![no_main]

use embassy_executor::Spawner;
use embassy_stm32::{gpio::{Level, Output, Speed}, time::Hertz};
use embassy_time::{Duration, Timer};
use panic_probe as _;

#[embassy_executor::main]
async fn main(_spawner: Spawner) {
    let p = embassy_stm32::init(Default::default());
    let mut led = Output::new(p.PC13, Level::High, Speed::Low);

    loop {
        led.set_high();
        Timer::after(Duration::from_millis(500)).await;
        led.set_low();
        Timer::after(Duration::from_millis(500)).await;
    }
}
```

### Embassy Multi-Task

```rust
#[embassy_executor::main]
async fn main(spawner: Spawner) {
    let p = embassy_stm32::init(Default::default());

    spawner.spawn(blink_task(p.PC13)).unwrap();
    spawner.spawn(uart_task(p.USART1, p.PA9, p.PA10)).unwrap();

    // main can do work too or just wait
    loop {
        Timer::after(Duration::from_secs(60)).await;
    }
}

#[embassy_executor::task]
async fn blink_task(pin: embassy_stm32::gpio::AnyPin) {
    let mut led = Output::new(pin, Level::High, Speed::Low);
    loop {
        led.toggle();
        Timer::after(Duration::from_millis(500)).await;
    }
}
```

## RTIC: Real-Time Interrupt-Driven Concurrency

```rust
#[rtic::app(device = stm32f4xx_hal::pac, peripherals = true)]
mod app {
    use stm32f4xx_hal::{gpio::*, prelude::*};

    #[shared]
    struct Shared {
        led: PC13<Output<PushPull>>,
    }

    #[local]
    struct Local {
        button: PA0<Input<PullDown>>,
    }

    #[init]
    fn init(cx: init::Context) -> (Shared, Local) {
        let dp = cx.device;
        let gpioc = dp.GPIOC.split();
        let led = gpioc.pc13.into_push_pull_output();
        let button = dp.GPIOA.split().pa0.into_pull_down_input();

        (Shared { led }, Local { button })
    }

    #[task(binds = EXTI0, local = [button], shared = [led])]
    fn button_pressed(cx: button_pressed::Context) {
        cx.shared.led.lock(|led| led.toggle());
    }
}
```

## no_std Alternatives

Common `std` types have `no_std` alternatives:

| Need | std | no_std alternative |
|------|-----|--------------------|
| Dynamic allocation | `Vec`, `String`, `Box` | `alloc` crate (requires allocator) |
| Fixed-size buffer | — | `heapless::Vec`, `heapless::String` |
| HashMap | `HashMap` | `heapless::FnvIndexMap`, `hashbrown::HashMap` |
| String formatting | `format!` | `ufmt`, `write!` to a buffer |
| Logging | `log` + env-logger | `defmt` (deferred formatting over RTT) |
| Random | `rand` | `rand_core` + hardware RNG |
| Float math | `std::f32` | `libm` crate |

```toml
[dependencies]
heapless = "0.8"          # Fixed-capacity data structures
defmt = "0.3"             # Efficient logging for embedded
defmt-rtt = "0.4"         # Transport over RTT
```

```rust
use heapless::Vec;
use heapless::String;

// Fixed-capacity Vec — no heap allocation
let mut buf: Vec<u8, 256> = Vec::new(); // Max 256 elements
buf.push(42).unwrap(); // Returns Err if full

// Fixed-capacity String
let mut s: String<64> = String::new();
s.push_str("hello").unwrap();
```

## defmt Logging

The embedded logging standard — serializes format strings on-device, decodes on host:

```rust
use defmt::{debug, error, info, warn};

info!("Initializing...");
debug!("Counter value: {}", count);
warn!("Low battery: {}%", battery_level);
error!("I2C error: {:?}", err);
```

```toml
# .cargo/config.toml — add defmt runner
[target.thumbv7em-none-eabihf]
runner = "probe-rs run --chip STM32F411CEUx"
```

## Volatile / Register Access

Use the PAC for register access — never hand-roll `*ptr = val`:

```rust
use stm32f4::stm32f411::Peripherals;

let dp = Peripherals::take().unwrap();

// Safe, typed register access via PAC
dp.RCC.ahb1enr.modify(|_, w| w.gpiocen().set_bit());

// Never do this (even if you know the address):
// *(0x40023830 as *mut u32) |= 1 << 2; // Wrong! Use PAC.
```

For truly custom register access:

```rust
use core::ptr::{read_volatile, write_volatile};

const MY_REGISTER: *mut u32 = 0x4002_0000 as *mut u32;

unsafe {
    // SAFETY: Register address is valid for this chip, single-core, no aliasing
    write_volatile(MY_REGISTER, 0b1100);
    let value = read_volatile(MY_REGISTER);
}
```

## Memory Layout

```
# memory.x (linker script for stm32f411)
MEMORY
{
  FLASH : ORIGIN = 0x08000000, LENGTH = 512K
  RAM   : ORIGIN = 0x20000000, LENGTH = 128K
}
```

```rust
// Placing data in specific sections
#[link_section = ".ccmram"]
static mut DMA_BUFFER: [u8; 1024] = [0; 1024];

// Zero-cost static allocation
static STATE: cortex_m::interrupt::Mutex<
    core::cell::RefCell<Option<State>>
> = cortex_m::interrupt::Mutex::new(core::cell::RefCell::new(None));
```

## Interrupt Safety Patterns

```rust
use cortex_m::interrupt::Mutex;
use core::cell::RefCell;

// Shared data between main and ISR
static SHARED: Mutex<RefCell<u32>> = Mutex::new(RefCell::new(0));

fn main() -> ! {
    cortex_m::interrupt::free(|cs| {
        *SHARED.borrow(cs).borrow_mut() = 42;
    });
    loop {}
}

#[interrupt]
fn TIM2() {
    cortex_m::interrupt::free(|cs| {
        let val = *SHARED.borrow(cs).borrow();
        // use val
    });
}
```

## Common Targets

| Target triple | CPU | Used for |
|---------------|-----|---------|
| `thumbv6m-none-eabi` | Cortex-M0/M0+ | RP2040, nRF51 |
| `thumbv7m-none-eabi` | Cortex-M3 | STM32F1xx |
| `thumbv7em-none-eabihf` | Cortex-M4F/M7F | STM32F4xx, nRF52 |
| `riscv32imac-unknown-none-elf` | RISC-V 32 | ESP32-C3, GD32VF103 |
| `xtensa-esp32-none-elf` | Xtensa LX6 | ESP32 (needs esp toolchain) |

```bash
# Install targets
rustup target add thumbv7em-none-eabihf
rustup target add riscv32imac-unknown-none-elf

# Build for target
cargo build --target thumbv7em-none-eabihf --release

# Flash with probe-rs
probe-rs run --chip STM32F411CEUx target/thumbv7em-none-eabihf/release/firmware
```

## Anti-Patterns

```rust
// Bad: blocking forever in an interrupt handler.
#[interrupt]
fn USART1() {
    while !tx_ready() {}
    send_byte(0x42);
}

// Good: keep ISRs short; signal work to the main loop or executor.
#[interrupt]
fn USART1() {
    EVENTS.signal(Event::UartReady);
}
```

```rust
// Bad: heap-dependent design in constrained no_std firmware.
let mut log = String::new();

// Good: fixed-capacity buffers make failure explicit.
let mut log: heapless::String<128> = heapless::String::new();
```

## Bring-Up Checklist

- Confirm target triple, linker script, memory layout, and probe configuration.
- Start with GPIO blink before enabling clocks, DMA, networking, or RTOS features.
- Keep interrupt handlers bounded; defer work to tasks or main loop.
- Use `defmt`/RTT logging in debug builds and size-check release builds.
- Model peripheral ownership so two drivers cannot own the same register block.
- Test HAL-independent logic on host with normal unit tests.

## References

- [The Embedded Rust Book](https://docs.rust-embedded.org/book/)
- [Embassy docs](https://embassy.dev/)
- [RTIC book](https://rtic.rs/)
- [awesome-embedded-rust](https://github.com/rust-embedded/awesome-embedded-rust)
- [probe-rs](https://probe.rs/)
