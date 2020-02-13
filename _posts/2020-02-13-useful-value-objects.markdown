---
layout: post
title:  "Some Useful Value Objects"
date:   2020-02-13 18:00:00 +0000
categories: C# Architecture DDD Domain CQRS ValueObjects
---
I've recently started working on a greenfield project to replace an existing platform we have. When starting new projects in recent years I've applied more of a pure DDD (Domain Driven Design) architectural approach. One of the fundamental ideas within DDD is the [ValueObject](https://www.martinfowler.com/bliki/ValueObject.html), a term coined by Martin Fowler. There is a [great article about implementing value objects](https://enterprisecraftsmanship.com/posts/value-object-better-implementation/) that you should check out if you're not familiar with them.

My ORM of choice has always been Entity Framework (EF). One of the limitations of EF (version 6.4 and earlier) is that it does not support saving value objects in entities. However, with this new project I will be using EF Core, and one of the great features of EF Core is that there is now support for value objects in entities. There is a [great article on how to get it to work](https://medium.com/@austin.davies0101/using-value-objects-with-entity-framework-core-5cead49dbf9c) which I'd recommend checking out.

With that in mind, I have created some new value objects to replace common primitive types (such as name, email etc), that I plan to use within my entities. I've listed the source code for these below in case they come in handy for anyone else.

The main thing to note is my use of the NuGet package CSharpFunctionalExtensions, which has a type called [Result](https://enterprisecraftsmanship.com/posts/functional-c-handling-failures-input-errors/) that provides a useful way of handling failures and input errors. I've used Result as the return type for value object creation. This allows me to handle creation failures gracefully without having to throw an exception.

## Jump To
- [DateRange](#daterange)
- [Email](#email)
- [PersonName](#personname)
- [PhoneNumber](#phonenumber)

## Dependencies
The only thing you will need to do is install the NuGet package CSharpFunctionalExtensions and then you are good to go.

```
Install-Package CSharpFunctionalExtensions -Version 2.3.0
```

## DateRange
The DateRange type simply represents a start and end date. I found myself frequently doing the following within entities or other POCO classes:

```c#
public DateTime StartDate { get; protected set; }

public DateTime EndDate { get; protected set; }
```

So why not wrap them up into a single type? No only does it make our code cleaner, but it also allows us to provide methods for things like checking if a given date is within the date range.

```c#
using CSharpFunctionalExtensions;
using System;
using System.Collections.Generic;

public class DateRange : ValueObject
{
    private DateRange() { }

    private DateRange
        (
            DateTime startDate,
            DateTime? endDate = null
        )
    {
        this.StartDate = startDate;
        this.EndDate = endDate ?? DateTime.MaxValue;
    }

    public static Result<DateRange> Create
        (
            DateTime startDate,
            DateTime? endDate = null
        )
    {
        if (endDate.HasValue)
        {
            if (endDate.Value < startDate)
            {
                return Result.Failure<DateRange>
                (
                    "The end date cannot be before the start date."
                );
            }
        }

        var range = new DateRange(startDate, endDate);

        return Result.Success(range);
    }

    public DateTime StartDate { get; private set; }

    public DateTime EndDate { get; private set; }

    public bool IsWithinRange(DateTime date)
    {
        return date >= this.StartDate && date <= this.EndDate;
    }

    protected override IEnumerable<object> GetEqualityComponents()
    {
        yield return this.StartDate;
        yield return this.EndDate;
    }

    public override string ToString()
    {
        var startDisplay = FormatDate(this.StartDate);
        var endDisplay = FormatDate(this.EndDate);

        string FormatDate(DateTime date)
        {
            return date.ToShortDateString();
        }

        return $"{startDisplay} to {endDisplay}";
    }
}
```

Then we end up with something like:

```c#
public DateRange Range { get; protected set; }
```

## Email
The Email type encapsulates validation in one place. Now there is no need to repeat validation logic in multiple places, or create a separate helper class for reuse.

```c#
using CSharpFunctionalExtensions;
using System;
using System.Collections.Generic;
using System.Text.RegularExpressions;

public class Email : ValueObject
{
    private Email() { }

    private Email(string email)
    {
        var parts = email.Split('@');

        this.Address = email;
        this.User = parts[0];
        this.Host = parts[1];
    }

    public static Result<Email> Create
        (
            string email
        )
    {
        if (String.IsNullOrWhiteSpace(email))
        {
            return Result.Failure<Email>
            (
                "The email address must contain a value."
            );
        }

        var isValid = Regex.IsMatch
        (
            email,
            @"^([\w\.\-]+)@([\w\-]+)((\.(\w){2,3})+)$"
        );

        if (isValid)
        {
            var address = new Email(email);

            return Result.Success(address);
        }
        else
        {
            return Result.Failure<Email>
            (
                $"The email address '{email}' is invalid."
            );
        }
    }

    public string Address { get; private set; }

    public string Host { get; private set; }

    public string User { get; private set; }

    protected override IEnumerable<object> GetEqualityComponents()
    {
        yield return this.Address.ToUpper();
    }

    public override string ToString()
    {
        return this.Address;
    }
}
```

## PersonName
A persons name can be broken own into multiple parts: first name, middle name, last name, title and suffix. Why not wrap all of these into a single type?

```c#
using CSharpFunctionalExtensions;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;

public class PersonName : ValueObject
{
    private PersonName() { }

    private PersonName
        (
            string firstName,
            string middleName,
            string lastName,
            string title = null,
            string suffix = null
        )
    {
        this.FirstName = firstName;
        this.MiddleName = middleName;
        this.LastName = lastName;
        this.Title = title;
        this.Suffix = suffix;
    }

    public static Result<PersonName> Create
        (
            string firstName,
            string middleName,
            string lastName,
            string title = null,
            string suffix = null
        )
    {
        var allNames = new string[]
        {
            firstName,
            middleName,
            lastName
        };

        var allEmpty = allNames.All
        (
            name => String.IsNullOrWhiteSpace(name)
        );

        if (allEmpty)
        {
            return Result.Failure<PersonName>
            (
                "At least one name (first, middle or last) is required."
            );
        }
        else
        {
            var personName = new PersonName
            (
                firstName,
                middleName,
                lastName,
                title,
                suffix
            );

            return Result.Success(personName);
        }
    }

    public string FirstName { get; private set; }

    public string MiddleName { get; private set; }

    public string LastName { get; private set; }
    
    public string Title { get; private set; }

    public string Suffix { get; private set; }

    /// <summary>
    /// Gets a display name for presentation purposes (excluding title and suffix)
    /// </summary>
    public string DisplayName
    {
        get
        {
            return Concat
            (
                this.FirstName,
                this.MiddleName,
                this.LastName
            );
        }
    }

    /// <summary>
    /// Gets the full name of the person (including title and suffix)
    /// </summary>
    public string FullName
    {
        get
        {
            return Concat
            (
                this.Title,
                this.FirstName,
                this.MiddleName,
                this.LastName,
                this.Suffix
            );
        }
    }

    /// <summary>
    /// Concatenates an array of words into a single string separated by spaces
    /// </summary>
    /// <param name="words">The words to concatenate</param>
    /// <returns>The concatenated string</returns>
    private string Concat
        (
            params string[] words
        )
    {
        var builder = new StringBuilder();

        foreach (var word in words)
        {
            if (false == String.IsNullOrWhiteSpace(word))
            {
                builder.Append($"{word} ");
            }
        }

        return builder.ToString().Trim();
    }

    protected override IEnumerable<object> GetEqualityComponents()
    {
        yield return this.FirstName.ToUpper();
        yield return this.MiddleName.ToUpper();
        yield return this.LastName.ToUpper();
        yield return this.Title.ToUpper();
        yield return this.Suffix.ToUpper();
    }

    public override string ToString()
    {
        return this.DisplayName;
    }
}
```

## PhoneNumber
As with email addresses, telephone numbers require validation. However, validating phone numbers is very complex and can vary depending on country, region, if it's a landline or mobile etc. So, I've kept the validation very simple for this implementation. Feel free to add your own custom validation.

```c#
using CSharpFunctionalExtensions;
using System;
using System.Collections.Generic;
using System.Linq;

public class PhoneNumber : ValueObject
{
    private PhoneNumber() { }

    private PhoneNumber(string number)
    {
        this.Number = number.Trim();
    }

    public static Result<PhoneNumber> Create
        (
            string number
        )
    {
        var isValid = true;
        var failureMessage = String.Empty;

        if (String.IsNullOrWhiteSpace(number))
        {
            isValid = false;
            failureMessage = "The phone number must contain a value.";
        }

        if (false == number.Any(Char.IsDigit))
        {
            isValid = false;
            failureMessage = "The phone number must contain at least one digit.";
        }

        if (isValid)
        {
            var pn = new PhoneNumber(number);

            return Result.Success(pn);
        }
        else
        {
            return Result.Failure<PhoneNumber>
            (
                failureMessage
            );
        }
    }

    public string Number { get; private set; }

    protected override IEnumerable<object> GetEqualityComponents()
    {
        yield return this.Number.ToUpper();
    }

    public override string ToString()
    {
        return this.Number;
    }
}
```