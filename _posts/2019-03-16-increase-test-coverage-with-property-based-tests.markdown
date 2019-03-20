---
title:  "Use property-based testing to increase your test coverage by ∞"
date:   2019-03-16 09:47:00 +0100
categories: fsharp
classes: wide
toc: true
header: 
    overlay_image: /assets/images/ny_skyline3.jpg
    overlay_filter: rgba(242, 19, 104, 0.2)
---

# Background

A few weeks ago, a heated discussion about TDD at work got me thinking more and more about *property-based testing*.
I was also in the process of migrating our tests for [ActiveLogin.Identity](https://github.com/ActiveLogin/ActiveLogin.Identity) library from C# to F# (we had already migrated the implementation to F#). It seemed like a great opportunity to try writing some property-based tests and see how that turned out.

A short background on ActiveLogin.Identity; it's a library for parsing and validating Swedish identities, such as a Personal Identity Number, let's call it a **pin**. For this blog we can be satisfied with the following simplified model: the format for a pin is YYYYMMDDBBBC, i.e. **Y**ear, **M**onth, **D**ay, **B**irth number and a checksum. The birth number is any number three digit number from 001 to 999, the checksum is the luhn sum of the preceding numbers[^1].

The tests I started with were the ones related to creating a pin, [original code here](https://github.com/ActiveLogin/ActiveLogin.Identity/blob/master/test/ActiveLogin.Identity.Swedish.Test/SwedishPersonalIdentityNumber_Constructor.cs). The tests verify that you can create a valid pin and that invalid input throws an exception.

# What are property-based tests
## Testing by example
So what are property-based tests? Maybe it is easier to explain if put them in contrast with *normal* unit tests. The first test from the original code looks like this:

```csharp
[Theory]
[InlineData(-1, 01, 01, 239, 2)]
[InlineData(int.MaxValue, 01, 01, 239, 2)]
public void Throws_When_Invalid_Year(int year, int month, int day, int birthNumber, int checksum)
{
    var ex = Assert.Throws<ArgumentOutOfRangeException>(() 
        => new SwedishPersonalIdentityNumber(year, month, day, birthNumber, checksum));
    Assert.Contains("Invalid year.", ex.Message);
}
```

This is what we can call an **example-based test**. We are providing the tests with two known inputs (through the InlineData attributes) that we expect to have a certain outcome. In this case we except the `SwedishPersonalIdentityNumber` constructor to throw an Exception.

The problem is that a really evil developer could actually make this test pass by writing an implementation like this:
```csharp
public SwedishPersonalIdentity(int year, int month, int day, int birthNumber, int checksum)
{
    if (year == -1 || year == int.MaxValue) 
    {
        throw new ArgumentOutOfRangeException("Invalid year");
    }
    <...rest of implementation...>
}
```

So what should we do then? Add more example inputs? How many do we need to be satisfied that the evil developer has written the implementation that we *want*?

## Testing by properties

Property-based testing turns this on its head. Instead of us, the test writers, providing the example inputs, we let a library do it using a *generators*. In the simplest case this could just be a generator that returns a random integer. For our domain types we usually have to write our own custom generators. 

The testing library would then run our tests several times using input from the *generator*. Either it will find an input where the test fails and report that to us, or after running the tests a certain number of times it would be satisfied that the *property* holds.

We call these tests property-based since instead of providing *example inputs* where we know the result, we will be stating *properties that we know are true* about the code we are testing. 

For example, instead of providing two example inputs to the "Throws_When_Invalid_Year" test above we would just state the property "Throws when input year is invalid". I.e. given **any** *invalid year* our code should throw an exception.

Maybe it get's clearer if we put it into practice.

# Migrating to property-based tests

The complete implementation for the examples below can be found in [this GitHub repository](https://github.com/viktorvan/ActiveLoginPropertyBasedTestExample).

Since there now is an F# api for ActiveLogin.Identity we will test against that. The function under test is going to be create:

```fsharp
// SwedishPersonalIdentityNumberValues -> Result<SwedishPersonalIdentityNumber, Error>
let create (values : SwedishPersonalIdentityNumberValues) = ...
```

where `SwedishPersonalIdentityNumberValues` is a record with the input values:
```fsharp
type SwedishPersonalIdentityNumberValues =
    { Year : int
      Month : int
      Day : int
      BirthNumber : int
      Checksum : int }
```

and `Error` is a discriminated union with some error cases specific to our domain.

So it's a function that given the number values input will return a value wrapped in the built-in `Result`type. Result is a discriminated union that can have either an `Ok` value or an `Error`. We are not throwing any exceptions like we did in the C# version.

## Unit testing in F#
A quick note on writing unit tests in F#. My test library of choice for F# is [Expecto](https://github.com/haf/expecto). I also like [Unquote](https://github.com/SwensenSoftware/unquote) for writing my assertions, since I think it makes for really expressive assertions. 

For those unfamiliar with Unquote it makes it possible to write test assertions using F# quotations. For example, the following Expecto assertion (which shouldn't be too unfamiliar for users of other Assertion libraries):

```fsharp
Expect.equal result 42 "result should be equal to 42"
```

could with Unquote be written as:

```fsharp
test <@ result = 42 @>
```

or even simpler, using one of Unquote's custom operators:

```fsharp
result =! 42
```

The library we are going to use for property-based testing is [FsCheck](https://fscheck.github.io/FsCheck/). Expecto has an api for writing property-based tests with FsCheck so we will use that.

## The first test, the happy path

It turns out that we are going to need a *generator* for valid Personal Identity Numbers, so we might as well start with the "happy path" test which in C# looked like this:

```csharp
[Theory]
[InlineData(1899, 09, 13, 980, 1)]
[InlineData(1999, 08, 07, 239, 1)]
[InlineData(2000, 01, 02, 239, 1)]
[InlineData(2018, 01, 01, 239, 2)]
public void Accepts_Valid_Personal_Identity_Number(int year, int month, int day, int birthNumber, int checksum)
{
    var personalIdentityNumber = new SwedishPersonalIdentityNumber(year, month, day, birthNumber, checksum);
    Assert.Equal(year, personalIdentityNumber.Year);
    Assert.Equal(month, personalIdentityNumber.Month);
    Assert.Equal(day, personalIdentityNumber.Day);
    Assert.Equal(birthNumber, personalIdentityNumber.BirthNumber);
    Assert.Equal(checksum, personalIdentityNumber.Checksum);
}
```

Let's write a property-based version in F#:

```fsharp
testProperty "valid input values returns Result.Ok" <| fun input ->
    let result = SwedishPersonalIdentityNumber.create input
    Expect.isOk result "should be Result.Ok"
```

Ok, there´s a lot to go through here. 
First, we are using the function `testProperty` from Expecto, its' signature is

```fsharp
// (name: string) -> a' -> Test
let testProperty name = ...
```

which means that it will take a name for the test and then a function `a' -> Test`, i.e. a function that takes a generic type and returns a `Test`.

And as the value for that function we are passing in

```fsharp
// SwedishPersonalIdentityNumberValues -> Test
fun input ->
    let result = SwedishPersonalIdentityNumber.create input
    Expect.isOk result "should be Result.Ok"
```

So in our case the type of the function will be `SwedishPersonalIdentityNumberValues -> Test` since we are passing `input` to the `create` function.

We then use `Expect.isOk` to verify that the result of calling create is Successful, i.e. `result` is `Ok` and not an `Error`.

As you can see we have not specified anywhere how to generate the `input` SwedishPersonalIdentityNumberValues. Let's run the test and see what happens.

![result1](/assets/images/20190317/PropertyBasedTests20190316_1.png)

We get the message that the test failed after 1 test, and we can see that instead of `Ok` we got `Error(InvalidYear 0)`. We can see the values that FsCheck used as the input, and it used all zeros for Year, month, day, etc. Which clearly is not the valid inputs that we require for this test to pass.

It comes as no surprise that FsCheck does not know how to generate the valid input for us, it just sees that the properties on the `SwedishPersonalIdentityNumberValues` record are ints and tries to populate them with ints.

We need to write custom generator.

## A custom SwedishPersonalIdentityNumberValues-generator

It turns out, due to GDPR, that you can just use any Personal Identity Number values in your tests since the pins could actully belong to a real person. The Swedish National Tax Board that is in charge of the Personal Identity Numbers actually provide a list of valid numbers to use for testing purposes. To simplify the usage of these test numbers [ActiveLogin.Identity](https://github.com/ActiveLogin/ActiveLogin.Identity) actually provides access to the test numbers in the package `ActiveLogin.Identity.Swedish.TestData`. So we are going to be using that to make sure we are only testing with valid test numbers.

Using the TestData package we can get a string array of all test numbers, `SwedishPersonalIdentityNumberTestData.raw12DigitStrings`.
Let's build a generator that uses that array to build a valid `SwedishPersonalIdentityNumberValues` record.

First we create a generator that given an array will return a random element:

```fsharp
// a' [] -> Gen<a'>
let chooseFromArray xs =
    gen { let! index = Gen.choose (0, (Array.length xs) - 1)
          return xs.[index] }
```

Here we are using the `gen` computation expression from FsCheck to create our generator. We use `Gen.choose` to generate a random index. `Gen.choose` is a function from the FsCheck library that generates a value from within the range we provide, in this case `[0, n-1]` where n is then length of the array.

Now we can build a generator that returns one of the test number strings:

```fsharp
// Gen<string>
let valid12Digit = chooseFromArray SwedishPersonalIdentityNumberTestData.raw12DigitStrings
```

And finally we can build a generator that transforms the string into `SwedishPersonalIdentityNumberValues`:

```fsharp
// string -> SwedishPersonalIdentityNumberValues
let stringToValues (pin : string) =
    { Year = pin.[0..3] |> int
      Month = pin.[4..5] |> int
      Day = pin.[6..7] |> int
      BirthNumber = pin.[8..10] |> int
      Checksum = pin.[11..11] |> int }

// Gen<SwedishPersonalIdentityNumberValues>
let validValues =
    gen { let! str = valid12Digit
          return stringToValues str }
```

FsCheck actually packages a *generator* together with a *shrinker* into something called an *arbitrary* instance. I will not go further into the details of those concepts in this blog post, but feel free to take a deep dive into [FsCheck's documentation on generating test data](https://fscheck.github.io/FsCheck/TestData.html). 

But nevertheless we need to expose our custom generator as an arbitrary instance to be able to use it from Expecto:
```fsharp
type ValidNumberValuesGen() =
    static member Gen() : Arbitrary<SwedishPersonalIdentityNumberValues> =
        validValues |> Arb.fromGen
```

## Using our custom generator

To use the generator we need to configure Expecto to use it.

```fsharp
testPropertyWithConfig { FsCheckConfig.defaultConfig with arbitrary = [ typeof<Generators.ValidNumberValuesGen> ]} 
    "valid input values returns Result.Ok" <| fun input ->
        let result = SwedishPersonalIdentityNumber.create input
        Expect.isOk "should be Result.Ok" result 
```

Let's add some functions to make the configuration easier on the eyes:

```fsharp
let config = FsCheckConfig.defaultConfig
let addToConfig arbTypes = { config with arbitrary = arbTypes @ config.arbitrary }
let testProp arbTypes = testPropertyWithConfig (addToConfig arbTypes)

[<Tests>]
let tests =
    testList "create" [
        testProp [ typeof<Generators.ValidNumberValuesGen> ] 
            "valid input values returns Result.Ok" <| fun input ->
                let result = SwedishPersonalIdentityNumber.create input
                Expect.isOk "should be Result.Ok" result 
    ]
```

And if we run that test:

![result2](/assets/images/20190317/PropertyBasedTests20190316_2.png)

Success![^2]  

So, now instead of the just testing with 4 examples we know to be valid, like in the C# test above. We are now letting FsCheck test our function with random values until it is satisfied that the property holds. In this case this means running the test for a configurable amount of iterations, the default being 200.

## Adding type safety to our generators

A problem with our generator is that it is of type `Arbitrary<SwedishPersonalIdentityNumberValues>`. Just by looking at that type signature it is not actually clear what type of Values we are generating, is it valid or invalid or just completely random. 

This would be even worse if we had written a generator for a primitive type, like we are going to do later in this post when we need an InvalidYear-generator, InvalidMonth-generator, etc. Then it's going to be really hard to understand if our `Arbitrary<int>` is returning a year or a month. Also we would not be able to configure our property test to use two different `Arbitrary<int>` generators, it would just use the last one we configured it with.

A common approach to solve this problem is to use single-case discriminated unions. Instead of writing a generator for invalid year to be of type `Gen<int>` we can create a new type `type InvalidYear = InvalidYear of int` and make our generator be of type `Gen<InvalidYear>`.

Let's rewrite our ValidValuesGen using this approach:

```fsharp
type ValidValues = ValidValues of SwedishPersonalIdentityNumberValues
let validValues =
    gen { let! str = valid12Digit
          return stringToValues str |> ValidValues }

type ValidNumberValuesGen() =
    static member Gen() : Arbitrary<ValidValues> =
        validValues |> Arb.fromGen
```

And update the test where we use it:
```fsharp
testProp "valid input values returns Result.Ok" <| fun (Gen.ValidValues input) ->
    let result = SwedishPersonalIdentityNumber.create input
    Expect.isOk "should be Result.Ok" result 
```

So in our test function we are deconstructing the `ValidValues` discriminated union directly in the function definition: `(Gen.Validvalues input)`.

## Next test, invalid year

Now let's move on to the test for invalid year. This is the test we will be replacing:

```csharp
[Theory]
[InlineData(-1, 01, 01, 239, 2)]
[InlineData(int.MaxValue, 01, 01, 239, 2)]
public void Throws_When_Invalid_Year(int year, int month, int day, int birthNumber, int checksum)
{
    var ex = Assert.Throws<ArgumentOutOfRangeException>(() 
        => new SwedishPersonalIdentityNumber(year, month, day, birthNumber, checksum));
    Assert.Contains("Invalid year.", ex.Message);
}
```

First we need to create a generator for invalid year.
```fsharp
let invalidYear =
    let tooSmall = Gen.choose(Int32.MinValue, 0)
    let tooBig = Gen.choose(10000, Int32.MaxValue)
    Gen.oneof [ tooSmall; tooBig ]

type InvalidYearGen() =
    static member Gen() : Arbitrary<int> =
        invalidYear |> Arb.fromGen
```

First we are using the `Gen.choose` function to create two generators for too small and too big values for a year. We then combine them using `Gen.oneof` which will randomly pick one of the generators.

And here's the test using `InvalidYearGen`:
```fsharp
testProp [ typeof<Generators.ValidNumberValuesGen>; typeof<InvalidYearGen> ]
    "with invalid year returns InvalidYear Error" 
        <| fun (validValues: SwedishPersonalIdentityNumberValues, invalidYear) ->
            let input = { validValues with Year = invalidYear }
            let result = SwedishPersonalIdentityNumber.create input
            result =! Error(InvalidYear invalidYear)
```

For this test first we want a valid `SwedishPersonalIdentityNumberValues` record and then an invalid year so we can make a new record where the `Year` property is invalid. As we saw before, the test function have a signature of `a' -> Test` and in this case the generic type `a'` will be a tuple `SwedishPersonalIdentityNumberValues * int`. We have configured two generators, one for each of those types. And if we run the test:

![result3](/assets/images/20190317/PropertyBasedTests20190316_3.png)

Success! Two passing tests!  
And just to be sure whats going on here, let's add some debug printing and run the test again:
```fsharp
testProp [ typeof<Generators.ValidNumberValuesGen>; typeof<InvalidYearGen> ]
    "with invalid year" 
        <| fun (validValues: SwedishPersonalIdentityNumberValues, invalidYear) ->
            printfn "values: %A" validValues
            printfn "year: %i" invalidYear
            let input = { validValues with Year = invalidYear }
            let result = SwedishPersonalIdentityNumber.create input
            result =! Error(InvalidYear invalidYear)
```
![result4](/assets/images/20190317/PropertyBasedTests20190316_4.png)

## Invalid month

Now we set out to replace the next test:

```csharp
[Theory]
[InlineData(2018, 0, 01, 239, 2)]
[InlineData(2018, 13, 01, 239, 2)]
public void Throws_When_Invalid_Month(int year, int month, int day, int birthNumber, int checksum)
{
    var ex = Assert.Throws<ArgumentOutOfRangeException>(() => new SwedishPersonalIdentityNumber(year, month, day, birthNumber, checksum));
    Assert.Contains("Invalid month.", ex.Message);
}
```

So we build a new custom generator, `InvalidMonthGen`:

```fsharp
let invalidMonth =
    let tooSmall = Gen.choose(Int32.MinValue, 0)
    let tooBig = Gen.choose(13, Int32.MaxValue)
    Gen.oneof [ tooSmall; tooBig ]

type InvalidMonthGen() =
    static member Gen() : Arbitrary<int> =
        invalidMonth |> Arb.fromGen
```

Very similar to the generator for invalid year, and so is the test:

```fsharp
testProp [ typeof<Generators.ValidNumberValuesGen>; typeof<InvalidMonthGen> ]
    "with invalid month" <| 
        fun (validValues: SwedishPersonalIdentityNumberValues, invalidMonth) ->
            let input = { validValues with Month = invalidMonth }
            let result = SwedishPersonalIdentityNumber.create input
            result =! Error(InvalidMonth invalidMonth)
```
![result5](/assets/images/20190317/PropertyBasedTests20190316_5.png)

Success!

## Invalid day, building a generator with a computation expression

The test for invalid day looks like this in C#:
```csharp
[Theory]
[InlineData(2018, 01, 0, 239, 2)]
[InlineData(2018, 01, 32, 239, 2)]
[InlineData(2018, 02, 30, 239, 2)]
public void Throws_When_Invalid_Day(int year, int month, int day, int birthNumber, int checksum)
{
    var ex = Assert.Throws<ArgumentOutOfRangeException>(() => new SwedishPersonalIdentityNumber(year, month, day, birthNumber, checksum));
    Assert.Contains("Invalid day of month.", ex.Message);
}
```

The custom generator for an invalid day will be a little bit more complicated. To generate an invalid day we need to look at the year and month to figure out how many days the month has:

```fsharp
let invalidDay =
    gen {
        let! validValues = validValues
        let daysInMonth = DateTime.DaysInMonth(values.Year, values.Month)
        let! invalidDay = Gen.oneof [ Gen.choose (Int32.MinValue, 0)
                                      Gen.choose (daysInMonth + 1, Int32.MaxValue) ]
        return { validValues with Day = invalidDay }
    }

type InvalidDayGen() =
    static member Gen() : Arbitrary<SwedishPersonalIdentityNumberValues> =
        invalidDay |> Arb.fromGen
```

Here we are using the generator computation expression to be able to first get a set of validValues using `let!`. We can then get the expected number of days in the month and generate an invalid value. Since we need the valid year, valid month and invalid day for this test we could have returned all three as a tuple: `(validValues.Year, validValues.Month, invalidDay)` or we could even have created a new record for it `type InvalidDay = { ValidYear: int; ValidMonth: int; InvalidDay: int }`. But I opted to just use the `SwedishPersonalIdentityNumberValues` record that already was available to us. So this generator will return a `SwedishPersonalIdentityNumberValues` with an invalid day.

Our test will look like this:
```fsharp
testProp [ typeof<Generators.InvalidDayGen> ]
    "with invalid day" <| fun input ->
        let result = SwedishPersonalIdentityNumber.create input
        result =! Error(InvalidDayAndCoordinationDay input.Day )
``` 

In this case there's less setup to do in the test, since the generator `InvalidDayGen` has already done the job of setting up an invalid `SwedishPersonalIdentityNumberValues`.

![result6](/assets/images/20190317/PropertyBasedTests20190316_6.png)

Success!

## Invalid Birthnumber

This is again very similar to the tests for invalid year and month, so I will just present the code snippets:

```csharp

``1`

[^1]: In reality it is a little bit more complex than that, and there are other valid formats for a pin.

[^2]: Although we should really be checking that the pin we have created has the expected values, not just that it's not an Error. But I will leave that as exercise for the reader, or you can look at the [solution on GitHub](https://github.com/viktorvan/ActiveLoginPropertyBasedTestExample) to see how that can be done.
