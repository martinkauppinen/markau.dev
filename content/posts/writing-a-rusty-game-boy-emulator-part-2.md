---
title: "Writing a Rusty Game Boy Emulator (Part 2)"
date: 2022-02-10T17:06:22+01:00
author: Martin Kauppinen
series: ["Rusty Game Boy Emulator"]
---

Having implemented the simple opcodes, I have sat down and implemented _a lot_
of instructions. Pure grunt work at this stage. I may have gone a bit
macro-crazy.

{{< toc >}}

## Simple loading
After implementing incrementing and decrementing of registers, I set my sights
on being able to move data _between_ the registers. Now, the Game Boy has 7
regular data registers: A, B, C, D, E, H, and L. There are separate instructions
for every combination of sending data from one source register to a destination
register, which doesn't have to be different from the source register. That's 49 
different instructions for just copying a byte from one place to another. If
every instruction has its own dedicated function in the `OpCode` table described
in the previous part, I'd end up having to write 49 separate functions doing
almost the same thing, just varying in which register is the source and which is
the destination.

That is, if macros didn't exist.

Luckily, they do! My first iteration of the macro for intra-register loading
looked like this:

{{< labelled-highlight lang="rust" filename="src/cpu/opcodes/ld.rs" >}}
macro_rules! ld {
    ($dst:ident, $src:ident) => {
        pub fn $src(cpu: &mut crate::cpu::Cpu) {
            cpu.registers.$dst = cpu.registers.$src;
        }
    };
}

pub mod a {
    ld!(a, b);
    ld!(a, c);
    ld!(a, d);
    // ...
}
{{</ labelled-highlight >}}

To use this, one simply plugs in `ld::a::b` into the proper place in the
`OpCode` table to implement the instruction loading A with the value of B. This
works, but it's a bit clunky having to still type out all 49 macros in 7
separate `mod` blocks, no matter how short they are. This is where I learned
about [Rust macro argument
repetition](https://doc.rust-lang.org/rust-by-example/macros/repeat.html).

The change to the macro is so simple, yet incredibly powerful. Simply adding
`$(...)+` in proper places, and the macro suddenly accepts a variable number of
arguments, generating code for each argument! Here's the modified macro, very
little has changed:

```rust
macro_rules! ld {
    ($dst:ident; $( $src:ident ),+) => {
        $(
            pub fn $src(cpu: &mut crate::cpu::Cpu) {
                cpu.registers.$dst = cpu.registers.$src;
            }
        )+
    };
}
```

It's essentially unchanged, save for a handful of `$(),+` characters. I also
changed the argument separator after the destination register to a semi-colon
because it just felt right to indicate which argument functions are being
generated for. Implementing _all_ intra-register load instructcions can now be
done as so:

```rust
pub mod a {
    ld!(a; b, c, d, e, h, l, a);
}

pub mod b {
    ld!(b; b, c, d, e, h, l, a);
}

// ...
```

Hm. We still have to type out all 7 `mod` blocks. Macro time!

```rust
macro_rules! gen_ld {
    ($( $dst:ident ),+) => {
        $(
            pub mod $dst {
                ld!($dst; b, c, d, e, h, l, a);
            }
         )+
    };
}

gen_ld!(b, c, d, e, h, l, a);
```

This generates a `mod` block for each argument to the macro and in turn calls
the `ld!` macro to generate all the functions. Is this overkill and unnecessary?
Yes. But I like it! And that's as good a reason as any to do it. The less code I
have to repeat, the fewer places there are to make mistakes.

Speaking of overkill, I also wrote macros to generate the unit tests for all
these functions. I won't go over that here, though. It's the same concept, but
it gives me loads of unit tests and that feels good if nothing else.

## A memory of a memory
At this point, a lot of the instructions that were left needed to interact with
the memory somehow, so it was time to tackle the implementation of that. The
Game Boy has a 16-bit address space, so one could just slap a `[0u8; 0x10000]`
array into the CPU model and call it a day for now. That is essentially what I
did as well, but with a bit more logic, since the Game Boy has a memory map and
different regions are responsible for different things.

A region of memory is essentially just a wrapped `Vec<u8>`:
```rust
pub struct Ram {
    start: u16,
    end: u16,
    memory: Vec<u8>,
}
```
The memory keeps track of its start and end addresses, so that it can subtract
its own start address from whatever address it is given to read/write in order
to index into the `Vec` properly. The `impl` block for this struct simply takes
care of that logic and exposes functions for reading/writing
bytes/words[^words?] from
the memory region.

[^words?]: I realize I haven't defined what a word is. It' a 16-bit (2-byte)
  value. I don't know if this is proper terminology vis-a-vis the Game Boy, but
it makes sense to me.

Wrapping a bunch of theses into a `MemoryMap`[^MemoryMap] struct, setting their start and
end addresses to the ones described in the previous post and bam! We have a
memory thing we can slap into the CPU model and be on our merry way implementing
more instructions.

[^MemoryMap]: This simply exposes the same read/write functions as the structs
  it wraps, but it takes care to call the correct region depending on the
address it is given.

## Loading... Loading... Loading...
And this is the stage of the project where I stayed up until nearly midnight on
a work day. The Game Boy's processor has ***92*** load instructions. Sure, I had
already implemented 49 of them by virtue of macros, and many of the rest can be
implemented with macros as well to cut down on typing, but still. That leaves 53
instructions to implement, and they don't all share the same nice common
functionality as the first 49. What follows are some highlights.

### `LD reg, d8`
The first two-byte instruction so far! It simply loads the next byte in memory
after the instruction into the register the instruction codes for. This was a
nice start to having to deal with the newly implemented memory. All that had to
be done was assign the register with the value in memory pointed to by the PC
register + 1. In order to read instruction arguments for future instructions, I
implemented two functions in the CPU: `get_byte_argument()` and
`get_word_argument()`, which simply retrieve the next byte/word after the
instruction.

### `LD reg, (HL)` / `LD (HL), reg`
This is the first time the register pair HL comes up. Many instructions treat
the H and L 8-bit registers as a single 16-bit register, with H being the high
byte, and L the low byte.

These instructions take the value of the HL pair, interpret it as a memory
address and either loads or stores the value at that location in memory.

### `LD (HL+), reg`
This is a weird-looking instruction[^versions]. There's a mysterious plus sign that if one
doesn't know what the instruction does, could be interpreted in a number of
ways. I had some trouble finding out what this actually did for a little bit for
some reason, but I believe I know what it does now.

Basically, it loads the value of the register into the memory location pointed
to by HL, and the increments HL. I can see this being useful for example in a
loop where one wants to fill an array of bytes with values. No need to waste an
instruction on an extra increment instruction. Indeed, in the loop example this
is especially valuable, with how low-powered the CPU is. This instruction takes
the same number of cycles as `LD (HL), reg`, and so one saves two entire machine
cycle (8 clock cycles) per iteration.

### `LD (C), A` / `LD A, (C)`
These instructions do not do what you think they do. I thought they did, and I
had to spend some time debugging before I scrolled down a bit on the [opcode
table page](http://www.cochoy.fr/gb-doc/gameboy-opcodes.html) and
realized that these are special. `(C)` does not in this case mean "the memory
pointed to by the C register". No, in this case C is an offset from the address
`0xFF00`. The difference is simple to implement, but it was late at night and I
was tired from implementing everything in the macros, so this took me a while to
realize.[^and-yet]

[^and-yet]: Typing this up made me realize that the instructions `LD A, (a8)`
  work the same way and had to go back to the load instruction implementations
and change them. I really thought I went through everything when I finished it
two days ago, but apparently not.

### `LD HL, SP+r8`
Oh, load HL with the sum of SP and the argument to the instruction? Almost! not
quite, though. The problem is that the argument is an 8-bit _signed_ integer,
while the SP register is a 16-bit register. The fact that it's signed is
important; you cannot simply cast the `u8` as a `u16`. This will just pad the
number out to 16 bits with zeroes, and since the highest bit of a two's
complement number is the sign bit, we would lose the sign information. Not to
mention that it would be the completely wrong positive number as well. Padding
-1 in 8-bit (`1111 1111`) with zeroes will result in 16-bit +255. [Sign
extension](https://en.wikipedia.org/wiki/Sign_extension)[^sext] must be used
here. Instead of padding with zeroes, pad with copies of the highest bit. This
preserves the value and sign of the number.

But it doesn't end there! There's a little detail one should not miss when
implementing this instruction. It is the only load instruction that affects the
flags! I assume this is due to the addition of SP with the immediate value.

[^sext]: Apparently abbreviated as "sext". I find this humorous.

### What was so hard about that?
Oh, it wasn't necessarily difficult from a technical perspective. It's just that
I lost myself in all my macros quite a few times and I was tired from a long day
at work, _and_ I stayed up until nearly midnight doing all this grunt work. But
I got it done in the end! One could argue that the amount of macros I'm using is
overkill, but I would much rather not have to implement every single function
multiple times changing one or two things than not get confused by macros. If
nothing else, it forces me to learn more about Rust macros.

## Logic and arithmetic
The next day after implementing the load instructions, I sat down to implement
all variants of `ADD`, `SUB`, `AND`, `OR`, etc. This was simple enough, but I
had to take care to actually set the flags properly for them, and try to come up
with good tests. But overall it was a pretty easy job, since there weren't
really any specialized versions of these intructions that did anything
surprising.

## Where are we today?
Today I [implemented the `PUSH` and `POP`
instructions](https://github.com/martinkauppinen/gibberish/commit/e5934aa214205957680ade447572fb6a8838dea9)
so that the stack can be used. Nothing surprising there. Nothing I've found yet
at least. It will be interesting to in the future try to run a program written
for the Game Boy and see where my implementations are wrong. But that's a
problem for the future.

## Where to next?
I've implemented almost 200 instructions and I'm running out of ones I can
implement at this stage without implementing peripheral stuff. But I think next
is going to be the jump and call instructions. Can't have programs without
functions and subroutines, after all!

[^versions]: There are also versions of this instruction with a minus sign
  instead, or the operands reversed.
