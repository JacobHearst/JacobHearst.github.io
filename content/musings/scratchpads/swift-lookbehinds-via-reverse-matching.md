+++
title = 'Scratchpad: Lookbehinds via reverse matching'
date = 2024-06-11T07:03:22-05:00
draft = true
tags = ['swift', 'scratchpad']
+++

A month or two ago I discovered that Swift's fancy new regex builder and regex literals didn't support lookbehinds, a feature I wanted for a
project I'm working on. I decided to see what it would take to implement it myself and slowly worked my way to a (sort-of) (mostly) working
[initial implementation](https://forums.swift.org/t/swift-regex-lookbehind/58477/19). The next step, which is the subject of this musing, is to
replace that work with reverse matching since lookbehinds are just reverse matching with some extra save points on top.

<!--more-->

Reverse matching is basically exactly what it sounds like: matching a regular expression in reverse. For example, a forwards match of `(.*)(.*)z`
against `"abcdefgz"` would produce the capture groups of `("abcdefg", "")`. A reverse match of that input would produce the capture groups of
`("", "abcdefg")`.

