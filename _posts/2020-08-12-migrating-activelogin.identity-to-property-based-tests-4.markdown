---
title:  "Migrating a C# test suite to property based tests in F# - part 4"
date:   2020-06-29 07:50:00 +0100
categories: fsharp 
tags:
    - fsharp
    - tests
classes: wide
toc: true
header: 
    overlay_image: /assets/images/sansebastian2.jpg
published: false
---

This is part 4 in a series of blog post where I migrate a C# test suite to property based tests in F#. In this post we will be dealing with tests where we need to avoid to re-using the production code in the test code.

# Background

[Part 1 - Introduction to property based testing](https://viktorvan.github.io/fsharp/migrating-activelogin.identity-to-property-based-tests-1/)
[Part 2 - More about generators](https://viktorvan.github.io/fsharp/migrating-activelogin.identity-to-property-based-tests-2/)
[Part 3 - Generators with too many output values](https://viktorvan.github.io/fsharp/migrating-activelogin.identity-to-property-based-tests-3/)

I've been wanting to try property-based testing in a real-life situation for some time, and decided to try it out with the test suite for our open source library ActiveLogin.Identity.

A short background on ActiveLogin.Identity; it's a library for parsing and validating Swedish identities, such as a Personal Identity Number, let's call it a **pin**. For this blog we can be satisfied with the following simplified model: the format for a pin is YYYYMMDDBBBC, i.e. **Y**ear, **M**onth, **D**ay, **B**irth number and a checksum. The birth number is any number three digit number from 001 to 999, the checksum is calculated using the [Luhn algorithm](https://en.wikipedia.org/wiki/Luhn_algorithm). You can also write a pin using a 10 digit format YYMMDD-BBBC or YYMMDD+BBBC, where a "+" indicates that the person has turned or is turning 100 this year[^1].




[^1]: But as it turns out, sometimes it can actually be very useful to use the same code in both test and production code. Honestly, this might be a too important tip to hide in a foot note actually, but here we are... There are some good use cases for this type of testing. Refactoring legacy code, or trying to improve performance are two cases that comes to mind. When you are working with legacy code where there are no unit tests and you don't know what the requirements are, only that the legacy function is "doing the right thing". Then you copy the legacy production code to your property test. You are then free to refactor the production code and as long as the test is passing you know the functionality is the same. 
