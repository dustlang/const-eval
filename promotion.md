# Const promotion

"Promotion" is the act of splicing a part of a MIR computation out into a
separate self-contained MIR body which is evaluated at compile-time like a
constant.

## Promotion contexts

There are a few different contexts where promotion is beneficial.

### Lifetime extension

"Lifetime extension" is a mechanism that affects code like `&3`:
Instead of putting it on the stack, the `3` is allocated in global static memory
and a reference with lifetime `'static` is provided.  This is essentially an
automatic transformation turning `&EXPR` into `{ const _PROMOTED = &EXPR;
_PROMOTED }`, but only if `EXPR` qualifies. Topmost projections are not
promoted, so `&EXPR.proj1.proj2` turns into `{ const _PROMOTED = &EXPR;
&(*_PROMOTED).proj1.proj2 }`.

Note that promotion happens on the MIR, not on surface-level syntax.  This is
relevant when discussing e.g. handling of panics caused by overflowing
arithmetic.

Lifetime extension is described in [RFC 1414][promotion-rfc]. The RFC uses the
word "promotion" to refer exclusively to lifetime extension, since this was the
first context where promotion was done.

[promotion-rfc]: https://github.com/rust-lang/rfcs/blob/master/text/1414-rvalue_static_promotion.md

### Non-`Copy` array initialization

Another promotion context, the initializer of an array expression, was
introduced in [RFC 2203][]. Here, promotion allows arrays of
non-`Copy` types to be initialized idiomatically, for example
`[Option::<Box<i32>>::None; 32]`.

[RFC 2203]: https://github.com/rust-lang/rfcs/blob/master/text/2203-const-repeat-expr.md

### `#[rustc_args_required_const(...)]` and inline assembly `const` operands

Additionally, some platform intrinsics require certain parameters to be
immediates (known at compile-time). We use the `#[rustc_args_required_const]`
attribute, introduced in
[rust-lang/rust#48018](https://github.com/rust-lang/rust/pull/48018), to
specify these parameters and (aggressively, see below) try to promote the
corresponding arguments.

Similarly, inline assembly has `const` operands, which are treated the same way
as `rustc_args_required_const` arguments.

## Implicit and explicit promotion

On top of what applies to [consts](const.md), promoteds suffer from the additional issue that *the user did not ask for them to be evaluated at compile-time*.
Thus, if CTFE fails but the code would have worked fine at run-time, we broke the user's code for no good reason.
Even if we are sure we found an error in the user's code, we are only allowed to [emit a warning, not a hard error][warn-rfc].
We call this *implicit* promotion, and we have to be very conservative with what can and cannot be implicitly promoted.

CTFE of implicitly promoted code must never fail to evaluate except if the
run-time code also would have failed. This means we cannot permit calling
arbitrary `const fn`, as discussed in detail in
[rust-lang/const-eval#19](https://github.com/rust-lang/const-eval/issues/19).
Thus, only functions marked `#[rustc_promotable]` are implicitly promotable (see
below).

On the other hand, when a user passes an expression to a function with
`#[rustc_args_required_const]`, the only way for this code to compile is to
promote it.  In that sense, the user is explicitly asking for that expression to
be evaluated at compile-time even though they have not written it in a `const`
declaration. We can thus be less conservative. This is called *explicit*
promotion.

Currently, the following are considered explicit promotion contexts:
* `#[rustc_args_required_const]` arguments and inline assembly `const` operands everywhere.
* Everything inside the bodies of `const` and `static` items. (Note: this is handled separately from "explicit contexts" in promotion analysis, but the effect is the same.
The arguments given above for justifying explicit promotion do not apply here. Currently, this works out because failing to evaluate one of these promoteds just leads to a warning, but longer-term it would be desirable to turn evaluation failures into hard errors, which for these promoteds means we have to guarantee that we only evaluate them on-demand.)

In these contexts, we promote calls to arbitrary `const fn`.

[See below][static access] for another special case in promotion analysis:
accesses and references to statics are only promoted inside other statics.

[warn-rfc]: https://github.com/rust-lang/rfcs/blob/master/text/1229-compile-time-asserts.md

## "enclosing scope" rule

Notice that some code involving `&` *looks* like it relies on lifetime
extension but actually does not:

```rust
const EMPTY_BYTES: &Vec<u8> = &Vec::new(); // Ok without lifetime extension
```

`Vec::new()` cannot get promoted because it needs dropping. And yet this
compiles. Why that? The reason is that the reference obtains the lifetime of
the "enclosing scope", similar to how `let x = &mut x;` creates a reference
whose lifetime lasts for the enclosing scope. This is decided during MIR
building already, and does not involve lifetime extension.

In contrast, this does not compile:

```rust
const OPT_EMPTY_BYTES: Option<&Vec<u8>> = Some(&Vec::new());
```

The "enclosing scope" rule only fires for outermost `&`, just like in `fn` bodies.

## Promotability

We have described the circumstances where promotion is desirable, but what
expressions are actually eligible for promotion? We refer to eligible
expressions as "promotable" and describe the restrictions on such expressions
below.

First of all, expressions have to be [allowed in constants](const.md). The
restrictions described there are needed because we want `const` to behave the
same as copying the `const` initializer everywhere the constant is used; we need
the same property when promoting expressions. But we need more.

Note that there is no point in doing additional dynamic checks here.  The entire point of
the promotion restrictions is to avoid failing compilation for code that would
have been fine without promotion.  The best a dynamic check could do is tell us
after the fact that we should not have promoted something, but then it is
already too late -- and the dynamic checks for that are exactly the ones we are
already doing for constants and statics.

### Panics

Promotion is not allowed to throw away side effects.  This includes panicking.
Let us look at what happens when we promote `&(0_usize - 1)` in a debug build:
We have to avoid erroring at compile-time, because that would be promotion
breaking compilation, but we must be sure to error correctly at run-time.  In
the MIR, this looks roughly like

```
_tmp1 = CheckedSub (const 0usize) (const 1usize)
assert(!_tmp1.1) -> [success: bb2; unwind: ..]

bb2:
_tmp2 = tmp1.0
_res = &_tmp2
```

Both `_tmp1` and `_tmp2` are promoted.  `_tmp1` evaluates to `(~0, true)`, so
the assertion will always fail at run-time.  Computing `_tmp2` fails with a
panic, which is thrown away -- so we have no result.  In principle, we could
generate any code for this because we know the code is unreachable (the
assertion is going to fail).  Just to be safe, we generate a call to
`llvm.trap`.

As long as CTFE only panics when run-time code would also have panicked, this
works out correctly: The MIR already contains provisions for what to do on
panics (unwind edges etc.), so when CTFE panics we can generate code that
hard-codes a panic to happen at run-time.  In other words, *promotion relies on
CTFE correctly implementing both normal program behavior and panics*.  An
earlier version of miri used to panic on arithmetic overflow even in release
mode.  This breaks promotion, because now promoting code that would work (and
could not panic!) at run-time leads to a compile-time CTFE error.

### Const safety

We have explained what happens when evaluating a promoted panics, but what about
other kinds of failure -- what about hitting an unsupported operation or
undefined behavior?  To make sure this does not happen, only const safe code
gets promoted. The exact details for `const safety` are discussed in
[here](const_safety.md).

An example of this would be `&(&1 as *const i32 as usize % 16 == 0)`. The actual
location is not known at compile-time, so we cannot promote this.  Generally, we
can guarantee const-safety by not promoting when an unsafe or unconst operation
is performed -- if our const safety checker is correct, that has to cover
everything, so the only possible remaining failure are panics.

However, things get more tricky when `const` and `const fn` are involved.

For `const`, based on the const safety check described [here](const_safety.md#const-safety-check-on-values),
we can rely on there not being const-unsafe values in the `const`, so we should
be able to promote freely.  For example:

```rust
union Foo { x: &'static i32, y: usize }
const A: usize = unsafe { Foo { x: &1 }.y };
const B: usize = unsafe { Foo { x: &2 }.y };
let x: &bool = &(A < B);
```

Promoting `x` would lead to a compile failure because we cannot compare pointer
addresses.  However, we do not even get there -- computing `A` or `B` fails with
a const safety check error because these are values of type `usize` that contain
a `Scalar::Ptr`.

For `const fn`, however, there is no way to check anything in advance.  We can
either just not promote, or we can move responsibility to the `const fn` and
promote *if* all function arguments pass the const safety check.  So,
`foo(42usize)` would get promoted, but `foo(&1 as *const i32 as usize)` would
not.  When this call panics, compilation proceeds and we just hard-code a panic
to happen as well at run-time.  However, when const evaluation fails with
another error (unsupported operation or undefined behavior), we have no choice
but to abort compilation of a program that would have compiled fine if we would
not have decided to promote.  It is the responsibility of `foo` to not fail this
way when working with const-safe arguments.

For this reason, only `const fn` that were explicitly marked with the
`#[rustc_promotable]` attribute are subject to promotion. Those functions must
be manually reviewed to never raise CTFE errors.

### Drop

Expressions returning "needs drop" types can never be promoted. If such an
expression were promoted, the `Drop` impl would never get called on the value,
even though the user did not explicitly request such behavior by using an
explicit `const` or `static` item.

As expression promotion is essentially the silent insertion of a `static` item, and
`static` items never have their `Drop` impl called, the `Drop` impl of the promoted
value would never get called.

While it is sound to `std::mem::forget` any value and thus not call its `Drop` impl,
it is unlikely to be the desired behavior in most cases and very likey to be confusing
to the user. If such behavior is desired, the user can still use an explicit `static`
or `const` item and refer to that.

*Dynamic check.* The Miri engine could dynamically check this by ensuring that
the result of computing a promoted is a value that does not need dropping.

### Access to a `const` or `static`
[access-static]: #access-to-a-const-or-static

When accessing a `const` in a promotable context, its value gets computed
at compile-time anyway, so we do not have to check the initializer.  However, the
restrictions described above still apply for the *result* of the promoted
computation: in particular, it must be a valid `const` (i.e., it cannot
introduce interior mutability) and it must not require dropping.

For instance the following would be legal even though calls to `do_it` are not
eligible for implicit promotion:

```rust
const fn do_it(x: i32) -> i32 { 2*x }
const ANSWER: i32 = {
  let ret = do_it(21);
  ret
};

let x: &'static i32 = &ANSWER;
```

An access to a `static`, including just taking references to a `static`, is only
promotable within the initializer of another `static`. This is for the same
reason that `const` initializers
[cannot access statics](const.md#reading-statics).

Crucially, however, the following is *not* legal:

```rust
const X: Cell<i32> = Cell::new(5); // ok
const XREF: &Cell<i32> = &X; // not ok
fn main() {
    let x: &'static _ = &X; // not ok
}
```

Just like allowing `XREF` would be a problem because, by the inlining semantics,
every user of `XREF` should get their own `Cell`; it would also be a problem to
promote here because if that code gets executed multiple times (e.g. inside a
loop), it should get a new `Cell` each time.

### Named locals

Promotable expressions cannot refer to named locals. This is not a technical
limitation with the CTFE engine. While writing `let x = {expr}` outside of a
const context, the user likely expects that `x` will live on the stack and be
initialized at run-time.  Although this is not (to my knowledge) guaranteed by
the language, we do not wish to violate the user's expectations here.

Note that constant-folding still applies: the optimizer may compute `x` at
compile-time and even inline it everywhere if it can show that this does not
observably alter program behavior.  Promotion is very different from
constant-folding as promotion can introduce observable differences in behavior
(if const-evaluation fails) and as it is *guaranteed* to happen in some cases
(and thus exploited by the borrow checker).  This is reflected in the fact that
promotion affects lifetimes, but constant folding does not.

### Single assignment

We only promote temporaries that are assigned to exactly once. For example, the
lifetime of the temporary whose reference is assigned to `x` below will not be
extended.

```rust
let x: &'static i32 = &if cfg!(windows) { 0 } else { 1 };
```

Once again, this is not a fundamental limitation in the CTFE engine; we are
perfectly capable of evaluating such expressions at compile time. However,
determining the promotability of complex expressions would require more
resources for little benefit.

## Open questions

* There is a fourth kind of CTFE failure -- resource exhaustion.  What do we do
  when that happens while evaluating a promoted?
