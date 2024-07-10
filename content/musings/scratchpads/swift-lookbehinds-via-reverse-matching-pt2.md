+++
title = 'Swift Lookbehinds via Reverse Matching Part 2'
date = 2024-07-10T09:40:34-05:00
draft = false
tags = ['swift', 'scratchpad']
+++

[Part 1](/musings/scratchpads/swift-lookbehinds-via-reverse-matching)

Had a video call with Michael and was able to figure out what I need to do. I was actually pretty spot on with what reverse matching is supposed
to be (I mean the name kind of says it all), but what I inferred from his suggestion is that my approach is inefficient. Instead of performing
a reversing operation on the entire AST, his suggestion was to implement reverse variants of the following instructions: `matchScalar`, `matchBitset`,
`matchBuiltin`, `matchAnyNonNewline`, and `advance`. I could then simply emit the reversed instructions instead of their forwards counterparts
whenever we enter a lookbehind. So to summarize, my plan of attack for building a reverse match is:

1. Introduce a flag on `ByteCodeGen` to indicate when the instructions should be reversed
2. Flip said flag when we enter a lookbehind and flip it back when we leave
3. Introduce some logic in the various `emitX` functions to:
    1. Step back
    2. Emit the reversed version of an instruction when the flag is `true`
4. Profit?

## The Implementation

I cleverly named my reverse instruction variants by simply putting `reverse` in front of the corresponding forwards instruction name with the exception
of `advance` who's reverse is simply named `reverse`. Doing that got the compiler to yell at me about non-exhaustive `switch` statements so I commented
out all my new instructions except for `reverse`, `reverseMatch`, and `reverseMatchScalar`. Then I went through and made `buildReverseX` functions in
`MEProgram.Builder` so that I could call those from `ByteCodeGen` which is where the real fun starts.

### ByteCodeGen

Steps 1 and 2 of my above plan were simple enough. Declare a new flag, set that flag. Done.

```swift
struct ByteCodeGen {
  var reverse = false
  //...

  mutating func emitLookaround(
    _ kind: (forwards: Bool, positive: Bool),
    _ child: DSLTree.Node
  ) throws {
    reverse = !kind.forwards
    switch kind {
        // Emit the various kinds of lookarounds depending on the kind...
    }
    reverse = false
  }
  //...
}
```

Step 3 is where I start getting confused again. I _think_ that the lookahead and lookbehind instructions are actually going to be the exact same in terms of the
save point machinary so I replaced the `switch` with a simple `if-else` block. Now the full `emitLookaround` function looks like this:

```swift
mutating func emitLookaround(
    _ kind: (forwards: Bool, positive: Bool),
    _ child: DSLTree.Node
  ) throws {
    reverse = !kind.forwards
    if kind.positive {
      try emitPositiveLookaround(child) // Renamed from `emitPositiveLookahead`
    } else {
      try emitNegativeLookaround(child) // Renamed from `emitNegativeLookahead`
    }
    reverse = false
  }
```

Since the actual reversing happens inside the functions called from `emitNode` (which is called by `emitPositiveLookaround`/`emitNegativeLookaround`) I think re-using
these functions will be okay. My confidence that I'm not overlooking something here is low but for now it makes sense in my head that a lookbehind is just a reverse
lookahead so we'll give this a go and see what happens 🤷‍♂️.

On our meeting, Michael said something like "for reverse matching, we should step back and then match the _rightmost_ element of the concatenation". Because of this,
I decided to start modifying the bytecode generation in the `emitConcatenation` function.

```swift
mutating func emitConcatenation(_ children: [DSLTree.Node]) throws {
    // Prep the children for processing...
    if reverse {
      children.reverse()
    }
    for child in children {
      if reverse {
        builder.buildReverse(1)
      }
      try emitConcatenationComponent(child)
    }
  }
```

Reversing the children here makes sense to me because, as Michael said, we need to match the _rightmost_ element of the concatenation first. In other words, match
right-to-left. I don't think that by itself would work though because we don't want to match the current character, but the previous one. Hence the additional `if`
inside the `for` loop.

Essentially from here I just went down the tree of `emit` functions called from `emitConcatenationComponent` and wherever I found a `emitMatch`, `buildMatch`,
`emitMatchScalar`,  or `buildMatchScalar`, I added a conditional to call the reversed counterparts of those functions, building the counterparts as I went.

### Processor
With the bytecode generation done (although possibly incorrect), I need to write the implementation of the new instructions in the processor. This is where things are getting much more difficult.

#### Scalar Matching
Starting with `reverseMatchScalar` I've copied the tree of functions called to handle `matchScalar` and modified them to go
backwards so they look like this:

```swift
extension Processor {
  mutating func reverseMatchScalar(
    _ s: Unicode.Scalar,
    boundaryCheck: Bool,
    isCaseInsensitive: Bool
  ) -> Bool {
    guard let next = input.reverseMatchScalar( // `input` is a `String`
      s,
      at: currentPosition,
      limitedBy: start,
      boundaryCheck: boundaryCheck,
      isCaseInsensitive: isCaseInsensitive
    ) else {
      signalFailure()
      return false
    }
    currentPosition = next
    return true
  }
}

extension String {
  func reverseMatchScalar(
    _ scalar: Unicode.Scalar,
    at pos: Index,
    limitedBy start: String.Index,
    boundaryCheck: Bool,
    isCaseInsensitive: Bool
  ) -> Index? {
    guard pos >= start else { return nil }
    let curScalar = unicodeScalars[pos]

    if isCaseInsensitive {
      guard curScalar.properties.lowercaseMapping == scalar.properties.lowercaseMapping
      else {
        return nil
      }
    } else {
      guard curScalar == scalar else { return nil }
    }

    guard pos != start else { return nil }
    let idx = unicodeScalars.index(before: pos)
    assert(idx >= start, "Input is a substring with a sub-scalar startIndex.")

    if boundaryCheck && !isOnGraphemeClusterBoundary(idx) {
      return nil
    }

    return idx
  }
}
```

The problem with my `String.reverseMatchScalar` implementation is that returning `nil` when we've hit the start of the string
causes `Processor.reverseMatchScalar` to trigger the `guard` clause and fail the match. I need to somehow prevent execution of
`let idx = unicodeScalars.index(before: pos)` when we reach the start of the string (since it has a precondition that "the previous
index exists") but ALSO return something other than `nil` so that `Processor.reverseMatchScalar` doesn't fail.

Returning the current position in that case seems to work for my current test case of matching `"abcdef"` with `(?<=abc)def`.

_Some time later_

### Debugging

Unfortunately there are still issues with the approach I used above. Luckily, the authors of the regex engine added in some tracing
machinary. This machinary is locked behind a compilation flag which I was having trouble setting for xcode runs so I just inverted the
conditions 🤷‍♂️. I also had to modify the `_compileRegex` function I was using to enable tracing and metrics but then my next test run
spat out TONS of output. The game plan now is to just test a bunch of different regular expressions of varying complexity and see what
breaks!

#### Test case #1
```text
Input: "bdefg"
Regex: /(?<=a|b|c)defg/
```

Problem: This test case was crashing when I ran the match with the message `"Fatal error: String index is out of bounds"`.

Solution: Reading through the trace output below, I was able to see that I was never `reverse`-ing. Scattering some `buildReverse`s seemed to fix this.

{{< details title="Test case #1 trace output">}}

```text
--- cycle 0 ---

input: bdefg
       ^
position: 0
>[0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] save #6
 [4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
 [6] save #9


--- cycle 1 ---

input: bdefg
       ^
position: 0
 [0] beginCapture 0
>[1] save #14
 [2] save #12
 [3] save #6
 [4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
 [6] save #9
 [7] reverseMatchScalar 'b' boundaryCheck: true


--- cycle 2 ---
save points:
  pc: #14, pos: 0, stackEnd: #0

input: bdefg
       ^
position: 0
 [0] beginCapture 0
 [1] save #14
>[2] save #12
 [3] save #6
 [4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
 [6] save #9
 [7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10


--- cycle 3 ---
save points:
  pc: #14, pos: 0, stackEnd: #0
  pc: #12, pos: 0, stackEnd: #0

input: bdefg
       ^
position: 0
 [0] beginCapture 0
 [1] save #14
 [2] save #12
>[3] save #6
 [4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
 [6] save #9
 [7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10
 [9] reverseMatchScalar 'c' boundaryCheck: true


--- cycle 4 ---
save points:
  pc: #14, pos: 0, stackEnd: #0
  pc: #12, pos: 0, stackEnd: #0
  pc: #6, pos: 0, stackEnd: #0

input: bdefg
       ^
position: 0
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] save #6
>[4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
 [6] save #9
 [7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10
 [9] reverseMatchScalar 'c' boundaryCheck: true
 [10] clearThrough


--- cycle 5 ---
save points:
  pc: #14, pos: 0, stackEnd: #0
  pc: #12, pos: 0, stackEnd: #0

input: bdefg
       ^
position: 0
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] save #6
 [4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
>[6] save #9
 [7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10
 [9] reverseMatchScalar 'c' boundaryCheck: true
 [10] clearThrough
 [11] fail
 [12] clear


--- cycle 6 ---
save points:
  pc: #14, pos: 0, stackEnd: #0
  pc: #12, pos: 0, stackEnd: #0
  pc: #9, pos: 0, stackEnd: #0

input: bdefg
       ^
position: 0
 [1] save #14
 [2] save #12
 [3] save #6
 [4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
 [6] save #9
>[7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10
 [9] reverseMatchScalar 'c' boundaryCheck: true
 [10] clearThrough
 [11] fail
 [12] clear
 [13] fail


--- cycle 7 ---
save points:
  pc: #14, pos: 0, stackEnd: #0
  pc: #12, pos: 0, stackEnd: #0
  pc: #9, pos: 0, stackEnd: #0

input: bdefg
       ^
position: 0
 [2] save #12
 [3] save #6
 [4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
 [6] save #9
 [7] reverseMatchScalar 'b' boundaryCheck: true
>[8] branch #10
 [9] reverseMatchScalar 'c' boundaryCheck: true
 [10] clearThrough
 [11] fail
 [12] clear
 [13] fail
 [14] matchScalar 'd' boundaryCheck: false


--- cycle 8 ---
save points:
  pc: #14, pos: 0, stackEnd: #0
  pc: #12, pos: 0, stackEnd: #0
  pc: #9, pos: 0, stackEnd: #0

input: bdefg
       ^
position: 0
 [4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
 [6] save #9
 [7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10
 [9] reverseMatchScalar 'c' boundaryCheck: true
>[10] clearThrough
 [11] fail
 [12] clear
 [13] fail
 [14] matchScalar 'd' boundaryCheck: false
 [15] matchScalar 'e' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false


--- cycle 9 ---
save points:
  pc: #14, pos: 0, stackEnd: #0

input: bdefg
       ^
position: 0
 [5] branch #10
 [6] save #9
 [7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10
 [9] reverseMatchScalar 'c' boundaryCheck: true
 [10] clearThrough
>[11] fail
 [12] clear
 [13] fail
 [14] matchScalar 'd' boundaryCheck: false
 [15] matchScalar 'e' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'g' boundaryCheck: true


--- cycle 10 ---

input: bdefg
       ^
position: 0
 [8] branch #10
 [9] reverseMatchScalar 'c' boundaryCheck: true
 [10] clearThrough
 [11] fail
 [12] clear
 [13] fail
>[14] matchScalar 'd' boundaryCheck: false
 [15] matchScalar 'e' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'g' boundaryCheck: true
 [18] endCapture 0
 [19] accept


--- cycle 11 ---

input: bdefg
       ^
position: 0
FAIL

--- cycle 12 ---

input: bdefg
       ~^
position: 1
 [0] beginCapture 0
>[1] save #14
 [2] save #12
 [3] save #6
 [4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
 [6] save #9
 [7] reverseMatchScalar 'b' boundaryCheck: true


--- cycle 13 ---
save points:
  pc: #14, pos: 1, stackEnd: #0

input: bdefg
       ~^
position: 1
 [0] beginCapture 0
 [1] save #14
>[2] save #12
 [3] save #6
 [4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
 [6] save #9
 [7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10


--- cycle 14 ---
save points:
  pc: #14, pos: 1, stackEnd: #0
  pc: #12, pos: 1, stackEnd: #0

input: bdefg
       ~^
position: 1
 [0] beginCapture 0
 [1] save #14
 [2] save #12
>[3] save #6
 [4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
 [6] save #9
 [7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10
 [9] reverseMatchScalar 'c' boundaryCheck: true


--- cycle 15 ---
save points:
  pc: #14, pos: 1, stackEnd: #0
  pc: #12, pos: 1, stackEnd: #0
  pc: #6, pos: 1, stackEnd: #0

input: bdefg
       ~^
position: 1
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] save #6
>[4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
 [6] save #9
 [7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10
 [9] reverseMatchScalar 'c' boundaryCheck: true
 [10] clearThrough


--- cycle 16 ---
save points:
  pc: #14, pos: 1, stackEnd: #0
  pc: #12, pos: 1, stackEnd: #0

input: bdefg
       ~^
position: 1
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] save #6
 [4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
>[6] save #9
 [7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10
 [9] reverseMatchScalar 'c' boundaryCheck: true
 [10] clearThrough
 [11] fail
 [12] clear


--- cycle 17 ---
save points:
  pc: #14, pos: 1, stackEnd: #0
  pc: #12, pos: 1, stackEnd: #0
  pc: #9, pos: 1, stackEnd: #0

input: bdefg
       ~^
position: 1
 [1] save #14
 [2] save #12
 [3] save #6
 [4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
 [6] save #9
>[7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10
 [9] reverseMatchScalar 'c' boundaryCheck: true
 [10] clearThrough
 [11] fail
 [12] clear
 [13] fail


--- cycle 18 ---
save points:
  pc: #14, pos: 1, stackEnd: #0
  pc: #12, pos: 1, stackEnd: #0

input: bdefg
       ~^
position: 1
 [3] save #6
 [4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
 [6] save #9
 [7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10
>[9] reverseMatchScalar 'c' boundaryCheck: true
 [10] clearThrough
 [11] fail
 [12] clear
 [13] fail
 [14] matchScalar 'd' boundaryCheck: false
 [15] matchScalar 'e' boundaryCheck: false


--- cycle 19 ---
save points:
  pc: #14, pos: 1, stackEnd: #0

input: bdefg
       ~^
position: 1
 [6] save #9
 [7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10
 [9] reverseMatchScalar 'c' boundaryCheck: true
 [10] clearThrough
 [11] fail
>[12] clear
 [13] fail
 [14] matchScalar 'd' boundaryCheck: false
 [15] matchScalar 'e' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'g' boundaryCheck: true
 [18] endCapture 0


--- cycle 20 ---

input: bdefg
       ~^
position: 1
 [7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10
 [9] reverseMatchScalar 'c' boundaryCheck: true
 [10] clearThrough
 [11] fail
 [12] clear
>[13] fail
 [14] matchScalar 'd' boundaryCheck: false
 [15] matchScalar 'e' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'g' boundaryCheck: true
 [18] endCapture 0
 [19] accept


--- cycle 21 ---

input: bdefg
       ~^
position: 1
FAIL

--- cycle 22 ---

input: bdefg
       ~~^
position: 2
 [0] beginCapture 0
>[1] save #14
 [2] save #12
 [3] save #6
 [4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
 [6] save #9
 [7] reverseMatchScalar 'b' boundaryCheck: true


--- cycle 23 ---
save points:
  pc: #14, pos: 2, stackEnd: #0

input: bdefg
       ~~^
position: 2
 [0] beginCapture 0
 [1] save #14
>[2] save #12
 [3] save #6
 [4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
 [6] save #9
 [7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10


--- cycle 24 ---
save points:
  pc: #14, pos: 2, stackEnd: #0
  pc: #12, pos: 2, stackEnd: #0

input: bdefg
       ~~^
position: 2
 [0] beginCapture 0
 [1] save #14
 [2] save #12
>[3] save #6
 [4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
 [6] save #9
 [7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10
 [9] reverseMatchScalar 'c' boundaryCheck: true


--- cycle 25 ---
save points:
  pc: #14, pos: 2, stackEnd: #0
  pc: #12, pos: 2, stackEnd: #0
  pc: #6, pos: 2, stackEnd: #0

input: bdefg
       ~~^
position: 2
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] save #6
>[4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
 [6] save #9
 [7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10
 [9] reverseMatchScalar 'c' boundaryCheck: true
 [10] clearThrough


--- cycle 26 ---
save points:
  pc: #14, pos: 2, stackEnd: #0
  pc: #12, pos: 2, stackEnd: #0

input: bdefg
       ~~^
position: 2
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] save #6
 [4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
>[6] save #9
 [7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10
 [9] reverseMatchScalar 'c' boundaryCheck: true
 [10] clearThrough
 [11] fail
 [12] clear


--- cycle 27 ---
save points:
  pc: #14, pos: 2, stackEnd: #0
  pc: #12, pos: 2, stackEnd: #0
  pc: #9, pos: 2, stackEnd: #0

input: bdefg
       ~~^
position: 2
 [1] save #14
 [2] save #12
 [3] save #6
 [4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
 [6] save #9
>[7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10
 [9] reverseMatchScalar 'c' boundaryCheck: true
 [10] clearThrough
 [11] fail
 [12] clear
 [13] fail


--- cycle 28 ---
save points:
  pc: #14, pos: 2, stackEnd: #0
  pc: #12, pos: 2, stackEnd: #0

input: bdefg
       ~~^
position: 2
 [3] save #6
 [4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
 [6] save #9
 [7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10
>[9] reverseMatchScalar 'c' boundaryCheck: true
 [10] clearThrough
 [11] fail
 [12] clear
 [13] fail
 [14] matchScalar 'd' boundaryCheck: false
 [15] matchScalar 'e' boundaryCheck: false


--- cycle 29 ---
save points:
  pc: #14, pos: 2, stackEnd: #0

input: bdefg
       ~~^
position: 2
 [6] save #9
 [7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10
 [9] reverseMatchScalar 'c' boundaryCheck: true
 [10] clearThrough
 [11] fail
>[12] clear
 [13] fail
 [14] matchScalar 'd' boundaryCheck: false
 [15] matchScalar 'e' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'g' boundaryCheck: true
 [18] endCapture 0


--- cycle 30 ---

input: bdefg
       ~~^
position: 2
 [7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10
 [9] reverseMatchScalar 'c' boundaryCheck: true
 [10] clearThrough
 [11] fail
 [12] clear
>[13] fail
 [14] matchScalar 'd' boundaryCheck: false
 [15] matchScalar 'e' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'g' boundaryCheck: true
 [18] endCapture 0
 [19] accept


--- cycle 31 ---

input: bdefg
       ~~^
position: 2
FAIL

--- cycle 32 ---

input: bdefg
       ~~~^
position: 3
 [0] beginCapture 0
>[1] save #14
 [2] save #12
 [3] save #6
 [4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
 [6] save #9
 [7] reverseMatchScalar 'b' boundaryCheck: true


--- cycle 33 ---
save points:
  pc: #14, pos: 3, stackEnd: #0

input: bdefg
       ~~~^
position: 3
 [0] beginCapture 0
 [1] save #14
>[2] save #12
 [3] save #6
 [4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
 [6] save #9
 [7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10


--- cycle 34 ---
save points:
  pc: #14, pos: 3, stackEnd: #0
  pc: #12, pos: 3, stackEnd: #0

input: bdefg
       ~~~^
position: 3
 [0] beginCapture 0
 [1] save #14
 [2] save #12
>[3] save #6
 [4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
 [6] save #9
 [7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10
 [9] reverseMatchScalar 'c' boundaryCheck: true


--- cycle 35 ---
save points:
  pc: #14, pos: 3, stackEnd: #0
  pc: #12, pos: 3, stackEnd: #0
  pc: #6, pos: 3, stackEnd: #0

input: bdefg
       ~~~^
position: 3
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] save #6
>[4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
 [6] save #9
 [7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10
 [9] reverseMatchScalar 'c' boundaryCheck: true
 [10] clearThrough


--- cycle 36 ---
save points:
  pc: #14, pos: 3, stackEnd: #0
  pc: #12, pos: 3, stackEnd: #0

input: bdefg
       ~~~^
position: 3
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] save #6
 [4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
>[6] save #9
 [7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10
 [9] reverseMatchScalar 'c' boundaryCheck: true
 [10] clearThrough
 [11] fail
 [12] clear


--- cycle 37 ---
save points:
  pc: #14, pos: 3, stackEnd: #0
  pc: #12, pos: 3, stackEnd: #0
  pc: #9, pos: 3, stackEnd: #0

input: bdefg
       ~~~^
position: 3
 [1] save #14
 [2] save #12
 [3] save #6
 [4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
 [6] save #9
>[7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10
 [9] reverseMatchScalar 'c' boundaryCheck: true
 [10] clearThrough
 [11] fail
 [12] clear
 [13] fail


--- cycle 38 ---
save points:
  pc: #14, pos: 3, stackEnd: #0
  pc: #12, pos: 3, stackEnd: #0

input: bdefg
       ~~~^
position: 3
 [3] save #6
 [4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
 [6] save #9
 [7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10
>[9] reverseMatchScalar 'c' boundaryCheck: true
 [10] clearThrough
 [11] fail
 [12] clear
 [13] fail
 [14] matchScalar 'd' boundaryCheck: false
 [15] matchScalar 'e' boundaryCheck: false


--- cycle 39 ---
save points:
  pc: #14, pos: 3, stackEnd: #0

input: bdefg
       ~~~^
position: 3
 [6] save #9
 [7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10
 [9] reverseMatchScalar 'c' boundaryCheck: true
 [10] clearThrough
 [11] fail
>[12] clear
 [13] fail
 [14] matchScalar 'd' boundaryCheck: false
 [15] matchScalar 'e' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'g' boundaryCheck: true
 [18] endCapture 0


--- cycle 40 ---

input: bdefg
       ~~~^
position: 3
 [7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10
 [9] reverseMatchScalar 'c' boundaryCheck: true
 [10] clearThrough
 [11] fail
 [12] clear
>[13] fail
 [14] matchScalar 'd' boundaryCheck: false
 [15] matchScalar 'e' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'g' boundaryCheck: true
 [18] endCapture 0
 [19] accept


--- cycle 41 ---

input: bdefg
       ~~~^
position: 3
FAIL

--- cycle 42 ---

input: bdefg
       ~~~~^
position: 4
 [0] beginCapture 0
>[1] save #14
 [2] save #12
 [3] save #6
 [4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
 [6] save #9
 [7] reverseMatchScalar 'b' boundaryCheck: true


--- cycle 43 ---
save points:
  pc: #14, pos: 4, stackEnd: #0

input: bdefg
       ~~~~^
position: 4
 [0] beginCapture 0
 [1] save #14
>[2] save #12
 [3] save #6
 [4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
 [6] save #9
 [7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10


--- cycle 44 ---
save points:
  pc: #14, pos: 4, stackEnd: #0
  pc: #12, pos: 4, stackEnd: #0

input: bdefg
       ~~~~^
position: 4
 [0] beginCapture 0
 [1] save #14
 [2] save #12
>[3] save #6
 [4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
 [6] save #9
 [7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10
 [9] reverseMatchScalar 'c' boundaryCheck: true


--- cycle 45 ---
save points:
  pc: #14, pos: 4, stackEnd: #0
  pc: #12, pos: 4, stackEnd: #0
  pc: #6, pos: 4, stackEnd: #0

input: bdefg
       ~~~~^
position: 4
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] save #6
>[4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
 [6] save #9
 [7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10
 [9] reverseMatchScalar 'c' boundaryCheck: true
 [10] clearThrough


--- cycle 46 ---
save points:
  pc: #14, pos: 4, stackEnd: #0
  pc: #12, pos: 4, stackEnd: #0

input: bdefg
       ~~~~^
position: 4
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] save #6
 [4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
>[6] save #9
 [7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10
 [9] reverseMatchScalar 'c' boundaryCheck: true
 [10] clearThrough
 [11] fail
 [12] clear


--- cycle 47 ---
save points:
  pc: #14, pos: 4, stackEnd: #0
  pc: #12, pos: 4, stackEnd: #0
  pc: #9, pos: 4, stackEnd: #0

input: bdefg
       ~~~~^
position: 4
 [1] save #14
 [2] save #12
 [3] save #6
 [4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
 [6] save #9
>[7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10
 [9] reverseMatchScalar 'c' boundaryCheck: true
 [10] clearThrough
 [11] fail
 [12] clear
 [13] fail


--- cycle 48 ---
save points:
  pc: #14, pos: 4, stackEnd: #0
  pc: #12, pos: 4, stackEnd: #0

input: bdefg
       ~~~~^
position: 4
 [3] save #6
 [4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
 [6] save #9
 [7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10
>[9] reverseMatchScalar 'c' boundaryCheck: true
 [10] clearThrough
 [11] fail
 [12] clear
 [13] fail
 [14] matchScalar 'd' boundaryCheck: false
 [15] matchScalar 'e' boundaryCheck: false


--- cycle 49 ---
save points:
  pc: #14, pos: 4, stackEnd: #0

input: bdefg
       ~~~~^
position: 4
 [6] save #9
 [7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10
 [9] reverseMatchScalar 'c' boundaryCheck: true
 [10] clearThrough
 [11] fail
>[12] clear
 [13] fail
 [14] matchScalar 'd' boundaryCheck: false
 [15] matchScalar 'e' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'g' boundaryCheck: true
 [18] endCapture 0


--- cycle 50 ---

input: bdefg
       ~~~~^
position: 4
 [7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10
 [9] reverseMatchScalar 'c' boundaryCheck: true
 [10] clearThrough
 [11] fail
 [12] clear
>[13] fail
 [14] matchScalar 'd' boundaryCheck: false
 [15] matchScalar 'e' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'g' boundaryCheck: true
 [18] endCapture 0
 [19] accept


--- cycle 51 ---

input: bdefg
       ~~~~^
position: 4
FAIL

--- cycle 52 ---

input: bdefg
       .....^
position: 5
 [0] beginCapture 0
>[1] save #14
 [2] save #12
 [3] save #6
 [4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
 [6] save #9
 [7] reverseMatchScalar 'b' boundaryCheck: true


--- cycle 53 ---
save points:
  pc: #14, pos: 5, stackEnd: #0

input: bdefg
       .....^
position: 5
 [0] beginCapture 0
 [1] save #14
>[2] save #12
 [3] save #6
 [4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
 [6] save #9
 [7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10


--- cycle 54 ---
save points:
  pc: #14, pos: 5, stackEnd: #0
  pc: #12, pos: 5, stackEnd: #0

input: bdefg
       .....^
position: 5
 [0] beginCapture 0
 [1] save #14
 [2] save #12
>[3] save #6
 [4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
 [6] save #9
 [7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10
 [9] reverseMatchScalar 'c' boundaryCheck: true


--- cycle 55 ---
save points:
  pc: #14, pos: 5, stackEnd: #0
  pc: #12, pos: 5, stackEnd: #0
  pc: #6, pos: 5, stackEnd: #0

input: bdefg
       .....^
position: 5
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] save #6
>[4] reverseMatchScalar 'a' boundaryCheck: true
 [5] branch #10
 [6] save #9
 [7] reverseMatchScalar 'b' boundaryCheck: true
 [8] branch #10
 [9] reverseMatchScalar 'c' boundaryCheck: true
 [10] clearThrough

Swift/StringIndexValidation.swift:121: Fatal error: String index is out of bounds
Swift/StringIndexValidation.swift:121: Fatal error: String index is out of bounds
```

{{< /details >}}

<br/><br/>

#### Test case #2
```text
Inputs: "asdefg", "bdefg", "cdefg"
Regex: /(?<=as|b|c)defg/
```

Problem: This regex isn't matching any of the inputs

Solution: I went a little crazy adding the `reverse` instructions and was emitting a `reverse` for every character in `emitReverseQuotedLiteral` instead of one for the whole literal.

{{< details title="Test case #2 trace output" >}}
```text
--- cycle 0 ---

input: asdefg
       ^
position: 0
>[0] beginCapture 0
 [1] save #20
 [2] save #18
 [3] save #10
 [4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false


--- cycle 1 ---

input: asdefg
       ^
position: 0
 [0] beginCapture 0
>[1] save #20
 [2] save #18
 [3] save #10
 [4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)


--- cycle 2 ---
save points:
  pc: #20, pos: 0, stackEnd: #0

input: asdefg
       ^
position: 0
 [0] beginCapture 0
 [1] save #20
>[2] save #18
 [3] save #10
 [4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true


--- cycle 3 ---
save points:
  pc: #20, pos: 0, stackEnd: #0
  pc: #18, pos: 0, stackEnd: #0

input: asdefg
       ^
position: 0
 [0] beginCapture 0
 [1] save #20
 [2] save #18
>[3] save #10
 [4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16


--- cycle 4 ---
save points:
  pc: #20, pos: 0, stackEnd: #0
  pc: #18, pos: 0, stackEnd: #0
  pc: #10, pos: 0, stackEnd: #0

input: asdefg
       ^
position: 0
 [0] beginCapture 0
 [1] save #20
 [2] save #18
 [3] save #10
>[4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
 [10] save #14


--- cycle 5 ---
save points:
  pc: #20, pos: 0, stackEnd: #0
  pc: #18, pos: 0, stackEnd: #0

input: asdefg
       ^
position: 0
 [4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
>[10] save #14
 [11] reverse (isScalarDistance: false, #1)
 [12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16
 [14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough


--- cycle 6 ---
save points:
  pc: #20, pos: 0, stackEnd: #0
  pc: #18, pos: 0, stackEnd: #0
  pc: #14, pos: 0, stackEnd: #0

input: asdefg
       ^
position: 0
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
 [10] save #14
>[11] reverse (isScalarDistance: false, #1)
 [12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16
 [14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough
 [17] fail


--- cycle 7 ---
save points:
  pc: #20, pos: 0, stackEnd: #0
  pc: #18, pos: 0, stackEnd: #0

input: asdefg
       ^
position: 0
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
 [10] save #14
 [11] reverse (isScalarDistance: false, #1)
 [12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16
>[14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough
 [17] fail
 [18] clear
 [19] fail
 [20] matchScalar 'd' boundaryCheck: false


--- cycle 8 ---
save points:
  pc: #20, pos: 0, stackEnd: #0

input: asdefg
       ^
position: 0
 [12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16
 [14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough
 [17] fail
>[18] clear
 [19] fail
 [20] matchScalar 'd' boundaryCheck: false
 [21] matchScalar 'e' boundaryCheck: false
 [22] matchScalar 'f' boundaryCheck: false
 [23] matchScalar 'g' boundaryCheck: true
 [24] endCapture 0


--- cycle 9 ---

input: asdefg
       ^
position: 0
 [13] branch #16
 [14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough
 [17] fail
 [18] clear
>[19] fail
 [20] matchScalar 'd' boundaryCheck: false
 [21] matchScalar 'e' boundaryCheck: false
 [22] matchScalar 'f' boundaryCheck: false
 [23] matchScalar 'g' boundaryCheck: true
 [24] endCapture 0
 [25] accept


--- cycle 10 ---

input: asdefg
       ^
position: 0
FAIL

--- cycle 11 ---

input: asdefg
       ~^
position: 1
 [0] beginCapture 0
>[1] save #20
 [2] save #18
 [3] save #10
 [4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)


--- cycle 12 ---
save points:
  pc: #20, pos: 1, stackEnd: #0

input: asdefg
       ~^
position: 1
 [0] beginCapture 0
 [1] save #20
>[2] save #18
 [3] save #10
 [4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true


--- cycle 13 ---
save points:
  pc: #20, pos: 1, stackEnd: #0
  pc: #18, pos: 1, stackEnd: #0

input: asdefg
       ~^
position: 1
 [0] beginCapture 0
 [1] save #20
 [2] save #18
>[3] save #10
 [4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16


--- cycle 14 ---
save points:
  pc: #20, pos: 1, stackEnd: #0
  pc: #18, pos: 1, stackEnd: #0
  pc: #10, pos: 1, stackEnd: #0

input: asdefg
       ~^
position: 1
 [0] beginCapture 0
 [1] save #20
 [2] save #18
 [3] save #10
>[4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
 [10] save #14


--- cycle 15 ---
save points:
  pc: #20, pos: 1, stackEnd: #0
  pc: #18, pos: 1, stackEnd: #0
  pc: #10, pos: 1, stackEnd: #0

input: asdefg
       ^
position: 0
 [0] beginCapture 0
 [1] save #20
 [2] save #18
 [3] save #10
 [4] reverse (isScalarDistance: false, #1)
>[5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
 [10] save #14
 [11] reverse (isScalarDistance: false, #1)


--- cycle 16 ---
save points:
  pc: #20, pos: 1, stackEnd: #0
  pc: #18, pos: 1, stackEnd: #0

input: asdefg
       ~^
position: 1
 [4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
>[10] save #14
 [11] reverse (isScalarDistance: false, #1)
 [12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16
 [14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough


--- cycle 17 ---
save points:
  pc: #20, pos: 1, stackEnd: #0
  pc: #18, pos: 1, stackEnd: #0
  pc: #14, pos: 1, stackEnd: #0

input: asdefg
       ~^
position: 1
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
 [10] save #14
>[11] reverse (isScalarDistance: false, #1)
 [12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16
 [14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough
 [17] fail


--- cycle 18 ---
save points:
  pc: #20, pos: 1, stackEnd: #0
  pc: #18, pos: 1, stackEnd: #0
  pc: #14, pos: 1, stackEnd: #0

input: asdefg
       ^
position: 0
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
 [10] save #14
 [11] reverse (isScalarDistance: false, #1)
>[12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16
 [14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough
 [17] fail
 [18] clear


--- cycle 19 ---
save points:
  pc: #20, pos: 1, stackEnd: #0
  pc: #18, pos: 1, stackEnd: #0

input: asdefg
       ~^
position: 1
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
 [10] save #14
 [11] reverse (isScalarDistance: false, #1)
 [12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16
>[14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough
 [17] fail
 [18] clear
 [19] fail
 [20] matchScalar 'd' boundaryCheck: false


--- cycle 20 ---
save points:
  pc: #20, pos: 1, stackEnd: #0
  pc: #18, pos: 1, stackEnd: #0

input: asdefg
       ^
position: 0
 [9] branch #16
 [10] save #14
 [11] reverse (isScalarDistance: false, #1)
 [12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16
 [14] reverse (isScalarDistance: false, #1)
>[15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough
 [17] fail
 [18] clear
 [19] fail
 [20] matchScalar 'd' boundaryCheck: false
 [21] matchScalar 'e' boundaryCheck: false


--- cycle 21 ---
save points:
  pc: #20, pos: 1, stackEnd: #0

input: asdefg
       ~^
position: 1
 [12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16
 [14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough
 [17] fail
>[18] clear
 [19] fail
 [20] matchScalar 'd' boundaryCheck: false
 [21] matchScalar 'e' boundaryCheck: false
 [22] matchScalar 'f' boundaryCheck: false
 [23] matchScalar 'g' boundaryCheck: true
 [24] endCapture 0


--- cycle 22 ---

input: asdefg
       ~^
position: 1
 [13] branch #16
 [14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough
 [17] fail
 [18] clear
>[19] fail
 [20] matchScalar 'd' boundaryCheck: false
 [21] matchScalar 'e' boundaryCheck: false
 [22] matchScalar 'f' boundaryCheck: false
 [23] matchScalar 'g' boundaryCheck: true
 [24] endCapture 0
 [25] accept


--- cycle 23 ---

input: asdefg
       ~^
position: 1
FAIL

--- cycle 24 ---

input: asdefg
       ~~^
position: 2
 [0] beginCapture 0
>[1] save #20
 [2] save #18
 [3] save #10
 [4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)


--- cycle 25 ---
save points:
  pc: #20, pos: 2, stackEnd: #0

input: asdefg
       ~~^
position: 2
 [0] beginCapture 0
 [1] save #20
>[2] save #18
 [3] save #10
 [4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true


--- cycle 26 ---
save points:
  pc: #20, pos: 2, stackEnd: #0
  pc: #18, pos: 2, stackEnd: #0

input: asdefg
       ~~^
position: 2
 [0] beginCapture 0
 [1] save #20
 [2] save #18
>[3] save #10
 [4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16


--- cycle 27 ---
save points:
  pc: #20, pos: 2, stackEnd: #0
  pc: #18, pos: 2, stackEnd: #0
  pc: #10, pos: 2, stackEnd: #0

input: asdefg
       ~~^
position: 2
 [0] beginCapture 0
 [1] save #20
 [2] save #18
 [3] save #10
>[4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
 [10] save #14


--- cycle 28 ---
save points:
  pc: #20, pos: 2, stackEnd: #0
  pc: #18, pos: 2, stackEnd: #0
  pc: #10, pos: 2, stackEnd: #0

input: asdefg
       ~^
position: 1
 [0] beginCapture 0
 [1] save #20
 [2] save #18
 [3] save #10
 [4] reverse (isScalarDistance: false, #1)
>[5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
 [10] save #14
 [11] reverse (isScalarDistance: false, #1)


--- cycle 29 ---
save points:
  pc: #20, pos: 2, stackEnd: #0
  pc: #18, pos: 2, stackEnd: #0
  pc: #10, pos: 2, stackEnd: #0

input: asdefg
       ^
position: 0
 [0] beginCapture 0
 [1] save #20
 [2] save #18
 [3] save #10
 [4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
>[6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
 [10] save #14
 [11] reverse (isScalarDistance: false, #1)
 [12] reverseMatchScalar 'b' boundaryCheck: true


--- cycle 30 ---
save points:
  pc: #20, pos: 2, stackEnd: #0
  pc: #18, pos: 2, stackEnd: #0

input: asdefg
       ~~^
position: 2
 [4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
>[10] save #14
 [11] reverse (isScalarDistance: false, #1)
 [12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16
 [14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough


--- cycle 31 ---
save points:
  pc: #20, pos: 2, stackEnd: #0
  pc: #18, pos: 2, stackEnd: #0
  pc: #14, pos: 2, stackEnd: #0

input: asdefg
       ~~^
position: 2
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
 [10] save #14
>[11] reverse (isScalarDistance: false, #1)
 [12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16
 [14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough
 [17] fail


--- cycle 32 ---
save points:
  pc: #20, pos: 2, stackEnd: #0
  pc: #18, pos: 2, stackEnd: #0
  pc: #14, pos: 2, stackEnd: #0

input: asdefg
       ~^
position: 1
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
 [10] save #14
 [11] reverse (isScalarDistance: false, #1)
>[12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16
 [14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough
 [17] fail
 [18] clear


--- cycle 33 ---
save points:
  pc: #20, pos: 2, stackEnd: #0
  pc: #18, pos: 2, stackEnd: #0

input: asdefg
       ~~^
position: 2
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
 [10] save #14
 [11] reverse (isScalarDistance: false, #1)
 [12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16
>[14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough
 [17] fail
 [18] clear
 [19] fail
 [20] matchScalar 'd' boundaryCheck: false


--- cycle 34 ---
save points:
  pc: #20, pos: 2, stackEnd: #0
  pc: #18, pos: 2, stackEnd: #0

input: asdefg
       ~^
position: 1
 [9] branch #16
 [10] save #14
 [11] reverse (isScalarDistance: false, #1)
 [12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16
 [14] reverse (isScalarDistance: false, #1)
>[15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough
 [17] fail
 [18] clear
 [19] fail
 [20] matchScalar 'd' boundaryCheck: false
 [21] matchScalar 'e' boundaryCheck: false


--- cycle 35 ---
save points:
  pc: #20, pos: 2, stackEnd: #0

input: asdefg
       ~~^
position: 2
 [12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16
 [14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough
 [17] fail
>[18] clear
 [19] fail
 [20] matchScalar 'd' boundaryCheck: false
 [21] matchScalar 'e' boundaryCheck: false
 [22] matchScalar 'f' boundaryCheck: false
 [23] matchScalar 'g' boundaryCheck: true
 [24] endCapture 0


--- cycle 36 ---

input: asdefg
       ~~^
position: 2
 [13] branch #16
 [14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough
 [17] fail
 [18] clear
>[19] fail
 [20] matchScalar 'd' boundaryCheck: false
 [21] matchScalar 'e' boundaryCheck: false
 [22] matchScalar 'f' boundaryCheck: false
 [23] matchScalar 'g' boundaryCheck: true
 [24] endCapture 0
 [25] accept


--- cycle 37 ---

input: asdefg
       ~~^
position: 2
FAIL

--- cycle 38 ---

input: asdefg
       ~~~^
position: 3
 [0] beginCapture 0
>[1] save #20
 [2] save #18
 [3] save #10
 [4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)


--- cycle 39 ---
save points:
  pc: #20, pos: 3, stackEnd: #0

input: asdefg
       ~~~^
position: 3
 [0] beginCapture 0
 [1] save #20
>[2] save #18
 [3] save #10
 [4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true


--- cycle 40 ---
save points:
  pc: #20, pos: 3, stackEnd: #0
  pc: #18, pos: 3, stackEnd: #0

input: asdefg
       ~~~^
position: 3
 [0] beginCapture 0
 [1] save #20
 [2] save #18
>[3] save #10
 [4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16


--- cycle 41 ---
save points:
  pc: #20, pos: 3, stackEnd: #0
  pc: #18, pos: 3, stackEnd: #0
  pc: #10, pos: 3, stackEnd: #0

input: asdefg
       ~~~^
position: 3
 [0] beginCapture 0
 [1] save #20
 [2] save #18
 [3] save #10
>[4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
 [10] save #14


--- cycle 42 ---
save points:
  pc: #20, pos: 3, stackEnd: #0
  pc: #18, pos: 3, stackEnd: #0
  pc: #10, pos: 3, stackEnd: #0

input: asdefg
       ~~^
position: 2
 [0] beginCapture 0
 [1] save #20
 [2] save #18
 [3] save #10
 [4] reverse (isScalarDistance: false, #1)
>[5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
 [10] save #14
 [11] reverse (isScalarDistance: false, #1)


--- cycle 43 ---
save points:
  pc: #20, pos: 3, stackEnd: #0
  pc: #18, pos: 3, stackEnd: #0
  pc: #10, pos: 3, stackEnd: #0

input: asdefg
       ~^
position: 1
 [0] beginCapture 0
 [1] save #20
 [2] save #18
 [3] save #10
 [4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
>[6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
 [10] save #14
 [11] reverse (isScalarDistance: false, #1)
 [12] reverseMatchScalar 'b' boundaryCheck: true


--- cycle 44 ---
save points:
  pc: #20, pos: 3, stackEnd: #0
  pc: #18, pos: 3, stackEnd: #0
  pc: #10, pos: 3, stackEnd: #0

input: asdefg
       ^
position: 0
 [1] save #20
 [2] save #18
 [3] save #10
 [4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
>[7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
 [10] save #14
 [11] reverse (isScalarDistance: false, #1)
 [12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16


--- cycle 45 ---
save points:
  pc: #20, pos: 3, stackEnd: #0
  pc: #18, pos: 3, stackEnd: #0

input: asdefg
       ~~~^
position: 3
 [4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
>[10] save #14
 [11] reverse (isScalarDistance: false, #1)
 [12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16
 [14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough


--- cycle 46 ---
save points:
  pc: #20, pos: 3, stackEnd: #0
  pc: #18, pos: 3, stackEnd: #0
  pc: #14, pos: 3, stackEnd: #0

input: asdefg
       ~~~^
position: 3
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
 [10] save #14
>[11] reverse (isScalarDistance: false, #1)
 [12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16
 [14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough
 [17] fail


--- cycle 47 ---
save points:
  pc: #20, pos: 3, stackEnd: #0
  pc: #18, pos: 3, stackEnd: #0
  pc: #14, pos: 3, stackEnd: #0

input: asdefg
       ~~^
position: 2
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
 [10] save #14
 [11] reverse (isScalarDistance: false, #1)
>[12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16
 [14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough
 [17] fail
 [18] clear


--- cycle 48 ---
save points:
  pc: #20, pos: 3, stackEnd: #0
  pc: #18, pos: 3, stackEnd: #0

input: asdefg
       ~~~^
position: 3
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
 [10] save #14
 [11] reverse (isScalarDistance: false, #1)
 [12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16
>[14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough
 [17] fail
 [18] clear
 [19] fail
 [20] matchScalar 'd' boundaryCheck: false


--- cycle 49 ---
save points:
  pc: #20, pos: 3, stackEnd: #0
  pc: #18, pos: 3, stackEnd: #0

input: asdefg
       ~~^
position: 2
 [9] branch #16
 [10] save #14
 [11] reverse (isScalarDistance: false, #1)
 [12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16
 [14] reverse (isScalarDistance: false, #1)
>[15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough
 [17] fail
 [18] clear
 [19] fail
 [20] matchScalar 'd' boundaryCheck: false
 [21] matchScalar 'e' boundaryCheck: false


--- cycle 50 ---
save points:
  pc: #20, pos: 3, stackEnd: #0

input: asdefg
       ~~~^
position: 3
 [12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16
 [14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough
 [17] fail
>[18] clear
 [19] fail
 [20] matchScalar 'd' boundaryCheck: false
 [21] matchScalar 'e' boundaryCheck: false
 [22] matchScalar 'f' boundaryCheck: false
 [23] matchScalar 'g' boundaryCheck: true
 [24] endCapture 0


--- cycle 51 ---

input: asdefg
       ~~~^
position: 3
 [13] branch #16
 [14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough
 [17] fail
 [18] clear
>[19] fail
 [20] matchScalar 'd' boundaryCheck: false
 [21] matchScalar 'e' boundaryCheck: false
 [22] matchScalar 'f' boundaryCheck: false
 [23] matchScalar 'g' boundaryCheck: true
 [24] endCapture 0
 [25] accept


--- cycle 52 ---

input: asdefg
       ~~~^
position: 3
FAIL

--- cycle 53 ---

input: asdefg
       ~~~~^
position: 4
 [0] beginCapture 0
>[1] save #20
 [2] save #18
 [3] save #10
 [4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)


--- cycle 54 ---
save points:
  pc: #20, pos: 4, stackEnd: #0

input: asdefg
       ~~~~^
position: 4
 [0] beginCapture 0
 [1] save #20
>[2] save #18
 [3] save #10
 [4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true


--- cycle 55 ---
save points:
  pc: #20, pos: 4, stackEnd: #0
  pc: #18, pos: 4, stackEnd: #0

input: asdefg
       ~~~~^
position: 4
 [0] beginCapture 0
 [1] save #20
 [2] save #18
>[3] save #10
 [4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16


--- cycle 56 ---
save points:
  pc: #20, pos: 4, stackEnd: #0
  pc: #18, pos: 4, stackEnd: #0
  pc: #10, pos: 4, stackEnd: #0

input: asdefg
       ~~~~^
position: 4
 [0] beginCapture 0
 [1] save #20
 [2] save #18
 [3] save #10
>[4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
 [10] save #14


--- cycle 57 ---
save points:
  pc: #20, pos: 4, stackEnd: #0
  pc: #18, pos: 4, stackEnd: #0
  pc: #10, pos: 4, stackEnd: #0

input: asdefg
       ~~~^
position: 3
 [0] beginCapture 0
 [1] save #20
 [2] save #18
 [3] save #10
 [4] reverse (isScalarDistance: false, #1)
>[5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
 [10] save #14
 [11] reverse (isScalarDistance: false, #1)


--- cycle 58 ---
save points:
  pc: #20, pos: 4, stackEnd: #0
  pc: #18, pos: 4, stackEnd: #0
  pc: #10, pos: 4, stackEnd: #0

input: asdefg
       ~~^
position: 2
 [0] beginCapture 0
 [1] save #20
 [2] save #18
 [3] save #10
 [4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
>[6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
 [10] save #14
 [11] reverse (isScalarDistance: false, #1)
 [12] reverseMatchScalar 'b' boundaryCheck: true


--- cycle 59 ---
save points:
  pc: #20, pos: 4, stackEnd: #0
  pc: #18, pos: 4, stackEnd: #0

input: asdefg
       ~~~~^
position: 4
 [4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
>[10] save #14
 [11] reverse (isScalarDistance: false, #1)
 [12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16
 [14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough


--- cycle 60 ---
save points:
  pc: #20, pos: 4, stackEnd: #0
  pc: #18, pos: 4, stackEnd: #0
  pc: #14, pos: 4, stackEnd: #0

input: asdefg
       ~~~~^
position: 4
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
 [10] save #14
>[11] reverse (isScalarDistance: false, #1)
 [12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16
 [14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough
 [17] fail


--- cycle 61 ---
save points:
  pc: #20, pos: 4, stackEnd: #0
  pc: #18, pos: 4, stackEnd: #0
  pc: #14, pos: 4, stackEnd: #0

input: asdefg
       ~~~^
position: 3
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
 [10] save #14
 [11] reverse (isScalarDistance: false, #1)
>[12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16
 [14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough
 [17] fail
 [18] clear


--- cycle 62 ---
save points:
  pc: #20, pos: 4, stackEnd: #0
  pc: #18, pos: 4, stackEnd: #0

input: asdefg
       ~~~~^
position: 4
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
 [10] save #14
 [11] reverse (isScalarDistance: false, #1)
 [12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16
>[14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough
 [17] fail
 [18] clear
 [19] fail
 [20] matchScalar 'd' boundaryCheck: false


--- cycle 63 ---
save points:
  pc: #20, pos: 4, stackEnd: #0
  pc: #18, pos: 4, stackEnd: #0

input: asdefg
       ~~~^
position: 3
 [9] branch #16
 [10] save #14
 [11] reverse (isScalarDistance: false, #1)
 [12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16
 [14] reverse (isScalarDistance: false, #1)
>[15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough
 [17] fail
 [18] clear
 [19] fail
 [20] matchScalar 'd' boundaryCheck: false
 [21] matchScalar 'e' boundaryCheck: false


--- cycle 64 ---
save points:
  pc: #20, pos: 4, stackEnd: #0

input: asdefg
       ~~~~^
position: 4
 [12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16
 [14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough
 [17] fail
>[18] clear
 [19] fail
 [20] matchScalar 'd' boundaryCheck: false
 [21] matchScalar 'e' boundaryCheck: false
 [22] matchScalar 'f' boundaryCheck: false
 [23] matchScalar 'g' boundaryCheck: true
 [24] endCapture 0


--- cycle 65 ---

input: asdefg
       ~~~~^
position: 4
 [13] branch #16
 [14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough
 [17] fail
 [18] clear
>[19] fail
 [20] matchScalar 'd' boundaryCheck: false
 [21] matchScalar 'e' boundaryCheck: false
 [22] matchScalar 'f' boundaryCheck: false
 [23] matchScalar 'g' boundaryCheck: true
 [24] endCapture 0
 [25] accept


--- cycle 66 ---

input: asdefg
       ~~~~^
position: 4
FAIL

--- cycle 67 ---

input: asdefg
       ~~~~~^
position: 5
 [0] beginCapture 0
>[1] save #20
 [2] save #18
 [3] save #10
 [4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)


--- cycle 68 ---
save points:
  pc: #20, pos: 5, stackEnd: #0

input: asdefg
       ~~~~~^
position: 5
 [0] beginCapture 0
 [1] save #20
>[2] save #18
 [3] save #10
 [4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true


--- cycle 69 ---
save points:
  pc: #20, pos: 5, stackEnd: #0
  pc: #18, pos: 5, stackEnd: #0

input: asdefg
       ~~~~~^
position: 5
 [0] beginCapture 0
 [1] save #20
 [2] save #18
>[3] save #10
 [4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16


--- cycle 70 ---
save points:
  pc: #20, pos: 5, stackEnd: #0
  pc: #18, pos: 5, stackEnd: #0
  pc: #10, pos: 5, stackEnd: #0

input: asdefg
       ~~~~~^
position: 5
 [0] beginCapture 0
 [1] save #20
 [2] save #18
 [3] save #10
>[4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
 [10] save #14


--- cycle 71 ---
save points:
  pc: #20, pos: 5, stackEnd: #0
  pc: #18, pos: 5, stackEnd: #0
  pc: #10, pos: 5, stackEnd: #0

input: asdefg
       ~~~~^
position: 4
 [0] beginCapture 0
 [1] save #20
 [2] save #18
 [3] save #10
 [4] reverse (isScalarDistance: false, #1)
>[5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
 [10] save #14
 [11] reverse (isScalarDistance: false, #1)


--- cycle 72 ---
save points:
  pc: #20, pos: 5, stackEnd: #0
  pc: #18, pos: 5, stackEnd: #0
  pc: #10, pos: 5, stackEnd: #0

input: asdefg
       ~~~^
position: 3
 [0] beginCapture 0
 [1] save #20
 [2] save #18
 [3] save #10
 [4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
>[6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
 [10] save #14
 [11] reverse (isScalarDistance: false, #1)
 [12] reverseMatchScalar 'b' boundaryCheck: true


--- cycle 73 ---
save points:
  pc: #20, pos: 5, stackEnd: #0
  pc: #18, pos: 5, stackEnd: #0

input: asdefg
       ~~~~~^
position: 5
 [4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
>[10] save #14
 [11] reverse (isScalarDistance: false, #1)
 [12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16
 [14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough


--- cycle 74 ---
save points:
  pc: #20, pos: 5, stackEnd: #0
  pc: #18, pos: 5, stackEnd: #0
  pc: #14, pos: 5, stackEnd: #0

input: asdefg
       ~~~~~^
position: 5
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
 [10] save #14
>[11] reverse (isScalarDistance: false, #1)
 [12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16
 [14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough
 [17] fail


--- cycle 75 ---
save points:
  pc: #20, pos: 5, stackEnd: #0
  pc: #18, pos: 5, stackEnd: #0
  pc: #14, pos: 5, stackEnd: #0

input: asdefg
       ~~~~^
position: 4
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
 [10] save #14
 [11] reverse (isScalarDistance: false, #1)
>[12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16
 [14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough
 [17] fail
 [18] clear


--- cycle 76 ---
save points:
  pc: #20, pos: 5, stackEnd: #0
  pc: #18, pos: 5, stackEnd: #0

input: asdefg
       ~~~~~^
position: 5
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
 [10] save #14
 [11] reverse (isScalarDistance: false, #1)
 [12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16
>[14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough
 [17] fail
 [18] clear
 [19] fail
 [20] matchScalar 'd' boundaryCheck: false


--- cycle 77 ---
save points:
  pc: #20, pos: 5, stackEnd: #0
  pc: #18, pos: 5, stackEnd: #0

input: asdefg
       ~~~~^
position: 4
 [9] branch #16
 [10] save #14
 [11] reverse (isScalarDistance: false, #1)
 [12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16
 [14] reverse (isScalarDistance: false, #1)
>[15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough
 [17] fail
 [18] clear
 [19] fail
 [20] matchScalar 'd' boundaryCheck: false
 [21] matchScalar 'e' boundaryCheck: false


--- cycle 78 ---
save points:
  pc: #20, pos: 5, stackEnd: #0

input: asdefg
       ~~~~~^
position: 5
 [12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16
 [14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough
 [17] fail
>[18] clear
 [19] fail
 [20] matchScalar 'd' boundaryCheck: false
 [21] matchScalar 'e' boundaryCheck: false
 [22] matchScalar 'f' boundaryCheck: false
 [23] matchScalar 'g' boundaryCheck: true
 [24] endCapture 0


--- cycle 79 ---

input: asdefg
       ~~~~~^
position: 5
 [13] branch #16
 [14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough
 [17] fail
 [18] clear
>[19] fail
 [20] matchScalar 'd' boundaryCheck: false
 [21] matchScalar 'e' boundaryCheck: false
 [22] matchScalar 'f' boundaryCheck: false
 [23] matchScalar 'g' boundaryCheck: true
 [24] endCapture 0
 [25] accept


--- cycle 80 ---

input: asdefg
       ~~~~~^
position: 5
FAIL

--- cycle 81 ---

input: asdefg
       ......^
position: 6
 [0] beginCapture 0
>[1] save #20
 [2] save #18
 [3] save #10
 [4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)


--- cycle 82 ---
save points:
  pc: #20, pos: 6, stackEnd: #0

input: asdefg
       ......^
position: 6
 [0] beginCapture 0
 [1] save #20
>[2] save #18
 [3] save #10
 [4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true


--- cycle 83 ---
save points:
  pc: #20, pos: 6, stackEnd: #0
  pc: #18, pos: 6, stackEnd: #0

input: asdefg
       ......^
position: 6
 [0] beginCapture 0
 [1] save #20
 [2] save #18
>[3] save #10
 [4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16


--- cycle 84 ---
save points:
  pc: #20, pos: 6, stackEnd: #0
  pc: #18, pos: 6, stackEnd: #0
  pc: #10, pos: 6, stackEnd: #0

input: asdefg
       ......^
position: 6
 [0] beginCapture 0
 [1] save #20
 [2] save #18
 [3] save #10
>[4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
 [10] save #14


--- cycle 85 ---
save points:
  pc: #20, pos: 6, stackEnd: #0
  pc: #18, pos: 6, stackEnd: #0
  pc: #10, pos: 6, stackEnd: #0

input: asdefg
       ~~~~~^
position: 5
 [0] beginCapture 0
 [1] save #20
 [2] save #18
 [3] save #10
 [4] reverse (isScalarDistance: false, #1)
>[5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
 [10] save #14
 [11] reverse (isScalarDistance: false, #1)


--- cycle 86 ---
save points:
  pc: #20, pos: 6, stackEnd: #0
  pc: #18, pos: 6, stackEnd: #0
  pc: #10, pos: 6, stackEnd: #0

input: asdefg
       ~~~~^
position: 4
 [0] beginCapture 0
 [1] save #20
 [2] save #18
 [3] save #10
 [4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
>[6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
 [10] save #14
 [11] reverse (isScalarDistance: false, #1)
 [12] reverseMatchScalar 'b' boundaryCheck: true


--- cycle 87 ---
save points:
  pc: #20, pos: 6, stackEnd: #0
  pc: #18, pos: 6, stackEnd: #0

input: asdefg
       ......^
position: 6
 [4] reverse (isScalarDistance: false, #1)
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
>[10] save #14
 [11] reverse (isScalarDistance: false, #1)
 [12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16
 [14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough


--- cycle 88 ---
save points:
  pc: #20, pos: 6, stackEnd: #0
  pc: #18, pos: 6, stackEnd: #0
  pc: #14, pos: 6, stackEnd: #0

input: asdefg
       ......^
position: 6
 [5] reverse (isScalarDistance: false, #1)
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
 [10] save #14
>[11] reverse (isScalarDistance: false, #1)
 [12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16
 [14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough
 [17] fail


--- cycle 89 ---
save points:
  pc: #20, pos: 6, stackEnd: #0
  pc: #18, pos: 6, stackEnd: #0
  pc: #14, pos: 6, stackEnd: #0

input: asdefg
       ~~~~~^
position: 5
 [6] reverseMatchScalar 's' boundaryCheck: false
 [7] reverse (isScalarDistance: false, #1)
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
 [10] save #14
 [11] reverse (isScalarDistance: false, #1)
>[12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16
 [14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough
 [17] fail
 [18] clear


--- cycle 90 ---
save points:
  pc: #20, pos: 6, stackEnd: #0
  pc: #18, pos: 6, stackEnd: #0

input: asdefg
       ......^
position: 6
 [8] reverseMatchScalar 'a' boundaryCheck: true
 [9] branch #16
 [10] save #14
 [11] reverse (isScalarDistance: false, #1)
 [12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16
>[14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough
 [17] fail
 [18] clear
 [19] fail
 [20] matchScalar 'd' boundaryCheck: false


--- cycle 91 ---
save points:
  pc: #20, pos: 6, stackEnd: #0
  pc: #18, pos: 6, stackEnd: #0

input: asdefg
       ~~~~~^
position: 5
 [9] branch #16
 [10] save #14
 [11] reverse (isScalarDistance: false, #1)
 [12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16
 [14] reverse (isScalarDistance: false, #1)
>[15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough
 [17] fail
 [18] clear
 [19] fail
 [20] matchScalar 'd' boundaryCheck: false
 [21] matchScalar 'e' boundaryCheck: false


--- cycle 92 ---
save points:
  pc: #20, pos: 6, stackEnd: #0

input: asdefg
       ......^
position: 6
 [12] reverseMatchScalar 'b' boundaryCheck: true
 [13] branch #16
 [14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough
 [17] fail
>[18] clear
 [19] fail
 [20] matchScalar 'd' boundaryCheck: false
 [21] matchScalar 'e' boundaryCheck: false
 [22] matchScalar 'f' boundaryCheck: false
 [23] matchScalar 'g' boundaryCheck: true
 [24] endCapture 0


--- cycle 93 ---

input: asdefg
       ......^
position: 6
 [13] branch #16
 [14] reverse (isScalarDistance: false, #1)
 [15] reverseMatchScalar 'c' boundaryCheck: true
 [16] clearThrough
 [17] fail
 [18] clear
>[19] fail
 [20] matchScalar 'd' boundaryCheck: false
 [21] matchScalar 'e' boundaryCheck: false
 [22] matchScalar 'f' boundaryCheck: false
 [23] matchScalar 'g' boundaryCheck: true
 [24] endCapture 0
 [25] accept


--- cycle 94 ---

input: asdefg
       ......^
position: 6
FAIL
===
Total cycle count: 94
Backtracks: 21
Resets: 6
Instructions:
> save: 28
> reverse: 28
> reverseMatchScalar: 17
> fail: 14
> clear: 7
> beginCapture: 1
===
```
{{< /details >}}

<br/><br/>
#### Test case #3

```text
Inputs: "123-_+/-789suffix"
Regex: /(?<=\d{1,3}-.{1,3}-\d{1,3})suffix/
```

Problem: This regex isn't matching the input

Solution: It looks like this expression uses builtins which I haven't addressed at all yet. Sounds like a problem for future Jacob.

{{< details title="Test case #3 trace output" >}}
```text
--- cycle 0 ---

input: 123-_+/-789suffix
       ^
position: 0
>[0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2


--- cycle 1 ---

input: 123-_+/-789suffix
       ^
position: 0
 [0] beginCapture 0
>[1] save #14
 [2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)


--- cycle 2 ---
save points:
  pc: #14, pos: 0, stackEnd: #0

input: 123-_+/-789suffix
       ^
position: 0
 [0] beginCapture 0
 [1] save #14
>[2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true


--- cycle 3 ---
save points:
  pc: #14, pos: 0, stackEnd: #0
  pc: #12, pos: 0, stackEnd: #0

input: 123-_+/-789suffix
       ^
position: 0
 [0] beginCapture 0
 [1] save #14
 [2] save #12
>[3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2


--- cycle 4 ---
save points:
  pc: #14, pos: 0, stackEnd: #0
  pc: #12, pos: 0, stackEnd: #0
  pc: #4, pos: 1...2, stackEnd: #0

input: 123-_+/-789suffix
       ~~~^
position: 3
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] quantify builtin 1 2
>[4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough


--- cycle 5 ---
save points:
  pc: #14, pos: 0, stackEnd: #0
  pc: #12, pos: 0, stackEnd: #0
  pc: #4, pos: 1...2, stackEnd: #0

input: 123-_+/-789suffix
       ~~^
position: 2
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
>[5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail


--- cycle 6 ---
save points:
  pc: #14, pos: 0, stackEnd: #0
  pc: #12, pos: 0, stackEnd: #0
  pc: #4, pos: 2, stackEnd: #0

input: 123-_+/-789suffix
       ~~^
position: 2
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] quantify builtin 1 2
>[4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough


--- cycle 7 ---
save points:
  pc: #14, pos: 0, stackEnd: #0
  pc: #12, pos: 0, stackEnd: #0
  pc: #4, pos: 2, stackEnd: #0

input: 123-_+/-789suffix
       ~^
position: 1
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
>[5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail


--- cycle 8 ---
save points:
  pc: #14, pos: 0, stackEnd: #0
  pc: #12, pos: 0, stackEnd: #0
  pc: #4, pos: 1, stackEnd: #0

input: 123-_+/-789suffix
       ~^
position: 1
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] quantify builtin 1 2
>[4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough


--- cycle 9 ---
save points:
  pc: #14, pos: 0, stackEnd: #0
  pc: #12, pos: 0, stackEnd: #0
  pc: #4, pos: 1, stackEnd: #0

input: 123-_+/-789suffix
       ^
position: 0
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
>[5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail


--- cycle 10 ---
save points:
  pc: #14, pos: 0, stackEnd: #0
  pc: #12, pos: 0, stackEnd: #0

input: 123-_+/-789suffix
       ~^
position: 1
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] quantify builtin 1 2
>[4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough


--- cycle 11 ---
save points:
  pc: #14, pos: 0, stackEnd: #0
  pc: #12, pos: 0, stackEnd: #0

input: 123-_+/-789suffix
       ^
position: 0
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
>[5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail


--- cycle 12 ---
save points:
  pc: #14, pos: 0, stackEnd: #0

input: 123-_+/-789suffix
       ^
position: 0
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail
>[12] clear
 [13] fail
 [14] matchScalar 's' boundaryCheck: false
 [15] matchScalar 'u' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'f' boundaryCheck: false
 [18] matchScalar 'i' boundaryCheck: false


--- cycle 13 ---

input: 123-_+/-789suffix
       ^
position: 0
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail
 [12] clear
>[13] fail
 [14] matchScalar 's' boundaryCheck: false
 [15] matchScalar 'u' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'f' boundaryCheck: false
 [18] matchScalar 'i' boundaryCheck: false
 [19] matchScalar 'x' boundaryCheck: true


--- cycle 14 ---

input: 123-_+/-789suffix
       ^
position: 0
FAIL

--- cycle 15 ---

input: 123-_+/-789suffix
       ~^
position: 1
 [0] beginCapture 0
>[1] save #14
 [2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)


--- cycle 16 ---
save points:
  pc: #14, pos: 1, stackEnd: #0

input: 123-_+/-789suffix
       ~^
position: 1
 [0] beginCapture 0
 [1] save #14
>[2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true


--- cycle 17 ---
save points:
  pc: #14, pos: 1, stackEnd: #0
  pc: #12, pos: 1, stackEnd: #0

input: 123-_+/-789suffix
       ~^
position: 1
 [0] beginCapture 0
 [1] save #14
 [2] save #12
>[3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2


--- cycle 18 ---
save points:
  pc: #14, pos: 1, stackEnd: #0
  pc: #12, pos: 1, stackEnd: #0
  pc: #4, pos: 2...2, stackEnd: #0

input: 123-_+/-789suffix
       ~~~^
position: 3
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] quantify builtin 1 2
>[4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough


--- cycle 19 ---
save points:
  pc: #14, pos: 1, stackEnd: #0
  pc: #12, pos: 1, stackEnd: #0
  pc: #4, pos: 2...2, stackEnd: #0

input: 123-_+/-789suffix
       ~~^
position: 2
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
>[5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail


--- cycle 20 ---
save points:
  pc: #14, pos: 1, stackEnd: #0
  pc: #12, pos: 1, stackEnd: #0
  pc: #4, pos: 2, stackEnd: #0

input: 123-_+/-789suffix
       ~~^
position: 2
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] quantify builtin 1 2
>[4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough


--- cycle 21 ---
save points:
  pc: #14, pos: 1, stackEnd: #0
  pc: #12, pos: 1, stackEnd: #0
  pc: #4, pos: 2, stackEnd: #0

input: 123-_+/-789suffix
       ~^
position: 1
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
>[5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail


--- cycle 22 ---
save points:
  pc: #14, pos: 1, stackEnd: #0
  pc: #12, pos: 1, stackEnd: #0

input: 123-_+/-789suffix
       ~~^
position: 2
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] quantify builtin 1 2
>[4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough


--- cycle 23 ---
save points:
  pc: #14, pos: 1, stackEnd: #0
  pc: #12, pos: 1, stackEnd: #0

input: 123-_+/-789suffix
       ~^
position: 1
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
>[5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail


--- cycle 24 ---
save points:
  pc: #14, pos: 1, stackEnd: #0

input: 123-_+/-789suffix
       ~^
position: 1
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail
>[12] clear
 [13] fail
 [14] matchScalar 's' boundaryCheck: false
 [15] matchScalar 'u' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'f' boundaryCheck: false
 [18] matchScalar 'i' boundaryCheck: false


--- cycle 25 ---

input: 123-_+/-789suffix
       ~^
position: 1
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail
 [12] clear
>[13] fail
 [14] matchScalar 's' boundaryCheck: false
 [15] matchScalar 'u' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'f' boundaryCheck: false
 [18] matchScalar 'i' boundaryCheck: false
 [19] matchScalar 'x' boundaryCheck: true


--- cycle 26 ---

input: 123-_+/-789suffix
       ~^
position: 1
FAIL

--- cycle 27 ---

input: 123-_+/-789suffix
       ~~^
position: 2
 [0] beginCapture 0
>[1] save #14
 [2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)


--- cycle 28 ---
save points:
  pc: #14, pos: 2, stackEnd: #0

input: 123-_+/-789suffix
       ~~^
position: 2
 [0] beginCapture 0
 [1] save #14
>[2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true


--- cycle 29 ---
save points:
  pc: #14, pos: 2, stackEnd: #0
  pc: #12, pos: 2, stackEnd: #0

input: 123-_+/-789suffix
       ~~^
position: 2
 [0] beginCapture 0
 [1] save #14
 [2] save #12
>[3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2


--- cycle 30 ---
save points:
  pc: #14, pos: 2, stackEnd: #0
  pc: #12, pos: 2, stackEnd: #0

input: 123-_+/-789suffix
       ~~~^
position: 3
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] quantify builtin 1 2
>[4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough


--- cycle 31 ---
save points:
  pc: #14, pos: 2, stackEnd: #0
  pc: #12, pos: 2, stackEnd: #0

input: 123-_+/-789suffix
       ~~^
position: 2
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
>[5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail


--- cycle 32 ---
save points:
  pc: #14, pos: 2, stackEnd: #0

input: 123-_+/-789suffix
       ~~^
position: 2
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail
>[12] clear
 [13] fail
 [14] matchScalar 's' boundaryCheck: false
 [15] matchScalar 'u' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'f' boundaryCheck: false
 [18] matchScalar 'i' boundaryCheck: false


--- cycle 33 ---

input: 123-_+/-789suffix
       ~~^
position: 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail
 [12] clear
>[13] fail
 [14] matchScalar 's' boundaryCheck: false
 [15] matchScalar 'u' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'f' boundaryCheck: false
 [18] matchScalar 'i' boundaryCheck: false
 [19] matchScalar 'x' boundaryCheck: true


--- cycle 34 ---

input: 123-_+/-789suffix
       ~~^
position: 2
FAIL

--- cycle 35 ---

input: 123-_+/-789suffix
       ~~~^
position: 3
 [0] beginCapture 0
>[1] save #14
 [2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)


--- cycle 36 ---
save points:
  pc: #14, pos: 3, stackEnd: #0

input: 123-_+/-789suffix
       ~~~^
position: 3
 [0] beginCapture 0
 [1] save #14
>[2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true


--- cycle 37 ---
save points:
  pc: #14, pos: 3, stackEnd: #0
  pc: #12, pos: 3, stackEnd: #0

input: 123-_+/-789suffix
       ~~~^
position: 3
 [0] beginCapture 0
 [1] save #14
 [2] save #12
>[3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2


--- cycle 38 ---
save points:
  pc: #14, pos: 3, stackEnd: #0

input: 123-_+/-789suffix
       ~~~^
position: 3
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail
>[12] clear
 [13] fail
 [14] matchScalar 's' boundaryCheck: false
 [15] matchScalar 'u' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'f' boundaryCheck: false
 [18] matchScalar 'i' boundaryCheck: false


--- cycle 39 ---

input: 123-_+/-789suffix
       ~~~^
position: 3
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail
 [12] clear
>[13] fail
 [14] matchScalar 's' boundaryCheck: false
 [15] matchScalar 'u' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'f' boundaryCheck: false
 [18] matchScalar 'i' boundaryCheck: false
 [19] matchScalar 'x' boundaryCheck: true


--- cycle 40 ---

input: 123-_+/-789suffix
       ~~~^
position: 3
FAIL

--- cycle 41 ---

input: 123-_+/-789suffix
       ~~~~^
position: 4
 [0] beginCapture 0
>[1] save #14
 [2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)


--- cycle 42 ---
save points:
  pc: #14, pos: 4, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~^
position: 4
 [0] beginCapture 0
 [1] save #14
>[2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true


--- cycle 43 ---
save points:
  pc: #14, pos: 4, stackEnd: #0
  pc: #12, pos: 4, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~^
position: 4
 [0] beginCapture 0
 [1] save #14
 [2] save #12
>[3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2


--- cycle 44 ---
save points:
  pc: #14, pos: 4, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~^
position: 4
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail
>[12] clear
 [13] fail
 [14] matchScalar 's' boundaryCheck: false
 [15] matchScalar 'u' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'f' boundaryCheck: false
 [18] matchScalar 'i' boundaryCheck: false


--- cycle 45 ---

input: 123-_+/-789suffix
       ~~~~^
position: 4
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail
 [12] clear
>[13] fail
 [14] matchScalar 's' boundaryCheck: false
 [15] matchScalar 'u' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'f' boundaryCheck: false
 [18] matchScalar 'i' boundaryCheck: false
 [19] matchScalar 'x' boundaryCheck: true


--- cycle 46 ---

input: 123-_+/-789suffix
       ~~~~^
position: 4
FAIL

--- cycle 47 ---

input: 123-_+/-789suffix
       ~~~~~^
position: 5
 [0] beginCapture 0
>[1] save #14
 [2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)


--- cycle 48 ---
save points:
  pc: #14, pos: 5, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~^
position: 5
 [0] beginCapture 0
 [1] save #14
>[2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true


--- cycle 49 ---
save points:
  pc: #14, pos: 5, stackEnd: #0
  pc: #12, pos: 5, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~^
position: 5
 [0] beginCapture 0
 [1] save #14
 [2] save #12
>[3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2


--- cycle 50 ---
save points:
  pc: #14, pos: 5, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~^
position: 5
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail
>[12] clear
 [13] fail
 [14] matchScalar 's' boundaryCheck: false
 [15] matchScalar 'u' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'f' boundaryCheck: false
 [18] matchScalar 'i' boundaryCheck: false


--- cycle 51 ---

input: 123-_+/-789suffix
       ~~~~~^
position: 5
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail
 [12] clear
>[13] fail
 [14] matchScalar 's' boundaryCheck: false
 [15] matchScalar 'u' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'f' boundaryCheck: false
 [18] matchScalar 'i' boundaryCheck: false
 [19] matchScalar 'x' boundaryCheck: true


--- cycle 52 ---

input: 123-_+/-789suffix
       ~~~~~^
position: 5
FAIL

--- cycle 53 ---

input: 123-_+/-789suffix
       ~~~~~~^
position: 6
 [0] beginCapture 0
>[1] save #14
 [2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)


--- cycle 54 ---
save points:
  pc: #14, pos: 6, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~^
position: 6
 [0] beginCapture 0
 [1] save #14
>[2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true


--- cycle 55 ---
save points:
  pc: #14, pos: 6, stackEnd: #0
  pc: #12, pos: 6, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~^
position: 6
 [0] beginCapture 0
 [1] save #14
 [2] save #12
>[3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2


--- cycle 56 ---
save points:
  pc: #14, pos: 6, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~^
position: 6
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail
>[12] clear
 [13] fail
 [14] matchScalar 's' boundaryCheck: false
 [15] matchScalar 'u' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'f' boundaryCheck: false
 [18] matchScalar 'i' boundaryCheck: false


--- cycle 57 ---

input: 123-_+/-789suffix
       ~~~~~~^
position: 6
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail
 [12] clear
>[13] fail
 [14] matchScalar 's' boundaryCheck: false
 [15] matchScalar 'u' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'f' boundaryCheck: false
 [18] matchScalar 'i' boundaryCheck: false
 [19] matchScalar 'x' boundaryCheck: true


--- cycle 58 ---

input: 123-_+/-789suffix
       ~~~~~~^
position: 6
FAIL

--- cycle 59 ---

input: 123-_+/-789suffix
       ~~~~~~~^
position: 7
 [0] beginCapture 0
>[1] save #14
 [2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)


--- cycle 60 ---
save points:
  pc: #14, pos: 7, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~^
position: 7
 [0] beginCapture 0
 [1] save #14
>[2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true


--- cycle 61 ---
save points:
  pc: #14, pos: 7, stackEnd: #0
  pc: #12, pos: 7, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~^
position: 7
 [0] beginCapture 0
 [1] save #14
 [2] save #12
>[3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2


--- cycle 62 ---
save points:
  pc: #14, pos: 7, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~^
position: 7
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail
>[12] clear
 [13] fail
 [14] matchScalar 's' boundaryCheck: false
 [15] matchScalar 'u' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'f' boundaryCheck: false
 [18] matchScalar 'i' boundaryCheck: false


--- cycle 63 ---

input: 123-_+/-789suffix
       ~~~~~~~^
position: 7
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail
 [12] clear
>[13] fail
 [14] matchScalar 's' boundaryCheck: false
 [15] matchScalar 'u' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'f' boundaryCheck: false
 [18] matchScalar 'i' boundaryCheck: false
 [19] matchScalar 'x' boundaryCheck: true


--- cycle 64 ---

input: 123-_+/-789suffix
       ~~~~~~~^
position: 7
FAIL

--- cycle 65 ---

input: 123-_+/-789suffix
       ~~~~~~~~^
position: 8
 [0] beginCapture 0
>[1] save #14
 [2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)


--- cycle 66 ---
save points:
  pc: #14, pos: 8, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~^
position: 8
 [0] beginCapture 0
 [1] save #14
>[2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true


--- cycle 67 ---
save points:
  pc: #14, pos: 8, stackEnd: #0
  pc: #12, pos: 8, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~^
position: 8
 [0] beginCapture 0
 [1] save #14
 [2] save #12
>[3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2


--- cycle 68 ---
save points:
  pc: #14, pos: 8, stackEnd: #0
  pc: #12, pos: 8, stackEnd: #0
  pc: #4, pos: 9...10, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~~~~^
position: 11
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] quantify builtin 1 2
>[4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough


--- cycle 69 ---
save points:
  pc: #14, pos: 8, stackEnd: #0
  pc: #12, pos: 8, stackEnd: #0
  pc: #4, pos: 9...10, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~~~^
position: 10
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
>[5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail


--- cycle 70 ---
save points:
  pc: #14, pos: 8, stackEnd: #0
  pc: #12, pos: 8, stackEnd: #0
  pc: #4, pos: 10, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~~~^
position: 10
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] quantify builtin 1 2
>[4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough


--- cycle 71 ---
save points:
  pc: #14, pos: 8, stackEnd: #0
  pc: #12, pos: 8, stackEnd: #0
  pc: #4, pos: 10, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~~^
position: 9
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
>[5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail


--- cycle 72 ---
save points:
  pc: #14, pos: 8, stackEnd: #0
  pc: #12, pos: 8, stackEnd: #0
  pc: #4, pos: 9, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~~^
position: 9
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] quantify builtin 1 2
>[4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough


--- cycle 73 ---
save points:
  pc: #14, pos: 8, stackEnd: #0
  pc: #12, pos: 8, stackEnd: #0
  pc: #4, pos: 9, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~^
position: 8
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
>[5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail


--- cycle 74 ---
save points:
  pc: #14, pos: 8, stackEnd: #0
  pc: #12, pos: 8, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~~^
position: 9
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] quantify builtin 1 2
>[4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough


--- cycle 75 ---
save points:
  pc: #14, pos: 8, stackEnd: #0
  pc: #12, pos: 8, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~^
position: 8
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
>[5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail


--- cycle 76 ---
save points:
  pc: #14, pos: 8, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~^
position: 8
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail
>[12] clear
 [13] fail
 [14] matchScalar 's' boundaryCheck: false
 [15] matchScalar 'u' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'f' boundaryCheck: false
 [18] matchScalar 'i' boundaryCheck: false


--- cycle 77 ---

input: 123-_+/-789suffix
       ~~~~~~~~^
position: 8
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail
 [12] clear
>[13] fail
 [14] matchScalar 's' boundaryCheck: false
 [15] matchScalar 'u' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'f' boundaryCheck: false
 [18] matchScalar 'i' boundaryCheck: false
 [19] matchScalar 'x' boundaryCheck: true


--- cycle 78 ---

input: 123-_+/-789suffix
       ~~~~~~~~^
position: 8
FAIL

--- cycle 79 ---

input: 123-_+/-789suffix
       ~~~~~~~~~^
position: 9
 [0] beginCapture 0
>[1] save #14
 [2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)


--- cycle 80 ---
save points:
  pc: #14, pos: 9, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~~^
position: 9
 [0] beginCapture 0
 [1] save #14
>[2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true


--- cycle 81 ---
save points:
  pc: #14, pos: 9, stackEnd: #0
  pc: #12, pos: 9, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~~^
position: 9
 [0] beginCapture 0
 [1] save #14
 [2] save #12
>[3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2


--- cycle 82 ---
save points:
  pc: #14, pos: 9, stackEnd: #0
  pc: #12, pos: 9, stackEnd: #0
  pc: #4, pos: 10...10, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~~~~^
position: 11
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] quantify builtin 1 2
>[4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough


--- cycle 83 ---
save points:
  pc: #14, pos: 9, stackEnd: #0
  pc: #12, pos: 9, stackEnd: #0
  pc: #4, pos: 10...10, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~~~^
position: 10
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
>[5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail


--- cycle 84 ---
save points:
  pc: #14, pos: 9, stackEnd: #0
  pc: #12, pos: 9, stackEnd: #0
  pc: #4, pos: 10, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~~~^
position: 10
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] quantify builtin 1 2
>[4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough


--- cycle 85 ---
save points:
  pc: #14, pos: 9, stackEnd: #0
  pc: #12, pos: 9, stackEnd: #0
  pc: #4, pos: 10, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~~^
position: 9
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
>[5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail


--- cycle 86 ---
save points:
  pc: #14, pos: 9, stackEnd: #0
  pc: #12, pos: 9, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~~~^
position: 10
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] quantify builtin 1 2
>[4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough


--- cycle 87 ---
save points:
  pc: #14, pos: 9, stackEnd: #0
  pc: #12, pos: 9, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~~^
position: 9
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
>[5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail


--- cycle 88 ---
save points:
  pc: #14, pos: 9, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~~^
position: 9
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail
>[12] clear
 [13] fail
 [14] matchScalar 's' boundaryCheck: false
 [15] matchScalar 'u' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'f' boundaryCheck: false
 [18] matchScalar 'i' boundaryCheck: false


--- cycle 89 ---

input: 123-_+/-789suffix
       ~~~~~~~~~^
position: 9
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail
 [12] clear
>[13] fail
 [14] matchScalar 's' boundaryCheck: false
 [15] matchScalar 'u' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'f' boundaryCheck: false
 [18] matchScalar 'i' boundaryCheck: false
 [19] matchScalar 'x' boundaryCheck: true


--- cycle 90 ---

input: 123-_+/-789suffix
       ~~~~~~~~~^
position: 9
FAIL

--- cycle 91 ---

input: 123-_+/-789suffix
       ~~~~~~~~~~^
position: 10
 [0] beginCapture 0
>[1] save #14
 [2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)


--- cycle 92 ---
save points:
  pc: #14, pos: 10, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~~~^
position: 10
 [0] beginCapture 0
 [1] save #14
>[2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true


--- cycle 93 ---
save points:
  pc: #14, pos: 10, stackEnd: #0
  pc: #12, pos: 10, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~~~^
position: 10
 [0] beginCapture 0
 [1] save #14
 [2] save #12
>[3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2


--- cycle 94 ---
save points:
  pc: #14, pos: 10, stackEnd: #0
  pc: #12, pos: 10, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~~~~^
position: 11
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] quantify builtin 1 2
>[4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough


--- cycle 95 ---
save points:
  pc: #14, pos: 10, stackEnd: #0
  pc: #12, pos: 10, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~~~^
position: 10
 [0] beginCapture 0
 [1] save #14
 [2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
>[5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail


--- cycle 96 ---
save points:
  pc: #14, pos: 10, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~~~^
position: 10
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail
>[12] clear
 [13] fail
 [14] matchScalar 's' boundaryCheck: false
 [15] matchScalar 'u' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'f' boundaryCheck: false
 [18] matchScalar 'i' boundaryCheck: false


--- cycle 97 ---

input: 123-_+/-789suffix
       ~~~~~~~~~~^
position: 10
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail
 [12] clear
>[13] fail
 [14] matchScalar 's' boundaryCheck: false
 [15] matchScalar 'u' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'f' boundaryCheck: false
 [18] matchScalar 'i' boundaryCheck: false
 [19] matchScalar 'x' boundaryCheck: true


--- cycle 98 ---

input: 123-_+/-789suffix
       ~~~~~~~~~~^
position: 10
FAIL

--- cycle 99 ---

input: 123-_+/-789suffix
       ~~~~~~~~~~~^
position: 11
 [0] beginCapture 0
>[1] save #14
 [2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)


--- cycle 100 ---
save points:
  pc: #14, pos: 11, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~~~~^
position: 11
 [0] beginCapture 0
 [1] save #14
>[2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true


--- cycle 101 ---
save points:
  pc: #14, pos: 11, stackEnd: #0
  pc: #12, pos: 11, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~~~~^
position: 11
 [0] beginCapture 0
 [1] save #14
 [2] save #12
>[3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2


--- cycle 102 ---
save points:
  pc: #14, pos: 11, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~~~~^
position: 11
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail
>[12] clear
 [13] fail
 [14] matchScalar 's' boundaryCheck: false
 [15] matchScalar 'u' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'f' boundaryCheck: false
 [18] matchScalar 'i' boundaryCheck: false


--- cycle 103 ---

input: 123-_+/-789suffix
       ~~~~~~~~~~~^
position: 11
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail
 [12] clear
>[13] fail
 [14] matchScalar 's' boundaryCheck: false
 [15] matchScalar 'u' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'f' boundaryCheck: false
 [18] matchScalar 'i' boundaryCheck: false
 [19] matchScalar 'x' boundaryCheck: true


--- cycle 104 ---

input: 123-_+/-789suffix
       ~~~~~~~~~~~^
position: 11
FAIL

--- cycle 105 ---

input: 123-_+/-789suffix
       ~~~~~~~~~~~~^
position: 12
 [0] beginCapture 0
>[1] save #14
 [2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)


--- cycle 106 ---
save points:
  pc: #14, pos: 12, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~~~~~^
position: 12
 [0] beginCapture 0
 [1] save #14
>[2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true


--- cycle 107 ---
save points:
  pc: #14, pos: 12, stackEnd: #0
  pc: #12, pos: 12, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~~~~~^
position: 12
 [0] beginCapture 0
 [1] save #14
 [2] save #12
>[3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2


--- cycle 108 ---
save points:
  pc: #14, pos: 12, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~~~~~^
position: 12
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail
>[12] clear
 [13] fail
 [14] matchScalar 's' boundaryCheck: false
 [15] matchScalar 'u' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'f' boundaryCheck: false
 [18] matchScalar 'i' boundaryCheck: false


--- cycle 109 ---

input: 123-_+/-789suffix
       ~~~~~~~~~~~~^
position: 12
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail
 [12] clear
>[13] fail
 [14] matchScalar 's' boundaryCheck: false
 [15] matchScalar 'u' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'f' boundaryCheck: false
 [18] matchScalar 'i' boundaryCheck: false
 [19] matchScalar 'x' boundaryCheck: true


--- cycle 110 ---

input: 123-_+/-789suffix
       ~~~~~~~~~~~~^
position: 12
FAIL

--- cycle 111 ---

input: 123-_+/-789suffix
       ~~~~~~~~~~~~~^
position: 13
 [0] beginCapture 0
>[1] save #14
 [2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)


--- cycle 112 ---
save points:
  pc: #14, pos: 13, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~~~~~~^
position: 13
 [0] beginCapture 0
 [1] save #14
>[2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true


--- cycle 113 ---
save points:
  pc: #14, pos: 13, stackEnd: #0
  pc: #12, pos: 13, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~~~~~~^
position: 13
 [0] beginCapture 0
 [1] save #14
 [2] save #12
>[3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2


--- cycle 114 ---
save points:
  pc: #14, pos: 13, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~~~~~~^
position: 13
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail
>[12] clear
 [13] fail
 [14] matchScalar 's' boundaryCheck: false
 [15] matchScalar 'u' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'f' boundaryCheck: false
 [18] matchScalar 'i' boundaryCheck: false


--- cycle 115 ---

input: 123-_+/-789suffix
       ~~~~~~~~~~~~~^
position: 13
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail
 [12] clear
>[13] fail
 [14] matchScalar 's' boundaryCheck: false
 [15] matchScalar 'u' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'f' boundaryCheck: false
 [18] matchScalar 'i' boundaryCheck: false
 [19] matchScalar 'x' boundaryCheck: true


--- cycle 116 ---

input: 123-_+/-789suffix
       ~~~~~~~~~~~~~^
position: 13
FAIL

--- cycle 117 ---

input: 123-_+/-789suffix
       ~~~~~~~~~~~~~~^
position: 14
 [0] beginCapture 0
>[1] save #14
 [2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)


--- cycle 118 ---
save points:
  pc: #14, pos: 14, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~~~~~~~^
position: 14
 [0] beginCapture 0
 [1] save #14
>[2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true


--- cycle 119 ---
save points:
  pc: #14, pos: 14, stackEnd: #0
  pc: #12, pos: 14, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~~~~~~~^
position: 14
 [0] beginCapture 0
 [1] save #14
 [2] save #12
>[3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2


--- cycle 120 ---
save points:
  pc: #14, pos: 14, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~~~~~~~^
position: 14
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail
>[12] clear
 [13] fail
 [14] matchScalar 's' boundaryCheck: false
 [15] matchScalar 'u' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'f' boundaryCheck: false
 [18] matchScalar 'i' boundaryCheck: false


--- cycle 121 ---

input: 123-_+/-789suffix
       ~~~~~~~~~~~~~~^
position: 14
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail
 [12] clear
>[13] fail
 [14] matchScalar 's' boundaryCheck: false
 [15] matchScalar 'u' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'f' boundaryCheck: false
 [18] matchScalar 'i' boundaryCheck: false
 [19] matchScalar 'x' boundaryCheck: true


--- cycle 122 ---

input: 123-_+/-789suffix
       ~~~~~~~~~~~~~~^
position: 14
FAIL

--- cycle 123 ---

input: 123-_+/-789suffix
       ~~~~~~~~~~~~~~~^
position: 15
 [0] beginCapture 0
>[1] save #14
 [2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)


--- cycle 124 ---
save points:
  pc: #14, pos: 15, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~~~~~~~~^
position: 15
 [0] beginCapture 0
 [1] save #14
>[2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true


--- cycle 125 ---
save points:
  pc: #14, pos: 15, stackEnd: #0
  pc: #12, pos: 15, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~~~~~~~~^
position: 15
 [0] beginCapture 0
 [1] save #14
 [2] save #12
>[3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2


--- cycle 126 ---
save points:
  pc: #14, pos: 15, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~~~~~~~~^
position: 15
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail
>[12] clear
 [13] fail
 [14] matchScalar 's' boundaryCheck: false
 [15] matchScalar 'u' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'f' boundaryCheck: false
 [18] matchScalar 'i' boundaryCheck: false


--- cycle 127 ---

input: 123-_+/-789suffix
       ~~~~~~~~~~~~~~~^
position: 15
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail
 [12] clear
>[13] fail
 [14] matchScalar 's' boundaryCheck: false
 [15] matchScalar 'u' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'f' boundaryCheck: false
 [18] matchScalar 'i' boundaryCheck: false
 [19] matchScalar 'x' boundaryCheck: true


--- cycle 128 ---

input: 123-_+/-789suffix
       ~~~~~~~~~~~~~~~^
position: 15
FAIL

--- cycle 129 ---

input: 123-_+/-789suffix
       ~~~~~~~~~~~~~~~~^
position: 16
 [0] beginCapture 0
>[1] save #14
 [2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)


--- cycle 130 ---
save points:
  pc: #14, pos: 16, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~~~~~~~~~^
position: 16
 [0] beginCapture 0
 [1] save #14
>[2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true


--- cycle 131 ---
save points:
  pc: #14, pos: 16, stackEnd: #0
  pc: #12, pos: 16, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~~~~~~~~~^
position: 16
 [0] beginCapture 0
 [1] save #14
 [2] save #12
>[3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2


--- cycle 132 ---
save points:
  pc: #14, pos: 16, stackEnd: #0

input: 123-_+/-789suffix
       ~~~~~~~~~~~~~~~~^
position: 16
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail
>[12] clear
 [13] fail
 [14] matchScalar 's' boundaryCheck: false
 [15] matchScalar 'u' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'f' boundaryCheck: false
 [18] matchScalar 'i' boundaryCheck: false


--- cycle 133 ---

input: 123-_+/-789suffix
       ~~~~~~~~~~~~~~~~^
position: 16
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail
 [12] clear
>[13] fail
 [14] matchScalar 's' boundaryCheck: false
 [15] matchScalar 'u' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'f' boundaryCheck: false
 [18] matchScalar 'i' boundaryCheck: false
 [19] matchScalar 'x' boundaryCheck: true


--- cycle 134 ---

input: 123-_+/-789suffix
       ~~~~~~~~~~~~~~~~^
position: 16
FAIL

--- cycle 135 ---

input: 123-_+/-789suffix
       .................^
position: 17
 [0] beginCapture 0
>[1] save #14
 [2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)


--- cycle 136 ---
save points:
  pc: #14, pos: 17, stackEnd: #0

input: 123-_+/-789suffix
       .................^
position: 17
 [0] beginCapture 0
 [1] save #14
>[2] save #12
 [3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true


--- cycle 137 ---
save points:
  pc: #14, pos: 17, stackEnd: #0
  pc: #12, pos: 17, stackEnd: #0

input: 123-_+/-789suffix
       .................^
position: 17
 [0] beginCapture 0
 [1] save #14
 [2] save #12
>[3] quantify builtin 1 2
 [4] reverse (isScalarDistance: true, #1)
 [5] reverseMatchScalar '-' boundaryCheck: true
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2


--- cycle 138 ---
save points:
  pc: #14, pos: 17, stackEnd: #0

input: 123-_+/-789suffix
       .................^
position: 17
 [6] quantify any 1 2
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail
>[12] clear
 [13] fail
 [14] matchScalar 's' boundaryCheck: false
 [15] matchScalar 'u' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'f' boundaryCheck: false
 [18] matchScalar 'i' boundaryCheck: false


--- cycle 139 ---

input: 123-_+/-789suffix
       .................^
position: 17
 [7] reverse (isScalarDistance: true, #1)
 [8] reverseMatchScalar '-' boundaryCheck: true
 [9] quantify builtin 1 2
 [10] clearThrough
 [11] fail
 [12] clear
>[13] fail
 [14] matchScalar 's' boundaryCheck: false
 [15] matchScalar 'u' boundaryCheck: false
 [16] matchScalar 'f' boundaryCheck: false
 [17] matchScalar 'f' boundaryCheck: false
 [18] matchScalar 'i' boundaryCheck: false
 [19] matchScalar 'x' boundaryCheck: true


--- cycle 140 ---

input: 123-_+/-789suffix
       .................^
position: 17
FAIL
===
Total cycle count: 140
Backtracks: 28
Resets: 17
Instructions:
> fail: 36
> save: 36
> clear: 18
> quantify: 18
> reverseMatchScalar: 16
> reverse: 16
> beginCapture: 1
===
```
{{< /details >}}