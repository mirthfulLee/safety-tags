# `std.annotated_params.v2.json`

This file is a reviewed, consumer-friendly view of the safety-tag dataset for unsafe standard-library functions.

## Schema

Each entry has the form:

```json
{
  "fn_path": {
    "params": ["..."],
    "contracts": {
      "<param_name_or_reserved_key>": ["Tag1", "Tag2"]
    }
  }
}
```

## Meaning of `params`

- `params` is the ordered parameter-name list for the target function.
- For methods, the receiver is included as parameter `0` and is written as `self`.
- For free functions and associated functions, `params[0]` is the first explicit parameter in the Rust signature.

Examples:

```json
{
  "core::alloc::Allocator::grow": {
    "params": ["ptr", "old_layout", "new_layout"],
    "contracts": {
      "ptr": ["Allocated", "Layout", "ValidNum"],
      "old_layout": ["Allocated", "Layout", "ValidNum"],
      "new_layout": ["Layout", "ValidNum"]
    }
  }
}
```

```json
{
  "alloc::collections::btree::map::insert_after_unchecked": {
    "params": ["self", "key", "value"],
    "contracts": {
      "self": ["Function_sp"],
      "key": ["Function_sp"],
      "value": ["Function_sp"]
    }
  }
}
```

## Meaning of `contracts`

`contracts` maps either:

- a concrete parameter name, such as `ptr`, `len`, `self`, `key`, `value`
- or a reserved function-level key

to the safety tags relevant to that parameter or function-level requirement.

The tag values are copied from `data/std.json`; this file only rewrites how those tags are attached to targets.

## Reserved Keys

Currently the following reserved keys may appear:

- `function_precondition`
  Used when the function has no explicit parameter target for the contract, but the contract is still a precondition on calling the unsafe function.

- `environment_requirement`
  Used for contracts that primarily constrain process/thread/environment state rather than a specific explicit parameter.

Examples:

```json
{
  "core::hint::unreachable_unchecked": {
    "params": [],
    "contracts": {
      "function_precondition": ["Unreachable"]
    }
  }
}
```

## Provenance

This file is derived from:

- `data/std.json`
- `data/std.reviewed_indexed_params.json`

The conversion replaces numeric parameter slots (`"0"`, `"1"`, ...) with concrete parameter names or reserved keys.

## Intended Use

Use this file if your analysis wants to answer questions such as:

- which contracts are related to a given explicit parameter?
- which contracts are receiver-related?
- which contracts are function-level rather than parameter-level?

If you need the original slot-based data, use `data/std.json`.

## Expressiveness Notes

This file is still limited by the coarse tag vocabulary inherited from `data/std.json`.
Some unsafe APIs have safety requirements that are fundamentally:

- type-level rather than value-level
- relation-level across multiple parameters
- region-local rather than whole-parameter predicates
- representation-specific and not equivalent to UTF-8 / ordinary string validity

Typical examples:

- `mem::transmute`
  The real contract depends on source/destination type size and validity of the destination type.
  The current tag system cannot precisely encode the destination-type side of this requirement.

- `task::Waker::{new, from_raw}` and `task::LocalWaker::{new, from_raw}`
  The true precondition is “the `RawWaker` / `RawWakerVTable` contract is upheld”.
  In this dataset that requirement is attached conservatively to all directly involved parameters
  using coarse function-specific tags such as `Function_sp`.

- `ffi::OsStr::from_encoded_bytes_unchecked` / `ffi::OsString::from_encoded_bytes_unchecked`
  The actual requirement is about platform-specific self-synchronizing OS-string encodings
  originating from compatible `as_encoded_bytes` outputs, not generic UTF-8 validity.

- `io::BorrowedBuf::set_init` / `io::BorrowedCursor::{as_mut, advance_unchecked, set_init}`
  The real preconditions talk about the first `n` bytes or the already-initialized region.
  The dataset therefore prefers attaching tags to `n` or to a function-level precondition
  rather than incorrectly claiming that the whole receiver is fully initialized.
