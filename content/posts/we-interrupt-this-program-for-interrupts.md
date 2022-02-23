---
title: "We Interrupt This Program for... Interrupts!"
date: 2022-02-22T18:42:25+01:00
author: Martin Kauppinen
series: ["Rusty Game Boy Emulator"]
---

Taking a break from all the opcodes, I wanted to turn my attention to interrupt
handling. When certain hardware events happen on the Game Boy, the CPU should
stop executing the current program and handle whatever event occurred. This is
done with interrupts, which I have modelled into the CPU, but they're not
actually fully implemented yet, since I have not implemented the emulations of
the hardware systems that generate the interrupts. One step at a time! This post
will serve as a reference for when I do get around to implementing them.

## Interrupt registers and flags
There are two relevant memory mapped registers and one flag when it comes to
interrupts. The IME (interrupt master enable) is the flag controlling whether or
not interrupts will be processed by the CPU. It is not memory mapped and can
only be set by the `EI` and `RETI` instructions and only reset by the `DI`
instruction.

Next we have the IE (interrupt enable) register, memory mapped at 0xFFFF, which
controls what interrupts the CPU will acknowledge when IME is set.

Finally, we have the IF (interrupt flags) memory mapped at 0xFF0F. The lower 5
bits are set whenever one of the five interrupting subsystem requests an
interrupt from the CPU. Or whenever the programmer of a game writes to the
memory address. Either way, an interrupt will be handled if the corresponding
bit in IE is also set.

## Available interrupts
The five different interrupts available on the Game Boy are as follows:

### Vertical Blanking
The Game Boy display has 144 scanlines, and once the LCD controller has finished
all of them, it enters the V-Blank state, and sends this interrupt to the CPU.
The interrupt means that the display has finished drawing the current frame and
so writing to video memory can be freely done, meaning it's a good time to
update graphics and sprites.

The V-Blank period lasts for 1140 machine cycles, or roughly 1.1 milliseconds,
and is triggered at 59.7 Hz from the info I've managed to find. This is also the
top priority interrupt.

### LCD STAT
This interrupt is triggered when some configured LCD events happen.

-   If the current scanline to be drawn is equal to the one in the `LYC` register
    (memory mapped at 0xFF45), and bit 6 of the `STAT` register (memory mapped at
    0xFF41) is set.

-   __OR__ the LCD Screen mode is 0 (H-Blank) and bit 3 in the `STAT` register is
    set.

-   __OR__ the LCD Screen mode is 1 (V-Blank) and bits 4 or 5 in the `STAT`
    register are set.

-   __OR__ the LCD Screen mode is 2 (Searching OAM-RAM) and bit 5 in the `STAT`
    register is set.

### Timer
This interrupt is triggered when the TIMA timer register (memory mapped at
0xFF05) overflows from 0xFF to 0x00, before resetting to the value of the TMA
timer register (memory mapped at 0xFF05).

### Serial
When transferring data over serial cable, this interrupt is triggered after
sending 8 bits.

### Joypad
The P1 register (0xFF00) holds the state of the buttons of the Game Boy in the
lower nybble. Since there are 8 buttons but only 4 bits are used, there is some
multiplexing going on when reading this register. In any case, the joypad
interrupt is triggered when at least one bit in the lower nybble of P1 goes from
high to low.

## Interrupt sequence
Before fetching an instruction, the CPU checks for interrupts. If IME is set and
at least one IF bit is set _and_ the corresponding bit in IE is set, the
interrupt handling starts. When handling an interrupt the current program
counter is pushed to the stack, set to the interrupt service routine for the
specific interrupt that happened , and the IME flag is reset so that interrupts
don't occur while processing the interrupt [^nested].

[^nested]: Technically it's possible to set IME with `EI` during an interrupt
  handler, but we don't have to think too much about that in the emulator. We'll
  simply run the instructions as given and whatever happens, happens.

The five different interrupt service routines are at 0x0040, 0x0048, 0x0050,
0x0058, and 0x0060. If multiple interrupt bits are set in IF at the same time,
only the one with the lowest address of its interrupt service handler will be
taken. The priority order is the same as the order I listed the interrupts in
the previous section.

Modelling interrupts in Rust, I decided to base everything on an `enum` to have
semantic names for the interrupts:

```rust
enum Interrupt {
    Vblank,
    Lcdc,
    Timer,
    Serial,
    Joypad,
}
```

This could all just be modeled with `u8`s and bitmasks, but I felt like going
the extra mile with real names and types would help when coding things up.
Speaking of masks, the interrupt masks are modeled as a simple `struct` of
`bools`, mirroring the `enum`:

```rust
struct InterruptMask {
    vblank: bool,
    lcdc: bool,
    timer: bool,
    serial: bool,
    joypad: bool,
}
```

For both of these types, I implemented the `From<u8>` trait and the opposite
conversion trait. I'm going a bit overkill here, but that's because I had an
idea for how I wanted to implement the whole "only process the interrupt with
the highest priority" bit which would work if I modeled it with `u8`s, but I
still wanted the semantic types, so I needed the conversion routines.

Anyway, finally the IF and IE registers were modelled in a simple controller
struct as follows:

```rust
struct InterruptController {
    request: InterruptMask,
    enabled: InterruptMask,
}
```

This was simply put in the `Cpu` struct along with a `bool` for the IME flag.

Now for the idea I had for how to implement this if it were just modelled with
`u8`s. Since the lowest set bit is the one with the highest priority, we can
count how many trailing zeroes there are in the binary representation of the IF
register. The integer primitives in Rust have a function called
[`trailing_zeros`](https://doc.rust-lang.org/std/primitive.u8.html#method.trailing_zeros)
in their implementations which does exactly this. If there are 5 or more
trailing zeros, none of the interrupt request bits in IF were set, so we have
not received an interrupt request. However, if there are 4 or fewer trailing
zeroes, at least one interrupt request has been set. Then we simply loop from
the trailing zeros up to 4 and check whether the corresponding bit is set in the
IE register, returning the first one we find. It looks like this:

```rust
pub fn get_pending_interrupt(&mut self) -> Option<Interrupt> {
    let bits: u8 = self.request.into();

    // None of the 5 interrupts set
    if bits.trailing_zeros() > 4 {
        return None;
    }

    let mut highest_prio: Option<u8> = None;

    // Loop through bits from least to most significant
    for i in bits.trailing_zeros()..=4 {
        let interrupt_bit = 1 << i;

        if u8::from(self.enabled) & interrupt_bit > 0 {
            // Found a match
            highest_prio = Some(interrupt_bit);
            break;
        }
    }

    // Check that an enabled interrupt was found, otherwise return None
    highest_prio?;

    // Reset interrupt flag
    self.request = (u8::from(self.request) & !highest_prio.unwrap()).into();

    let interrupt = highest_prio.unwrap().into();
    Some(interrupt)
}
```

Could this simply have been implemented with a bunch of `if` statements? Yes. Is
it? Also kind of, yes (they're hidden in the `impl From` blocks), but I wanted
to see if this idea worked, and it did! If it turns out that this is a slow/bad
way to do it I'll change it, but projects like this are for having fun anyway.
The code _looks_ pretty short and neat at least.

Now for implementing the interrupt handling in the `Cpu` struct. Simply adding a
line to the `step` function to call the interrupt handling:

```diff
pub fn step(&mut self) {
+   self.handle_interrupts();
    self.current_instruction = self.read_byte(self.registers.pc);
    // ...
}
```

And implementing said function, calling the `get_pending_interrupt` implemented
above:

```rust
fn handle_interrupts(&mut self) {
    if !self.interrupt_master_enable {
        return;
    }

    if let Some(interrupt) = self.interrupts.get_pending_interrupt() {
        self.push(self.registers.pc);
        self.interrupt_master_enable = false;
        match interrupt {
            Interrupt::Vblank => self.registers.pc = 0x0040,
            Interrupt::Lcdc => self.registers.pc = 0x0048,
            Interrupt::Timer => self.registers.pc = 0x0050,
            Interrupt::Serial => self.registers.pc = 0x0058,
            Interrupt::Joypad => self.registers.pc = 0x0060,
        }

        // Dispatching takes 5 machine cycles
        self.machine_cycles = 5;
    }
}
```
And we're done with interrupt implementation! Well, at least handling the
interrupts being generated and jumping to their service routines. None of the
subsystems that can generate interrupts have actually been implemented yet. But
we'll get there! The groundwork has been laid, so it should be pretty simple to
generate interrupts when the time comes.

Speaking of simple, the `EI` and `DI` instructions couldn't be easier:

```rust
pub fn ei(cpu: &mut crate::cpu::Cpu) {
    cpu.interrupt_master_enable = true;
}

pub fn di(cpu: &mut crate::cpu::Cpu) {
    cpu.interrupt_master_enable = false;
}
```

## STOP, HALT!
Worth mentioning in the same post, is that there are two instructions which put
the CPU into a state where it waits, for interrupts or resets: `STOP` and
`HALT`. They work slightly differently, despite their similar names.

`HALT` is the lesser instruction, which I find backwards due to its name. When
the instruction is executed, the system is put into a state where the system
clock is stopped, but the oscillator and LCD controller continue to run
normally. It was recommended to use this state as often as possible to save on
battery power. This mode is cancelled when receiving an interrupt request and
the corresponding flag in IE is set, regardless of the state of IME.

`STOP` stops all functionality. System clock, oscillator, LCD controller, all of
it. Stopped. The only thing that will take the Game Boy out of this mode is a
reset.

I have modelled these states, but not yet implemented them fully in the
emulator.

## The HALT Bug
The above description of `HALT` was a simplification. There are three different
possible outcomes of running the `HALT` instruction:

1.  When IME is set, `HALT` works normally. The CPU stops until there is a
    requested interrupt in IF with the corresponding bit set in IE, calling the
    proper service routine when this happens. Everybody is happy.

2.  When IME is cleared, and there are no correspondences between IF and IE,
    `HALT` makes the CPU go into halt mode until an interrupt is requested in
    IF, regardless of if the corresponding flag is set in IE.

3.  Oh boy, here we go. If IME is cleared and there _are_ correspondences
    between IF and IE, the `HALT` bug occurs. Halt mode is not entered in this
    case, and when the next instruction is executed, the CPU fails to increment
    the program counter, leading to the instruction immediately following `HALT`
    to be executed twice.

The workaround for this was to _always_ remember to put a `NOP` after every
`HALT` instruction, so that the worst thing that could happen was wasting one
machine cycle. But I wonder how many developer hours went into fixing this bug
in the making of games for the Game Boy back in the day (and even now in the
retro scene). It's a very interesting bug, and I think I am going to put it in
the emulator, despite the fact that I'm pretty sure I don't have to. Simply
because the bug is so interesting to me. Very simple and subtle, but with
potentially really bad consequences. Imagine that the very next instruction is
an innocent increment of a register used for indexing into memory. Suddenly you
potentially have buffer overflow issues! Or what if a `HALT` instruction is
followed by another `HALT` instruction?

Here's one fun example I found in [The Cycle-Accurate Game Boy
Docs](https://github.com/AntonioND/giibiiadvance/blob/master/docs/TCAGBD.pdf)
with instructions requiring more than one byte:
```asm
halt
ld  a,$14   ; $3E $14, but executed as $3E $3E $14
ld  [hl+],a
```
Since the program counter doesn't increment properly, the second line is
actually executed as the following:
```asm
ld  a,$3E
inc d
```
So the A register has a completely different value than what we expect, and
nothing is visibly wrong with the code! You would have to know about this bug to
even be able to begin debugging it. Which of course is why the practice of `NOP`
after `HALT` was mandated.

