- Feature Name: osstring_string_interface
- Start Date: 2015-10-05
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary

Add a string-like API to the `OsString` and `OsStr` types.  This RFC
focuses on creating a string-like interface, as opposed to RFC #1307,
which focuses more on container-like features.

# Motivation

As mentioned in the `std::ffi::os_str` documentation: "**Note**: At
the moment, these types are extremely bare-bones, usable only for
conversion to/from various other string types. Eventually these types
will offer a full-fledged string API."  This is intended as a step in
that direction.

Having an ergonomic way to manipulate OS strings is needed to allow
programs to easily handle non-UTF-8 data received from the operating
system.  Currently, it is common for programs to just convert OS data
to `String`s, which leads to undesirable panics in the unusual case
where the input is not UTF-8.  For example, currently, calling rustc
with a non-UTF-8 command line argument will result in an immediate
panic.  Fixing that in a way that actually handles non-UTF-8 data
correctly (as opposed to, for example, just interpreting it lossily as
UTF-8) would be very difficult with the current OS string API.  Most
of the functions proposed here were motivated by the OS string
processing needs of rustc.

# Detailed design

## `OsString`

`OsString` will get the following new method:
```rust
/// Converts an `OsString` into a `String`, avoiding a copy if possible.
///
/// Any non-Unicode sequences are replaced with U+FFFD REPLACEMENT CHARACTER.
pub fn into_string_lossy(self) -> String;

```

This is analogous to the existing `OsStr::to_string_lossy` method, but
transfers ownership.  This operation can be done without a copy if the
`OsString` contains UTF-8 data or if the platform is Windows.

## `OsStr`

OsStr will get the following new methods:
```rust
/// Returns true if `needle` is a substring of `self`.
fn contains_os<S: AsRef<OsStr>>(&self, needle: S) -> bool;

/// Returns true if `needle` is a prefix of `self`.
fn starts_with_os<S: AsRef<OsStr>>(&self, needle: S) -> bool;

/// Returns true if `needle` is a suffix of `self`.
fn ends_with_os<S: AsRef<OsStr>>(&self, needle: S) -> bool;

use std::str::pattern::{DoubleEndedSearcher, Pattern, ReverseSearcher};

/// Returns true if `self` matches `pat`.
///
/// Note that patterns can only match UTF-8 sections of the `OsStr`.
fn contains<'a, P>(&'a self, pat: P) -> bool where P: Pattern<'a> + Clone;

/// Returns true if the beginning of `self` matches `pat`.
///
/// Note that patterns can only match UTF-8 sections of the `OsStr`.
fn starts_with<'a, P>(&'a self, pat: P) -> bool where P: Pattern<'a>;

/// Returns true if the end of `self` matches `pat`.
///
/// Note that patterns can only match UTF-8 sections of the `OsStr`.
fn ends_with<'a, P>(&'a self, pat: P) -> bool
    where P: Pattern<'a>, P::Searcher: ReverseSearcher<'a>;

/// An iterator over substrings of `self` separated by characters
/// matched by a pattern.  See `str::split` for details.
///
/// Note that patterns can only match UTF-8 sections of the `OsStr`.
fn split<'a, P>(&'a self, pat: P) -> Split<'a, P> where P: Pattern<'a>;

struct Split<'a, P> where P: Pattern<'a> { ... }
impl<'a, P> Clone for Split<'a, P> where P: Pattern<'a> + Clone, P::Searcher: Clone { ... }
impl<'a, P> Iterator for Split<'a, P> where P: Pattern<'a> + Clone {
    type Item = &'a OsStr;
    ...
}
impl<'a, P> DoubleEndedIterator for Split<'a, P>
    where P: Pattern<'a> + Clone, P::Searcher: DoubleEndedSearcher<'a> { ... }

/// An iterator over substrings of `self` separated by characters
/// matched by a pattern, in reverse order.  See `str::rsplit` for
/// details.
///
/// Note that patterns can only match UTF-8 sections of the `OsStr`.
fn rsplit<'a, P>(&'a self, pat: P) -> RSplit<'a, P> where P: Pattern<'a>;

struct RSplit<'a, P> where P: Pattern<'a> { ... }
impl<'a, P> Clone for RSplit<'a, P> where P: Pattern<'a> + Clone, P::Searcher: Clone { ... }
impl<'a, P> Iterator for RSplit<'a, P>
    where P: Pattern<'a> + Clone, P::Searcher: ReverseSearcher<'a> {
    type Item = &'a OsStr;
    ...
}
impl<'a, P> DoubleEndedIterator for RSplit<'a, P>
    where P: Pattern<'a> + Clone, P::Searcher: DoubleEndedSearcher<'a> { ... }

/// Returns true if the string starts with a valid UTF-8 sequence
/// equal to the given `&str`.
fn starts_with_str(&self, prefix: &str) -> bool;

/// If the string starts with the given `&str`, returns the rest
/// of the string.  Otherwise returns `None`.
fn remove_prefix_str(&self, prefix: &str) -> Option<&OsStr>;

/// Retrieves the first character from the `OsStr` and returns it
/// and the remainder of the `OsStr`.  Returns `None` if the
/// `OsStr` does not start with a character (either because it it
/// empty or because it starts with non-UTF-8 data).
fn slice_shift_char(&self) -> Option<(char, &OsStr)>;

/// If the `OsStr` starts with a UTF-8 section followed by
/// `boundary`, returns the sections before and after the boundary
/// character.  Otherwise returns `None`.
fn split_off_str(&self, boundary: char) -> Option<(&str, &OsStr)>;
```

The first three of these (`contains_os`, `starts_with_os`, and
`ends_with_os`) test for `OsStr` substrings of an `OsStr`.

The remainder implement a subset of the string pattern matching
functionality of `OsStr`.  These functions act the same as the `str`
versions, except that some of them require an additional `Clone` bound
on the pattern.  Note that patterns can only match UTF-8 sections of
the `OsStr`.

FIXME
These methods fall into two categories.  The first four
(`starts_with_str`, `remove_prefix_str`, `slice_shift_char`, and
`split_off_str`) interpret a prefix of the `OsStr` as UTF-8 data,
while ignoring any non-UTF-8 parts later in the string.  The last is a
restricted splitting operation.

### `starts_with_str`

`string.starts_with_str(prefix)` is logically equivalent to
`string.remove_prefix_str(prefix).is_some()`, but is likely to be a
common enough special case to warrant it's own clearer syntax.

### `remove_prefix_str`

This could be used for things such as removing the leading "--" from
command line options as is common to enable simpler processing.
Example:
```rust
let opt = OsString::from("--path=/some/path");
assert_eq!(opt.remove_prefix_str("--"), Some(OsStr::new("path=/some/path")));
```

### `slice_shift_char`

This performs the same function as the similarly named method on
`str`, except that it also returns `None` if the `OsStr` does not
start with a valid UTF-8 character.  While the `str` version of this
function may be removed for being redundant with `str::chars`, the
functionality is still needed here because it is not clear how an
iterator over the contents of an `OsStr` could be defined in a
platform-independent way.

An intended use for this function is for interpreting bundled
command-line switches.  For example, with switches from rustc:

```rust
let mut opts = &OsString::from("vL/path")[..]; // Leading '-' has already been removed
while let Some((ch, rest)) = opts.slice_shift_char() {
    opts = rest;
    match ch {
        'v' => { verbose = true; }
        'L' => { /* interpret remainder as a link path */ }
        ....
    }
}
```

### `split_off_str`

This is intended for interpreting "tagged" OS strings, for example
rustc's `-L [KIND=]PATH` arguments.  It is expected that such tags
will usually be UTF-8.  Example:
```rust
let s = OsString::from("dylib=/path");

let (name, kind) = match s.split_off_str('=') {
    None => (&*s, cstore::NativeUnknown),
    Some(("dylib", name)) => (name, cstore::NativeUnknown),
    Some(("framework", name)) => (name, cstore::NativeFramework),
    Some(("static", name)) => (name, cstore::NativeStatic),
    Some((s, _)) => { error(...) }
};
```

## `SliceConcatExt`

Implement the trait
```rust
impl<S> SliceConcatExt<OsStr> for [S] where S: Borrow<OsStr> {
    type Output = OsString;
    ...
}
```

This has the same behavior as the `str` version, except that it works
on OS strings.  It is intended as a more convenient and efficient way
of building up an `OsString` from parts than repeatedly calling
`push`.

# Drawbacks

This is a somewhat unusual string interface in that many of the
functions only accepts UTF-8 encoded data, while the type can encode
more general strings.  Unfortunately, in many cases it is not possible
to generalize the interface to accept non-UTF-8 data.  For example, on
Windows, the following should hold using a hypothetical
`split(&self, &OsStr) -> Split`:

```rust
let string = OsString::from("😺"); // [0xD83D, 0xDE3A] in UTF-16
let prefix: OsString = OsStringExt::from_wide(&[0xD83D]);
let suffix: OsString = OsStringExt::from_wide(&[0xDE3A]);

assert_eq!(string.split(&suffix[..]).next(), Some(&prefix[..]));
```

However, the slice `&prefix[..]` (internally `[0xED, 0xA0, 0xBD]`)
does not occur anywhere in `string` (internally `[0xF0, 0x9F, 0x98,
0xBA]`), so there would be no way to construct the values returned by
such an iterator.

# Alternatives

## Stricter bounds on the pattern-accepting iterator constructors

The proposed bounds on the pattern-accepting functions are the weakest
possible.  This means that one can often construct an "iterator" that
does not actually implement the `Iterator` trait.  For example, one
can call `split` with any `P: Pattern<'a>`, but the resulting `Split`
struct only implements the `Iterator` trait if `P` is additionally
`Clone`. This is likely to be confusing, so tightening the bounds may
be desirable.

# Unresolved questions

The correct behavior of `split` with a pattern that matches the empty
string is not clear.  Possibilities include:

* panic
* match on "character boundaries", probably defined as the ends of the
  string and adjacent to each UTF-8 character.
* define the behavior to commute with `to_string_lossy` (assuming the
  pattern does not match anything including the replacement character)

In any case, care should be taken to handle patterns that can match
both the empty string and non-empty strings correctly.
