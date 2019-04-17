- Feature Name: fadvise-unix-extension
- Start Date: 2019-04-16
<!-- - RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000) -->
<!-- - Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000) -->

# Summary
[summary]: #summary

Add a method to `std::os::unix::fs::FileExt` exposing the `posix_fadvise()` Unix systemcall.

# Motivation
[motivation]: #motivation

Working with raw file descriptors is clunky, and makes for ugly code. What I propose
is an intuitive interface to the `posix_fadvise()` systemcall, without the need
of working with raw file descriptors.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## What is `posix_fadvise` anyway?

Using the `posix_fadvise()` system call, programs can announce an intention to
access file data in a specific pattern in the future, thus allowing the kernel to
perform appropriate optimizations not possible otherwise.

In the C standard library, `posix_favise()` is included in `fcntl.h` and takes
the following arguments:
```c
int posix_fadvise(int fd, off_t offset, off_t len, int advice);
```
Quoting the Linux man pages:[1](https://linux.die.net/man/2/posix_fadvise)
> The advice applies to a (not necessarily existent) region starting at offset and extending for len bytes (or until the end of the file if len is 0) within the file referred to by fd. The advice is not binding; it merely constitutes an expectation on behalf of the application.

> Permissible values for advice include:
> * POSIX_FADV_NORMAL: Indicates that the application has no advice to give about its access pattern for the specified data. If no advice is given for an open file, this is the default assumption.
> * POSIX_FADV_SEQUENTIAL: The application expects to access the specified data sequentially (with lower offsets read before higher ones).
> * POSIX_FADV_RANDOM: The specified data will be accessed in random order.
> * POSIX_FADV_NOREUSE: The specified data will be accessed only once.
> * POSIX_FADV_WILLNEED: The specified data will be accessed in the near future.
> * POSIX_FADV_DONTNEED: The specified data will not be accessed in the near future.

## How it would work:

We'd have two new methods, `my_file.advise(advice)`, and
`my_file.advise_at(offset, len, advice)`. The former is the same as the latter,
except the offset and length are zero, meaning the advice will apply to the entirety
of the file.

### Example usage:
The usage is pretty self explainatory.
```rust
fn my_function(file: &mut File) {
    file.advise(Advice::Sequential);
    read_file_sequentially(file);
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation
We'd have two new methods in `std::sys::unix::ext::fs::FileExt` and a new
enum, looking something like this:

```rust
enum Advice {
    Normal,
    Sequential,
    Random,
    NoReuse,
    WillNeed,
    DontNeed,
}
```

```rust
fn advise(&self, advice: Advice) -> Error {
    self.advise_at(0, 0, advice)
}

fn advise(&) -> Error {
    // ...
}
```

The `advise` method would then be implemented using `libc::posix_fadvise()`.

# Drawbacks
[drawbacks]: #drawbacks

Unlike in C, the system call is not widely used in Rust, but that may simply be
the lack of a good interface. Please mention any other potential drawbacks in
a comment.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Why two methods instead of one?
As will be discussed below, most of the time, most of the time you'll want the
advice to apply to the entirety of the file, not just a portion of it — that is,
the `offset`, and `len` parameters will both be `0`. This is why I think that
should be the default behaviour and offer a seperate `advise_at` method.

# Prior art
[prior-art]: #prior-art
In GNU Coreutils, the systemcall is widely used, namely in: `sum`, `md5sum`,
`nl`, `fold`, `paste`, `cat`, `basenc`, `wc`, `copy`, `comm`, `join`, `shuf`,
`fmt`, `dd`, `ptx`, `tr`, `cut`, `sort`, `expand`, `tsort`, `pr`, `cksum`,
`uniq`, `tee`. Since their use of the system call is mostly on an entire file,
they — using a wrapping function — created the same bisection as discussed above.
[2](http://git.savannah.gnu.org/cgit/coreutils.git/tree/gl/lib/fadvise.h)

# Unresolved questions
[unresolved-questions]: #unresolved-questions

To me, it's pretty clear we should place the methods in
`std::sys::unix::ext::fs::FileExt`, but it's not as clear where to put the enum.

# Future possibilities
[future-possibilities]: #future-possibilities
I can't think of anything right now.
