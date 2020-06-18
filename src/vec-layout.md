# Layout

First off, we need to come up with the struct layout. A Vec has three parts:
a pointer to the allocation, the size of the allocation, and the number of
elements that have been initialized.

Naively, this means we just want this design:

```rust
pub struct Vec<T> {
    ptr: *mut T,
    cap: usize,
    len: usize,
}
# fn main() {}
```

And indeed this would compile. Unfortunately, it would be incorrect. First, the
compiler will give us too strict variance. So a `&Vec<&'static str>`
couldn't be used where an `&Vec<&'a str>` was expected. More importantly, it
will give incorrect ownership information to the drop checker, as it will
conservatively assume we don't own any values of type `T`. See [the chapter
on ownership and lifetimes][ownership] for all the details on variance and
drop check.

As we saw in the ownership chapter, we should use `Unique<T>` in place of
`*mut T` when we have a raw pointer to an allocation we own. Unique is unstable,
so we'd like to not use it if possible, though.

As a recap, Unique is a wrapper around a raw pointer that declares that:

* We are covariant over `T`
* We may own a value of type `T` (for drop check)
* We are Send/Sync if `T` is Send/Sync
* Our pointer is never null (so `Option<Vec<T>>` is null-pointer-optimized)

We can implement all of the above requirements in stable Rust. To do this, instead of using `Unique<T>` we will use [`NonNull<T>`](https://doc.rust-lang.org/stable/std/ptr/struct.NonNull.html), another wrapper around a raw pointer, which gives us two of the above properties, namely it is covariant over `T` and is declared to never be null. By
adding a `PhantomData<T>` (for drop check) and implementing Send/Sync if `T` is, we then get the same results as if we had used `Unique<T>`:

```rust
pub struct Vec<T> {
    ptr: NonNull<T>,
    cap: usize,
    len: usize,
    _marker: PhantomData<T>,
}

unsafe impl<T: Send> Send for Vec<T> {}
unsafe impl<T: Sync> Sync for Vec<T> {}
```

[ownership]: ownership.html
