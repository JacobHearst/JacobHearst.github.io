+++
title = 'Scratchpad: Lookbehinds via reverse matching'
date = 2024-06-12T07:03:22-05:00
draft = true
tags = ['swift', 'scratchpad']
+++

A month or two ago I discovered that Swift's fancy new regex builder and regex literals didn't support lookbehinds, a feature I wanted for a
project I'm working on. I decided to see what it would take to implement it myself and slowly worked my way to a (sort-of) (mostly) working
[initial implementation](https://forums.swift.org/t/swift-regex-lookbehind/58477/19). Michael Ilseman (one of the original authors of the Swift 5.
7 regex work) has been kindly giving me pointers that were essential for getting as far as I have. In his eyes, the core concept behind
lookbehinds, is reverse matching. This is the focus of this scratchpad.

<!--more-->

## What _exactly_ is reverse matching?
Michael messaged me this example:

> For example, forwards match of `(.*)(.*)z` against `"abcdefgz"` would produce the capture groups of `("abcdefg", "")`. A reverse-match of that input would produce the capture groups of `("", "abcdefg")`.

My initial thought is that this means reverse matching is just the equivalent of reversing the regex and running it from the end of the input to
the start.

This is what the forwards match looks like in Swift:
```swift
let expr = #/(.*)(.*)z/#
let input = "abcdefgzfac" // Added some extra characters to make it more clear what match!.output.0 was

let match = input.firstMatch(of: expr)
print(match!.output.0) // Full match: "abcdefgz"
print(match!.output.1) // Group 1:    "abcdefg"
print(match!.output.2) // Group 2:    ""
```

Obviously reverse-matching doesn't exist yet in Swift but I thought I could simulate it by reversing the regex and applying it to the reversed input:
```swift
let expr = #/z(.*)(.*)/#
let input = "cafzgfedcba"

let match = input.firstMatch(of: expr)
print(match!.output.0) // Full match: "zgfedcba"
print(match!.output.1) // Group 1:    "gfedcba" <-- This should be ""
print(match!.output.2) // Group 2:    ""        <-- This should be "gfedcba"
```

Unfortunately that didn't quite work as in Michael's example but I might just need to reverse the capture group order? This seems a bit strange
but I suppose it makes some sense since we reversed the expression and the input so to get back into forwards-land we need to reverse the output
of the reversed match. Thinking backwards hurts my brain and something about flipping the output seems weird so I decided to reach out to Michael
for some clarification.