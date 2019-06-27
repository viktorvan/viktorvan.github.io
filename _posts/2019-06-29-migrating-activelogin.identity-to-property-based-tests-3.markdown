---
title:  "Migrating a C# test suite to property based tests in F# - part 3"
date:   2019-06-29 07:50:00 +0100
categories: fsharp
classes: wide
toc: true
header: 
    overlay_image: /assets/images/ny_skyline3.jpg
    overlay_filter: rgba(242, 19, 104, 0.2)
---

# Background

This is part 3 in a series of blog post where I migrate a C# test suite to property based tests in F#.

link to part 1.
link to part 2.

I've been wanting to try property-based testing in a real-life situation for some time, and decided to try it out with the test suite for our open source library ActiveLogin.Identity.

A short background on ActiveLogin.Identity; it's a library for parsing and validating Swedish identities, such as a Personal Identity Number, let's call it a **pin**. For this blog we can be satisfied with the following simplified model: the format for a pin is YYYYMMDDBBBC, i.e. **Y**ear, **M**onth, **D**ay, **B**irth number and a checksum. The birth number is any number three digit number from 001 to 999, the checksum is calculated using the [Luhn algorithm](https://en.wikipedia.org/wiki/Luhn_algorithm). You can also write a pin using a 10 digit format YYMMDD-BBBC or YYMMDD+BBBC, where a "+" indicates that the person has turned or is turning 100 this year[^1].

In the previous posts we have looked at parsing valid and invalid pin numbers. In this post we will look at the `create` function and have to deal with ranges of inputs that are very large.
