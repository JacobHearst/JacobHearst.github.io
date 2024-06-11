+++
title = 'Unit Testing NZB/Torrent Parsers with Custom String Interpolations'
date = 2024-06-11T09:42:29-05:00
draft = false
tags = ['swift', 'jarr']
+++

_Disclaimer: This work is something I pursue on my own time and is neither endorsed nor supported by my employer_

I'm working on an all-in one alternative to Sonarr/Radarr/Prowlarr written entirely in Swift. I call it "Jarr" (as in Jacob's *arr). It's been
a very interesting project to work on and has let me explore lots new concepts. There are a lot of parts to the various pieces of *arr software
but the one I'll be focusing on in this post is the parsing of various details (which I'll call 'features') from the titles of NZB and Torrent
files.

Some of the challenges that these kinds of software face are that uploaders don't follow a standard naming format, nor do they all include
the same features in their titles. For example:
- Some uploaders might decide that including the video codec used is important while others might decide to include the audio codec instead.
- Some might put the resolution at the beginning of the title, others at the end.

You can probably see where I'm going with this: How do we extract the features of an upload and, more importantly, how do we test our feature extracting code?

<!--more-->

## Identifying Features

So how do we extract the features from an upload's title? Well at least for Sonarr/Radarr (and now Jarr), the answer is RegEx. It's excellent at
pattern matching, extremely flexible, and allows for matching only when some context around the target pattern is present. So RegEx allows us to
match arbitrary portions of a string and we can use the presence of those matches to determine the various features of a given upload, great!
Luckily for me, Sonarr and Radarr are both open source so rather than having to come up with my own regular expressions, I was able to just slice
and dice theirs into the pieces that I wanted.

I ultimately built my parsing code on top of Point Free's [swift-parsing](https://github.com/pointfreeco/swift-parsing) library which focuses on
modular, composable parsers. One of the core components of their parser design is that parsers inputs are `inout` values and _they consume the
piece of the string that they match_. This means that if I have a regex parser with the pattern `abc` and feed it the string `"abcdef"`, the parser will remove `"abc"` from the input variable and leave it with just `"def"`. This detail will be important later.

The basic strategy I took when building my parsers was to have lots of parsers that each parse the input using a small regular expression. Then
I was able to combine those parsers into a single, higher-order parser. For example, combining a 1080p parser, a 720p parser, a 480p parser, and a
360p parser would create a single, higher-order resolution parser that would match any of those resolutions. By combining my resolution
parser with a source parser that matched things like "WebDL" or "BluRay" and any other feature parsers I can write, I can create a single parser
that matches on all the features I'm looking for.

## Unit Testing the Feature Parsers

Regular expressions and parsers are all well and good, but I wanted to be able to validate my parsing code based on two important factors:
1. The regular expressions match the strings I expect them to
2. The parsers consume the portions of the string that I expect them to

I also wanted to make sure I was testing against input strings that closely resemble actual NZB/Torrent titles. This is another place I was able to
draw on the work done by Sonarr and Radarr devs as both projects have pretty decent test coverage with plenty of test cases.

While it was nice to have a bunch of test cases already written, I saw two areas for improvement:
1. Some test cases had features in them that weren't being used in the tests they were associated with. For example, there were some tests that
included the video codec in the title but the test was simply checking for resolution.
2. Each test had its own pool of test cases so for tests like I described in point 1, there was some "free" coverage that I felt was being missed
out on.

To summarize, these are the requirements that I wanted to meet:
1. For a given parser, a unit test should validate that:
   * The regular expression matches the portion of a string that I expect it to.
   * The parser consumes ONLY the matched portion of the string.
2. Unit tests should be able to share test cases.
   * To this end, test cases should be filterable by the known features that they contain.

If you look at a title like this: `"The Series S04E87 REPACK 720p HDTV x264 aAF"` There are a few pieces of information that we want our parser to pull out:
- Whether the upload was repacked: `"REPACK"`
- Resolution: `"720p"`
- Source: `"HDTV"`
- Video codec: `"x264"`

If we look at the problem from the test's point of view instead of the parser's, what we really want to do is add some markers into the string that
we can use in our assertions later on. Another way to say this is: how can we _interpolate_ some markers into our string. The answer? You guessed it: Custom String Interpolations

### Custom String Interpolations
There are [tons](https://www.avanderlee.com/swift/string-interpolation/) of [articles](https://www.hackingwithswift.com/articles/163/how-to-use-custom-string-interpolation-in-swift) written by much more knowledgable folks than I on this topic so this post won't go into the
mechanics of string interpolations or how to create your own. Instead I'll focus on the solution I came up with to address the points I outlined
above.

Given that I knew what parts of the string I expected my parsers to both match on and consume, I decided that my interpolations should have 2
parameters: The string that should be matched and consumed, and the enum value I expected my parser to return. With this in mind, I wrote up two
types:
- `TestCase` which conforms to `ExpressibleByStringLiteral` and  `ExpressibleByStringInterpolation`
- `TestCase.FeatureInterpolation` which conforms to `StringInterpolationProtocol`.

Here's what those types look like, along with some comments highlighting the more interesting portions of the code:

```swift
public struct TestCase: ExpressibleByStringLiteral, ExpressibleByStringInterpolation {
    public let input: String
    public let features: Features

    public init(stringLiteral value: StringLiteralType) {
        input = value
        features = []
        self.interpolation = nil
    }

    public init(stringInterpolation: FeatureInterpolation) {
        self.interpolation = stringInterpolation
        input = stringInterpolation.output
        self.features = stringInterpolation.features
    }

    // As I mentioned earlier, parsers actually consume the portion of the string that they match so in order to be able to assert
    // that the string looks how it should after the parser has been run, I implemented this function to return the input minus
    // whichever features the parser should have extracted.
    public func expectedRest(excluding features: Features) -> String {
        guard let interpolation else { return input }

        let rangesToExclude = interpolation.ranges(for: features).sorted { $0.lowerBound < $1.lowerBound }
        return input.excluding(ranges: rangesToExclude)
    }

    public func expectedRest(excluding feature: Feature) -> String {
        expectedRest(excluding: [feature])
    }

    private let interpolation: FeatureInterpolation?
}
```

```swift
extension TestCase {
    public struct FeatureInterpolation: StringInterpolationProtocol {
        var output = ""

        // These variables just hold onto the location of the feature and the corresponding enum type that the
        // parser is expected to recognize from that portion of the string
        typealias TitleComponent<T> = (range: Range<String.Index>, value: T)
        var resolutions = [TitleComponent<Resolution>]()
        var sources = [TitleComponent<Source>]()
        var codecs = [TitleComponent<Codec>]()
        // Repack/rerip, proper, and remux are just boolean flags so they don't have a corresponding enum
        var repackRerip: Range<String.Index>?
        var proper: Range<String.Index>?
        var remux: Range<String.Index>?

        private(set) var features = Features()

        public init(literalCapacity: Int, interpolationCount: Int) {
            output.reserveCapacity(literalCapacity)
        }

        public mutating func appendLiteral(_ literal: StringLiteralType) {
            output.append(literal)
        }

        // MARK: Enum interpolations
        // These functions are all essentially the same except they take different features as their arguments
        public mutating func appendInterpolation(_ value: String, resolution: Resolution) {
            let range = appendInterpolation(string: value)
            features.insert(resolution.feature)
            self.resolutions.append((range, resolution))
        }

        public mutating func appendInterpolation(_ value: String, source: Source) {
            let range = appendInterpolation(string: value)
            features.insert(source.feature)
            self.sources.append((range, source))
        }

        public mutating func appendInterpolation(_ value: String, codec: Codec) {
            let range = appendInterpolation(string: value)
            features.insert(codec.feature)
            self.codecs.append((range, codec))
        }

        // MARK: Bool interpolations
        // These functions are all essentially the same except they correspond to different boolean flags
        public mutating func appendInterpolation(repackRerip: String) {
            assert(self.repackRerip == nil)
            features.insert(.repackRerip)
            self.repackRerip = appendInterpolation(string: repackRerip)
        }

        public mutating func appendInterpolation(proper: String) {
            assert(self.proper == nil)
            features.insert(.proper)
            self.proper = appendInterpolation(string: proper)
        }

        public mutating func appendInterpolation(remux: String) {
            assert(self.remux == nil)
            features.insert(.remux)
            self.remux = appendInterpolation(string: remux)
        }

        // This is used by `TestCase` to get the interpolated string minus the features passed as a parameter here
        func ranges(for features: Features) -> [Range<String.Index>] {
            let sourceRanges = sources
                .filter { features.contains($0.value.feature) }
                .map(\.range)

            let resolutionRanges = resolutions
                .filter { features.contains($0.value.feature) }
                .map(\.range)

            let codecRanges = codecs
                .filter { features.contains($0.value.feature) }
                .map(\.range)

            var ranges = sourceRanges + resolutionRanges + codecRanges

            if features.contains(.repackRerip), let repackRerip { ranges.append(repackRerip) }
            if features.contains(.proper), let proper { ranges.append(proper) }
            if features.contains(.remux), let remux { ranges.append(remux) }

            return ranges
        }

        // A helper function that figures out the range of the string being appended and returns it for the caller
        private mutating func appendInterpolation(string: String) -> Range<String.Index> {
            let lowerBound = output.endIndex
            output.append(string)
            return lowerBound..<output.endIndex
        }
    }
}
```

`TestCase` and `FeatureInterpolation` combine to let me create one big pool of test cases like so:
```swift
enum TestCases {
    // Easy to filter based on the parser being tested
    static func cases(features: Features) -> [TestCase] {
        all.filter { $0.features.contains(features) }
    }

    static let all: Set<TestCase> = [
        "S07E23 .avi ",
        "The.Series.S01E13.\("x264", codec: .x264)-CtrlSD",
        "The Series S02E01 \("HDTV", source: .television) \("XviD", codec: .xvid) 2HD",
        "The Series S05E11 \(proper: "PROPER") \("HDTV", source: .television) \("XviD", codec: .xvid) 2HD",
        "The Series Show S02E08 \("HDTV", source: .television) \("x264", codec: .x264) FTP",
        "The.Series.2011.S02E01.WS.\("PDTV", source: .television).\("x264", codec: .x264)-TLA",
        "The.Series.2011.S02E01.WS.\("PDTV", source: .television).\("x264", codec: .x264)-\(repackRerip: "REPACK")-TLA",
        "The Series S01E04 DSR \("x264", codec: .x264) 2HD",
        "The Series S01E04 Series Death Train DSR \("x264", codec: .x264) MiNDTHEGAP",
        "The Series S11E03 has no periods or extension \("HDTV", source: .television)",
        "The.Series.S04E05.\("HDTV", source: .television).\("XviD", codec: .xvid)-LOL",
        "The.Series.S02E15.avi",
        "The.Series.S02E15.\("xvid", codec: .xvid)",
        "The.Series.S02E15.\("divx", codec: .divx)",
        // Plus tons more test cases ...
    ]
}
```

Add a little `XCTestCase` extension:
```swift
extension XCTestCase {
    func run<Output: Equatable>(
        parser: any Parser<Substring, Output>,
        on testCases: [TestCase],
        features: Features,
        expecting expectedOutput: Output
    ) throws {
        print("==Checking \(testCases.count) test cases==")

        for testCase in testCases {
            print(" - Checking '\(testCase.input)'")
            var input = testCase.input[...]
            do {
                let output = try parser.parse(&input)

                XCTAssertEqual(output, expectedOutput)
                XCTAssertEqual(String(input), testCase.expectedRest(excluding: features), "Incorrect consumption of '\(testCase.input)'")
            } catch let error as LocalizedError {
                XCTFail("\(error.errorDescription!)")
            }
        }
    }
}
```

And now my actual unit tests look like this!
```swift
func testParse360() throws {
    try run(
        parser: Resolution.threeSixtyParser,
        onCasesWith: .threeSixty,
        expecting: Resolution.threeSixty
    )
}
```

## Conclusion
The result of all this work is that I now have one giant filterable pool of test cases that I can continue to grow as I find edge cases or bugs.
As a bonus, each new test case I add also adds coverage for any of the parsers that parse any of the features in that test case _for free_.