# Embedded HAL & Driver Specifications

Rust uses trait-based contracts (`embedded-hal`) to build hardware-agnostic drivers that function across different MCU architectures.

## 1. Hardware Abstraction Layers (HAL)

Trait bindings allow writing driver libraries that compile for STM32, ESP32, and Arduino alike.

```rust
use embedded_hal::digital::OutputPin;

pub struct Led<PIN> {
    pin: PIN,
}

impl<PIN> Led<PIN>
where
    PIN: OutputPin,
{
    pub fn new(pin: PIN) -> Self {
        Self { pin }
    }

    pub fn turn_on(&mut self) -> Result<(), PIN::Error> {
        self.pin.set_high()
    }

    pub fn turn_off(&mut self) -> Result<(), PIN::Error> {
        self.pin.set_low()
    }
}
```

## 2. Register-Level Safe Modifications

When writing low-level drivers, use Peripheral Access Crates (PAC) created via `svd2rust` to safely write values to hardware memory registers.

```rust
// Safely manipulate CPU hardware configurations without manual pointer casting
device.RCC.apb2enr.modify(|_, w| w.iopaen().set_bit());
device.GPIOA.odr.write(|w| w.odr5().set_bit());
```

## 3. embedded-hal Traits Overview

| Trait | Category | Methods | Common Implementors |
|-------|----------|---------|-------------------|
| `OutputPin` | Digital output | `set_high`, `set_low`, `toggle` | GPIO pins, port expanders |
| `InputPin` | Digital input | `is_high`, `is_low` | Buttons, sensors, switches |
| `SpiDevice` | Bus | `read`, `write`, `transfer` | SPI sensors, displays, SD cards |
| `I2cDevice` | Bus | `read`, `write`, `write_read` | I2C sensors, EEPROM, RTC |
| `DelayUs` | Timing | `delay_us` | Blocking delay |
| `PwmPin` | Analog output | `set_duty`, `get_period` | LEDs, motors, servos |
| `CountDown` | Timer | `start`, `wait` | Blink, timeouts |
| `Serial` (`Write`/`ReadType`) | UART | `write`, `read`, `flush` | GPS, LoRa, debugging |

## 4. Driver Design Patterns

```rust
// Type-state pattern: sensor configuration at compile time
pub struct Sensor<I2C, MODE> {
    i2c: I2C,
    addr: u8,
    _mode: PhantomData<MODE>,
}

pub struct Configuring;
pub struct Running;

impl<I2C, MODE> Sensor<I2C, MODE>
where
    I2C: I2cDevice,
{
    pub fn new(i2c: I2C, addr: u8) -> Sensor<I2C, Configuring> {
        Sensor { i2c, addr, _mode: PhantomData }
    }
}

impl<I2C> Sensor<I2C, Configuring>
where
    I2C: I2cDevice,
{
    pub fn init(mut self) -> Result<Sensor<I2C, Running>, I2C::Error> {
        self.i2c.write(self.addr, &[0x00, 0x01])?; // config register
        Ok(Sensor { i2c: self.i2c, addr: self.addr, _mode: PhantomData })
    }
}

impl<I2C> Sensor<I2C, Running>
where
    I2C: I2cDevice,
{
    pub fn read_value(&mut self) -> Result<u16, I2C::Error> {
        let mut buf = [0u8; 2];
        self.i2c.read(self.addr, &mut buf)?;
        Ok(u16::from_be_bytes(buf))
    }
}
```

## 5. DMA and Interrupt Patterns

```rust
// DMA-based peripheral transfer (pseudo-code)
let dma = DMA::new(peripherals.DMA);
let mut spi = Spi::new(peripherals.SPI1, dma::Channel::Ch1);

// Transfer completes in background, CPU is free
spi.transfer_dma(&tx_buffer, &mut rx_buffer);

// Poll or wait for interrupt
while !dma.is_complete(dma::Channel::Ch1) {
    // execute other tasks or WFI
}
```

## 6. RTIC for Real-Time Systems

```rust
#[rtic::app(device = stm32f4xx_hal::stm32, peripherals = true)]
mod app {
    #[shared]
    struct Shared {
        led: Led<PB0<Output<PushPull>>>,
    }

    #[local]
    struct Local {
        timer: Timer<TIM2>,
    }

    #[init]
    fn init(cx: init::Context) -> (Shared, Local) {
        let dp = cx.device;
        let timer = Timer::new(dp.TIM2, 1.hz(), &mut dp.RCC);
        timer.enable_interrupt();
        (Shared { led: /* ... */ }, Local { timer })
    }

    #[task(binds = TIM2, shared = [led])]
    fn blink(mut cx: blink::Context) {
        cx.shared.led.lock(|led| led.toggle().ok());
    }
}
```

## 7. Memory Layout for Cortex-M

```linker
/* memory.x — chip-specific */
MEMORY
{
  FLASH : ORIGIN = 0x08000000, LENGTH = 512K
  RAM   : ORIGIN = 0x20000000, LENGTH = 128K
}
```

```rust
// Stack and heap placement
use core::mem::MaybeUninit;

// Place large buffers in specific RAM regions
#[link_section = ".ram2"]
static mut AUDIO_BUFFER: MaybeUninit<[u8; 8192]> = MaybeUninit::uninit();
```

## 8. Debugging Embedded Rust

- `probe-rs` / `probe-run`: flash and debug via SWD/JTAG.
- `defmt`: efficient logging — messages are format strings at compile time, binary payload over wire.
- `itmin` / `semihosting`: print to host console without UART.
- `panic-probe`: panic handler that also defmt-logs the call site.

```rust
// RTT (Real-Time Transfer) with defmt
#[entry]
fn main() -> ! {
    defmt::info!("System booted at {} MHz", CLOCK_FREQ);
    loop { /* ... */ }
}
```

## 9. Common Driver Test Harnesses

```rust
// Test with mock HAL in host tests
use embedded_hal_mock::eh1::digital::{
    Mock as PinMock,
    Transaction as PinTransaction,
};

#[test]
fn led_toggles() {
    let expectations = [
        PinTransaction::set_high(Mock::Generic(Err(())).into()),
        PinTransaction::set_low(Mock::Generic(Err(())).into()),
    ];
    let mut pin = PinMock::new(&expectations);
    let mut led = Led::new(pin);
    let _ = led.turn_on();
    let _ = led.turn_off();
}
```
