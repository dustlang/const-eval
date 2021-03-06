# Const safety

The miri engine, which is used to execute code at compile time, can fail in
four possible ways:

* The program performs an unsupported operation (e.g., calling an unimplemented
  intrinsics, or doing an operation that would observe the integer address of a
  pointer).
* The program causes undefined behavior (e.g., dereferencing an out-of-bounds
  pointer).
* The program panics (e.g., a failed bounds check).
* The program exhausts its resources: It might overflow the stack, allocation
  too much memory or loops forever.  Note that detecting these conditions
  happens on a best-effort basis only.

Just like panics and non-termination are acceptable in safe run-time Rust code,
we also consider these acceptable in safe compile-time Rust code.  However, we
would like to rule out the first two kinds of failures in safe code.  Following
the terminology in [this blog post], we call a program that does not fail in the
first two ways *const safe*.

[this blog post]: https://www.ralfj.de/blog/2018/07/19/const.html

The goal of the const safety check, then, is to ensure that a program is const
safe.  What makes this tricky is that there are some operations that are safe as
far as run-time Rust is concerned, but unsupported in the miri engine and hence
not const safe (they fall in the first category of failures above).  We call these operations *unconst*.  The purpose
of the following section is to explain this in more detail, before proceeding
with the main definitions.

## Miri background

A very simple example of an unconst operation is
```rust
static S:i32 = 0;
const BAD:bool = (&S as *const i32 as usize) % 16 == 0;
```
The modulo operation here is not supported by the miri engine because evaluating
it requires knowing the actual integer address of `S`.

The way miri handles this is by treating pointer and integer values separately.
The most primitive kind of value in miri is a `Scalar`, and a scalar is *either*
a pointer (`Scalar::Ptr`) or a bunch of bits representing an integer
(`Scalar::Bits`).  Every value of a variable of primitive type is stored as a
`Scalar`.  In the code above, casting the pointer `&S` to `*const i32` and then
to `usize` does not actually change the value -- we end up with a local variable
of type `usize` whose value is a `Scalar::Ptr`.  This is not a problem in
itself, but then executing `%` on this *pointer value* is unsupported.

However, it does not seem appropriate to blame the `%` operation above for this
failure. `%` on "normal" `usize` values (`Scalar::Bits`) is perfectly fine, just using it on
values computed from pointers is an issue.  Essentially, `&i32 as *const i32 as
usize` is a "safe" `usize` at run-time (meaning that applying safe operations to
this `usize` cannot lead to misbehavior, following terminology [suggested here])
-- but the same value is *not* "safe" at compile-time, because we can cause a
const safety violation by applying a safe operation (namely, `%`).

[suggested here]: https://www.ralfj.de/blog/2018/08/22/two-kinds-of-invariants.html

## Const safety check on values

The result of any const computation (`const`, `static`, promoteds) is subject to
a "sanity check" which enforces const safety.  (A sanity check is already
happening, but it is not exactly checking const safety currently.)  Const safety
is defined as follows:

* Integer and floating point types are const-safe if they are a `Scalar::Bits`.
  This makes sure that we can run `%` and other operations without violating
  const safety.  In particular, the value must *not* be uninitialized.
* References are const-safe if they are `Scalar::Ptr` into allocated memory, and
  the data stored there is const-safe.  (Technically, we would also like to
  require `&mut` to be unique and `&` to not be mutable unless there is an
  `UnsafeCell`, but it seems infeasible to check that.)  For fat pointers, the
  length of a slice must be a valid `usize` and the vtable of a `dyn Trait` must
  be a valid vtable.
* `bool` is const-safe if it is `Scalar::Bits` with a value of `0` or `1`.
* `char` is const-safe if it is a valid unicode codepoint.
* `()` is always const-safe.
* `!` is never const-safe.
* Tuples, structs, arrays and slices are const-safe if all their fields are
  const-safe.
* Enums are const-safe if they have a valid discriminant and the fields of the
  active variant are const-safe.
* Unions are always const-safe; the data does not matter.
* `dyn Trait` is const-safe if the value is const-safe at the type indicated by
  the vtable.
* Function pointers are const-safe if they point to an actual function.  A
  `const fn` pointer (when/if we have those) must point to a `const fn`.

For example:
```rust
static S: i32 = 0;
const BAD: usize = &S as *const i32 as usize;
```
Here, `S` is const-safe because `0` is a `Scalar::Bits`.  However, `BAD` is *not* const-safe because it is a `Scalar::Ptr`.

## Const safety check on code

The purpose of the const safety check on code is to prohibit construction of
non-const-safe values in safe code.  We can allow *almost* all safe operations,
except for unconst operations -- which are all related to raw pointers:
Comparing raw pointers for (in)equality, converting them to integers, hashing
them (including hashing references) and so on must be prohibited.  Basically, we
should not permit any raw pointer operations to begin with, and carefully
evaluate any that we permit to make sure they are fully supported by miri and do
not permit constructing non-const-safe values.

There should also be a mechanism akin to `unsafe` blocks to opt-in to using
unconst operations.  At this point, it becomes the responsibility of the
programmer to preserve const safety.  In particular, a *safe* `const fn` must
always execute const-safely when called with const-safe arguments, and produce a
const-safe result.  For example, the following function is const-safe (after
some extensions of the miri engine that are already implemented in miri) even
though it uses raw pointer operations:
```rust
const fn test_eq<T>(x: &T, y: &T) -> bool {
    x as *const _ == y as *const _
}
```
On the other hand, the following function is *not* const-safe and hence it is considered a bug to mark it as such:
```
const fn convert<T>(x: &T) -> usize {
  x as *const _ as usize
}
```

## Open questions

* Do we allow unconst operations in `unsafe` blocks, or do we have some other
  mechanism for opting in to them (like `unconst` blocks)?

* How do we communicate that the rules for safe `const fn` using unsafe code are
  different than the ones for "runtime" functions?  The good news here is that
  violating the rules, at worst, leads to a compile-time error in a dependency.
  No UB can arise.  However, thanks to [promotion](promotion.md), compile-time
  errors can arise even if no `const` or `static` is involved.
