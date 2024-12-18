---
title: "Integer Promotion was a Mistake"
date: 2024-12-18T19:07:47+01:00
author: Martin Kauppinen
---

At work this week, I had to write a simple function (in C++) that was just
supposed to pack eight `uint8_t`s into a single `uint64_t`. Extremely simple
stuff, just a bunch of bit shifts and bitwise or-operations. But since this is
C++-land, there's [footguns](https://en.wiktionary.org/wiki/footgun) here too.

## The function
It looked a little something like this:

```cpp
uint64_t packBytes(/*...*/)
{
    auto packed = 0;

    packed |= byte0 << 0;
    packed |= byte1 << 8;
    packed |= byte2 << 16;
    packed |= byte3 << 24;
    packed |= byte4 << 32;
    packed |= byte5 << 40;
    packed |= byte6 << 48;
    packed |= byte7 << 56;

    return packed;
}
```

Looks pretty innocent, right? I thought so. But then my tests started failing. I
was expecting something like `0x0102030405060708`, but only got `0x05060708`. It
seemed like the top _half_ of my bytes were just going in the bit bucket?

## The problem
_Well_. The thing about C++ is that it has inherited a bunch of baggage from C.
One piece of such baggage is [integer
promotion](https://en.cppreference.com/w/cpp/language/implicit_conversion#Integral_promotion).
Where basically, an integer type can be _implicitly_ converted to a different --
larger -- integer type.

This, combined with my use of `auto` for the declaration of the return value,
made `packed` initialize as an `int` (aka `int32_t`). Because that's what C++
defaults to if you just initialize with a simple number.

In each of the lines following, each byte is then implicitly cast to an `int`,
shifted by some amount, then bitwise or-ed into `packed`. But the last 4 bytes
are shifted by more than 32, thus the bits are shifted right off the end of the
promoted 32-bit value and thus the function essentially looks like this:

```cpp
uint64_t packBytes(/*...*/)
{
    auto packed = 0;

    packed |= byte0 << 0;
    packed |= byte1 << 8;
    packed |= byte2 << 16;
    packed |= byte3 << 24;
    packed |= 0;
    packed |= 0;
    packed |= 0;
    packed |= 0;

    return packed;
}
```

No wonder the top 4 bytes weren't there.

Finally, at the return statement, the compiler looks at the return type of the
function and goes _"Huh. That's a bigger integral type than I have here. Oh
well, here I go implicitly converting again!"_, casts `packed` to a `uint64_t`,
and calls it a day.

## The fix
So how do you fix it? Simple. Make sure `packed` gets initialized as a 64-bit
unsigned value from the start. There are three ways you could tweak the
initialization line above to achieve this:

  1. `uint64_t packed = 0;` -- Just do away with `auto`.
  2. `auto packed = uint64_t{0};` -- Be explicit (making `auto` pointless).
  3. `auto packed = 0ULL;` -- Use an [integer
     suffix](https://en.cppreference.com/w/cpp/language/integer_literal#The_type_of_the_literal).

That's it. Now, the bytes will get promoted to the proper 64-bit type, won't get
shifted off the end, and will actually end up in the `packed` value.

I went with option 1 because it is the most explicit. Option 2 just makes `auto`
redundant, and option 3 is _ugly_. I really dislike integer suffixes in C++.

## The hypocrisy
> _"Aha, Martin! You're complaining about integer promotion yet your
> implementation ***uses*** integer promotion during the bit-shifting. Curious."_
> - You, probably

Okay, you got me. I didn't even consider that initially. But yes, implicit
conversion and integer promotion is happening there as well. To be honest, it's
been a bit since I was doing stuff like this in C++ and forgot about the
details.

Another argument you can make is that I use `auto` too much in C++. Sure, it has
its place when a function is returning a type whose name is long and tedious to
type out, but was it really necessary with a `uint64_t`? I would still argue
yes, since `auto` is 4 quick characters to type, and keeps it consistent with
the rest of my variable declarations. Whereas `uint64_t` is twice as long, with
one of the characters being a special character that I have to use Shift to get
to. Ew.

## The iron oxide

Basically, I'm spoiled by Rust[^rust], with the compiler _actually_ caring about
types properly. No implicit conversions, `let` is even shorter and easier, and
the compiler will ___scream___ at you if you don't take the necessary step to
prevent bugs like this by being explicit with your casts.

[^rust]: You didn't _really_ think I was going to go an entire post without
    mentioning Rust, did you?

Here's the same function, written in a similar way in Rust (foregoing any
niceties like `u64::from_be_bytes`):

```rust
fn pack_bytes(bytes: [u8; 8]) -> u64 {
    let mut packed = 0;

    packed |= bytes[0] << 0;
    packed |= bytes[1] << 8;
    packed |= bytes[2] << 16;
    packed |= bytes[3] << 24;
    packed |= bytes[4] << 32;
    packed |= bytes[5] << 40;
    packed |= bytes[6] << 48;
    packed |= bytes[7] << 56;

    packed
}
```

Let's just _attempt_ to compile this.
```
$ cargo build
   Compiling example v0.1.0 (/home/martin/dev/example)
error[E0308]: mismatched types
  --> src/lib.rs:17:5
   |
5  | pub fn pack(bytes: &[u8; 8]) -> u64 {
   |                                 --- expected `u64` because of return type
...
17 |     packed
   |     ^^^^^^ expected `u64`, found `u8`
   |
help: you can convert a `u8` to a `u64`
   |
17 |     packed.into()
   |           +++++++

For more information about this error, try `rustc --explain E0308`.
error: could not compile `example` (lib) due to 1 previous error
```

Oh. Looks like `packed` was deduced to be a `u8`. That's not what we want, we
want a `u64`. Let's fix that.

```diff
fn pack_bytes(bytes: [u8; 8]) -> u64 {
-   let mut packed = 0;
+   let mut packed: u64 = 0;

    packed |= bytes[0] << 0;
    packed |= bytes[1] << 8;
    packed |= bytes[2] << 16;
    packed |= bytes[3] << 24;
    packed |= bytes[4] << 32;
    packed |= bytes[5] << 40;
    packed |= bytes[6] << 48;
    packed |= bytes[7] << 56;

    packed
}
```

And compile again:

```
$ cargo build
   Compiling example v0.1.0 (/home/martin/dev/example)
error[E0308]: mismatched types
 --> src/lib.rs:8:15
  |
8 |     packed |= bytes[0] << 0;
  |               ^^^^^^^^^^^^^ expected `u64`, found `u8`

error[E0277]: no implementation for `u64 |= u8`
 --> src/lib.rs:8:12
  |
8 |     packed |= bytes[0] << 0;
  |            ^^ no implementation for `u64 |= u8`
  |
  = help: the trait `BitOrAssign<u8>` is not implemented for `u64`
  = help: the following other types implement trait `BitOrAssign<Rhs>`:
            `u64` implements `BitOrAssign<&u64>`
            `u64` implements `BitOrAssign`

... (repeated 7 more times)
```

Oh right. Those are all `u8`s and I'm trying to mash them up with a `u64`. Silly
me. Let's fix that too:

```diff
fn pack_bytes(bytes: [u8; 8]) -> u64 {
    let mut packed: u64 = 0;

-   packed |= bytes[0] << 0;
-   packed |= bytes[1] << 8;
-   packed |= bytes[2] << 16;
-   packed |= bytes[3] << 24;
-   packed |= bytes[4] << 32;
-   packed |= bytes[5] << 40;
-   packed |= bytes[6] << 48;
-   packed |= bytes[7] << 56;
+   packed |= (bytes[0] as u64) << 0;
+   packed |= (bytes[1] as u64) << 8;
+   packed |= (bytes[2] as u64) << 16;
+   packed |= (bytes[3] as u64) << 24;
+   packed |= (bytes[4] as u64) << 32;
+   packed |= (bytes[5] as u64) << 40;
+   packed |= (bytes[6] as u64) << 48;
+   packed |= (bytes[7] as u64) << 56;

    packed
}
```

Compile again:
```
$ cargo build
   Compiling example v0.1.0 (/home/martin/dev/example)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.07s
```

There we go! Easy. No footguns here.

## The conclusion

I will enable the proper `clang-tidy` checks to prevent stuff like this in the
future. Can you believe that checking implicit conversions is [_off by
default_](https://clang.llvm.org/extra/clang-tidy/checks/bugprone/misplaced-widening-cast.html#cmdoption-arg-CheckImplicitCasts)?!

I was really hoping that the type deduction would realize that since I'm
returning an `auto` from a function whose signature declares that it should
return a `uint64_t`, then `auto` = `uint64_t` in this case. That, or that it
would give me a compiler warning/error. Clearly my hopes were too high.

What's even funnier is that I ended up completely removing this function in the
end because it was unnecessary to do things that way. Oh well.
