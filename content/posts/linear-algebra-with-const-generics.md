---
title: "Linear Algebra with Const Generics"
date: 2023-09-19T15:46:26+02:00
author: Martin Kauppinen
---
I was recently doing a project with some friends where we extracted the pixel
data from [RAW images](https://en.wikipedia.org/wiki/Raw_image_format). One step
in extracting a picture from a RAW image is demosaicing, which boils down to
doing some kind of convolution over the pixel intensities in the image with a
convolution kernel defined by the color filter array. That's when I had the idea
to just tinker with creating a linear algebra/image processing library in Rust
that utilizes [const
generics](https://practice.rs/generics-traits/const-generics.html) in order to
encode some properties of matrices in the type system. This post chronicles the
creation of one such library. It is very bare bones, but I find it interesting
what you can encode in the type system.

This is not useful as a general linear algebra/image processing library, since
const generics are compile-time. Thus creating matrices of arbitrary dimensions
at runtime can't be done with const generics. This is purely an academic
exercise in what you can do with them. But it was fun nonetheless!

{{< toc >}}

## Earlier (better) work
There are a couple of crates that do linear algebra, multidimensional arrays,
and computer graphics. Here's a small selection:

* [ndarray](https://crates.io/crates/ndarray)
* [nalgebra](https://crates.io/crates/nalgebra)
* [cgmath](https://crates.io/crates/cgmath)

None of these utilize const generics the way I'll be doing, and for good reason.
We will get to that in due time, though.

## In the beginning, there was the matrix
I will be restricting this to 2D matrices [^2d], as the original domain where I got
the idea was relating to 2D photographs. There are multiple ways we could define
a matrix with `M` rows and `N` columns, encoding the dimensions with const
generics. There are a couple of data types that you could base a `Matrix<T>` on.
They all have their pros and cons.
[^2d]: Vectors are just 2D matrices where one of the dimensions is 1. So
  matrix-vector operations will be supported.

### Option 1: `Vec<Vec<T>>`
A nested `Vec` is very flexible and easy to reason about when it comes to
indexing, as there is no need to manually calculate a linear index from a
two-dimensional one, and its size is completely dynamic. However, given that it
is a double `Vec`, there is also potentially a double indirection happening,
having to address memory into the outer `Vec` to find the inner one and then
index into that one. So I opted not to use this representation.

### Option 2: `[[T; N]; M]`
A two-dimensional array of `T` has the same advantage of intuitive indexing as a
nested `Vec`, but without the disadvantage of a double indirection, since its
dimensions are known statically. This means the compiler will take care of the
calculation of a linear index for us! However, an array lives on the stack and
perhaps we want to work with some really large matrices in certain applications.
So I opted not to use this representation either.

### Option 3: `[[T; M]; N]`
Wait, did I accidentally put option 2 in twice? No, look again! The dimensions
are switched. Option 2 contained `M` arrays of length `N`, whereas this one has
`N` arrays of length `M`. This is known as [row-major and
column-major](https://en.wikipedia.org/wiki/Row-_and_column-major_order) order.
Option 2 stored the elements in memory sequentially, row by row. This option
instead stores them column by column. The choice between them is essentially
arbitrary, but it is important to know. I elected to store my matrices in
row-major order, so this option is not that relevant but very important to
mention. There will be a cool little trick regarding row-major vs. column-major
later.

### Option 4: `[T; M * N]`
Ah, a linear array. Pretty simple, though you have to calculate the linear index
manually to use it. That's not too bad though. It's really simple to do
operations on all elements at once since there is no nesting to think about and
row-major vs. column-major order is entirely arbitrary. Both options can be
represented by this type and you don't need to go through `std::mem::transmute`
to get from one to the other. Wonderful!

However, you cannot do this with const generics. They can only be used as
standalone arguments, not in const operations. This just leaves us with...

### Option 5: `Vec<T>`
This has the same properties of option 4, but with the huge advantage that it is
actually valid Rust! Also, since it's a `Vec` it lives on the heap and our
`Matrix<T>` type can own that memory. No large amounts of data on the stack. And
the information about the matrix dimensions is entirely on the `Matrix` type.
Our underlying type is completely agnostic to the size and layout of the
`Matrix` type, which I think is pretty nice. No double bookkeeping. What relates
to the matrix, is defined in the type of the matrix. All of these reasons (and
the row-major vs. column-major trick I will get to later) made me decide on this
as the underlying type of the matrix:

```rust
#[derive(Debug, Clone)]
pub struct Matrix<
    T,
    const ROWS: usize,
    const COLS: usize
>(Vec<T>);
```

You can probably already start to see another reason why other libraries do not
use const generics this way. That is one long type definition. But this is just
for fun, so we'll roll with it. Besides, it's going to get so, so much worse :)

## `impl`-blocks from Hell
Let's implement some functionality for our shiny new `Matrix<T>`. How about
being able to construct a matrix from a two-dimensional array? That would enable
us to do things like
```rust
let matrix = Matrix::new([
    [1, 2],
    [3, 4],
]);
```
Pretty nice and readable! Now let's see what the implementation of that would
look like:
```rust
impl<T: Element, const ROWS: usize, const COLS: usize>
Matrix<T, ROWS, COLS> {
    pub fn new(array: [[T; COLS]; ROWS]) -> Self {
        Self(array.concat())
    }
}
```
Oh. Oh no, this is going to get out of hand fast. And that's exactly what's so
fun about it. :)

`Element` is just a combination trait of `Copy + Clone + PartialEq` that I
created and implemented for `i64` and `f64`. It's going to show up in every
single `impl` block.

### Indexing
Continuing on with this syntax-driven development, one of the simplest
operations I would like to be able to do is access elements of the matrix. Say I
have a matrix `m`. I would like to get the element at row 0, column 1 by doing
something like this:
```rust
#[test]
fn index() {
    let m = Matrix::new([
        [1, 2],
        [3, 4],
    ]);
    let element = m[(0, 1)];
    assert_eq!(element, 2);
}
```

This means we'll have to implement the `Index<(usize, usize)>` trait for our
`Matrix<T>`. However, it's not a `Matrix<T>`, is it? It's a `Matrix<T, ROWS,
COLS>`. Oh no, time for another `impl` block!
```rust
impl<T: Element, const ROWS: usize, const COLS: usize>
Index<(usize, usize)> for Matrix<T, ROWS, COLS> {
    type Output = T;

    fn index(&self, (row, col): (usize, usize)) -> &T {
        assert!(row < ROWS && col < COLS);
        &self.0[row * COLS + col]
    }
}
```
Okay, the implementation itself is straight-forward. But are all the `impl<...>`
lines going to look like that? No, no. They're going to get worse ;)

`IndexMut<(usize, usize)>` is essentially identical, but pretty important to
implement as well if we want to be able to change our matrices.

The `assert!` is pretty ugly, but I didn't want to introduce even more syntax
(there will be plenty of that) by making the indexing return an `Option<&T>`. It
also makes operations on the numbers simpler by not having to check and unwrap
them all the time, and since this is not intended to be a good, production-ready
library I will just do the easy thing here.

### Transposition
Transposing a matrix is a pretty common operation. We want to be able to reflect
it along its diagonal, swapping the rows and columns. So do we have to reach
into our underlying `Vec<T>` and shuffle it around? Remember that it is stored
in row-major order, so if the columns are now considered the rows, we have to
swap all the elements around, right?

Wrong!

Here's the sneaky little row-major vs. column-major trick: Don't touch the
underlying memory at all. Just add more information into the type system. That's
right, we're adding another const generic to our matrix type, baby!
```rust
pub struct Matrix<
    T,
    const ROWS: usize,
    const COLS: usize
    const TRANSPOSED: bool,
>(Vec<T>);
```
This means the two previous `impl` blocks have to be updated. For the one
containing the constructor, I will simply restrict it to always have this new
`bool` set to `false`, meaning all matrices start out non-transposed, in
row-major order:
```diff
impl<T: Element, const ROWS: usize, const COLS: usize>
- Matrix<T, ROWS, COLS> {
+ Matrix<T, ROWS, COLS, false> {
    pub fn new(array: [[T; COLS]; ROWS]) -> Self {
        Self(array.concat())
    }
}
```

For the `Index<(usize, usize)>` implementation, there are two possible ways to
go about it. Either you just add the same `false` as for the constructor above
and implement it again for `true` (we still want to be able to index transposed
matrices after all), or you put the checking of this bool inside the `index()`
function. Either way to go about it is fine. I'm just going to arbitrarily put
the check inside to keep the number of `impl` blocks down.
```diff
- impl<T: Element, const ROWS: usize, const COLS: usize>
- Index<(usize, usize)> for Matrix<T, ROWS, COLS> {
+ impl<T: Element, const ROWS: usize, const COLS: usize, const TRANSPOSED: bool>
+ Index<(usize, usize)> for Matrix<T, ROWS, COLS, TRANSPOSED> {
    type Output = T;

    fn index(&self, (row, col): (usize, usize)) -> &T {
        assert!(row < ROWS && col < COLS);
+       if TRANSPOSED {
+           &self.0[col * ROWS + row]
+       } else {
            &self.0[row * COLS + col]
+       }
    }
}
```
Pretty simple, just flip `col`/`COLS` with `row`/`ROWS`
[^what-about-the-assert]! Remember, the underlying `Vec` is always stored in
row-major order, so we're just pretending to actually have it be transposed to
column-major.

[^what-about-the-assert]: What about the `assert!`? Well, the `ROWS/COLS` _is_
  the `COLS/ROWS` of the non-transposed matrix. So we don't have to do the swap
  there. It's a bit confusing, but given some thought and playing around gives a
  good feeling for it.

So where's the trick? How do we actually do this pretend transposition? It's
really simple. It's just this pair of `impl` blocks [^const-return]:
```rust
impl<T: Element, const ROWS: usize, const COLS: usize>
Matrix<T, ROWS, COLS, false> {
    pub fn transpose(self) -> Matrix<T, COLS, ROWS, true> {
        Matrix(self.0)
    }
}

impl<T: Element, const ROWS: usize, const COLS: usize>
Matrix<T, ROWS, COLS, true> {
    pub fn transpose(self) -> Matrix<T, COLS, ROWS, false> {
        Matrix(self.0)
    }
}
```

[^const-return]: I wish it could be one `impl` block that returns `Matrix<T,
  COLS, ROWS, !TRANSPOSED>`. But that doesn't work for basically the same reason
  why `[T; ROWS * COLS]` doesn't.

One for each value of `TRANSPOSED`. They just return a `Matrix` with `ROWS` and
`COLS` swapped, and `TRANSPOSED` inverted. It takes ownership of `self`,
transferring ownership of the memory to the transposed matrix. No
copying/cloning at all! The memory is not touched. The underlying `Vec` points
to the exact same memory!
```rust
#[test]
fn transpose_moves() {
    let m = Matrix::new([[1, 2, 3, 4]]);
    assert_eq!(m.0, [1, 2, 3, 4]);
    assert_eq!(m[(0, 1)], 2);
    let initial_addr = m.0.as_ptr();
    let m = m.transpose();
    assert_eq!(m.0, [1, 2, 3, 4]);
    assert_eq!(m[(1, 0)], 2);
    let transposed_addr = m.0.as_ptr();
    assert_eq!(initial_addr, transposed_addr);
}
```

The test passes, meaning indexing works, and the memory is left where it is with
only ownership being transferred! Success! We can transpose matrices of any size
in constant time.

### Equality
Another thing I would like to be able to do is check whether two matrices are
equal[^float-equal]. That's easy, right? Just check if the underlying `Vec`:s are equal!
[^float-equal]: Ignore the floating point equality monster under the bed.

Not so fast.

Remember the whole thing about row-major vs. column-major? And the transposition
we just implemented? Yeah, since the transposition is pretend, and the
underlying memory is not touched, two matrices can have the same memory contents
but be unequal. Moreover, they can have different dimensions, so we shouldn't
even pass the type-check in that case. Expressed in a test, the following should
pass:
```rust
#[test]
fn equality() {
    let m1 = Matrix::new([
        [1, 2],
        [3, 4],
    ]);
    let m2 = m1.clone();
    let m3 = m1.clone().transpose();
    assert_eq!(m1, m2);
    assert_ne!(m1, m3); // asymmetric transpose
    assert_eq!(m1.0, m2.0);
    assert_eq!(m1.0, m3.0);

    let m1 = Matrix::new([
        [1, 2],
        [2, 1],
    ]);
    let m2 = m1.clone().transpose();
    assert_eq!(m1, m2); // symmetric transpose
    assert_eq!(m1.0, m2.0);
}
```

Equality is impossible for matrices of different dimensions, so this test sticks
to square matrices to prove that the asymmetric transposition equality check
should fail, even though all `Vec`:s are identical. Now, we need to implement
`PartialEq` for our matrix type. Oh no, I hear an `impl` block coming...

```rust
impl<
    T: Element,
    const ROWS: usize,
    const COLS: usize,
    const LHS_T: bool,
    const RHS_T: bool,
>
PartialEq<Matrix<T, ROWS, COLS, RHS_T>>
for Matrix<T, ROWS, COLS, LHS_T> {
    fn eq(&self, other: &Matrix<T, ROWS, COLS, RHS_T>) -> bool {
        for row in 0..ROWS {
            for col in 0..COLS {
                if self[(row, col)] != other[(row, col)] {
                    return false;
                }
            }
        }
        true
    }
}
```

That's right, this `impl`-block has _yet another const generic_. Either one of
the left and right hand sides of the equals sign could have been transposed, and
we need to make sure this is implemented for every possible combination. So the
`impl` needs another `bool` in this case. Will it ever end [^spoiler]? Other
than that, the implementation itself is very simple and utilizes the fact that
we already implemented `Index`.
[^spoiler]: Not yet.

### Element-wise addition
Now, let's actually start doing things with our matrices! How about being able
to add them? This simple test should suffice:
```rust
#[test]
fn addition() {
    let a = Matrix::new([
        [1, 2],
        [3, 4],
    ]).transpose();
    let b = Matrix::new([
        [1, 1],
        [1, 1],
    ]);
    let c = Matrix::new([
        [2, 3],
        [4, 5],
    ]).transpose();
    assert_eq!(a + b, c);
}
```

Okay, now let's brace for the `impl`...
```rust
impl<
    T: Element + Add<Output = T>,
    const ROWS: usize,
    const COLS: usize,
    const LHS_T: bool,
    const RHS_T: bool,
>
Add<Matrix<T, ROWS, COLS, RHS_T>>
for Matrix<T, ROWS, COLS, LHS_T> {
    type Output = Matrix<T, ROWS, COLS, LHS_T>;

    fn add(mut self, other: Matrix<T, ROWS, COLS, RHS_T>) -> Self::Output {
        for row in 0..ROWS {
            for col in 0..COLS {
                self[(row, col)] = self[(row, col)] + rhs[(row, col)];
            }
        }
        self
    }
}
```

Oh. That wasn't so bad. There's a new bound on `T`, it has to implement the
`Add` trait, outputting itself for the addition of the elements to work. Other
than that though, the implementation looks really similar to the one for
`PartialEq`. And in this one we're using the implementation of `IndexMut` that I
left out for brevity to assign into the left hand matrix. I only implemented the
trait for taking ownership of the added matrices, but it's straightforward to
implement for taking them by reference as well.

Well, since that wasn't so bad, I guess we've been through the worst now! The
trait `impl`:s surely won't get much more complex than this! :)

### Multiplication
Hey, did you know that matrix multiplication, unlike addition, doesn't require
the dimensions to be identical? That's right, the `impl` for `Mul` is going to
be more complex! But first, a test. Multiplication requires the left-hand side
to have as many columns as the right-hand side has rows. This means that a
matrix can always be multiplied with its transpose, which makes for a pretty
simple to write test:
```rust
#[test]
fn multiplication() {
    let a = Matrix::new([
        [1, 2, 3],
        [4, 5, 6],
    ]);
    let b = a.clone().transpose();
    let c = Matrix::new([
        [14, 32],
        [32, 77],
    ]);
    assert_eq!(a * b, c);
}
```

That should be good enough! Now we just need to `impl Mul`. And this time I will
just go through the trait bounds and const generic on their own before the
implementation, because it will be clearer that way and I want to talk about
some weird choices in the implementation. Here is the `impl<...>` part:
```rust
impl<
    T: Element + Mul<Output = T> + Add<Output = T>,
    const ROWS: usize,
    const COLS: usize,
    const MATCHING_DIM: usize,
    const LHS_T: bool,
    const RHS_T: bool,
>
```
Phew. Now, it's the most complex one in this post, but it's not _that_ bad. We
can see that `T` has a new `Mul` trait bound, but it also has the `Add` trait
bound. This is because matrix multiplication is defined by a bunch of
multiplications and additions, so both these bounds must be present. And then we
have _yet another const generic_. Last one, I promise. As I said before, the
left-hand side of the multiplication must have as many columns as the right-hand
side has rows. This is what `MATCHING_DIM` will represent. The remaining two
dimensions of the left and right side are arbitrary, but will become the
dimensions of the output matrix. So multiplying a 2x3 matrix with a 3x4 matrix
will yield a 2x4 matrix.

Now, before I show you the implementation, I have to warn you that it's a little
strange. It looks like this purely because I'm a nerd and liked the idea of it.
The implementation makes the following (non-)assumptions of the underlying type
`T` in our matrix:

1. `T` might not be numerical.
2. `T` might not have a multiplicative or additive identity. [^identity]
3. `T` might not `impl Default`.
[^identity]: I wish Rust had this, maybe as a trait of some kind. It would be
  very simple to implement myself, but I liked being more general for the fun of
  it in this case.

These are implicit in the trait bounds I have chosen for `T`, but I figured I
would make them explicit.

Now, on to the implementation (`impl` line from above excluded):
```rust
Mul<Matrix<T, MATCHING_DIM, COLS, RHS_T>>
for Matrix<T, ROWS, MATCHING_DIM, LHS_T> {
    type Output = Matrix<T, ROWS, COLS, LHS_T>;

    fn mul(self, rhs: Matrix<T, MATCHING_DIM, COLS, RHS_T>) -> Self::Output {
        let mut vec = Vec::with_capacity(ROWS * COLS);

        unsafe {
            vec.set_len(vec.capacity());
        }

        for row in 0..ROWS {
            for col in 0..COLS {
                let mut products = Vec::with_capacity(MATCHING_DIM);
                for i in 0..MATCHING_DIM {
                    products.push(
                        self[(row, i)] * rhs[(i, col)]
                    );
                }
                vec[row * COLS + col] = products
                    .iter()
                    .skip(1)
                    .fold(products[0], |acc, x| acc + *x);
            }
        }
        Matrix(vec)
    }

}
```

Whoa, is that an `unsafe` block?? Yes, but don't worry. It's fine. Really, I
promise [^trust].  Mathematically, the resulting matrix is _guaranteed_ to have
`ROWS * COLS` elements in it. And to avoid having a bunch of reallocations, I
initialize the matrix to have this capacity. But if I don't set the length, I
can't assign to indices in the `Vec`. Rust will `panic!` if I try. So into
`unsafe` land we go, and we pinky-promise Rust that the `Vec` does indeed have
the same length as it has capacity. So indexing into it and assigning things
there is fine. Of course, in reality this is uninitialized memory. We are just
manually initializing it.
[^trust]: Source: trust me bro

"Why not just call `Vec::resize` instead?", you may ask. To which I reply "And
fill it with what?". `Vec::resize` takes an element to pad the `Vec` with to the
desired length. And remember point 3 from above. `T` might not `impl Default`,
so what should we put there? Better to just leave it uninitialized since we're
immediately going to initialize it anyway.

The rest of the implementation is relatively straighforward. Each element in the
output matrix is a sum of products from the input matrices, so the `products`
vector is self-explanatory. But why am I summing `products` like that? Why not
just call `.sum()` or `.fold()` directly on the iterator? Well, that all comes
back to point 2 from above. `T` might not have a multiplicative or additive
identity. So we skip the first element, use that as the initial value for the
accumulator in `.fold()`, and do a folding addition like normal. This
implementation requires the dimensions to be non-zero, but we can just skip that
check for brevity [^brevity].
[^brevity]: "For brevity", he said. In the almost 600 line, 3700 word markdown
  document.

If I wanted to use `.sum()`, I could constrain `T` to require an additive
identity by adding another trait bound requiring `T` to implement
`std::iter::Sum`. If we also require `std::iter::Product`, we could force a
multiplicative identity to exist, and that could simplify the innermost loop of
the implementation. But again, I wanted to keep this general for fun.

And there we have it! A very simple linear algebra library. Or rather, a simple
matrix addition and multiplication library. But other operations are now pretty
trivial to implement and build on top of this foundation.

## Conclusion
This is not how you should do things. All the const generics and trait bounds
make it really hard to read and implement anything, and trying to stay too
general will result in strange implementations. But boy, is it fun!

The real libraries do not use const generics and implement this stuff in much
smarter ways in order to actually be useful at runtime. This was all just a fun
excursion into const generic land in order to learn and see how far I could take
it. If nothing else, it was a fun way to spend an evening.

## Use cases?
Very few. As mentioned, all matrix dimensions are created and checked at
compile-time, so any real world use for a library like this would be pretty
limited.

## Further work
Absolutely not. But if one wanted to expand on this, implementing subtraction,
inverses, convolutions, views, scalar multiplication, and all manner of fun
things should in theory be possible. Don't blame me for any loss of sanity if
you try, though. Keeping all these trait bounds and const generics in check
takes a good bit of care and energy.
