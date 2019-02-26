---
title: Life Before Main
theme: black
---

## Life Before Main

James Munns

@bitshiftmask

james.munns@ferrous-systems.com

2019-02-26

::: notes

:::

---

## Step One: Apply Power

---

## Step Two: ???

---

## Step Three: Run Code (profit!)

---

## Wait, What?

---

## Your Code

---

```rust
#[entry]
fn main() -> ! {
    let mut dwm1001 = DWM1001::take().unwrap();
    let mut timer = dwm1001.TIMER0.constrain();

    loop {
        dwm1001.leds.D12.enable();
        delay(&mut timer, 20_000);  // 20ms
        dwm1001.leds.D12.disable();
        delay(&mut timer, 230_000); // 230ms
    }
}
```

---

## But Also

---

```rust
#![no_std]

use dwm1001;
use cortex_m_rt::entry;
use panic_halt;

use dwm1001::{
    nrf52832_hal::{prelude::*, timer::Timer},
    DWM1001,
};
```

---

## What Makes This Happen?

---

## Code

> Mostly other people's code

---

## The Compiler

(`rustc` and `llvm`)

> Less than you might think

---

## The Linker

(`llvm-ld`)

> More than you might think

---

## The Hardware

> No thinking, just execution

---

## The Code (again)

> This time, as machine code

---

## What is a Runtime?

---

## Someone Else's code

> `cortex-m-rt` + `r0`

---

```rust
#[no_mangle]
pub unsafe extern "C" fn Reset() -> ! {
    // User pre-init hook
    __pre_init();
    // Initialize RAM
    r0::zero_bss();
    r0::init_data();
    #[cfg(has_fpu)]
    enable_fpu();

    main()
}
```

---

## The Reset Vector

---

## The Vector Table

---

## Code Sections

* `.text` - code and read only data
* `.bss` - RAM with uninitialized/Zeroed values
* `.data` - RAM with initial values

---

## But How Does It Know?

---

## The Compiler Knows

> Well, it knows how to categorize

---

## How does the Linker know?

---

## We tell it.

---

## The Linker Script

* `link.x.in` - The base template
* `device.x` - The interrupt table (generated)
* `memory.x` - How much RAM? How Much Flash?
* `link.x` - The final linker script

---

## `device.x`

```text
PROVIDE(POWER_CLOCK = DefaultHandler);
PROVIDE(RADIO = DefaultHandler);
// ...
```

---

## `memory.x`

```text
MEMORY
{
  /* NOTE K = KiBi = 1024 bytes */
  FLASH : ORIGIN = 0x00000000, LENGTH = 512K
  RAM   : ORIGIN = 0x20000000, LENGTH = 64K
}
```

---

## `link.x.in`

```text
SECTIONS
{
    .vector_table ORIGIN(FLASH) :
    {
        // ...
    } > FLASH

    .text _stext :
    {
        // ...
    } > FLASH

    .data : ALIGN(4)
    {
        // ...
    } > RAM AT > FLASH

    .bss : ALIGN(4)
    {
        // ...
    } > RAM
}
```

---

## An ELF is born

---

## Hardware

> Once you put things in the right place, the magic starts

---

## Launch Sequence

* Code exists in Flash
    * Includes the Vector Table
    * Reset Vector is Called
* cortex-m-rt runs Reset
* RAM is set up
* Your main is run!

---

## Let there be (blinky) light!

---

## Thank you!

---

## Plugs

![](./assets/plug.svg)

<font size="1">icon by [Freepik] from [flaticon.com]</font>

[Freepik]: https://www.flaticon.com/authors/freepik
[flaticon.com]: https://www.flaticon.com

---

github.com/rust-embedded/wg

![](./assets/ewg-logo.png)

---

## Ferrous Systems

![](./assets/ferrous.png)

ferrous-systems.com

---

> Life Before Main

James Munns

@bitshiftmask

james.munns@ferrous-systems.com

https://github.com/ferrous-systems/life-before-main


---

## Bonus Slides

> YAGNI

---


```rust
// In `fn Reset() -> !`
extern "C" {
    // These symbols come from `link.x`
    static mut __sbss: u32;
    static mut __ebss: u32;
    static mut __sdata: u32;
    static mut __edata: u32;
    static __sidata: u32;
}
```

---

```rust
// In `fn Reset() -> !`
extern "Rust" {
    // This symbol will be provided by the user
    // via `#[entry]`
    fn main() -> !;

    // This symbol will be provided by the user
    // via `#[pre_init]`
    fn __pre_init();
}
```

---

```rust
// In `fn Reset() -> !`
// User hook for pre-initialization
__pre_init();

// Initialize RAM
r0::zero_bss(&mut __sbss, &mut __ebss);
r0::init_data(
    &mut __sdata,
    &mut __edata,
    &__sidata
);
```

---

```rust
pub unsafe fn zero_bss<T>(mut sbss: *mut T, ebss: *mut T)
where
    T: Copy,
{
    while sbss < ebss {
        // NOTE(volatile) to prevent this from being transformed into `memclr`
        ptr::write_volatile(sbss, mem::zeroed());
        sbss = sbss.offset(1);
    }
}
```

---

```rust
pub unsafe fn init_data<T>(
    mut sdata: *mut T,
    edata: *mut T,
    mut sidata: *const T,
) where
    T: Copy,
{
    while sdata < edata {
        ptr::write(sdata, ptr::read(sidata));
        sdata = sdata.offset(1);
        sidata = sidata.offset(1);
    }
}
```

---
```rust
// In `fn Reset() -> !`

// enable the FPU
#[cfg(has_fpu)]
core::ptr::write_volatile(
    SCB_CPACR,
    *SCB_CPACR
        | SCB_CPACR_FPU_ENABLE
        | SCB_CPACR_FPU_USER,
);

main()
```
