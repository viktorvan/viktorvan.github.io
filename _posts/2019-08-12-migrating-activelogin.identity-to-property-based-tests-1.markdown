---
title:  "Migrating a C# test suite to property based tests in F# - part 1"
date:   2019-08-12 06:45:00 +0100
categories: fsharp 
tags:
    - fsharp
    - tests
classes: wide
toc: true
header: 
    overlay_image: /assets/images/sansebastian1.jpg
---

# Background


I've been wanting to try property-based testing in a real-life situation for some time, and decided to try it out with the test suite for our open source library [ActiveLogin.Identity](https://github.com/ActiveLogin/ActiveLogin.Identity).

A short background on ActiveLogin.Identity; it's a library for parsing and validating Swedish identities, such as a Personal Identity Number, let's call it a **pin**. For this blog we can be satisfied with the following simplified model: the format for a pin is YYYYMMDDBBBC, i.e. **Y**ear, **M**onth, **D**ay, **B**irth number and a **C**hecksum. The birth number is any number three digit number from 001 to 999, the checksum is calculated using the [Luhn algorithm](https://en.wikipedia.org/wiki/Luhn_algorithm). You can also write a pin using a 10 digit format YYMMDD-BBBC or YYMMDD+BBBC, where a "+" indicates that the person has turned or is turning 100 this year[^1].

There are a few functions of interest in the library that we are going to focus on in this blog post: `parse`, `to12DigitString`, and `to10DigitString`. What they do is pretty straight forward, `parse` takes a string input and returns a pin or an error. The `toXXDigitString` takes a pin and returns its string representation.

# What are property-based tests
## Testing by example
So what are property-based tests? Maybe it is easier to explain if put them in contrast with *normal* unit tests. The first test from the original code looks like this:

```csharp
[Theory]
[InlineData("990913+9801", 1899)]
[InlineData("120211+9986", 1912)]
[InlineData("990807-2391", 1999)]
[InlineData("180101-2392", 2018)]
public void Parses_Year_From_10_Digit_String(
    string personalIdentityNumberString, 
    int expectedYear)
{
    var personalIdentityNumber = 
        SwedishPersonalIdentityNumber.Parse(personalIdentityNumberString);
    Assert.Equal(expectedYear, personalIdentityNumber.Year);
}
```

This looks like a normal unit test and is what we can call an **example-based test**. We are providing the tests with four known **example** inputs (through the InlineData attributes) that we expect to have a certain outcome. In this case we expect the `SwedishPersonalIdentityNumber.Parse` method to correctly parse the year part of the pin.

But we are only testing with four inputs which is only a small subset of the actual inputs we can expect.

## Testing by properties

Property-based testing turns this on its head. Instead of us, the test writers, providing the example inputs, we let a library do it using a *generators*. In the simplest case this could just be a generator that returns a random integer. For our domain types we usually have to write our own custom generators. 

The testing library would then run our tests several times using input from the *generator*. Either it will find an input where the test fails and report that to us, or after running the tests a certain number of times it would be satisfied that the *property* holds.

We call these tests property-based since instead of providing *example inputs* where we know the result, we will be stating *properties* about the code we are testing. 

Often these property tests can look quite different from what (at least from my experience) unit tests normally look like. For example when it comes to testing the parse function that is tested above I ended up with a series of tests that actually are testing parse and the toXXDigitString functions together:
One example property would be, in plain english: "turning a pin into a 12-digit string and then parsing it, should return the same pin". And this is what the property-based test in F# would look like:

```fsharp
    testProp "roundtrip for 12 digit string" <| fun (Gen.ValidPin pin) ->
        pin
        |> SwedishPersonalIdentityNumber.to12DigitString
        |> SwedishPersonalIdentityNumber.parse =! Ok pin
```

But that's probably a lot to take in if you're not familiar with F# or unit testing in F#. 

## Unit testing in F#

The tests we are migrating from are written against the C# api of the library. Since there now is an F# implementation of the api I will be writing the new tests against the F# api.
A quick note on writing unit tests in F#. My test library of choice for F# is [Expecto](https://github.com/haf/expecto). I also like [Unquote](https://github.com/SwensenSoftware/unquote) for writing my assertions, since I think it makes for really expressive assertions. 

For those unfamiliar with Unquote it makes it possible to write test assertions using F# quotations. For example, the following Expecto assertion:

```fsharp
let result = 17
Expect.equal result 42 "result should be equal to 42"
// outputs:
// result should be equal to 42.
// expected: 42
// actual: 17
```

could with Unquote be written as:

```fsharp
let result = 17
test <@ result = 42 @>
// outputs:
// 17 = 42
// false
```

or even simpler, using one of Unquote's custom operators:

```fsharp
let result = 17
result =! 42
// outputs:
// 17 = 42
// false
```

The library we are going to use for property-based testing is [FsCheck](https://fscheck.github.io/FsCheck/). Expecto has an api for writing property-based tests with FsCheck so we will use that. Both FsCheck and Expecto can also be used from C#.

# Testing the parse function

The first function we will test is `parse`.

```fsharp
// string -> Result<SwedishPersonalIdentityNumber, Error>
let parse str = ...
```

where `Error` is a discriminated union with some error cases specific to our domain.

So it's a function that given a string input will return a value wrapped in the built-in `Result`type. Result is a discriminated union that can have either an `Ok` value or an `Error`. In the F# api we are not throwing any exceptions like we did in the C# api.

## The Happy Path: Parsing valid pin strings
In the original C# code we have several tests to verify that we can parse a valid pin. The idea is to replace them with a few of the "roundtrip" tests described above.

The first one is:
```fsharp
testProp "roundtrip for 12 digit string" <| fun pin ->
    pin
    |> SwedishPersonalIdentityNumber.to12DigitString
    |> SwedishPersonalIdentityNumber.parse =! Ok pin
```

I.e. given a valid pin, if we turn it in to a 12 digit string and parse it again, we should get the same pin back. If we try to run this test we get an ArgumentException: 

<div class="notice--danger">
    [16:10:45 INF] EXPECTO? Running tests...<br/>
    [16:10:46 ERR] parse/valid pins/roundtrip for 12 digit string errored in 00:00:00.4420000 <br/>
    System.ArgumentException: The type 'ActiveLogin.Identity.Swedish.FSharp.Types+Year' is an F# union type but its representation is private. You must specify BindingFlags.NonPublic to access private type representations. <br/>
    [16:10:46 INF] EXPECTO! 1 tests run in 00:00:00.5820007 for miscellaneous – 0 passed, 0 ignored, 0 failed, 1 errored.
</div>

Which is due to the fact that FsCheck doesn't yet know how to create the input `pin` for us[^2]. We need to write custom generator.

## Creating a custom generator for the SwedishPersonalIdentityNumber type

It turns out, due to GDPR, that you should not use any 'real' Personal Identity Number values in your tests since the pins could actully belong to a real person. The Swedish National Tax Board that is in charge of the Personal Identity Numbers actually provide a list of valid numbers to use for testing purposes. To simplify the usage of these test numbers [ActiveLogin.Identity](https://github.com/ActiveLogin/ActiveLogin.Identity) actually provides access to the test numbers in the package [`ActiveLogin.Identity.Swedish.TestData`](https://www.nuget.org/packages/ActiveLogin.Identity.Swedish.TestData/). So we'll use that package to ensure we are not using any 'real' pin numbers.

Using the TestData package we can get a random valid pin from: `SwedishPersonalIdentityNumberTestData.getRandom()`.
Now let's build our custom generator; In this case it is quite easy, we can use the generator computation expression from FsCheck and just call the getRandom-function.

```fsharp
// Gen<SwedishPersonalIdentityNumber>
let validPin() = gen { return SwedishPersonalIdentityNumberTestData.getRandom() }
```

FsCheck actually packages a *generator* together with a *shrinker* into something called an *arbitrary* instance. The *shrinker* has the following signature: `'a -> seq<'a>` and takes an input and returns new values of the same type that are in some way 'smaller' that the original value. The *shrinker* is used when a test fails. FsCheck will then use the shrink the input and try again until it finds the smallest value that will cause the property to fail.
I will not go further into the details of those concepts in this blog post, but feel free to take a deep dive into [FsCheck's documentation on generating test data](https://fscheck.github.io/FsCheck/TestData.html). We actually don't have to provide a shrinker, but we have to wrap our generator in the Arbitrary type like this:

```fsharp
// Arbitrary<SwedishPersonalIdentityNumber>
let validPin() = gen { 
    return SwedishPersonalIdentityNumberTestData.getRandom() } 
    |> Arb.fromGen
```

## Using our custom generator

To use the generator we need to configure Expecto to use it.

```fsharp
type IdentityNumberGenerators() =
    static member ValidPin() : Arbitrary<SwedishPersonalIdentityNumberValues> = validPin()
```

```fsharp
let config = 
    { FsCheckConfig.defaultConfig with arbitrary = [ typeof<IdentityNumberGenerators> ] } 

testPropertyWithConfig config "roundtrip for 12 digit string" <| fun pin ->
    pin
    |> SwedishPersonalIdentityNumber.to12DigitString
    |> SwedishPersonalIdentityNumber.parse =! Ok pin
```

We can run that test and now it will pass:

<div class="notice--success">
    [17:25:26 INF] EXPECTO? Running tests...<br />
    [17:25:27 INF] EXPECTO! 1 tests run in 00:00:01.0833124 for miscellaneous – 1 passed, 0 ignored, 0 failed, 0 errored. Success!
</div>

Let's refactor the test and add a helper function.

```fsharp
let config = 
    { FsCheckConfig.defaultConfig with arbitrary = [ typeof<IdentityNumberGenerators> ]} 
let testProp = testPropertyWithConfig config

testProp "roundtrip for 12 digit string" <| fun pin ->
    pin
    |> SwedishPersonalIdentityNumber.to12DigitString
    |> SwedishPersonalIdentityNumber.parse =! Ok pin
```

And if we run that test it still passes.

So what we have now is a passing property test that shows us that we can parse a valid 12 digit string, and that the to12DigitString function is working as expected. This one test actually has the same coverage as about 25 of our original example based tests.

## Adding type safety to our generators

The setup for the test above is fine as long as we only have one generator for `SwedishPersonalIdentityNumber`. If we where to have more than one generator for the same type and add both to the configuration. For example, let's say we write two new generators to generate valid and invalid years. Both will have the type `Arbitrary<int>` and if we were to add them to the configuration:
```fsharp
let config = { FsCheckConfig.defaultConfig with arbitrary = 
    [ typeof<ValidYearGenerator>; typeof<InvalidYearGenerator> ] }
```

then the only the second one would be used, i.e. if we wrote a property test that takes an `int` as an input, then all of them would be generated using the `InvalidYearGenerator`.

A common approach to deal with this problem is to wrap your generators in single-case discriminated unions. Instead of writing a generator for a valid pin to be of type `Gen<SwedishPersonalIdentityNumber>` we create a new single-case DU: `type ValidPin = ValidPin of SwedishPersonalIdentityNumber`. Then we rewrite our generator to be of type `Gen<ValidPin>`:

```fsharp
type ValidPin = ValidPin of SwedishPersonalIdentityNumber
// Arbitrary<ValidPin>
let validPinGen() =
    gen { return SwedishPersonalIdentityNumberTestData.getRandom() |> ValidPin }
    |> Arb.fromGen
```

And update the test where we use it:
```fsharp
testProp "roundtrip for 12 digit string" <| fun (ValidPin pin) ->
    pin
    |> SwedishPersonalIdentityNumber.to12DigitString
    |> SwedishPersonalIdentityNumber.parse =! Ok pin
```

So in our test function we are deconstructing the `ValidPin` discriminated union directly in the function definition: `(ValidPin pin)`. Now when we add more generators it will be much clearer which one we are actually using in which test: 

```fsharp
fun pin -> ...
```
versus
```fsharp
fun (ValidPin pin) -> ...
```

## Valid 10 digit strings

To complete our happy path we must deal with 10 digit strings as well. An additional business rule here is that 10 digit strings can contain a delimiter (either "-" or "+" where a missing delimiter is interpreted as "-").

```fsharp
testProp "roundtrip for 10 digit string with delimiter" <| fun (ValidPin pin) ->
    pin
    |> SwedishPersonalIdentityNumber.to10DigitString
    |> Result.bind SwedishPersonalIdentityNumber.parse =! Ok pin

testProp "roundtrip for 10 digit string without hyphen-delimiter" <| fun (ValidPin pin) ->
    let removeHyphen (str:string) =
        let isHyphen (c:char) = "-".Contains(c)
        String.filter (isHyphen >> not) str

    pin
    |> SwedishPersonalIdentityNumber.to10DigitString
    |> Result.map removeHyphen
    |> Result.bind SwedishPersonalIdentityNumber.parse =! Ok pin
```

As you can see we now have to use `Result.bind` and Result.`map` since `.to10DigitString` returns a Result<string,Error>. `to10DigitString` is an operation that actually can fail.

## Other valid pin strings

### 12 digit strings with extra "noise"
There are some other tests in the original test suite for inputs that actually are valid. For example the parse function should be able to handle leading and trailing whitespaces. Actually if you really dive down into the requirements you will find that for 12 digit strings we should be able to parse an input as long as it contains 12 valid digits. I.e. even if there are any leading, trailing, or interspersed characters between the digits. For example if *"192011149255"* is a valid pin, then *"   192011149255   "* with leading and trailing whitespace is also valid. And taken to its extreme: *" a1b9c2d0e1f1g1h4i9j2k5l5m "* is a valid pin[^3]. But how can we test this? 

We need to generate the valid input but with some noise. One way to do this is to generate a random string and then remove any digits it contains in our test. FsCheck can generate a string for us out of the box, but it will generate strings of any length and with nulls. To avoid any problems with too short strings we will write a generator that generates a string with length 100. Since we will be filtering the array for digits it is easier to keep it as a `char[]` for now:

```fsharp
type Char100 = Char100 of char[]

let char100() =
    Gen.arrayOfLength 100 Arb.generate<char>
    |> Gen.map Char100
    |> Arb.fromGen
```

That is, we are using the built-in functions `arrayOfLength` and `generate<'T>` to generate a char[] with length 100. We then use `Gen.map` to turn the char[] into our `Char100` type. 

We then use it in our property test like this:

```fsharp
testProp "roundtrip for 12 digit string mixed with 'non-digit' noise" 
    <| fun (ValidPin pin, Char100 charArray) ->
        let charsWithoutDigits =
            charArray
            |> Array.filter (isDigit >> not)

        pin
        |> SwedishPersonalIdentityNumber.to12DigitString
        |> surroundEachChar charsWithoutDigits
        |> SwedishPersonalIdentityNumber.parse =! Ok pin
```

`isDigit` and `surroundEachChar` are helper functions for filtering and surrounding every character in our pin string with characters from `charsWithoutDigits`:

```fsharp
let private isDigit (c:char) = "0123456789".Contains(c)

let private surroundEachChar (chars:char[]) (pin:string) =
    let getRandomChar = getRandomFromArray chars
    let surroundWith c = [| getRandomChar(); c; getRandomChar() |]

    Seq.collect surroundWith pin
    |> Array.ofSeq
    |> System.String
```

### Conclusion

That's almost all the tests we need for parsing valid strings. We do need tests for 10 digit strings with some random noise added, but those tests will be very similar to the tests for 12 digit strings. So I will leave those tests as an exercise to the reader.[^4]

As we can see we ended up with only 4 test cases, but we are really verifying a lot with those tests. We know the `parse` function and to10/12digitString functions are working for valid inputs. Comparing this with C# unit test suite we have replaced about 30 unit tests. Additionally, since our property based tests are using randomly generated inputs, we are actually testing many, many more inputs than we did in the C# unit test suite.

I feel we have covered enough ground in this blog post so I will stop here. In the next post we will look at the tests for invalid inputs.



[^1]: So given a 12 digit pin: 191012249833 the 10 digit representation would be 101224-9833 up to year 2009, from year 2010 an onwards the 10 digit representation would be 101224+9833.
[^2]: Actually the exception message is a bit cryptic, but it is due to the private constructor on our Year-type which is the first type it tries to create for our SwedishPersonalIdentityNumber.
[^3]: It may seem overzealous to test for pins with this kind of noise. But it actually comes from some specific test cases that we had to deal with in our code base, mostly dealing with issues with how users are writing the delimiter in the pin. This is just the generalization of all those types of examples. 
[^4]: Also the real test suite has some tests for a `parseInSpecificYear` function which is needed due to the nature of the 10 digit format. But it only adds complexity that is not really related to property based testing so I will also leave these tests out of this blog post. 

