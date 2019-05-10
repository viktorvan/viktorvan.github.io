---
title:  "Migrating a C# test suite to property based tests in F#"
date:   2019-05-05 20:45:00 +0100
categories: fsharp
classes: wide
toc: true
header: 
    overlay_image: /assets/images/ny_skyline3.jpg
    overlay_filter: rgba(242, 19, 104, 0.2)
---

# Outline
- Background
- What are property-based tests
- Unit testing in F#
- The parse function
- A custom generator
- Adding type safety to our generators


# Background


I've been wanting to try property-based testing in a real-life situation for some time, and decided to try it out with the test suite for our open source library ActiveLogin.Identity.

A short background on ActiveLogin.Identity; it's a library for parsing and validating Swedish identities, such as a Personal Identity Number, let's call it a **pin**. For this blog we can be satisfied with the following simplified model: the format for a pin is YYYYMMDDBBBC, i.e. **Y**ear, **M**onth, **D**ay, **B**irth number and a checksum. The birth number is any number three digit number from 001 to 999, the checksum is calculated using the [Luhn algorithm](https://en.wikipedia.org/wiki/Luhn_algorithm)[^1].

There are a few functions of interest in the library that we are going to focus on in this blog post: `parse`, `create`, `to12DigitString`, and `to10DigitString`. What they do is pretty straight forward, parse takes a string input and returns a pin or an error, create takes some input values and returns a pin or an error. The toXXDigitString takes a pin and returns it string representation.

# What are property-based tests
## Testing by example
So what are property-based tests? Maybe it is easier to explain if put them in contrast with *normal* unit tests. The first test from the original code looks like this:

```csharp
[Theory]
[InlineData("990913+9801", 1899)]
[InlineData("120211+9986", 1912)]
[InlineData("990807-2391", 1999)]
[InlineData("180101-2392", 2018)]
public void Parses_Year_From_10_Digit_String(string personalIdentityNumberString, int expectedYear)
{
    var personalIdentityNumber = SwedishPersonalIdentityNumber.Parse(personalIdentityNumberString);
    Assert.Equal(expectedYear, personalIdentityNumber.Year);
}
```

This looks like a normal unit test and is what we can call an **example-based test**. We are providing the tests with four known **example** inputs (through the InlineData attributes) that we expect to have a certain outcome. In this case we except the `SwedishPersonalIdentityNumber.Parse` method to correctly parse the year part of the pin.

But we are only testing with four inputs which is only a small subset of the actual inputs we can expect.

## Testing by properties

Property-based testing turns this on its head. Instead of us, the test writers, providing the example inputs, we let a library do it using a *generators*. In the simplest case this could just be a generator that returns a random integer. For our domain types we usually have to write our own custom generators. 

The testing library would then run our tests several times using input from the *generator*. Either it will find an input where the test fails and report that to us, or after running the tests a certain number of times it would be satisfied that the *property* holds.

We call these tests property-based since instead of providing *example inputs* where we know the result, we will be stating *properties* about the code we are testing. 

Often these property tests can look quite different from what (at least from my experience) unit tests normally looks like. For example when it comes to testing the parse function that is tested above I ended up with a series of tests that actually are testing parse and the toXXDigitString functions together:
One example property would be, in plain english: "turning a pin into a 12DigitString and then parsing it, should return the same pin". And this is what the property-based test in F# would look like:

```fsharp
    testProp "roundtrip for 12 digit string" <| fun (Gen.ValidPin pin) ->
        pin
        |> SwedishPersonalIdentityNumber.to12DigitString
        |> SwedishPersonalIdentityNumber.parse =! Ok pin
```

Hopefully it will be more clear what these properties can look like when we start writing the tests. But I am getting ahead of myself, let's break this down and start from the beginning.

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
//  actual: 17
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

The library we are going to use for property-based testing is [FsCheck](https://fscheck.github.io/FsCheck/). Expecto has an api for writing property-based tests with FsCheck so we will use that.

## The parse function

The first function we will test is `parse`.

```fsharp
// string -> Result<SwedishPersonalIdentityNumber, Error>
let parse str = ...
```

where `Error` is a discriminated union with some error cases specific to our domain.

So it's a function that given a string input will return a value wrapped in the built-in `Result`type. Result is a discriminated union that can have either an `Ok` value or an `Error`. In the F# api we are not throwing any exceptions like we did in the C# api.

### The happy path
In the original C# code we have a bunch of tests to verify that we can parse a valid pin. The idea is to replace them with a few of the "roundtrip" tests described above.

The first one is:
```fsharp
testProp "roundtrip for 12 digit string" <| fun pin ->
    pin
    |> SwedishPersonalIdentityNumber.to12DigitString
    |> SwedishPersonalIdentityNumber.parse =! Ok pin
```

I.e. given a valid pin, if we turn it in to a 12 digit string and parse it again, we should get the same pin back. If we try to run this test we get an ArgumentException: 
```
The type 'ActiveLogin.Identity.Swedish.FSharp.Types+Year' is an F# union type but its representation is private. You must specify BindingFlags.NonPublic to access private type representations.
```

Which is due to the fact that FsCheck doesn't yet know how to create the input `pin` for us[^2]. We need to write custom generator.

### A custom SwedishPersonalIdentityNumber-generator

It turns out, due to GDPR, that you can just use any Personal Identity Number values in your tests since the pins could actully belong to a real person. The Swedish National Tax Board that is in charge of the Personal Identity Numbers actually provide a list of valid numbers to use for testing purposes. To simplify the usage of these test numbers [ActiveLogin.Identity](https://github.com/ActiveLogin/ActiveLogin.Identity) actually provides access to the test numbers in the package `ActiveLogin.Identity.Swedish.TestData`. So we are going to be using that to make sure we are only testing with valid test numbers.

Using the TestData package we can get a random valid pin from: `SwedishPersonalIdentityNumberTestData.getRandom()`.
Let's build a generator that using that. In this case it is quite easy, we can use the generator computation expression from FsCheck and just call the getRandom-function.

```fsharp
// Gen<SwedishPersonalIdentityNumber>
let validPin() = gen { return SwedishPersonalIdentityNumberTestData.getRandom() }
```

FsCheck actually packages a *generator* together with a *shrinker* into something called an *arbitrary* instance. The *shrinker* has the following signature: `'a -> seq<'a>` and takes an input and returns new values of the same type that are in some way 'smaller' that the original value. The *shrinker* is used when a test fails. FsCheck will then use the shrink the input and try again until it finds the smallest value that will cause the property to fail.
I will not go further into the details of those concepts in this blog post, but feel free to take a deep dive into [FsCheck's documentation on generating test data](https://fscheck.github.io/FsCheck/TestData.html). We actually don't have to provide a shrinker, but we have to wrap our generator in the Arbitrary type like this:

```fsharp
// Arbitrary<SwedishPersonalIdentityNumber>
let validPin() = gen { return SwedishPersonalIdentityNumberTestData.getRandom() } |> Arb.fromGen
```

### Using our custom generator

To use the generator we need to configure Expecto to use it.

```fsharp
type IdentityNumberGenerators() =
    static member ValidPin() : Arbitrary<SwedishPersonalIdentityNumberValues> = validPin()
```

```fsharp
testPropertyWithConfig { FsCheckConfig.defaultConfig with arbitrary = [ typeof<Gen.IdentityNumberGenerators> ]} 
    "roundtrip for 12 digit string" <| fun pin ->
        pin
        |> SwedishPersonalIdentityNumber.to12DigitString
        |> SwedishPersonalIdentityNumber.parse =! Ok pin
```

Let's refactor the test and add some helper functions:

```fsharp
let testProp = testPropertyWithConfig { FsCheckConfig.defaultConfig with arbitrary = [ typeof<Gen.IdentityNumberGenerators> ] }

testProp "roundtrip for 12 digit string" <| fun pin ->
    pin
    |> SwedishPersonalIdentityNumber.to12DigitString
    |> SwedishPersonalIdentityNumber.parse =! Ok pin
```

And if we run that test:

Still a success!

So what we have now is a passing property test that shows us that we can parse a valid 12 digit string, and that the to12DigitString function is working as expected. This one test actually has the same coverage as about 25 of our example based tests.

## Adding type safety to our generators

The setup for the test above is fine as long as we only have one generator for `SwedishPersonalIdentityNumber`. If we where to have more than one generator for the same type and add both to the configuration. For example, let's say we write two new generators to generate valid and invalid years. Both will have the type `Arbitrary<int>` and if we were to add them to the configuration:
```fsharp
let config = { FsCheckConfig.defaultConfig with arbitrary = [ typeof<ValidYearGenerator>; typeof<InvalidYearGenerator> ]}
```

then the only the second one would be used, i.e. if we wrote a property test that takes an `int` as an input, then all of them would be generated using the `InvalidYearGenerator`.

A common approach to deal with this problem is to wrap your generators in single-case discriminated unions. Instead of writing a generator for invalid year to be of type `Gen<int>` we can create a new type `type InvalidYear = InvalidYear of int` and make our generator be of type `Gen<InvalidYear>`.

Let's rewrite our valid pin generator using this approach:

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

So in our test function we are deconstructing the `ValidPin` discriminated union directly in the function definition: `(ValidPin pin)`. Now when we add more generators it will be much clearer which one we are actually using in which test.



[^1]: In reality it is a little bit more complex than that, and there are other valid formats for a pin as well.
[^2]: Actually the exception message is a bit cryptic, but it is due to the private constructor on our Year-type which is used by SwedishPersonalIdentityNumber.