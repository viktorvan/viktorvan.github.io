---
title:  "Why you should model your domain with F#"
date:   2019-03-12 22:47:02 +0100
categories: fsharp
classes: wide
toc: true
header: 
    overlay_image: /assets/images/ny_skyline2.jpg
    overlay_filter: rgba(242, 19, 104, 0.2)
---

Recently when working on a feature for a client, writing the normal barrage of C# null-checks and unit-tests verifying that my model state wasn't invalid, a mantra was repeating in my mind. *I wish I was coding in F#.*  

Why? Let's do the exercise of implementing a feature in both languages and see how it turns out...

# The requirements
Let's say these are our requirements:

- A manager can `invite` a customer by registering a customer's `email`.
- A customer can `create` (and `update`) their `contact information` and `address`
    - `contact information` has **required** fields: `firstName`, `lastName` and `email`  
     and an **optional** field: `phone number`
    - `address` has **required** fields `streetAddress1`, `zipcode`, `city`, `country`  
    and an **optional** field `streetAddress2`
- A customer can `acceptGDPR`
- A customer can `checkIn`
- When a customer has done all of the above, an administrator can `verify` their registration.

For the sake of simplicity we will not be doing any validation of contact information and addresses other than to check that they are not empty.

# Using C#

Let's make a simple C# implementation[^1]: 

## Entities
```csharp
public class Customer
{
    public Guid Id { get; set; }
    public ContactInformation ContactInformation { get; set; }
    public SwedishPersonalIdentityNumber PersonalIdentityNumber { get; set; }
    public Address Address { get; set; }
    public DateTime? AcceptedGDPR { get; set; }
    public DateTime? CheckedIn { get; set; }
    public DateTime? Verified { get; set; }
}

public class ContactInformation
{
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public string Email { get; set; }
    public string PhoneNumber { get; set; }
}

public class Address
{
    public string StreetAddress1 { get; set; }
    public string StreetAddress2 { get; set; }
    public string ZipCode { get; set; }
    public string City { get; set; }
    public string Country { get; set; }
}
```

## Domain logic
And a `CustomerService` with our domain logic to handle the requirements:

```csharp
public class CustomerService
{
    private readonly IDatabase database;

    public CustomerService(IDatabase database)
    {
        _database = database;
    }

    public async Task InviteCustomer(Guid id, string email)
    {
        ValidateId(id);
        if (string.IsNullOrWhiteSpace(email)) 
        { 
            throw new ArgumentException("Invite email cannot be empty"); 
        }

        var customer = new Customer
        {
            Id = id,
            ContactInformation = new ContactInformation
            {
                Email = email
            }
        };

        await _database.CreateCustomer(customer);
    }

    public async Task UpdateCustomerDetails(
        Guid id,
        ContactInformation contactInformation,
        Address address)
    {
        ValidateId(id);
        ValidateContactInformation(contactInformation);
        ValidateAddress(address);
        var customer = await AssureCustomer(id);

        customer.ContactInformation = contactInformation;
        customer.Address = address;

        await _database.UpdateCustomer(customer);
    }

    public async Task AcceptGDPR(Guid id)
    {
        ValidateId(id);
        await AssureCustomer(id);

        await _database.SetAcceptedGDPR(id, DateTime.UtcNow);
    }


    public async Task CheckInCustomer(Guid id)
    {
        ValidateId(id);
        await AssureCustomer(id);

        await _database.SetCustomerCheckedIn(id, DateTime.UtcNow);
    }

    public async Task VerifyCustomer(Guid id)
    {
        ValidateId(id);
        var customer = await AssureCustomer(id);
        if (!customer.AcceptedGDPR.HasValue) 
        { 
            throw new InvalidOperationException(
                "Cannot verify customer if not accepted GDPR"); 
        }
        if (!existing.CheckedIn.HasValue) 
        { 
            throw new InvalidOperationException(
                "Cannot verify customer if not checked in"); 
        }

        await _database.SetCustomerVerified(id, DateTime.UtcNow);
    }

    private static void ValidateId(Guid id) 
    {
        if (id == Guid.Empty) 
        { 
            throw new ArgumentException("Must provide id"); 
        }
    }

    private static void ValidateContactInformation(ContactInformation contactInformation)
    {
        if (contactInformation == null) 
        { 
            throw new ArgumentNullException(nameof(contactInformation)); 
        }
        if (string.IsNullOrWhiteSpace(contactInformation.Email) ||
            string.IsNullOrWhiteSpace(contactInformation.FirstName) ||
            string.IsNullOrWhiteSpace(contactInformation.LastName))
        {
            throw new ArgumentException("Missing contact information");
        }
    }

    private static void ValidateAddress(Address address)
    {
        if (address == null) { throw new ArgumentNullException(nameof(address)); }
        if (string.IsNullOrWhiteSpace(address.StreetAddress1) ||
            string.IsNullOrWhiteSpace(address.ZipCode) ||
            string.IsNullOrWhiteSpace(address.City))
        {
            throw new ArgumentException("Missing address information");
        }
    }

    private Task<Customer> AssureCustomer(Guid id)
    {
        var existing = await _database.GetCustomer(id);
        if (existing == null) 
        {
            throw new InvalidOperationException(
                "Customer not found, operation requires an customer");
        }
        return existing;
    }
}
```

Lovely, look at all those validations! Actually half of that code is validations.

*I wish I was coding in F#*

## Testing the C# code

These are the tests I had to write, by the way. I'll only list the test names, but they should be pretty self explanatory. It ended up being almost 300 lines of code.

<sub>
    *InviteCustomer_ShouldSetCustomerEmail*  
    *InviteCustomer_WithEmptyEmail_Throws*  
    *InviteCustomer_WithInvalidId_Throws*  
    *UpdateCustomerDetails_ShouldUpdateDetails*  
    *UpdateCustomerDetails_WhenNotExists_Throws*  
    *UpdateCustomerDetails_WithInvalidId_Throws*  
    *UpdateCustomerDetails_WithInvalidContactInformation_Throws*  
    *UpdateCustomerDetails_WithoutContactInformation_Throws*  
    *UpdateCustomerDetails_WithInvalidAddress_Throws*  
    *UpdateCustomerDetails_WithoutAddress_Throws*  
    *CheckInCustomer_ShouldSetCheckedIn*  
    *CheckInCustomer_WhenNotExists_Throws*  
    *CheckInCustomer_WithInvalidId_Throws*  
    *AcceptGDPRCustomer_ShouldSetAcceptedGDPR*  
    *AcceptGDPRCustomer_WhenNotExists_Throws*  
    *AcceptGDPRCustomer_WithInvalidId_Throws*  
    *SetVerifiedCustomer_ShouldSetVerified*  
    *SetVerifiedCustomer_WhenNotExists_Throws*  
    *SetVerifiedCustomer_WithInvalidId_Throws*  
    *SetVerifiedCustomer_WhenNotCheckedIn_Throws*  
    *SetVerifiedCustomer_WhenNotAcceptedGDPR_Throws*  
</sub>

# Using F#

## Making invalid states impossible

Let's try with F#.

We will start out by defining a bunch of types. Quite a lot of them actually, but one great thing about F# is how lightweight the type syntax is.

Let's start with creating some basic constrained types to prevent users from creating invalid values:
```fsharp
type NoneEmptyString = private NoneEmptyString of string
module NoneEmptyString =
    let create str =
        if String.IsNullOrWhiteSpace str then "Cannot be empty" |> Error
        else str |> NoneEmptyString |> Ok

type CustomerId = private CustomerId of Guid
module CustomerId =
    let create id =
        if id = Guid.Empty then "Cannot be empty" |> Error
        else id |> CustomerId |> Ok
```

The constructors are marked as private, so you cannot create the types without calling the `create` functions.

We also want some wrapper type to prevent users from making mistakes like mixing up strings and passing a us a NoneEmptyString that isn't an Email. Or to mixup the different type of dates that we use to indicate if a customer has accepted GDPR, checked in, etc[^2].

```fsharp
type Email = Email of NoneEmptyString
type CheckInDate = CheckInDate of DateTime
type AcceptDate = AcceptedDate of DateTime
type VerifiedDate = VerifiedDate of DateTime
```

Now we can create the types for ContactInformation and Address, and let's create a ContactDetails type to group them together:
```fsharp
type ContactInformation =
    { FirstName: NoneEmptyString
      LastName: NoneEmptyString
      Email: Email
      PhoneNumber: NoneEmptyString option } 

type Address =
    { StreetAddress1: NoneEmptyString
      StreetAddress2: NoneEmptyString option
      ZipCode: NoneEmptyString 
      City: NoneEmptyString
      Country: NoneEmptyString }

type CustomerDetails =
    { ContactInformation: ContactInformation 
      Address: Address }
```

Done! For the optional fields we are using the built-in Option type (a discriminated union that either has a value, or a None value). A nice feature of F# records is that all their fields are required. This means that when we get passed a ContactInformation or Address we know it must be valid, and we don't need to do any validation on our end. Or write any tests for it.

## Customer states

If we consider the requirements we can see that customer can actually have only a few valid states:
- First they are `invited`
- Then they can have `details only`
- Or they can have details and have `accepted GDPR`
- Or details and can be `checked in`
- Or they can have details and be **both** checked in and have accepted GDPR, let´s call that `complete`
- Finally they can be `verified`

In the C# model above it was possible to have a customer in other states that would not be valid, e.g. there is nothing in the C# model that prevents us from creating a customer that is verified, but has not accepted GDPR. Instead we had to rely on unit tests to verify our requirements.

In F# we can model just the valid states:
```fsharp
type InvitedCustomer = 
    private 
        { Id: CustomerId
          Email: Email }
type DetailsOnly = 
    private 
        { Id: CustomerId
          Details: CustomerDetails }
type AcceptedGDPR = 
    private 
        { Id: CustomerId
          AcceptDate: AcceptDate
          Details: CustomerDetails }
type CheckedIn = 
    private 
        { Id: CustomerId
          CheckInDate: CheckInDate
          Details: CustomerDetails }
type Complete = 
    private 
        { Id: CustomerId
          AcceptDate: AcceptDate
          CheckInDate: CheckInDate
          Details: CustomerDetails }
type VerifiedCustomer = 
    private 
        { Id: CustomerId
          AcceptDate: AcceptDate
          CheckedInDate: CheckInDate
          VerifiedDate: VerifiedDate
          Details: CustomerDetails }
```

The `private` keyword in this case is making the constructors to our types private, not the types themselves. This means that it is possible for users of our code to use our types, but they cannot create them. I.e. a user cannot create a VerifiedCustomer in any other way than calling our `verify` function (that we will define shortly).

Additionally we want to be able to distinguish customers that are verified, unverified and active, like this:

```fsharp
type ActiveCustomer =
    private 
        | DetailsOnly of DetailsOnly
        | AcceptedGDPR of AcceptedGDPR
        | CheckedIn of CheckedIn
        | Complete of Complete

type UnverifiedCustomer =
    private
        | Invited of InvitedCustomer
        | Active of ActiveCustomer
```

Now, we are finally ready to move on to the domain logic. I am a proponent of separating I/O from domain logic so I will assume that any database calls are handled outside of our domain logic[^3]. It won't be a "Service" in F# though, just a group of functions in a module. Let's go through them each in turn.

```fsharp
// CustomerId -> Email -> InvitedCustomer
let invite id email = { Id = id; Email = email }
```

From the signature we can see that `invite` takes one `CustomerId`, one `Email` and returns an `InvitedCustomer`. Not much that can go wrong there, and no validation required.

```fsharp
// UnverifiedCustomer -> CustomerDetails -> ActiveCustomer
let updateDetails (customer: UnverifiedCustomer) details =
    match customer with
    | Invited { Id = id } -> DetailsOnly { Id = id; Details = details }
    | Active activeCustomer ->
        match activeCustomer with
        | DetailsOnly c -> DetailsOnly { c with Details = details }
        | AcceptedGDPR c -> AcceptedGDPR { c with Details = details }
        | CheckedIn c -> CheckedIn { c with Details = details }
        | Complete c -> Complete { c with Details = details }
```

This is a bit more code, but from the signature we can see that `updateDetails` takes an `UnverifiedCustomer` some `CustomerDetails` and returns a customer that we now know to be in the `ActiveCustomer` state. We are using pattern matching to handle all the possible CustomerStates, and we get help from the compiler. It will remind us if we have forgotten to handle any states.

```fsharp
// ActiveCustomer -> AcceptDate -> ActiveCustomer
let acceptGDPR (customer: ActiveCustomer) date =
    match customer with
    | DetailsOnly c -> AcceptedGDPR { Id = c.Id
                                        Details = c.Details
                                        AcceptDate = date }
    | AcceptedGDPR c -> AcceptedGDPR { c with AcceptDate = date }
    | CheckedIn c -> Complete { Id = c.Id
                                Details = c.Details
                                CheckInDate = c.CheckInDate
                                AcceptDate = date }
    | Complete c -> Complete { c with AcceptDate = date }
```

By now it should be pretty clear from the function signature what is happening here. We take an `ActiveCustomer`, apply an `AcceptDate` and are given back a new `ActiveCustomer`. Again we are forced to handle all possible states.

```fsharp
// ActiveCustomer -> CheckInDate -> ActiveCustomer
let checkIn (customer: ActiveCustomer) date =
    match customer with
    | DetailsOnly c -> CheckedIn { Id = c.Id
                                    Details = c.Details
                                    CheckInDate = date }
    | AcceptedGDPR c -> Complete { Id = c.Id
                                    Details = c.Details
                                    AcceptDate = c.AcceptDate
                                    CheckInDate = date }
    | CheckedIn c -> CheckedIn { c with CheckInDate = date }
    | Complete c -> Complete { c with CheckInDate = date }
```

CheckIn has almost the same signature as acceptGDPR since they are almost doing the same thing. Good thing we made separate types for `AcceptDate` and `CheckInDate` so we don't accidentally mix them up. We couldn't do that now without getting a compiler error.

```fsharp   
// ActiveCustomer -> VerifiedDate -> VerifiedCustomer
let verify (customer: Complete) date =
    { Id = customer.Id
      Details = customer.Details
      AcceptDate = customer.AcceptDate
      CheckedInDate = customer.CheckInDate
      VerifiedDate = date }
```

And finally `verify`, it takes a `Complete` customer and a `VerifiedDate` and returns a `VerifiedCustomer`. One requirement was that a customer needed to have accepted GDPR and checked in before we could mark them as verified. We don't need any tests to verify that because we cannot even call this function unless we have a `Complete` customer. 

## Testing the F# code
Actually, the only tests that we now need to write will be very focused on the requirements, we no longer need to worry about stuff like empty strings, or that customers must be in the correct state. The type system is enforcing all that for us.

We need some tests for our constrained types:  
<sub>
    *Can create NoneEmptyString*  
    *Cannot create NoneEmptyString with invalid input*  
    *Can create CustomerId*  
    *Cannot create CustomerId with Empty Guid*  
</sub>

And then for the domain logic:  
<sub>
    *invite returns an Invited Customer*  
    *update transitions Invited Customer to DetailsOnly*  
    *update does not change state for DetailsOnly customer*  
    *update does not change state for AcceptedGDPR customer*  
    *update does not change state for CheckedIn customer*  
    *update does not change state for Complete customer*  
    *acceptGDPR transitions Details customer to AcceptedGDPR*  
    *acceptGDPR does not change state for AcceptedGDPR customer*  
    *acceptGDPR transitions CheckedIn customer to Complete*  
    *acceptGDPR does not change state for Complete customer*  
    *checkIn transitions DetailsOnly customer to CheckedIn*  
    *checkIn transitions AcceptedGDPR customer to Complete*  
    *checkIn does not change state for CheckedIn customer*  
    *checkIn dose not change state for Complete customer*  
    *verify returns VerifiedCustomer*
</sub>

# Conclusions
To be fair, at first glance it may not be obvious why the F# code is preferable. The number of lines of code is almost the same for both implementations with the F# just being slightly shorter[^4]. But we are accomplishing **so much more** in the F# code. It is more type safe, letting the compiler deal with things that we were forced to write unit tests for in C#. 

By modeling only the possible states for a customer we make it much harder for anyone to (unintentionally) use our code in the "wrong" way. Anyone calling `invite` will have to provide us with a valid id and email. And they will have to deal with the fact that they get an `InvitedCustomer` back. Or that you cannot call `verify` without a `Complete` customer. And since the constructor for the `Complete` state is private, the only way you can get a `Complete` customer is by calling our functions `updateDetails`, `acceptGDPR` and `checkIn`. Hopefully you can see the benefits of all this[^5].

If you want to read more about modeling your domain with F# I strongly recommend the book [Domain Modeling Made Functional](https://pragprog.com/book/swdddf/domain-modeling-made-functional) by Scott Wlaschin.

[^1]: Sure this code can be improved quite a bit, and be made more like the proposed F# solution with immutability and similar Customer states, but I want to write **less** code, not more. I will leave this as an exercise to the reader 😉.
[^2]: In "real" code we would probably add some validations here, for example to check that the email is valid email and that the dates are not in the future.
[^3]: Mark Seemann refers to this as Dependency Rejection, you really should read his [series of posts](https://blog.ploeh.dk/2017/02/02/dependency-rejection/) on the subject. I could have introduced this concept in the C# code as well, but I did want to write as recognizable object oriented code as possible.
[^4]: To be fair we have moved the database calls outside the scope of the F# solution. But also I am not counting the tests that tend to be much more verbose in C#.
[^5]: And as an aside, I actually discovered a missing requirement when rewriting the code in F#; should it really be possible to make any changes to a customer after they have been verified? Probably not.
