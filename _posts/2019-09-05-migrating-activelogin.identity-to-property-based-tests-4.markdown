---
title:  "Migrating a C# test suite to property based tests in F# - part 4"
date:   2019-09-05 16:00:00 +0100
categories: fsharp 
tags:
    - fsharp
    - tests
    - property based tests
classes: wide
toc: true
header: 
    overlay_image: /assets/images/sansebastian2.jpg 
    overlay_filter: rgba(0, 0, 0, 0.4)
published: true
---

Property based tests part 4 - production code repeated in tests.

# Background

This is part 4 in a four part series: 

[Part 1 - Introduction to property based testing](https://viktorvan.github.io/fsharp/migrating-activelogin.identity-to-property-based-tests-1/)

[Part 2 - More about generators](https://viktorvan.github.io/fsharp/migrating-activelogin.identity-to-property-based-tests-2/)

[Part 3 - Generators with too many output values](https://viktorvan.github.io/fsharp/migrating-activelogin.identity-to-property-based-tests-3/)

I've been wanting to try property-based testing in a real-life situation for some time, and decided to try it out with the test suite for our open source library ActiveLogin.Identity.

A short background on ActiveLogin.Identity; it's a library for parsing and validating Swedish identities, such as a Personal Identity Number, let's call it a **pin**. For this blog we can be satisfied with the following simplified model: the format for a pin is YYYYMMDDBBBC, i.e. **Y**ear, **M**onth, **D**ay, **B**irth number and a checksum. The birth number is any number three digit number from 001 to 999, the checksum is calculated using the [Luhn algorithm](https://en.wikipedia.org/wiki/Luhn_algorithm). You can also write a pin using a 10 digit format YYMMDD-BBBC or YYMMDD+BBBC, where a "+" indicates that the person has turned or is turning 100 this year.

In the previous posts we have written property tests for valid and invalid pin numbers. We have had to deal with ranges of inputs that are very large. In this final post we will deal with tests where we feel the need to repeat the production code in the tests. We will discuss when this is ok or not. 

# "Hard to test properties"

The final group of tests I will discuss in this series of blog posts is tests where we would like to repeat the logic from the production code in the test just to figure out what the expected result should be. For example, since the first 8 digits (or 6 in the 10-digit version) of the swedish personal identity number corresponds to your date of birth (most of the time[^1]) the Active.Identity library provides a helper function to get a person's age given an identity number. Let's write a test for that function.

## Getting a persons age from a pin

Getting a persons date of birth is just a matter of looking at the Year, Month and Day properties of the pin. Calculating the age is a bit more complicated, mostly due to leap years. This is the implementation in the library:

```fsharp
let getAgeHintOnDate date pin =
    let dateOfBirth = getDateOfBirth pin
    if date >= dateOfBirth then
        let months = 12 * (date.Year - dateOfBirth.Year) + (date.Month - dateOfBirth.Month)
        match date.Day < dateOfBirth.Day with
        | true ->
            let years = (months - 1) / 12
            years |> Some
        | false -> months / 12 |> Some
    else None
```

Now, if we were going to tests this function using an example based unit test, it wouldn't be so hard. We would just take some examples where we know what the expected age should be and test with that; from the C# test suite[^2]:

```csharp
[Theory]
// Birth non leap year, before "leap day"
[InlineData("201701052399", "20180104", 0)]
[InlineData("201701052399", "20180105", 1)]
[InlineData("201701052399", "20190104", 1)]
[InlineData("201701052399", "20190105", 2)]
[InlineData("201701052399", "20200104", 2)]
[InlineData("201701052399", "20200105", 3)]

// Birth non leap year, after "leap day"
[InlineData("201703052397", "20180304", 0)]
[InlineData("201703052397", "20180305", 1)]
[InlineData("201703052397", "20190304", 1)]
[InlineData("201703052397", "20190305", 2)]
[InlineData("201703052397", "20200304", 2)]
[InlineData("201703052397", "20200305", 3)]

// Birth leap year, after leap day
[InlineData("201603102383", "20170309", 0)]
[InlineData("201603102383", "20170310", 1)]
[InlineData("201603102383", "20180309", 1)]
[InlineData("201603102383", "20180310", 2)]
[InlineData("201603102383", "20190309", 2)]
[InlineData("201603102383", "20190310", 3)]
[InlineData("201603102383", "20200309", 3)]
[InlineData("201603102383", "20200310", 4)]

// Birth leap year, on leap day
[InlineData("201602292383", "20160229", 0)]
[InlineData("201602292383", "20170228", 0)]
[InlineData("201602292383", "20170301", 1)]
[InlineData("201602292383", "20200228", 3)]
[InlineData("201602292383", "20200229", 4)]
public void GetAgeHint_Handles_LeapYears_Correctly(string personalIdentityNumber, string actualDate, int expectedAge)
{
    // Arrange
    var swedishPersonalIdentityNumber = SwedishPersonalIdentityNumber.Parse(personalIdentityNumber);
    var date = DateTime.ParseExact(actualDate, "yyyyMMdd", CultureInfo.InvariantCulture, DateTimeStyles.AssumeLocal);

    // Act
    var age = swedishPersonalIdentityNumber.GetAgeHint(date);

    // Assert
    Assert.Equal(expectedAge, age);
}
```

if we tried to write a property based test in the same way we would end up with something like this:

```fsharp
testProp "getAgeHintOnDate calculates the age from a pin" 
    <| fun (Gen.ValidPin pin, DateTime date) ->
        let expectedAge = calculateExpectedAge date pin
        Hints.getAgeHintOnDate date pin =! expectedAge
```

But what would calculateExpectedAge look like? It's exactly the same thing as the code we want to test. We can't just copy the production code, then we aren't really testing anything. It's almost like stating the property "a person **x** years old is **x** years old". 

In some cases maybe we can think of an alternative implementation and use in the test instead, but in many cases that will not be feasible. Otherwise we need to think of another property to test to fulfill our requirements.

In this case another property for the age calculation could be expressed as "a person ages by years counting from their date of birth". Maybe it's not clear straight away why that is a better property to test. Let's write the test and see. Actually since the logic for people born on leap days makes it complicated we'll actually split this into two separate tests. 

First we will deal with people not born on a leap day and for this test we need a generator for Age, like this:

```fsharp
type Age = Age of Years : int * Months : int * Days : double

let age() =
    gen {
        let! years = Gen.choose(0, 199)
        let! months = Gen.choose(0, 11)
        let! days = Gen.choose(0,27) |> Gen.map float

        return Age (Years = years, Months = months,Days = days)
    } |> Arb.fromGen
``` 

The 10-digit format actually breaks down for ages > 199[^3] so we will limit our generator to only yield ages in the range [0,199]. For the months we pick any number of months from 0 to 11, so the person has not aged a full year. And in the same way we don't want the person to have aged a full month in days, so we pick any day in the range [0,27] since the shortest month has 28 days.

We can then write our property test.

```fsharp
testProp "A person ages by years counting from their date of birth"
    <| fun (ValidPin pin, Age (years, months, days)) ->
        not (pin.Month.Value = 2 && pin.Day.Value = 29) ==>
        lazy
            let dateOfBirth = DateTime(pin.Year.Value, pin.Month.Value, pin.Day.Value)
            let checkDate =
                dateOfBirth.Date
                    .AddYears(years)
                    .AddMonths(months)
                    .AddDays(days)
            
            Hints.getAgeHintOnDate checkDate pin =! Some years
```

We have added a condition to the input value using the `==>` operator from FsCheck that we discussed in [Part 2 - More about generators](https://viktorvan.github.io/fsharp/migrating-activelogin.identity-to-property-based-tests-2/) to not run the test for leap days. We then extract the date of birth. We add the expected age to the date of birth and then use that as the date on which to check the age of the person. In this way we are testing the business logic calculate a person's age, but we do not need to repeat any code from the production code.

Next we need to do the same thing for people born on leap days. For this test we also need a generator for leap day pins. We could try and use the same conditional approach as above, i.e.:

```fsharp
testProp "A person ages by years counting from their date of birth"
    <| fun (ValidPin pin, Age (years, months, days)) ->
        (pin.Month.Value = 2 && pin.Day.Value = 29) ==>
        lazy
            // test implementation
```

To only run the test for leap days. But there are a lot less pins for leap days than not in the test data. Most of the values FsCheck will get from the generator will cause the condition to evaluate to false, and after a certain amount of tries it will fail the test and state that it was **exhausted**:

<div class="notice--danger">
    [18:40:34 INF] EXPECTO? Running tests... <br/>
    [18:40:36 ERR] hints/getAgeHint/A person ages by years counting from their date of birth failed in 00:00:01.3020000. 
    Exhausted after 1 test <br/>
    [18:40:36 INF] EXPECTO! 1 tests run in 00:00:01.3755183 for miscellaneous – 0 passed, 0 ignored, 1 failed, 0 errored.
</div>

To avoid this we can create a new generator where we take the array of pins from the test data and filter out the ones corresponding to leap days:

```fsharp
type LeapDayPin = LeapDayPin of SwedishPersonalIdentityNumber

let leapDayPins =
    let isLeapDay (pin: SwedishPersonalIdentityNumber) =
        pin.Month.Value = 2 && pin.Day.Value = 29

    SwedishPersonalIdentityNumberTestData.allPinsShuffled()
    |> Seq.filter isLeapDay
    |> Seq.toArray

let leapDayPinGen() =
    leapDayPins
    |> chooseFromArray
    |> Gen.map LeapDayPin
    |> Arb.fromGen
```

We use a helper function `isLeapDay` to filter out the leap day pins, then wrap them in our single case discriminated union LeapDayPin. The test will look exactly like the test for non leap days except for the calculation of check date where we need to account for the leap day.

```fsharp
testProp "A person born on a leap day also ages by years counting from their date of birth"
    <| fun (LeapDayPin pin, Age (years, months, days)) ->
        let dateOfBirth = DateTime(pin.Year.Value, pin.Month.Value, pin.Day.Value)
        // Since there isn't a leap day every year we need to add 1 extra day to the checkdate
        let checkDate =
            dateOfBirth.Date
                .AddYears(years)
                .AddMonths(months)
                .AddDays(days + 1.)
        pin
        |> Hints.getAgeHintOnDate checkDate =! Some years
```

# Duplicating production code in a test

There are a few usecases worth mentioning when it can be useful to copy the production code implementation to the test.

## Working with legacy code

When working with legacy code where there are no tests, we can write a test with the production code ase the expected result. Let's say we are tasked with refactoring the getAgeHint function from above, but that there are no tests for it.

```fsharp
testProp "The new function should work as the new function"
    <| fun (ValidPin pin, DateTime date) ->
        Hints.getAgeHintOnDate date pin =! newImplementation date pin
```

The test uses our ValidPin generator to run the test for any pin. We also let FsCheck generate a DateTime to use as the checkdate. If the new implementation does not return the same result as the original implementation the test will fail.

We are now safe to implement a new version of the business logic without being worried of changing any behaviours.

## Performance

If we are tasked with improving the performance of a function we can use the same approach to make sure we don't introduce any behaviour changes. And there are some convenient built-in functions in the [Expecto performance module](https://github.com/haf/expecto#performance-module) that we can use to compare performance:

```fsharp
[<Tests>]
let perfTests =
    testSequenced <| testList "performance tests" [
            testProp "The new implementation should be faster"
                <| fun (Gen.ValidPin pin, DateTime date) ->
                    Expect.isFasterThan 
                        (fun () -> newImplementation date pin) 
                        (fun () -> Hints.getAgeHintOnDate date pin) 
                        "new implementation is faster"
    ]
```

This test will fail as long as the new implementation is not faster than the old function. I am showing the full test setup here because it is important to remember to run any performance tests in sequence using `testSequenced`. By default Expecto will run all tests in parallel which will not work with performance tests.

# Conclusions

So, in the end, what are my thoughts on property based testing after migrating a unit test suite to property tests?

I like it a lot 🙂. It does require a new mindset and it still takes longer for me to think of the right tests to write. But in the end I was able to test the same requirements with a lot less actual test code. Comparing the test suites, with generators and all, the C# test suite had 1035 lines of code compared to 748 for the F# property tests. Maybe even more telling is to compare the number of tests, 91 with example based tests in C# and 43 with property based tests in F#.

Besides having less tests to maintain I also discovered two bugs in the process of writing the new tests. Having to think of properties of your code can give you some useful insights.

This codebase was pretty well suited to be tested by properties, and I imagine that will not be the case for all types of code. Without experience it can take some time to express your requirements as properties. For example the test for `getAgeHintOnDate` above did actually take me a few nights to figure out and get right. So I am not advocating that you **have** to replace all your tests with property based tests. Example based unit tests are better than **no** tests. 

So if you are struggling to write property tests at first, go ahead and write normal example based unit tests. Then, later, when you realise that some requirements can be expressed as a testable property, do a refactoring and replace the unit tests with property tests! 

[^1]: This is true in most cases, but not always. The identity numbers for any given day can actually run out and then you would be "moved" to another nearby day within the same month. The functions the [Active.Identity]() uses to determine date of birth and age are therefore placed in a `Hints` namespace to make it clear that the results must not be used as an absolute truth.

[^2]: There is a wrapper we are not seeing here that handles the conversion of age from `int option` to int when calling from C#. It makes the conversion of `None` to 0.

[^3]: In the 10-digit format YYMMDD-bbbc the delimiter changes to a **+** the year you turn 100. But there is no delimiter change when your age changes from 199 to 200. For historical reasons 😉.