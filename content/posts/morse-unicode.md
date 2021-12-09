---
title: "Morse Unicode"
date: 2021-12-09
author: Martin Kauppinen
---

I have had a delightfully terrible idea: What if you could transmit unicode
characters as Morse code?

## Exploratory phase
I started out as most non-serious research starts out, by consulting my favorite
search engines and online encyclopedias, to learn more about the basics of the
subject at hand. Everyone knows about Morse code, but to implement my idea, I
had to *know*[^know] Morse code.

[^know]: Well, I had to look up the tables of codes.

Morse code is a variable width code, much like the UTF-8 encoding of the Unicode
standard. You might see where I'm going with this, and yes, it is probably as
bad of an idea as it sounds. But I'm not doing this because it's practical.

### Previous work
In my search I found one javascript library that I could not figure out the
actual extended encoding for, and an April Fool's thread from a [Unicode mailing
list](http://unicode.org/mail-arch/unicode-ml/y2002-m11/0502.html). 

The mailing list thread is interesting, but the proposed encoding has
problems[^serious-joke]. The main problem is that it is slightly incompatible
with Morse code! For example: a full stop (.) in Morse code is `.-.-.-`, which
in the April Fool's encoding corresponds to the ASCII `ENQ` control character.
And if you wanted to send a dash followed by a full stop (.-) in Morse, it would
be `-....- .-.-.-`, which is ASCII carriage return! And that's just simple
Morse. If we're working with Japanese [Wabun
code](https://en.wikipedia.org/wiki/Wabun_code), the Wabun full stop is encoded
as `.-.-..`, which in the April fool's scheme clashes with ASCII `EOT`, the
character sent when entering `CTRL-D` at a shell.

[^serious-joke]: I am aware that it's a joke, but I'm trying to do this for real
  ;).

### Standard Morse, extensions, and extending
I will base my thought-experiment on international (ITU) Morse code, as opposed
to the American or Gerke variants. This code consists of the letters A to Z in
the standard latin alphabet (with no distinction between uppercase and
lowercase), the digits 0 to 9, the punctuation marks .,?'!/()&:;=+-\_"$@, some
control characters (called prosigns), and a whole bunch of non-latin extensions.
The list I based my research on can be found [on
Wikipedia](https://en.wikipedia.org/wiki/Morse_code#Letters,_numbers,_punctuation,_prosigns_for_Morse_code_and_non-Latin_variants)

In order to extend Morse code, I needed to find a sequence of 'dits' and 'dahs'
that had not been used before. Ideally it would be as short as possible, to
facilitate high bandwidth sending of 💩 over telegraph. The first shortest
unused code I could find was `-..--`. It does not clash with either the ITU
Morse code, or any other [Morse code for non-Latin
alphabets](https://en.wikipedia.org/wiki/Morse_code_for_non-Latin_alphabets).
I later found out that it *does* in fact clash with the
character for ユ in Wabun code. After some more searching I found
another code that doesn't clash with this one either, though: `-.....`.

So `-.....` could be used as some sort of control character (prosign) to signify
that what follows is encoded Unicode. I'm taking a page out of Wabun code's book
here, as when Wabun code is intermixed with International Morse Code, a prosign
is used to indicate the beginning and end of Wabun. Let's use the reverse of our
new prosign as an ending prosign: `.....-`. This doesn't clash with anything
either.

At this point I examined the length of each Morse code letter. The shortest
letters are a single 'dit' or 'dah', and it goes all the way up to eight
(`........`, signifying error or correction). So if all Unicode Morse codes are
encoded with a combination of groups of eight 'dits' and 'dahs', they cannot be
confused for a regular Morse code letter (except for the error code, but I'll
get to that). In other words: we can use standard 8-bit bytes!

### Code points, code units, glyphs, and grapheme clusters, oh my!
So the next step was figuring out what the heck Unicode actually is, and the
different ways there are of encoding it. In so doing, I was immediately hit with
words like *grapheme clusters* and *code points* and I kept thinking "dangit can
someone please just tell me what those mean!?", but eventually figured it out
despite some similar-sounding terms and preconceptions I had about text and
computers.

This is my understanding in short:

- Code point: A number assigned to a grapheme by the Unicode Consortium.
- Grapheme: Smallest visual unit of text. Can be for example a character, or a
  combining diacritic.
- Grapheme cluster: A collection of graphemes that make up one rendered visual
  unit of text. For example: `ȧ̩͖̦̗̹̘̟ͥ̇̂͝` is a grapheme cluster made up of the letter and a
  bunch of diacritics.

#### UTF-8
A short description of the variable width UTF-8 encoding follows here. Let's use
the code point for 🦀 (U+1F980) as an example.

If the highest bit of a UTF-8 byte is 0, the byte is interpreted as ASCII. If it
is 1, the number of leading ones describes the number of bytes the encoded code
point takes up. Followup bytes after the first one start with `10`. Our example,
the crab emoji, consists of four bytes, and thus looks like the following:
```
11110xxx 10xxxxxx 10xxxxxx 10xxxxxx
```
The `x` bits are simply filled in with the bits from the code point (`0x1F980 =
1 1111 1001 1000 0000`), and there we have it![^utf8]

[^utf8]: The [Wikipedia page for UTF-8](https://en.wikipedia.org/wiki/UTF-8) is
  good for a more thorough understanding of the encoding.

Now we need to decide whether a 1 should be a 'dit' or a 'dah'. Remember the
error/correction prosign? We want to be able to distinguish this from the ASCII
null (`00000000`), which means that the ASCII null should be made up of 8
'dahs'. Eight 'dits' (ones) is not a valid UTF-8 byte, so we should be fine with
this small kludge of the UTF-8 Morse encoding. So encoding our crab emoji, we
get the following Morse code sequence:
```
-..... ....---- .--..... .-.--..- .------- .....-
```
That's a crab in Morse! We have now extended Morse code in a way that is
compatible with the original, the non-latin alphabet versions, and Wabun code.
Furthermore, since UTF-8 is ASCII-compatible, we're backwards compatible in more
ways than one. This was a silly exercise, but I had a lot of fun
[nerd-sniping](https://xkcd.com/356/) myself.

## Implementation
You better believe I wrote a hacky Morse <-> Unicode converter! If you want to
use it/read terrible Rust code, you can find it
[here](https://github.com/martinkauppinen/umf-8).
