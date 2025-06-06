---
title: "We have Option and Result at home"
date: 2025-02-03
draft: false
summary: "My thoughts on how Go's language structures emulate some Rust concepts in a vastly simpler (in my opinion) way"
tags: ["go"]
---

This one will be nice and short! I've done both Go and Rust, although admittedly much more Go than Rust. I'm yet to reach fluency with Rust (perhaps I'm not alone?) but I feel like
I have to a greater extent with Go. Regardless, I appreciate them both, this is not a flame war, both have a place, I won't be comparing them in detail here.

This post is about two concepts that people (including me) *love* in Rust and how we actually have the same thing in Go... except it doesn't require all the fancy language features and
complexity of monads, pattern matching, generics etc.

It hadn't really ever occurred to me in this way before so I figured I'd write about it 🙂

## Result

I'm sure most people will know this already, but let's cover it briefly.

A `Result` in Rust is the standard error handling type:

```rust
enum Result<T, E> {
   Ok(T),
   Err(E),
}
```

It's an `enum`, generic over 2 types (`T` and `E`) with two variants:

- `Err(E)` -> Something has gone wrong, you have an error of type `E`, you can inspect it to find out more
- `Ok(T)` -> Everything is fine, here's the thing you wanted `T`

The Rust language provides many helpful things for working with these types including pattern matching, the `?` operator, it implements `FromIterator` meaning you can collect a list of Results into a single one, lots of attached methods etc. and those are all great.

But let's just take a step back for a second. The message this type is trying to express is that something *might* have gone wrong, you don't know unless you check (and the compiler makes you check).

Well... we have this in Go right?

```go
func canFail() (string, error) {
    // ...
}
```

And this is *delightfully* more simple. It requires no fancy types, no monads, no pattern matching, no methods, no special syntax. It's just two return values. And yet this perfectly
expresses the same message as the `Result`:

- `return "", err` -> Something has gone wrong, you have an error of type `error`, you can inspect it to find out more
- `return "success", nil` -> Everything is fine, here's the thing you wanted `string`

None of this is particularly groundbreaking but I hadn't ever really thought about it in quite this way before.

## Option

The same thing applies to `Option`.

`Option` is Rust's way of saying "this *could* be a thing, but we're not sure".

```rust
enum Option<T> {
    None,
    Some(T),
}
```

Again, it's an `enum`, generic over `T` with two variants:

- `None` -> There's nothing here
- `Some(T)` -> There's something here, and it's of type `T`

And again, Rust provides numerous helpful patterns and mechanism for working with options, just like results.

And just like `Result`, we have *this* in Go too:

```go
func findString() (found string, ok bool) {
    // ...
}
```

- `return "", false` -> There's nothing here
- `return "bingo", true` -> There's something here, and it's a string

{{<alert "lightbulb">}}
You could also imagine generic versions of this too, I'm just cheating and using a `string` as an example
{{</alert>}}

## Takeaway

This actually got me thinking about a bit of a rule on when to use the `ok` idiom in Go:

- Use `(T, error)` when there could be a number of possible errors:
  - Reading from a file (could be lots of errors)
  - Making HTTP requests
  - Basically most normal functions

- Use `(val T, ok bool)` when there is only *one* possible error, and it's *obvious* what that error is:
  - Looking up a value from a map, the only real error being it's missing
  - Type conversions, only real error is that it cannot be converted
