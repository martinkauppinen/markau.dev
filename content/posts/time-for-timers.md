---
title: "Time for Timers"
date: 2022-05-16T21:25:56+02:00
author: Martin Kauppinen
series: ["Rusty Game Boy Emulator"]
---

Woops, seems like I forgot to write stuff here for three months. I can explain!
Elden Ring came out and stole all my free time, and also I'm getting married
soon so I have to plan my wedding!

Anyways, that's not what we're here for today. Game Boy emulation time!

I decided to take it a bit easy, coming back to this project after three months.
So today we'll implement something relatively simple: the timer.

{{< toc >}}

## Overview
The timer functionality consists of four 8-bit registers mapped to the memory
region 0xFF04-0xFF07, and are summarized in the table below.

|Register|Address|Function                                      |
|--------|-------|----------------------------------------------|
| DIV    | 0xFF04|Incremented at constant frequency of 16384 Hz.|
| TIMA   | 0xFF05|Incremented at frequency configured by TAC.   |
| TMA    | 0xFF06|Modulo. Loaded into TIMA after overflow.      |
| TAC    | 0xFF07|Control register.                             |

### DIV - The divider register
As specified in the table, DIV is simply incremented at a constant frequency of
16384 Hz. As I understand it, other circuitry in the Game Boy triggers on some
rising/falling edges of this register to perform their functions at a set rate.
I have not implemented this yet.

An interesting thing about this register is that it is not only readable, but
also writable. However, writing any value to it will simply reset it to 0. I'm
guessing this is to prevent developers from setting it to arbitrary values
constantly as other subsystems could start misbehaving if they did so. This
forces developers to use the TIMA register instead and forces them to write
proper interrupt handlers.

### TIMA - The timer counter
This is the main register in the timer system. It gets incremented at a rate set
by TAC - the control register - if it is enabled. When the counter overflows
from 0xFF to 0x00, the timer interrupt is generated.

### TMA - The modulo register
When TIMA overflows, the value of this register is loaded back into TIMA. There
are some details and quirks regarding delays and such at overflow. As I
understand it, there is one cycle before TIMA is loaded with TMA where TIMA is
equal to zero. I'm not sure if this will have any noticeable effect on the
emulator when it's in a more finished state. I guess we shall see. Keep that on
the todo list for now. :)

### TAC - The control register
The control register sets up the TIMA register for different modes of operation.
Apart from the obvious enabled/disabled, the TIMA register can be incremented at
4 set frequencies:  4096 Hz, 262144 Hz, 65536 Hz, and 16384 Hz.

Only the three least significant bits are used, as follows:
```
xxxxx000
     |``- Input clock select
     |      00:   4096 Hz (CPU clock / 1024)
     |      01: 262144 Hz (CPU clock /   16)
     |      10:  65536 Hz (CPU clock /   64)
     |      11:  16384 Hz (CPU clock /  256)
     |
     `--- Timer stop
            0: Stop timer
            1: Start timer
```
Setting the timer stop bit to 0 only stops the TIMA counter. The DIV counter
always continues incrementing at 16384 Hz regardless of the value of TAC.

## Rust sketch
Sketching out an implementation of this in Rust, the following struct is very
natural:
```rust
struct TimerRegisters {
    div:  u8,
    tima: u8,
    tma:  u8,
    tac:  u8,
}
```
And to make the timers increment we need a method that, given how many cycles
have passed since the last time the timers were incremented, increments the
timers appropriately. Since an interrupt should be generated when TIMA
overflows, it should return an
[`Interrupt`]({{< ref
"/posts/we-interrupt-this-program-for-interrupts#interrupt-enum" >}}).
However, since TIMA may or may not overflow, we'll return an `Option<Interrupt>`
instead. That way it's the caller's job to handle actually calling the handling
code and the `TimerRegisters` struct can focus on only doing the incrementing.
```rust
impl TimerRegisters {
    // ...
    pub fn tick(&mut self, machine_cycles: usize) -> Option<Interrupt> {
        // ...
    }
    // ...
}
```
I'm choosing to pass in machine cycles rather than CPU cycles, as the `CPU`
struct keeps track of how many machine cycles have passed after executing the
current instruction. Inside the method they're converted to CPU cycles by
multiplying by 4. The code is very simple, so I won't repeat it all here. If
you're interested, the full code for this method can be found [in the git repo](https://github.com/martinkauppinen/gibberish/blob/7dcfb34751b6ded467a1757212a52adee3b79bd0/src/memory/timer.rs#L73-L107).
Again, it should be noted that it does not behave exactly as the Game Boy does
for now. It immediatly loads TIMA with the value in TMA on overflow, never
really letting it go to zero in any way that is visible to users of the struct.

## Memory region trait
Before implementing the `TimerRegisters` struct, I realized that there are a lot
of memory-mapped regions and registers in the Game Boy and I wanted to have a
common interface to them all so that I didn't have to worry about internal
details when coding on the higher level. So for once, I actually defined a
trait! The interface I wanted was very simple: only 4 functions. They allow me
to read and write bytes and words from specific memory addresses. Here's the
full trait:
```rust
pub trait MemoryRegion {
    fn read_byte(&self, addr: u16) -> u8;
    fn write_byte(&mut self, byte: u8, addr: u16);
    fn read_word(&self, addr: u16) -> u16;
    fn write_word(&mut self, word: u16, addr: u16);
}
```

I have already used functions with the same names and signatures in several
places in the code base, so it made sense to abstract them out into a trait. And
when I get around to refactoring everything to use this trait, I will have
shorter and neater `impl`-blocks that only concern themselves with internal
logic. Nice.

Implementing this trait is really simple as well. The word variants simply calls
the byte variants twice. And within the read/write functions I have checks that
make sure that the address that was passed in matches the addresses the trait
implementor is mapped to. Currently I've made the emulator panic if it is passed
an incorrect address. I might revisit that in the future.

## Assembly example
After writing the simpler unit tests, I wanted to make sure that this worked for
real. So of course I wrote [a short assembly program](https://github.com/martinkauppinen/gibberish/blob/7dcfb34751b6ded467a1757212a52adee3b79bd0/test-roms/timer.asm)
that can be loaded into the emulator as it is right now. It's exceedingly
simple: the code sets up and enables timer interrupts and the TIMA counter. Then
it just waits in a busy loop until the B register is non-zero. The B register is
only ever set to anything non-zero in the interrupt handler defined for the
timer interrupt. Once the timer interrupt fires and the interrupt handler sets B
and returns, the main program will break out of the loop and then stop the CPU.

The program can be run on the current commit
(7dcfb34751b6ded467a1757212a52adee3b79bd0) by building `test-roms/timer.gb` and
passing it as the argument to the main program. This requires
[rgbds](https://github.com/gbdev/rgbds) to compile everything.

Here's the list of commands to do this:
```
$ git clone https://github.com/martinkauppinen/gibberish
$ cd gibberish/
$ git checkout 7dcfb34751b6ded467a1757212a52adee3b79bd0
$ make -C test-roms
$ cargo run -- test-roms/timer.gb
```

This will print out a bunch of information about the registers and the current
instruction being executed. The exact details of what is printed is not too
interesting, but what is interesting is that it doesn't crash, and moreover, it
terminates! This means the timer is counting up properly and the interrupt fired
on overflow, breaking out of the loop!

We can also confirm that the frequency selection is working by editing
`test-roms/timer.asm`. For example, when we run the program as it is now, we get
this many lines of output:

```
$ cargo run -- test-roms/timer.gb | wc -l
425
```

This is because TIMA is set to its fastest increment frequency. However, if we
set it to its slowest frequency, 4096 Hz, by editing line 19 to load the A
register with 4 instead of 5 and recompiling, we get:

```
$ make -C test-roms/
$ cargo run -- test-roms/timer.gb | wc -l
26229
```

Quite a bit more. Since it takes a lot more cycles before overflow, and the
emulator prints every instruction.

## Future work
We'll see if it's necessary, but in the future I might implement the weird
inbetween cycles between overflow and TMA loading.

Also, as it stands now this code is completely synchronous. The timers are
incremented after the current machine instruction has executed. This may pose
some problem in the future and the timers might have to move into their own
thread. For now it works, though. And it's good to have the simple version
done and ready to be refactored if and when it's necessary.

## Repo
As always, the code can be found [in the git
repo](https://github.com/martinkauppinen/gibberish).
