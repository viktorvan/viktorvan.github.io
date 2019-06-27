---
title:  "Migrating a C# test suite to property based tests in F# - part 2"
date:   2019-06-28 07:50:00 +0100
categories: fsharp 
tags:
    - fsharp
    - tests
classes: wide
toc: true
header: 
    overlay_image: /assets/images/spain_bridge.jpg
---

# Background

This is part 2 in a series of blog post where I migrate a C# test suite to property based tests in F#.

link to part1.

I've been wanting to try property-based testing in a real-life situation for some time, and decided to try it out with the test suite for our open source library ActiveLogin.Identity.

A short background on ActiveLogin.Identity; it's a library for parsing and validating Swedish identities, such as a Personal Identity Number, let's call it a **pin**. For this blog we can be satisfied with the following simplified model: the format for a pin is YYYYMMDDBBBC, i.e. **Y**ear, **M**onth, **D**ay, **B**irth number and a checksum. The birth number is any number three digit number from 001 to 999, the checksum is calculated using the [Luhn algorithm](https://en.wikipedia.org/wiki/Luhn_algorithm). You can also write a pin using a 10 digit format YYMMDD-BBBC or YYMMDD+BBBC, where a "+" indicates that the person has turned or is turning 100 this year[^1].

In the last blog post we wrote some property tests for verifying the `parse` function for valid inputs. In this post we will handle som invalid inputs. 

<!-- ## Parsing invalid pin strings

### null or empty strings

The first invalid string we will test is `null` and for this test we don't need to use property based testing, there is only one input to test. So this is just a straight forward unit test:

```fsharp
test "null string returns argument null error" {
    null
    |> SwedishPersonalIdentityNumber.parse =! Error ArgumentNullError }
```

The next test is for empty strings. To write a generator for empty strings we are going to use a feature in FsCheck where a generator can take a size parameter. When running the tests the test framework will begin by generating small test cases, and gradually increases the size as testing progresses.

A generator for an empty string of random size could then look like this:

```fsharp
let emptyStringGen() =
    type EmptyString = EmptyString of string
    let emptyStringWithLength length = String.replicate length " "

    Gen.sized(fun s -> Gen.choose(0,s) |> Gen.map emptyStringWithLength)
    |> Gen.map EmptyString
    |> Arb.fromGen
```

Gen.choose generates a random integer in the range `[0,s]` we then `map` it to an emptyString with that length using our helper function `emptyStringWithLengh`.

And then we use the generator:
```fsharp
testProp "empty string returns parsing error" <| fun (Gen.EmptyString str) ->
    str
    |> SwedishPersonalIdentityNumber.parse =! Error (ParsingError Empty)
```

### An invalid number of digits
Another form of invalid pin strings are when there is an invalid number of digits. A pin can only have 10 or 12 digits. Let's write a generator for some random digits:

```fsharp
let digitsGen() =
    let createDigits = String.concat "" >> Digits
    Gen.choose(0,9)
    |> Gen.map string
    |> Gen.listOf
    |> Gen.map createDigits
    |> Arb.fromGen
```

First we create a helper function `createDigits` with type signature: `string list -> Digits`, i.e. given a list of strings it will return our `Digits` type.

Then we create our generator: First pick a random integer between 0 and 9 and turn it into a string. Then use `Gen.listOf` to get a list of those random strings. And finally we use our helper function `createDigits` to turn wrap the list of strings in the `Digits` type.

As you can see we have not constrained the generator to return a specific number of digits at this point. `Gen.listOf` will generate lists of random length, including empty lists. We will handle this in the test by adding a condition to the input using the `==>` operator from FsCheck. 

First we create a function to check if a string contains an invalid number of digits:

```fsharp
let private isInvalidNumberOfDigits (str: string) =
    if System.String.IsNullOrWhiteSpace str then false
    else
        str
        |> String.filter isDigit
        |> (fun s -> s.Length <> 10 && s.Length <> 12)
```

Check that the string is not null or empty, and then check that it is not of valid length (10 or 12 digits).

Then the property test with a condition on the input:

```fsharp
testProp "invalid number of digits returns parsing error" <| fun (Gen.Digits digits) ->
    isInvalidNumberOfDigits digits ==>
        lazy (digits
                |> SwedishPersonalIdentityNumber.parse =! Error (ParsingError Length))
```

By using the `==>` operator and wrapping the right-hand-side in a lazy expression then FsCheck will stop if the condition on the left-hand-side is false. And our test assertions will not be evaluated.
IF THE LEFT-HAND-SIDE IS TRUE THEN THE LAZY EXPRESSION WILL BE EVALUATED. -->