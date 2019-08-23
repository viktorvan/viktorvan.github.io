---
title:  "Migrating a C# test suite to property based tests in F# - part 2"
date:   2019-08-22 07:51:00 +0100
categories: fsharp 
tags:
    - fsharp
    - tests
classes: wide
toc: true
header: 
    overlay_image: /assets/images/spain_bridge.jpg
---

This is part 2 in a series of blog post where I migrate a C# test suite to property based tests in F#.

# Background

[Part 1 - Introduction to property based testing](https://viktorvan.github.io/fsharp/migrating-activelogin.identity-to-property-based-tests-1/)

I've been wanting to try property-based testing in a real-life situation for some time, and decided to try it out with the test suite for our open source library [ActiveLogin.Identity](https://github.com/ActiveLogin/ActiveLogin.Identity).

A short background on ActiveLogin.Identity; it's a library for parsing and validating Swedish identities, such as a Personal Identity Number, let's call it a **pin**. For this blog we can be satisfied with the following simplified model: the format for a pin is YYYYMMDDBBBC, i.e. **Y**ear, **M**onth, **D**ay, **B**irth number and a checksum. The birth number is any number three digit number from 001 to 999, the checksum is calculated using the [Luhn algorithm](https://en.wikipedia.org/wiki/Luhn_algorithm). You can also write a pin using a 10 digit format YYMMDD-BBBC or YYMMDD+BBBC, where a "+" indicates that the person has turned or is turning 100 this year.

In the last blog post we wrote some property tests for verifying the `parse` function for valid inputs. In this post we will handle som invalid inputs. 

As a reminder, this is the type signature of `parse`:

```fsharp
// string -> Result<SwedishPersonalIdentityNumber, Error>
```

# Empty input

First let's deal with empty input strings. The first example is simply if the input string is null. There isn't really anything to gain from using property based tests here, so let's just write a normal unit test.

```fsharp
test "null string returns argument null error" {
    null
    |> SwedishPersonalIdentityNumber.parse =! Error ArgumentNullError }
```

It's when we move on to empty strings that it get's a little bit more interesting. If we were writing normal 'example' based unit tests we would probably provide some example inputs like `""`, and maybe some examples with whitespace, `" "`, `"   "`. With property based tests we can actually do one better, and generate empty strings with different length and test with that. Maybe not a super valuable test, but it will showcase some FsCheck[^1] features. 

So we need to write a generator that returns empty strings of different lengths. We are going to use a feature of FsCheck where a generator can take a size parameter. When running the tests the test framework will begin by generating small test cases, and gradually increases the size as testing progresses.

A generator for an empty string of random size could then look like this[^2]:

```fsharp
let emptyStringGen() =
    type EmptyString = EmptyString of string
    let emptyStringWithLength length = String.replicate length " "

    Gen.sized(fun s -> Gen.choose(0,s) |> Gen.map emptyStringWithLength)
    |> Gen.map EmptyString
    |> Arb.fromGen
```

Gen.choose generates a random integer in the range `[0,s]` we then `map` it to an emptyString with that length using our helper function `emptyStringWithLength`.

And then we write a test with the generator:
```fsharp
testProp "empty string returns parsing error" 
    <| fun (EmptyString str) ->
        str
        |> SwedishPersonalIdentityNumber.parse =! Error (ParsingError Empty)
```

# Input with invalid number of digits

Another form of invalid pin strings is when there is an invalid number of digits. A pin can only have 10 or 12 digits[^3]. Let's write a generator that generates a string with a random number of digits:

```fsharp
type Digits = Digits of string
let digitsGen() =
    let createDigits (strs: string list) =
        System.String.Join("", strs) 
        |> Digits

    Gen.choose(0,9)
    |> Gen.map string
    |> Gen.listOf
    |> Gen.map createDigits
    |> Arb.fromGen
```

First we create a helper function `createDigits` with type signature: `string list -> Digits`, i.e. given a list of strings it will join the list and wrap it in our `Digits` type.

Then we create our generator: First pick a random integer between 0 and 9 and turn it into a string. Then use the FsCheck function `Gen.listOf` to generate a list of those random strings. The type signature of `listOf` is `Gen<'a> -> Gen<'a list>`, i.e. given a generator for a type it will return a generator for a list of that type. Finally we use our helper function `createDigits` to turn wrap the list of strings in the `Digits` type.

As you can see we have not constrained the generator to return a specific number of digits at this point. `Gen.listOf` will generate lists of random length, including empty lists. The problem is that it will also generate lists of 10 and 12 digits that we don't want for this test. We will handle this in the test by adding a condition to the input using the `==>` operator from FsCheck. 

The `==>` operator takes a condition on the left-hand-side and a lazy[^4]-expression on the right-hand-side. If the condition is true then the lazy-expression is evaluated, if it is false it will not be evaluated. In our case the left-hand-side will be some condition that we require for the test to run, e.g. the length of the string should not be 10 or 12. The right-hand-side would be the actual test to run:

First we create a function to check if a string contains an invalid number of digits:

```fsharp
let private isInvalidNumberOfDigits (str: string) =
    let isDigit (c:char) = "0123456789".Contains(c)
    if System.String.IsNullOrWhiteSpace str then false
    else
        str
        |> String.filter isDigit
        |> (fun s -> s.Length <> 10 && s.Length <> 12)
```

Check that the string is not null or empty, and then check that it is not of valid length (10 or 12 digits).

Then the property test with a condition on the input:

```fsharp
testProp "invalid number of digits returns parsing error" 
    <| fun (Digits digits) ->
        isInvalidNumberOfDigits digits ==>
            lazy (digits
                    |> SwedishPersonalIdentityNumber.parse =! Error (ParsingError Length))
```

# Input with invalid digits

The last test for invalid inputs to the parse method is going to be for invalid values. For example month number 13 or day number 42 and so on.

We start as always with writing a generator to provide us with the invalid value for our test. This generator has quite a bit of code in it. But it's all broken down into small functions and we will go through them all.

```fsharp
type InvalidPinString = InvalidPinString of string

let chooseFromArray xs =
    gen { let! index = Gen.choose (0, (Array.length xs) - 1)
          return xs.[index] }

let valid12Digit = chooseFromArray SwedishPersonalIdentityNumberTestData.raw12DigitStrings

let invalidPinStringGen() =
    gen {
        let! valid = valid12Digit
        let withInvalidYear =
            gen {
                return "0000" + valid.[ 4.. ]
            }
        let withInvalidMonth =
            gen {
                let! month = Gen.choose(13,99) |> Gen.map string
                return valid.[ 0..3 ] + month + valid.[ 6.. ]
            }
        let withInvalidDay =
            gen {
                let year = valid.[ 0..3 ] |> int
                let month = valid.[ 4..5 ] |> int
                let daysInMonth = DateTime.DaysInMonth(year, month)
                let! day = Gen.choose(daysInMonth + 1, 99) |> Gen.map string
                return valid.[ 0..5 ] + day + valid.[ 8.. ]
            }
        let withInvalidBirthNumber =
            gen {
                return valid.[ 0..7 ] + "000" + valid.[ 11.. ]
            }
        let withInvalidChecksum =
            let checksum = valid.[ 11.. ]
            let invalid = checksum |> int |> fun i -> (i + 1) % 10 |> string
            gen { return valid.[ 0..10 ] + invalid }

        return! Gen.oneof [ withInvalidYear; withInvalidMonth; withInvalidDay; withInvalidBirthNumber; withInvalidChecksum ]
    } |> Gen.map InvalidPinString |> Arb.fromGen
```

We are using a lot of the stuff we have previously learnt. We use a helper function `chooseFromArray`, that picks a random value from an array, to get a random from 12-digit string from the SwedishPersonalIdentityNumberTestData library.

Then we create generators for invalid values for all the individual parts of the pin: 

**Invalid year**

It turns out that the only invalid 4-digit year is 0000, so we just need to replace the valid year with "0000"[^5].

**Invalid month** 

To generate an invalid 2-digit month number is easy, we just pick a random number in the range [13, 99] and turn it into a string and replace the month in the valid pin.

**Invalid day**

To generate an invalid 2-digit day number we first need to get the valid year and month so we can figure out the actual number of days in that month. Then we can pick a random number in the range [daysInMonth + 1, 99], turn it to a string and replace the day in the valid pin.

**Invalid birth number**

The only invalid birth number is 000 so we replace the valid birthNumber with "000".

**Invalid checksum**

The checksum of the pin calculated using the [Luhn algorithm](https://en.wikipedia.org/wiki/Luhn_algorithm) so to generate an invalid value we can just take the valid checksum and add 1, and use the modulo operator to make sure 9 + 1 = 10 will give result in 0. Maybe we feel that this isn't enough and that we want to test with all invalid checksums and we will do that in the next blog post, when we are testing the `create` function.

**The full test**

Then, when we need to generate the invalid pin we do not need to apply all these generators at once to the valid pin. We are satisfied with using one at a time. FsCheck provides us with the function Gen.oneOf which will randomly select one of the generators in a list. 

The actual test can be written like this:

```fsharp
testProp "invalid pin returns parsing error" <| fun (InvalidPinString str) ->
    match SwedishPersonalIdentityNumber.parse str with
    | Error (ParsingError (Invalid _)) -> true
    | _ -> failwith "Did not return expected error"
```

Here we don't care what the actual error message is, we just check that it the result is a ParsingError of type Invalid. If it's not we throw an exception.
If we actually were interested in also verifying the error messages we could split this test into separate tests for invalid year, invalid month, and so on...

# Conclusion
That's it for this post where we have seen some more examples of how to create generators. We used sized generators, turned generators into lists, added conditions to generators, and more. In the next post we will look at writing tests for the `create` function which allows clients to create pins from number inputs. We will run into some new issues where we have to deal with generators that can generate too many different values to really be useful.


[^1]: See [part 1](https://viktorvan.github.io/fsharp/migrating-activelogin.identity-to-property-based-tests-1/) for an introduction to the [FsCheck](https://fscheck.github.io/FsCheck/) library. 
[^2]: See [part 1](https://viktorvan.github.io/fsharp/migrating-activelogin.identity-to-property-based-tests-1/) for an introduction to generators and fsharp unit testing in general.
[^3]: As we saw in [part 1](https://viktorvan.github.io/fsharp/migrating-activelogin.identity-to-property-based-tests-1/) anything except digits and a plus (+) delimiter for 10 digit numbers can be stripped away from the pin.
[^4]: See the [documentation](https://docs.microsoft.com/en-us/dotnet/api/system.lazy-1?view=netcore-2.2) for .net Lazy.
[^5]: Whether it's really a good idea to accept years from 0001 to 9999 as valid will be discussed further in part 3.