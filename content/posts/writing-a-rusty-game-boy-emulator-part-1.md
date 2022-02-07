---
title: "Writing a Rusty Game Boy Emulator (Part 1)"
date: 2022-02-07T20:25:38+01:00
author: Martin Kauppinen
---

Sometimes I get the urge to write an emulator. This has happened several times
in the past, but rarely have I actually finished any. I have made starts and
attempts for the NES, the CHIP-8, the 6502 CPU in general, the 8086 to play
Space Invaders. All of these I've eventually given up on. Not because I lack the
ability, but simply because I eventually grew tired of them. Maybe if I write
blog posts documenting my progress, I will get the push I need to make something
somewhat usable.

{{< toc >}}

## The sources
* [Wikipedia](https://en.wikipedia.org/wiki/Game_Boy)
* [GbdevWiki](https://gbdev.gg8.se/wiki/articles/Main_Page)
* [The Cycle-Accurate Game Boy
  Docs](https://github.com/AntonioND/giibiiadvance/blob/master/docs/TCAGBD.pdf)                                   
* [Opcode table](http://www.cochoy.fr/gb-doc/gameboy-opcodes.html)                                                
* [Game Boy CPU Manual](http://marc.rawer.de/Gameboy/Docs/GBCPUman.pdf)    
* Whatever other datasheets and information I can stumble upon

## The platform
The Game Boy is based on the Sharp LR35902 CPU, an 8-bit chip based on a
modified Intel 8080 and Z80 running at 4.19 MHz. Being 8-bit, the number of
opcodes is limited to 256. Or it would be, if it weren't for opcode 0xCB, a
prefix opcode which extends the instruction set to a maximum of 512. However,
some opcodes are unused. A full table I will reference a lot when writing this
emulator (or at least implementing the opcodes) [can be found
here](http://www.cochoy.fr/gb-doc/gameboy-opcodes.html).

64 KiB of address space is available, and the memory map looks like follows:

| Address range   | Size     | Description                              |
|-----------------|---------:|------------------------------------------|
| 0x0000 - 0x7FFF |   32 KiB | Cartridge                                |
| 0x8000 - 0x9FFF |    8 KiB | Video RAM                                |
| 0xA000 - 0xBFFF |    8 KiB | External, switchable RAM bank            |
| 0xC000 - 0xDFFF |    8 KiB | Internal RAM                             |
| 0xE000 - 0xFDFF |  7.5 KiB | Echo of Internal RAM 0xC000 - 0xDDFF     |
| 0xFE00 - 0xFE9F | 160 bytes| Sprite Attribute Table                   |
| 0xFEA0 - 0xFEFF |  96 bytes| Not usable                               |
| 0xFF00 - 0xFF7F | 128 bytes| I/O Registers                            |
| 0xFF80 - 0xFFFE | 127 bytes| High RAM                                 |
| 0xFFFF - 0xFFFF |   1 byte | Interrupts Enable Register               |

I'm not exactly sure what all those regions do yet, but we'll get there in due
time. Other than processor and memory, the display is 160 pixels wide by 144
pixels high and capable of 2-bit grey-scale. Then there's some sound and input
which I will worry about at a later date.

## Modelling the CPU
To start, I will just model the basic registers and implement some opcodes. No
graphics, input, sound, or anything fancy yet. Just the most bare-bones
structure to get started.

### The registers
The Game Boy has eight 8-bit registers: A, B, C, D, E, F, H, L and two 16-bit
registers: the program counter and the stack pointer. Several instructions allow
treating certain pairs of 8-bit registers as a 16-bit register. The available
pairs are AF, BC, DE, and HL. The registers can be modelled with a simple
struct as follows:

```rust
struct Registers {
    a: u8,
    b: u8,
    c: u8,
    d: u8,
    e: u8,
    f: FlagRegister,
    h: u8,
    l: u8,
    pc: u16,
    sp: u16,
}
```
Some simple glue code in the `impl`-block can take care of the handling of pairs
as 16-bit registers. I created a special struct for the flag register, just to
abstract away some bit-masking which would be necessary otherwise. Instead of a
`u8`, it's a struct of four `bool`s:

```rust
struct FlagRegister {
    z: bool,
    n: bool,
    h: bool,
    c: bool,
}
```

This struct models the zero flag (`z`), which is set when some arithmetic
operations result in 0; the negative flag (`n`), which is set when the last
arithmetic operation involved subtraction; the half-carry flag (`h`), which is
set when an arithmetic operation caused the high nybble to change[^halfcarry];
and the carry flag (`c`), which of course is set when an arithmetic operation
caused a carry.

[^halfcarry]: Sometimes. I've read that some 16-bit operations set the `h` flag
  when a carry happens from the low to the high byte. We shall see.

### The cycles

There are two kinds of cycles when modelling the duration of Game Boy opcodes:
machine cycles and clock cycles. As far as I can tell, 1 machine cycle = 4 clock
cycles. I don't yet know the practical difference between emulating with respect
to one or the other, so I will arbitrarily just encode the amount of machine
cycles an opcode takes as a `u8`. The opcode table lists clock cycles.

### The struct so far
Putting these together, the `Cpu` struct is very simple:

```rust
struct Cpu {
    registers: Registers,
    machine_cycles: u8,
}
```

## Modelling opcodes
Looking around at a few other Rust Game Boy emulators, I [found
one](https://github.com/simias/gb-rs/blob/8a817af85bcf279b15be7f2c0e79e173b044244b/src/cpu/instructions.rs#L40-L313)
which simply modelled the OpCodes as a big table of tuples of function pointers
and mnemonics. This emulator also served as inspiration for the above CPU model,
but I will modify it a bit. Instead of a raw tuple, I will introduce a new type,
a tuple struct, and it will contain a bit more information:

```rust
struct OpCode<'a>(
    /// Mnemonic
    pub &'a str,

    /// Function to call
    pub fn(&mut Cpu),

    /// Instruction size in bytes
    pub u8,

    /// Machine cycles
    pub u8,
);
```

Now, the big table of opcodes can be constructed as follows:

```rust
pub const OPCODES: [OpCode; 256] = [
    OpCode("NOP", nop, 1, 1),
    OpCode("NOP", nop, 1, 1),
    OpCode("NOP", nop, 1, 1),
    // ...
];

pub fn nop(_: &mut Cpu) {
    // No operation
}
```
Making sure that each opcode gets an index in the array corresponding to the
opcode itself. Thus, fetching an instruction is simply indexing this array with
the byte of memory that the program counter currently points to. Pretty clean
and simple. Although it makes for quite a long block of array initialization
code.

### Implementing some opcodes
A lot of opcodes in [the
table](http://www.cochoy.fr/gb-doc/gameboy-opcodes.html) are the same, just
operating on different registers and with slightly different functionality. I
feel like this calls for macros!

As an example, let's implement the `INC` instruction for all supported
registers. Referencing the table, the instruction is 1 byte, takes 1 or 2
machine cycles depending on if we're incrementing an 8-bit or 16-bit value, and
updates the Z, N, and H flags when incrementing 8-bit registers.

```rust
macro_rules! inc {
    ($reg:ident) => {
        pub fn $reg(cpu: &mut Cpu) {
            cpu.registers.f.n = false;

            let reg_old = cpu.registers.$reg;

            cpu.registers.$reg = reg_old.wrapping_add(1);
            cpu.registers.f.h = half_carry(reg_old, cpu.registers.$reg);
            cpu.registers.f.z = cpu.registers.$reg == 0;
        }
    };

    ($name:ident, $reg_hi:ident, $reg_lo:ident) => {
        pub fn $name(cpu: &mut Cpu) {
            cpu.registers.$reg_lo = cpu.registers.$reg_lo.wrapping_add(1);
            if cpu.registers.$reg_lo == 0 {
                cpu.registers.$reg_hi = cpu.registers.$reg_hi.wrapping_add(1);
            }
        }
    };
}

inc!(a);
inc!(b);
inc!(c);
inc!(d);
inc!(e);
inc!(h);
inc!(l);
inc!(bc, b, c);
inc!(de, d, e);
inc!(hl, h, l);
```

I'm not the greatest macro-writer in the world, so I'm sure this could be
improved a bit, but overall I'm pretty happy to not have to repeat the same code
ten times.
`INC (HL)` and `INC SP` need special treatment, so they don't get to be part of
the macros, instead getting their own functions.

Now, the instructions can be put into the table as follows (in their appropriate
places, of course):

```rust
pub const OPCODES: [OpCode; 256] = [
    // ...
    OpCode("INC BC",   inc_register::bc,     1, 2),
    OpCode("INC B",    inc_register::b,      1, 1),
    OpCode("INC C",    inc_register::c,      1, 1),
    OpCode("INC DE",   inc_register::de,     1, 2),
    OpCode("INC D",    inc_register::d,      1, 1),
    OpCode("INC E",    inc_register::e,      1, 1),
    OpCode("INC HL",   inc_register::hl,     1, 2),
    OpCode("INC H",    inc_register::h,      1, 1),
    OpCode("INC L",    inc_register::l,      1, 1),
    OpCode("INC SP",   inc_register::sp,     1, 2),
    OpCode("INC (HL)", inc_register::hl_ind, 1, 3),
    OpCode("INC A",    inc_register::a,      1, 1),
    // ...
];
```

Now, `DEC` can be implemented in much the same way. I won't go through how to
implement every single instruction in this post, but this gives an overview of
how I plan to implement the instructions going forward.

## Running an instruction

To run an instruction, a function like the following can be added to the CPU's
`impl`-block:

```rust
pub fn step_op(&mut self, op: usize) {
    let OpCode(mnemonic, func, size, cycles) = OPCODES[op];
    func(self);
    self.registers.pc = self.registers.pc.wrapping_add(size as u16);
    self.machine_cycles = cycles;
    println!("{}", mnemonic);
}
```

Testing this out with the following `main` function to run the `INC A`
instruction:

```rust
fn main() {
    let op = 0x3C; // INC A
    let mut cpu = Cpu::reset();
    println!("Before: {:?}", cpu);
    cpu.step_op(op);
    println!("After: {:?}", cpu);
}
```

Result:
```
Before: Cpu { registers: Registers { a: 0, b: 0, c: 0, d: 0, e: 0, f: FlagRegister { z: false, n: false, h: false, c: false }, h: 0, l: 0, pc: 256, sp: 65534 }, machine_cycles: 0 }
INC A
After: Cpu { registers: Registers { a: 1, b: 0, c: 0, d: 0, e: 0, f: FlagRegister { z: false, n: false, h: false, c: false }, h: 0, l: 0, pc: 257, sp: 65534 }, machine_cycles: 1 }
```
Success! The A register was incremented by 1!

## Progress?

You can follow my progress at the [GitHub
repo](https://github.com/martinkauppinen/gibberish) if you want. I'll do my best
to update that and this blog in tandem.
