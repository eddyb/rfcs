- Start Date: 2015-01-25
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary

Remove `static mut` and replace uses with `static` in combination with interior
mutability, such as atomics, or `UnsafeCell` directly.

# Motivation

Ever since `static` gained the ability to contain interior mutability, which
would place it into writable static memory, as opposed to always read-only,
`static mut` has become redundant with it.

There is also an inconsistency around `&mut` borrows, they are never allowed
in a global, except for `&mut [...]` in `static mut`, to support holding an
array in a `static mut` without specifying the length in the type.

Last, but not least, `static mut` is an unsafety trap. It is too easy to end
up with aliasing `&mut` borrows from it or have unsynchronized reads/writes
in a multi-threaded applications, both of which are undefined behavior.
Keeping a feature that is more unsafe than one would initially assume, and
which we see people abuse all the time, could turn out to a mistake in the
long run, and no doubt some may even call it an irresponsible move.

# Detailed design

Remove `static mut` and require all users to move to `static` combined with
interior mutability:
```rust
static mut FOO: Foo = Foo { x: 0, y: 1 };
/* very */ unsafe { FOO.x += FOO.y; }

// becomes (first approximation)

// somewhere in libstd:
struct RacyCell<T>(pub UnsafeCell<T>);
unsafe impl<T> Sync for RacyCell<T> {}
impl<T> RacyCell<T> { pub fn get(&self) -> *T { self.0.get() } }
macro_rules! racy_cell {
    ($x:expr) => (RacyCell(UnsafeCell { value: $x }))
}

static FOO: RacyCell<Foo> = racy_cell!(Foo { x: 0, y: 1 });
unsafe {
    (*FOO.get()).x += (*FOO.get()).y;
}
```

In some cases, the replacement is entirely safe:
```rust
static mut COUNTER: usize = 0;
unsafe { COUNTER += 1; }
// can be replaced with:
static COUNTER: AtomicUsize = ATOMIC_USIZE_INIT;
COUNTER.fetch_add(1, Ordering::SeqCst);
```

There are also designs of containers which are unlocked with an entirely safe
and zero-cost proof that interrupts have been disabled, being used in kernels
written in Rust. It works something like this:
```rust
struct InterruptGuarantor<'a> { marker: InvariantLifetime<'a> }
struct InterruptLocked<T> { value: T }
unsafe impl<T> Sync for InterruptLocked<T> {}
impl<'a, T> Index<InterruptGuarantor<'a>> InterruptLocked<T> {
    fn index<'b>(&'b self, _: &InterruptGuarantor<'a>) -> &'b T {
        &self.value
    }
}

static COUNTER: InterruptLocked<Cell<usize>> = InterruptLocked {
    value: CELL_USIZE_ZERO // pretending this exists
};
// assembly could call this function as if it had no arguments:
fn timer_interrupt_handler(guarantor: InterruptGuarantor) {
    COUNTER[guarantor].set(COUNTER[guarantor].get() + 1);
}
```

# Drawbacks

Certain cases would be more verbose, though this would force users to consider
moving away from globals where possible, and prevent new uses out of habit,
which tend to become (more or less) subtly unsafe.

Initializing globals is still problematic, as we don't have an UFCS story.
As such, we can't use `Mutex` or `RefCell` (the latter in a `#[thread_local]`
`static` only) at all right now. However, this isn't a downgrade from the
status quo, just an impediment in making even more usecases safe.

# Alternatives

Don't do anything, keep `static mut` in 1.0, alongside `const` and `static`.

Feature-gate `static mut` to allow its removal after 1.0.

# Unresolved questions

We need a better story for creating values of encapsulated types. This would
allow `static` globals to contain atomics with arbitrary initial values, like
`Mutex`, `RwLock` - or in the case of `#[thread_local]` (or other abstractions
like the interrupt-based one above) - `Cell` and `RefCell`.

Could we have a simplistic CTFE system with `const` functions and leave the
`trait`/`impl` interactions to be resolved at a later time?
Associated constants could also provide similar functionality, albeit less
clean and compact.
I should point out that this can be implemented rather easily, as long as they
are not true type-level constants, e.g. `[u8; pow(2, 12)]` is disallowed.
