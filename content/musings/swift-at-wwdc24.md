+++
date = 2024-06-14T08:46:56-05:00
title = 'Swift @ WWDC 2024'
draft = false
tags = ['swift']
+++

Another year, another WWDC. Here are my notes on improvements to the Swift language from watching the various keynotes and sessions.

<!--more-->

## Swift 6
Swift 6 being included in Xcode 16 is probably the biggest Swift-related news from WWDC. I'm not going to go into detail on all of the cool new
things that come with it, Paul Hudson wrote up a great article [here][7], the main story though is that full concurrency checking is now turned
on by default which should help to eliminate low level data races and make it easier to write safe concurrently executing code.

## Swift Testing
This is a new package for writing unit tests that I actually followed a bit through the pitches on the Swift Forums but there's two really great
sessions covering how to work with it: [Meet Swift Testing][4], and [Go further with Swift Testing][5]. It's a much more modern take on unit
testing with some super cool features a greatly simplified interface for writing assertions that also have power assert-style messages on failures.
So long `XCTAssert`s, hello `#expect()` and `#require()`!

On top of the new assertion interfaces, `swift-testing` introduces:
- Test tagging
- Nestable test suites
- Better parallelization
- Better assertions for when you expect a function to throw
- Identifying tests by macro instead of test name. This one is huge, it allows for all these features:
  - Display names
  - Runtime controls over whether a test should run using a boolean variable or an `@available` annotation
  - Setting time limits for tests
  - Parameterized tests. You can now declare a set of test cases in the `@Test` call, give your test function matching parameters, and your test
  will be run with all of those cases **and each run will show up as a different test in the test explorer**. This is a HUGE quality of life
  improvement for unit testing in Swift. No more hand coding for loops and writing helper functions.


I particularly like `#require()` as it combines the interface for asserting on a value and unwrapping some optional value. With `XCTest` you would write something like this:

```swift
let someOptional = Optional.some("some")

let unwrapped = try XCTUnwrap(someOptional)
XCTAssertEqual(someOptional, "some")

// Or if you just want to assert that the value isn't nil:
XCTAssertNotNil(someOptional)
```

With `swift-testing`, you can just write:

```swift
let someOptional = Optional<String>.some("some")

let unwrapped = #require(someOptional)
// Or if you just want to assert that the value isn't nil:
#require(someOptional)
```

This is a really small change but I always found the `XCTAssert[XYZ]` interfaces kind of cumbersome so I'm excited by the streamlined approach.

Another amazing piece of streamlining is the new assertions on throwing functions. With XCTest, if you wanted to assert that a function threw a
specific error, you would write something like this:

```swift
func testThrowingFunction() throws {
    do {
        let result = try throwingFunction()
        XCTFail("Should have thrown")
    } catch error is SomeError {
        // Test passed
    }
}
```

Now, you can write this:

```swift
func testThrowingFunction() {
    #expect(throws: SomeError.self) {
        try throwingFunction()
    }
}
```

SO MUCH BETTER! You can also use `(any Error)` if you don't care what kind of error is thrown or you can get even more specific and use a specific
case of your `Error` conforming type. `swift-testing` will handle asserting on the error and you can just call your throwing function. Want to
assert on some property of your error? You can do that too. I really recommend checking out the sessions I linked above for more details on this
awesome addition to Swift 6.

## Noncopyable Types
This is a language feature introduced in Swift 5.9 that I don't think I've ever personally come across a use case for but is nonetheless very
interesting and watching [Consume noncopyable types in Swift][8] taught me some new things about what happens when you create copies of reference
and value types.

### Copy-on-write
[Copy-on-write][10] is a very common compiler optimization that I had heard of before but didn't know anything about, much less that Swift was
using it. I knew that copying a reference type simply copies the reference to that object's location in memory but I didn't know that the same
thing is true for value types! This optimization prevents deep copies of value types until the copied value is written to. This example, copied
from [StackOverflow][9], helped me visualize this concept (keep in mind that arrays are value types in Swift).

```swift
let array1 = [1, 2, 3]
var array2 = array1

// Will print the same address twice.
array1.withUnsafeBytes { print($0.baseAddress!) }
array2.withUnsafeBytes { print($0.baseAddress!) }

// _Any_ write will cause the value to be copied, regardless of whether the value actually changes
array2[0] = 1

// Will print a different address.
array2.withUnsafeBytes { print($0.baseAddress!) }
```

### Copyable
Swift 5.9 introduced a new protocol called `Copyable` and as of Swift 6,  _everything_ conforms to it by default. This even includes protocols and
their associated types as well as generic argument types. Types in Swift can already by copied by default so this is really just making that
behavior explicit. The reason for introducing the `Copyable` protocol is that it gives you a way to mark types as _non-copyable_

### Non-Copyable
A non-copyable type is one that, well, can't be copied. This means that when you perform an operation, like initializing `array2` to the value of
`array1` above, instead of copying the value, `array2` now becomes the owner of the value of `array1`'s contents and you can no longer reference
`array1`'s value. I found this a bit hard to reason about through words so here's a code example from the session on non-copyable types:

```swift
sstruct FloppyDisk: ~Copyable {}

func copyFloppy() -> FloppyDisk {
    let system = FloppyDisk()
    let backup = consume system // `consume` here is implicit, you don't actually have to write it
    load(system) // Compiler errors here with "'system' used after consume"
    return backup
}

func load(_ disk: borrowing FloppyDisk) {}
```

What I found particularly interesting here is that `system` isn't inferred to be an optional since it holds a non-copyable type. It's an honest
`FloppyDisk` type. Instead, if you `consume` its value, the ownership of `system`'s value is passed to `backup` and compiler just throws an error
if you try to reference `system` again.

The `borrowing` keyword in `load`'s signature tells the compiler that it just needs to reference the values on `disk` and it won't make a copy of
it. In fact, because `disk` is marked `borrowing` and not `consuming`, the compiler will throw an error if you even _try_ to copy it. The compiler
also requires you to mark parameters of non-copyable types as `borrowing`, `consuming`, or `inout` to make the ownership explicit.

```swift
func load(_ disk: borrowing FloppyDisk) {  // Compiler errors here with "'disk' is borrowed and cannot be consumed"
    var copy = disk
}
```

The session linked above goes over these examples and more so I would highly recommend watching it yourself.

## Embedded Swift
This is another thing I've been loosely following via the forums and while I personally don't do any embedded programming (yet), I think it's
super cool to see Swift's awesome language features being brought to more places. I think the memory safety of the language will be particularly
nice for applications where memory is at a premium. Watch [Go small with Embedded Swift][6] for more info.

## Package Scoped Access Levels
Okay this feature was actually released in Swift 5.9 ([here's the pitch acceptance post][1]) but I somehow missed it and only found out it existed
while watching [A Swift Tour: Explore Swift's features and design][2]. This feature is pretty much what it says on the tin: An access modifier
that makes symbols accessible from anywhere in the package. If, like me, you had no idea this existed you can read a bit more about it [here][3]

[1]: https://forums.swift.org/t/accepted-se-0386-package-access-modifier/64904
[2]: https://developer.apple.com/wwdc24/10184
[3]: https://docs.swift.org/swift-book/documentation/the-swift-programming-language/accesscontrol/#Access-Levels
[4]: https://developer.apple.com/wwdc24/10179
[5]: https://developer.apple.com/wwdc24/10195
[6]: https://developer.apple.com/wwdc24/10197
[7]: https://www.hackingwithswift.com/articles/269/whats-new-in-swift-6
[8]: https://developer.apple.com/wwdc24/10170
[9]: https://stackoverflow.com/a/52375774
[10]: https://en.wikipedia.org/wiki/Copy-on-write