---
title: "X Marks the Macro"
date: 2025-08-27T15:31:42+02:00
author: Martin Kauppinen
draft: true
---

In this post I'd like to document a semi-obscure pattern that (ab)uses the C/C++
preprocessor to generate a bunch of code known as
[X-macros](https://en.wikipedia.org/wiki/X_macro). Probably slightly more useful
in C than C++ due to the latter being more featureful, having better ways of
doing metaprogramming than using the preprocessor. So for this post I will stick
to C.

{{< toc >}}

## The classic example
Imagine you have an enum representing different kinds of pet you might be
allowed to own in your jurisdiction in `legal_pets.h`.

```c
enum LegalPet
{
    Cat,
    Dog,
    Fish,
    Hamster
};
```

Maybe you want to print the list of pets somebody owns, so you write a helper
function to translate a `LegalPet` to a string (because C enum variants are just
named integers) with a simple `switch`:

```c
const char *pet_to_string(enum LegalPet pet)
{
    switch (pet)
    {
        case Cat: return "Cat";
        case Dog: return "Dog";
        case Fish: return "Fish";
        case Hamster: return "Hamster";
        default: return "Unknown or illegal pet";
    }
}
```

You test your program and everything is great until that fateful day when your
government decides to relax pet ownership laws, adding a whole bunch more pets
to the list of legally allowed pets. For some reason they banned owning hamsters
though. Something about it being suspicious how much they can hide in their
cheeks.

Now you have to go and update your code in at least two places. You have to
expand the enum (and remember to delete Hamster from it), and adapt your string
translation function to account for all the new kinds of pet you're allowed to
own (and remember to delete the Hamster case). If you're switching on the enum
elsewhere, you have to update your code there too. _If only there was another
way_.

### Introducing the X
With some clever preprocessor trickery, you can avoid having to update your code
in multiple places. If you can extract some pattern common to everywhere you use
the enum variants, you can have the preprocessor generate code for you instead
of having to hunt around the codebase yourself. So you modify `legal_pets.h`,
where you define this macro at the top:

```c
#define LEGAL_PETS  \
    X(Cat)          \
    X(Dog)          \
    X(Fish)         \
    X(Hamster)
```

`LEGAL_PETS` expands to a bunch of calls to a different macro called `X`, each
taking the legal pet as an argument. What is `X`? It's whatever you want,
whenever you want[^tm]! The trick here is that you define what `X` should be at
the point where you want to do something with the list of all legal pets.
[^tm]: Trademarked slogan, of course.

### Defining an enum
For example, when defining the enum from before you do this:

```c
#define X(pet) pet,
enum LegalPet
{
    LEGAL_PETS
};
#undef X
```

Since you define `X` before invoking the `LEGAL_PETS` macro, the recursive
macro expansion will take your definition of `X` and replace all calls to `X`
inside the `LEGAL_PETS` macro with your definition. In this case it's just
appending a comma to each pet so that the enum is syntactically correct. At the
end, you `#undef X` so that `X` can be redefined elsewhere.

### Generating code
Defining an enum is not that exciting, but the power of this pattern goes
further. You can now write your string translation function _once_ and never
have to touch it again, because you're going to generate all the necessary code
from the `LEGAL_PETS` macro! You just need to figure out how to define `X`. This
time, you define `X` as follows:

```c
const char *pet_to_string(enum LegalPet pet)
{
    switch (pet)
    {
        #define X(pet) case pet: return #pet;
        LEGAL_PETS
        #undef X
        default: return "Unknown or illegal pet";
    }
}
```

Using the stringification operator (`#`), the argument to `X` is expanded and
put in double quotes, leading to the definition of a valid string. Let's expand
`Cat` as an example:
```
#define X(pet) case pet: return #pet;

X(Cat) -> case Cat: return "Cat";
```

And this happens for all the pets in `LEGAL_PETS`. Suddenly you can change the
list of legal pets without ever having to touch this function again, since the
code is generated at the preprocessing step of compilation. Now you can simply
add the long list of newly-legal-to-own pets to `LEGAL_PETS` and not have to
worry about anything else (as long as you remember to also delete Hamster).

## Other example use cases
The previous example is pretty simplistic, but is basically the canonical
example of how to use X-macros, so I had to include it. You can do way more with
them, and the `X` macro can have as many arguments as you want. The arguments
can even be other X-macros!

### Multiple arguments
For example, let's define a macro that uses X-macros to define a struct and a
print function for the struct.

```c
/* format: X(type, name, format specifier) */
#define STRUCT_FOO_MEMBERS              \
    X(int, id, "%d")                    \
    X(float, percent_complete, "%.2f")  \
    X(char, grade, "%c")
```

Now, using this macro we can define the struct and a simple print function that
will print all struct members and their names, all according to the format
specified in the X-macro:

```c
struct Foo
{
    #define X(type, name, _fmt) type name;
    STRUCT_FOO_MEMBERS
    #undef X
};

void print_foo(const struct Foo *foo)
{
    #define X(_type, name, fmt) printf(#name " = " fmt "\n", foo->name);
    STRUCT_FOO_MEMBERS
    #undef X
}
```

And just like that, you have a struct that will print its fields nicely for
you, no matter how many fields you add/remove/reorder. To help visualize, this
expands to the following code[^except-string-concat]:
[^except-string-concat]: Truth with modification. It's actually expanded to a
    bunch of strings concatenated, but the compiler handles this and effectively
    works as a single string. See Compiler Explorer link at end of article.

```c
struct Foo
{
    int id;
    float percent_complete;
    char grade;
};

void print_foo(const struct Foo *foo)
{
    printf("id = %d\n", foo->id);
    printf("percent_complete = %.2f\n", foo->percent_complete);
    printf("grade = %c\n", foo->grade);
}
```

### Macros as arguments
To go even further, we can specify a struct-defining macro taking the `X` macro
as an argument. This eliminates the need to `#define` and `#undef` stuff if you
define well-named reusable macros instead. Let's go step-by-step to define some
macros that enable the automatic generation of a `print_struct` function, so you
don't have to write one for each struct you want to be printable in this way.

First, let's have some well-named helper macros. Our X-macros will have the same
type-name-format_spec format as we used in the previous example.

```c
#define STRUCT_FIELD(type, name, _fmt) type name;
#define PRINT_FIELD(_type, name, fmt) \
    printf(#name " = " fmt "\n", s->name);
```

These are the exact same X-macros we `#define`'d and `#undef`'d in the previous
example. Except this time we made them standalone macros.

Now let's make a list of X-macro members for a `Rectangle` struct. This will
differ from `STRUCT_FOO_MEMBERS` in that it will take the X-macro as an
argument:

```c
#define STRUCT_RECT_MEMBERS(X)  \
    X(int, width, "%d")         \
    X(int, height, "%d")
```

Let's stop here and imagine what passing one of the previous helper macros to
this macro as an argument would do. First, let's use `PRINT_FIELD` as an
example:

```c
/* 1. Call the macro */
STRUCT_RECT_MEMBERS(PRINT_FIELD)

/* 2. Expand the outer macro, replace X with PRINT_FIELD */
PRINT_FIELD(int, width, "%d")
PRINT_FIELD(int, height, "%d")

/* 3. Expand PRINT_FIELD */
printf("height = %d\n", s->height);
printf("width = %d\n", s->width);
```

`s` is not defined, but that's okay. We'll just make sure to define `s` in any
macro where we use this. This can be wrapped in a function block to define the
print function. For completeness, let's also look at passing `STRUCT_FIELD`:

```c
/* 1. Call the macro */
STRUCT_RECT_MEMBERS(STRUCT_FIELD)

/* 2. Expand the outer macro, replace X with STRUCT_FIELD */
STRUCT_FIELD(int, width, "%d")
STRUCT_FIELD(int, height, "%d")

/* 3. Expand STRUCT_FIELD */
int width;
int height;
```

That can be wrapped in a struct-defining block to define the fields of the
struct.

Let's put all of these together and create a macro that takes the list of X-macro
fields and generates a struct definition and print function definition using the
helper macros!

```c
#define PRINTABLE_STRUCT(struct_name, FIELDS)             \
    struct struct_name                                    \
    {                                                     \
        FIELDS(STRUCT_FIELD)                              \
    };                                                    \
    void print_##struct_name(const struct struct_name *s) \
    {                                                     \
        FIELDS(PRINT_FIELD)                               \
    }
```

As you can see, I have wrapped `STRUCT_FIELD` in a struct definition and
`PRINT_FIELD` in a print function definition. The print function is named
`print_StructName` according to whatever struct you define with it. Now we can
generate the `Rectangle` struct and print function like so:


```c
PRINTABLE_STRUCT(Rectangle, STRUCT_RECT_MEMBERS)
```

Let's expand this one step to help visualize what's going on:


```c
struct Rectangle
{
    STRUCT_RECT_MEMBERS(STRUCT_FIELD)
};
void print_Rectangle(const struct Rectangle *s)
{
    STRUCT_RECT_MEMBERS(PRINT_FIELD)
}
```

The inner macros are then expanded as explained before. And we're done with our
printable struct! Let's define another one for fun; here's `Circle`:

```c
STRUCT_CIRCLE_MEMBERS(X) \
    X(float, radius, "%f")
PRINTABLE_STRUCT(Circle, STRUCT_CIRCLE_MEMBERS)
```

Done. The `Circle` struct is defined with a print function that works the same
as the one for the `Rectangle` struct. You might be able to compress this even
more to a single macro definition with a bunch of indirection and variadic
arguments, but I haven't bothered figuring out how. I feel like it would also
complicate the understanding of X-macros, which is the goal of this post.

## Conclusion and further reading
You can keep going however far you want and complicate X-macros further, but at
some point you have to stop and I have decided that this is that point for me. I
also didn't want to look too much at prior work once I found out about them when
writing this post, so I'm sure others have done a better job than me at defining
these macros.

Here's some good references for further reading:
* [Wikipedia](https://en.wikipedia.org/wiki/X_macro)
* [Wikibooks](https://en.wikibooks.org/wiki/C_Programming/Preprocessor_directives_and_macros#X-Macros)
* [An excellent blog post going much further in-depth on the topic](https://danilafe.com/blog/chapel_x_macros/)

Also, I've put the examples from this post into a [Compiler Explorer
snippet](https://godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAMzwBtMA7AQwFtMQByARg9KtQYEAysib0QXACx8BBAKoBnTAAUAHpwAMvAFYTStJg1AB9U8lJL6yAngGVG6AMKpaAVxYMQkgGykHAGTwGTAA5dwAjTGIQAGZpAAdUBUJbBmc3Dy9fROSbAUDgsJZI6LiLTCs8hiECJmICdPdPH3LK1Jq6ggLQiKjY6QVa%2BsbMlsHO7qKS/oBKC1RXYmR2DgB6ACpNre2d3b39g8Oj4/WAUg0AQQBqdavlTAIrzFVWePor8%2Bv11c/TgCYYlgaMErv5sABxC7%2BYzKbAAFSEVw%2BAFZHJ8kUiABoQRxMAgzDGEjGnVHojHYgAiqGABKJhJJaMuhOxADE8AoELS6SjGddyRAABKsQZRGa/S7/QGYYGYK7Y%2BIPAkKgikT6MdygzDAMT3Ai/ADsACEyUiwZDobCEQaKacYsaJQDXAwgXLxRc0AxBldkAg6jdlcYiMZBsQgsAIOqWJrtbRdVdlWKJUaTVcFAB3Qg%2Bq4QBMp07JpncyVAoKy%2BWK71MJTxh4gK7EB6LBgfAHK232vl0s1QmHwoQp%2BmO53S12FulApiuWgEOsNghNlt/OQMADWDFQaebJCudHoMZrer%2Bf3bef1NqT54unw2J1vd/vD626NuAFkpzY3rK6sB3IwCAonheFhPw%2BJlvjdSUgmQNwsA%2BGJHEGfBUAAOgQW1sHRa9bn4YgWDxOtsQIABPBVSCuZg2DInC8MeBQFWQPAaFFG4fgdKUZSuIQ4QAJTkRw4WMFkAHkhOMF9sBfQ1sG4xFuTpBkU2xIIVR3dAyP%2BY8/mRdANK5OT9IUsc5WoWhUDxMiFSWP9jDQYD6AITB1KPf5kWQv4qF0pFDM7YyfTqMjgGIJgsCczTkWQXS3RDVxrCuFlUFQA0O0HdjS2M4jSPI1hHKuYwqBYfErgy2UKMwE8jK43j%2BMEkSxIkqSZLzIcXUxa1yqvS4ADdUDwdB41DQQ8oSiAPS9aLYvi1Abn4VBEyvAsfOLaU0uxQMSJy0qqIKpUBoIKgIElUrFzgiljvyx4NIZBgNKohKAFp0NKsU7RTSq%2BIE4TRPEyTpP7IzJSdFq2olS4b0fcGIYh58rhfJhkGIKbv1/QRANed5n1YjqrwBEsQTe6qWQASWwfwKQgYqyM23KCWKrK2HapaOOUbjCZCD7idJ8n1sp7KtsK%2BJdv2w7suO21To0q5zsXK6btTB6YmwJ6GZx5aQWZ1m4QuQ0wWMfG4QgcaCGMKmiZJikhD0/SvNJIzDdTAhiBio2jqt123fpG3FqNd2fd993vLk03SaECA9cEjmKUtv3XYD4kz3baPE6T2OkW63r%2BuU4xJUlQ3jeykaBDGh2nftx3rDztgbgUAl80NHkBwxIPzYgdW2fDs2o6TuSU4%2BeOQex1K8Z497jG47Bqu%2BhqQ8xGvPeZCBlLIjN0AIBBQpcnSj1n3l58Xq4EEwPBgAQFTLq0ze/jmz5W817XsF14f%2BIgbjMGsQxgHoMiw7Hif6t%2Bq%2B2K41lGHRwhNuKOB1pPX6EAZ71yMtiKgplzL1mCngVwCh15aQ8lva%2BLM2Zax1nrHEeAlif04o/ASoDwGQL/jJGYHA5i0E4MiXgngOBaFIKgTgjhUwLCsi2GIPBSAznYQwuYK4QDIg0PoTgkhWGaF4FwjgvAFAgGkSIrQcw4CwCQPEeI5BKDQXfndai%2BFRF%2BHwEQUM6A9A0GnFEO6B9gpRFUeY/gggRBiHYFIGQghFAqHUOY3QXBhENkPjYjQjDmHyPMUo5QDYBaoGWAoBQ24hKuAIPEDJVxVAAA5vB3W8JIK4wBkDICuFwVyfxsy4EINuSUISrjOGAnQKIAiuD0OEQorRSBbLxFaWQCgBcWn0GiMgYAlS/h8DoA5YgqiIDhAUaQcIQQ6hEU4EIlZzBiBESEuEbQr8RFCNsmwQQQkGC0HWeYrA4RXDAFxLQWgrihFYDwkYcQ1ySGHLwJ1TAriOHPFfhklYHDlIVCWbQPA4Qgo7OcFgJZDs8AsA2bwX5xBwhJEwBSTAbyP5hm6XwAwwAFAADU8CYDTEJBUbChHuOEKIcQPi6X%2BLUEs4J%2Bh34gFMDZfQULVGQDmKgeIVRXGKLRdYv58A5iWG%2BXYCMzoRieBCQEUsUw%2BghJyCkAQiq9CaqqJMXo0QQkyusO0IYDQXBND0CaqoHR6gGuKOqiw5qdXGvNQ66YHT5iLGWBIKJHAWGkDYRwpROT8mFOKaU8plS3I1MsfUgEHTeAaLEaQCRUiZEcDkaQZFXANDSODYozgKi1FdNETMf1fwYkhuLWWzRcw0XJDsJIIAA%3D%3D%3D)
so you can play around with them, seeing how adding/removing fields
automatically expands to the proper struct/enum/function.
