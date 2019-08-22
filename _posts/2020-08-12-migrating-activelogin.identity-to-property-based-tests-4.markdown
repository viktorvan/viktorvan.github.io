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

# Background

This is part 4 in a series of blog post where I migrate a C# test suite to property based tests in F#.

[Part 1 - Introduction to property based testing](https://viktorvan.github.io/fsharp/migrating-activelogin.identity-to-property-based-tests-1/)
[Part 2 - More about generators](https://viktorvan.github.io/fsharp/migrating-activelogin.identity-to-property-based-tests-2/)
[Part 3 - Generators with too many output values](https://viktorvan.github.io/fsharp/migrating-activelogin.identity-to-property-based-tests-3/)

I've been wanting to try property-based testing in a real-life situation for some time, and decided to try it out with the test suite for our open source library ActiveLogin.Identity.

A short background on ActiveLogin.Identity; it's a library for parsing and validating Swedish identities, such as a Personal Identity Number, let's call it a **pin**. For this blog we can be satisfied with the following simplified model: the format for a pin is YYYYMMDDBBBC, i.e. **Y**ear, **M**onth, **D**ay, **B**irth number and a checksum. The birth number is any number three digit number from 001 to 999, the checksum is calculated using the [Luhn algorithm](https://en.wikipedia.org/wiki/Luhn_algorithm). You can also write a pin using a 10 digit format YYMMDD-BBBC or YYMMDD+BBBC, where a "+" indicates that the person has turned or is turning 100 this year[^1].

In the previous posts we have looked at parsing valid and invalid pin numbers. In this post we will look at the `create` function and have to deal with ranges of inputs that are very large.

# The create function

In (Part 3) we started writing tests for the `create` function:

```fsharp
SwedishPersonalIdentityNumberValues -> Result<SwedishPersonalIdentityNumber, Error>
```

In [Part-2](https://viktorvan.github.io/fsharp/migrating-activelogin.identity-to-property-based-tests-3/) we discussed invalid years, months, days and birth numbers but we didn't do invalid checksums.

## Invalid checksum
Writing a test for an invalid checksum turns out to be a bit more involved. The checksum is calculated using the [Luhn algorithm](https://en.wikipedia.org/wiki/Luhn_algorithm). If we take the same approach as we did for invalid year, month and days we would be doing the following: Take a random valid pin and replace the checksum with a random invalid value. But this would require us to calculate an invalid Luhn number using the randomized pin as input. And that code would most likely be the same production code that we are actually trying to test. 
I have found that this dilemma is not an uncommon occurence in property-based testing; to generate an input to test with, we have to reproduce the production code in the test. And then we will actually not be testing anything other than that copy-paste from production code to test code is working. Or that we can write the same implementation for a piece of code twice. So this is approach is not going to work in this case[^1].

This is usually when it can feel a bit tricky to write property based tests, at least for me. When I wrote example based unit tests I would just pick some example input values where I knew the checksum was invalid beforehand. And that would be it. Now we need to put some more thought into how we can express the business logic of our code in the form of a testable property.

After thinking for a while the test I ended up with was in plain english: 
"with all other values valid, only 1 of 10 checksums can be valid". I.e. if we take a valid pin and change the checksum (the last digit) to take on all values from 0 to 9 then we should end up with only 1 valid pin and 9 invalid pins[^2]. Coming from writing normal example based unit tests it can feel weird to think of properties like this to verify your business logic but it is often very useful. In doing the migration from unit tests to property based tests I discovered two bugs that the unit tests had not caught. In part due to the fact that writing property based tests forced me to look at the business logic with new eyes.

So now that we have figured out a property to test for checksums it is not so hard to write.

```fsharp
testProp "with all other values valid, only 1 of 10 checksums can be valid" <|
    fun (ValidValues values) ->
        let withChecksum values checksum = { values with SwedishPersonalIdentityNumberValues.Checksum = checksum }

        let hasInvalidChecksum = function Error(InvalidChecksum _) -> true | _ -> false

        let numberOfInvalidPins =
            [ 0..9 ]
            |> List.map (withChecksum values)
            |> List.map (SwedishPersonalIdentityNumber.create)
            |> List.filter hasInvalidChecksum
            |> List.length
        numberOfInvalidPins =! 9
```

We are using the `ValidValues` generator that we wrote in [Part 3](https://viktorvan.github.io/fsharp/migrating-activelogin.identity-to-property-based-tests-3/) to generate an input for the test. Then we have a helper function `withChecksum` that sets the checksum value in the Values record. The helper function `hasInvalidChecksum` takes a result from the `create` and returns true if it is a checksum error otherwise it returns false.

Then the test is performed as follows, take all numbers in the range [0, 9] and build new input values where the checksum has been replaced with that number. Pass those values to the `create` function to get 10 "create-results". We filter using `hasInvalidChecksum` and then count the number of elements in the list, which we expect to be 9.


[^1]: But as it turns out, sometimes it can actually be very useful to use the same code in both test and production code. Honestly, this might be a too important tip to hide in a foot note actually, but here we are... There are some good use cases for this type of testing. Refactoring legacy code, or trying to improve performance are two cases that comes to mind. When you are working with legacy code where there are no unit tests and you don't know what the requirements are, only that the legacy function is "doing the right thing". Then you copy the legacy production code to your property test. You are then free to refactor the production code and as long as the test is passing you know the functionality is the same. 

[^2]: The observant reader will of course see that this test does not verify that the valid checksum is the right one, but that requirement is already handled in a previous test we discussed, but did not actually write, in [Part 3](https://viktorvan.github.io/fsharp/migrating-activelogin.identity-to-property-based-tests-3/). The round-trip test were we verify that a valid pin can be created and turned back to the same input we started with.