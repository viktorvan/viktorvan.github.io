---
title:  "Migrating a C# test suite to property based tests in F# - part 3"
date:   2020-06-29 07:50:00 +0100
categories: fsharp 
tags:
    - fsharp
    - tests
classes: wide
toc: true
header: 
    overlay_image: /assets/images/spain_skyline.jpg
published: false
---

This is part 3 in a series of blog post where I migrate a C# test suite to property based tests in F#.

# Background

[Part 1 - Introduction to property based testing](https://viktorvan.github.io/fsharp/migrating-activelogin.identity-to-property-based-tests-1/)
[Part 2 - More about generators](https://viktorvan.github.io/fsharp/migrating-activelogin.identity-to-property-based-tests-2/)

I've been wanting to try property-based testing in a real-life situation for some time, and decided to try it out with the test suite for our open source library [ActiveLogin.Identity](https://github.com/ActiveLogin/ActiveLogin.Identity).

A short background on ActiveLogin.Identity; it's a library for parsing and validating Swedish identities, such as a Personal Identity Number, let's call it a **pin**. For this blog we can be satisfied with the following simplified model: the format for a pin is YYYYMMDDBBBC, i.e. **Y**ear, **M**onth, **D**ay, **B**irth number and a checksum. The birth number is any number three digit number from 001 to 999, the checksum is calculated using the [Luhn algorithm](https://en.wikipedia.org/wiki/Luhn_algorithm). You can also write a pin using a 10 digit format YYMMDD-BBBC or YYMMDD+BBBC, where a "+" indicates that the person has turned or is turning 100 this year[^1].

In the previous posts we have looked at parsing valid and invalid pin numbers. In this post we will look at the `create` function and have to deal with ranges of inputs that are very large.

# The create function

The type signature of the `create` function is:

```fsharp
SwedishPersonalIdentityNumberValues -> Result<SwedishPersonalIdentityNumber, Error>
```

where SwedishPersonalIdentityNumberValues is just a record for grouping all the input values together:

```fsharp
type SwedishPersonalIdentityNumberValues =
    { Year : int
      Month : int
      Day : int
      BirthNumber : int
      Checksum : int }
```

## Creating valid pins

The first test I would like to write would be a roundtrip test for valid input, much like what we did in [Part 1](https://viktorvan.github.io/fsharp/migrating-activelogin.identity-to-property-based-tests-1/) of this series. It would test the property:

Valid12DigitString -> SwedishPersonalIdentityNumbeValues -> create -> Valid12DigitString. 

I.e. starting with a valid 12 digit string and extracting the year, month, day, birthnumber and checksum we can create a pin and if we turn that back into a 12-digit string we end up with the same valid 12-digit string we started with.

But I will skip it in this blog post because it would be so similar to what we already have seen, instead let's move on to testing invalid inputs where we will run into problems!

## Creating a pin with an invalid year

My first thought was that I wanted to write a test like this:

```fsharp

testProp "invalid year returns InvalidYear Error" <|
    fun (ValidValues validValues, Gen.InvalidYear invalidYear) ->
        let input = { validValues with Year = invalidYear }
        let result = SwedishPersonalIdentityNumber.create input
        result =! Error(InvalidYear invalidYear)
```

It just requires us to write a generator for valid SwedishPersonalIdentityNumberValues and a generator for invalid years. They would look like this:

```fsharp
type ValidValues = ValidValues of SwedishPersonalIdentityNumberValues

// string -> SwedishPersonalIdentityNumberValues
let stringToValues (pin : string) =
    { Year = pin.[0..3] |> int
      Month = pin.[4..5] |> int
      Day = pin.[6..7] |> int
      BirthNumber = pin.[8..10] |> int
      Checksum = pin.[11..11] |> int }

// Gen<string>
let valid12Digit = 
    chooseFromArray SwedishPersonalIdentityNumberTestData.raw12DigitStrings

// Arb<ValidValues>
let validValues = 
    valid12Digit 
        |> Gen.map (stringToValues >> ValidValues)
        |> Arb.fromGen
```

where stringToValues is a helper function to extract a `SwedishPersonalIdentityNumberValues` record from a valid pin-string.

```fsharp
type InvalidYear = InvalidYear of int

// int -> int -> Gen<int>
let outsideRange min max =
    let low = Gen.choose (Int32.MinValue, min - 1)
    let high = Gen.choose (max + 1, Int32.MaxValue)
    Gen.oneof [ low; high ]

let invalidYearGen() =
    (1, 9999)
    ||> outsideRange
    |> Gen.map InvalidYear
    |> Arb.fromGen
```

where `outsideRange` is a helper function that takes a min and a max value and generates an int outside of that range. We then provide our range of valid years, and since we are using the `DateTime` type for parsing the year that will be 1 - 9999.

### How many valid valid input values are there?

Now, we can run this test, and it will pass, but let's about what FsCheck is doing for us here. Specifically the `InvalidYear` generator is a problem. By default FsCheck will try to falsify our property by generating 200 input values using our generator. How many different values can be generated by `InvalidYear` though? Ints outside the range [1, 9999], so that would be Int.MaxValue - Int.MinValue - 9999 = 4,294,957,296. Which is a more than 200. Quite a bit more actually. When writing a test like this we are probably more worried about tests close to our limits, e.g. invalid years near 1 or near 9999. For example, if we would introduce an bug in our `create` function so that 10000 is a valid year this test would most likely not catch it.

So what do we do. Well, in this case I would say that this is probably a code smell to be using an Int32 as input for a year. Even thinking about what we are considering valid years, [1, 9999], might be to big a range for our domain. I think the earliest year for a pin was around 1860. Also it's quite optimistic (or pessimistic) to expect this code to be running in 8000 years time. We would probably be fine if we only considered valid years in the range [1860, 2100].

But let's say we cannot change the business logic at this point, what can we do then. Like I said FsCheck will by default try to falsify a property 200 times. This is configurable, but to cover 4 billion possible input values we would probably have to run the test 8 billion times or more. Which is not possible of course.

So I guess what we need to do is to write a new generator that returns more reasonable ranges of inputs, for example [-100, 0] and [10000, 10100] instead of the full span of Int32.

### Testing the inverse property
In some other cases it might be useful to look at the inverse of a property when we run in to this issue where the range of invalid inputs is too large to test. E.g. if we cannot test that 'An invalid year should return InvalidYearError', then maybe we can test that 'A valid year should *not* return InvalidYearError'. It will not fulfill the same requirement, but maybe will be useful anyway. And as it turns out, in this our domain of personal identity numbers this would be a good test. We are probably more worried that a valid year would be interpreted as an invalid year than the other way around. It would be very inconvenient for a user to not be able to enter their valid pin in a form using our library for validation.

A generator for valid years is not so hard to write:

```fsharp
type ValidYear = ValidYear of int

let validYearGen() =
    Gen.choose (1, 9999)
    |> Gen.map ValidYear
    |> Arb.fromGen
```

And we would use it like this:

```fsharp

testProp "valid year does not return InvalidYear Error" <|
    fun (ValidValues values, ValidYear validYear) ->
        let input = { values with Year = validYear }
        let result = SwedishPersonalIdentityNumber.create input
        result <>! (Error(InvalidYear validYear))
```

But we should note that even in this case it will not be enough to run the test 200 times, we must increase the number of times to run the test which is done by setting the MaxTest property of FsCheck[^1].

# Conclusion

The tests for invalid months and days can be done in the same way as for years, so I will leave them out of this blog post. For invalid birth numbers there's only one invalid case: 000, so I will skip that as well.

Testing with an invalid checksum on the other hand turns out to be so interesting that we will leave that for the next blog post.

In this post we realised we need to be careful when writing generators to be aware of what their possible outputs can be. If it's too large there is a great chance we will have flaky tests, or never find any bugs simply because the tests will not run enough times. In the next post we will look at how to write tests without repeating our production code in the test logic.

[^1]: [FsCheck configuration](https://fscheck.github.io/FsCheck/reference/fscheck-config.html)