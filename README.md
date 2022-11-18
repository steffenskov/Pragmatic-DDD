# Pragmatic-DDD

A pragmatic to-the-point guide for implementing DDD in C#.

# Introduction

Domain-Driven Design (also known as DDD) is a lot more than just writing code. This guide expects you to already know about the DDD process, and have reached the point of starting to implement software using DDD.
If you don't yet know about the process I highly recommend relying on the books on the topic, rather than a brief guide like this. It's a rather large topic after all.

Finally here's a link to a sample solution, implementing much of what's described below: [DDD-CQRS-Template](https://github.com/steffenskov/DDD-CQRS-Template). I find it's often beneficial to see the actual code rather than just reading about it.

# Prerequisites for reading

In order to gain the most benefit from this, you should be familiar with the following concepts:

- [CQRS](https://en.wikipedia.org/wiki/Command%E2%80%93query_separation)
- [Onion Architecture](https://en.everybodywiki.com/Onion_Architecture)
- [MediatR / mediator pattern](https://github.com/jbogard/MediatR/wiki)

Note: CQRS and Mediator pattern aren't strictly necessary, however they provide a very nice decoupling, which makes following the [Open/Closed principle](https://en.wikipedia.org/wiki/Open%E2%80%93closed_principle) a lot easier.

# DDD Terminology

## Aggregate

An `Aggregate` in DDD consists of an `Aggregate root` as well as `0-n other aggregates`. For simplicity I usually make the `Aggregate` itself its root, rather than nesting the root into it. e.g.

```
class Order // Is its own root
{
	public OrderId Id { get; private set; } // Part of the root
	public OrderDetails Details { get; private set; } // Part of the root
	public IList<Orderline> Orderlines { get; private set; } // 0-n other aggregates
}
```

You could split the root from the `Aggregate` like this instead, but I don't recommend it as you IMHO gain little for the added complexicity:

```
class OrderAggregate // Has nested root
{
	public Order Root { get; private set; } // Root as a whole
	public IList<Orderline> Orderlines { get; private set; } // 0-n other aggregates
}
```

Furthermore an `Aggregate` is responsible for maintaining its own business rules and state. As such all data should be privately set, so mutations only occur via methods on the `Aggregate` that in turn maintains business rules and a valid state.
To ensure a valid state, the method should validate ALL incoming data before mutating anything. e.g.

```
class Order
{
	public OrderId Id { get; private set; }
	public OrderDetails Details { get; private set; }

	public static Order Create(OrderId id, OrderDetails details)
	{
		ValidateId(id);
		ValidateDetails(details);

		return new Order { Id = id, Details = details };
	}

	public Order UpdateDetails(OrderDetails newDetails)
	{
		ValidateDetails(newDetails);

		this.Details = newDetails;
	}
}
```

The simplest way to ensure state never becomes invalid, is IMHO to throw an exception in your `ValidateX` methods to stop execution prior to the mutation part.
When creating a new instance of an `Aggregate` feel free to use a static `Create` method like above, a `constructor` or even an instance method called upon an empty `Aggregate`.

To remember: An aggregate is itself responsible for ensuring its state is always valid according to the business rules.

## Aggregate root

An `Aggregate root` is an `Entity` (look that one up below, you'll see we come full circle back to `Aggregate`)
The rules for valid state are the same on the root as on `Aggregate`.

## Entity

An `Entity` is a combination of data that has an inherit identity. Typically in a relational database, an `Entity` represents a single row in a table with a primary key. As such this is an entity:

```
class Order
{
	public OrderId Id { get; private set; } // It has an identity
	public OrderDetails Details { get; private set; } // And other data as well
}
```

Notice how an `Aggregate` always has an `Aggregate root` and _may_ also have other aggregates. Since an `Aggregate root` _is_ an `Entity` you can simplify the language around these concepts if you merged the root into the `Aggregate` itself like suggested above.
This essentially means you completely ignore the term `Entity` and just call all of it an `Aggregate`. This is very much subject for debate, but I find it eases the communication in a team and I haven't really missed the `Entity` term. The primary thing here is probably making a choice and sticking to it, to avoid confusion in your teams.

To remember: Just call it an `Aggregate` ;-)

## Value Object

A `Value Object` is pretty much an `Entity` without a specified identity. It is data that as a whole describes a concept, and it can consist of `1-n` data points. A couple of examples:

```
struct Location
{
	public decimal Latitude { get; set; }
	public decimal Longitude { get; set; }
}
```

```
struct TemperatureCelcius
{
	public decimal Value { get; set; }
}
```

As you can see a Value Object can encapsulate multiple parts, that becomes a bigger whole. Or simply "wrap" a primitive value, because there's some inherent meaning to the concept that would otherwise be missed.
Any business rules regarding value objects should be implemented on the `Value Object` itself, to ensure it's never brought to an invalid state. For instance our `TemperatureCelcius` `Value Object` above should ensure its `Value` never dips below absolute zero, because that's an invalid/impossible temperature. Likewise there's a valid range of values for both `Latitude` and `Longitude` that Location should adhere too.

Food for thought: Should `Latitude` and `Longitude` each be their own `Value Objects` with their own rules for ensure a valid value, which are then further encapsulated into `Location`? YMMV here, but I'd actually say "yes".

To remember: Use fine granularity with `Value Objects` to gain the most benefit from them. And like `Aggregate` they should maintain a valid state at all times.

# CQRS Terminology

We'll be talking `Command-Query Request Segregation` (CQRS) soon as it works very well with DDD.

## Command

A command changes something on one or many `Aggregates`, it either doesn't return anything or just returns whatever `Aggregate(s)` it was issued on.
This is purely a `Write` operation.
When using `Mediator` (which I recommend) there must be exactly one `Handler` that deals with the `Command`.

## Query

A query finds something, it could be one or many `Aggregates`, `ValueObjects`, etc. It never changes anything.
This is purely a `Read` operation.
When using `Mediator` there must be exactly one `Handler` that deals with the `Query`.

## Notification

A notification is a broadcasted information, sent in a "to-whom-it-many-concern" fashion. This is really neither a `Read` nor `Write` operation, but it can lead to both depending on how any `Handlers` choose to deal with it.
When using `Mediator` there can be `0-n handlers` for notifications.

# Architecture

I like to match DDD with the `CQRS` pattern, as I find the commands make for excellent mutation methods on your `Aggregates`.
To this end I'd recommend using `Mediator` for implementing `CQRS`, it's great at it and it makes for a very `SOLID` architecture with very low coupling.

Furthermore I'd also suggest implenting the [Onion architecture](Onion-Architecture.md) in your solution, as this keeps responsibilities clearly seperated between your `Domain` code (where all the business takes place), your `Application` (could be a Web API for instance) and your `Infrastructure` (which often just boils down to persistence).

So our basic architecture has 3 projects:

- Application
- Domain
- Infrastructure

If you're dealing with a broad `Domain` consisting of many sub-domains, feel free to create multiples of both `Domain` and corresponding `Infrastructure` projects. Your `Application` should probably be just one, as if you're doing `Microservices`, you should ideally have multiple separate solutions instead.

## Domain architecture

This is really where the bulk of your DDD architecture work takes place, so we'll start here. I tend to group my code into namespaces (and directories) by "sub" domain. This is not to be confused with a `Sub-domain` from the DDD process, but rather a small subset of aggregates that, when combined, cover a specic part of your `Domain`. For instance you might be dealing with `Orders` and `Orderlines`, both of which are used for dealing with some sort of sale. They would make a fine "sub" domain namespace:

- Domain
  - Sale

Now for the actual structure of the `Sale` namespace I suggest the following (feel free to omit any that aren't relevant for your particular namespace):

- Domain
  - Sale
    - Aggregates
    - Commands
    - Queries
    - Repositories
    - ValueObjects

Supposing we're doing basic CRUD for `Order` and `Orderline` (and keeping the two as separate `Aggregates` - see below why), the final structure could look like this:

- Domain
  - Sale
    - Aggregates
      - Order.cs
      - Orderline.cs
    - Commands
      - Order
        - OrderCommandHandler.cs
        - OrderCreateCommand.cs
        - OrderDeleteCommand.cs
        - OrderUpdateCommand.cs
      - Orderline
        - OrderlineCommandHandler.cs
        - OrderlineCreateCommand.cs
        - OrderlineDeleteCommand.cs
        - OrderlineUpdateCommand.cs
    - Queries
      - Order
        - OrderGetAllQuery.cs
        - OrderGetSingleQuery.cs
      - Orderline
        - OrderlineGetAllQuery.cs
        - OrderlineGetSingleQuery.cs
    - Repositories
      - IOrderRepository.cs // Interface
      - IOrderlineRepository.cs // Interface
    - ValueObjects
      - OrderDetails.cs
      - OrderId.cs
      - OrderlineId.cs

Note: You'll rarely have just an `UpdateCommand`, rather you'd have specific mutational commands that adhere to the product specific workflows. It could be a `ShipOrderCommand`, `RefundOrderCommand` etc.

Note: The repositories are `interfaces`, it's the `Infrastructure` project that's in charge of their actual implementations as per the `Onion architecture`.

## Inter-domain communication

Whilst the above does introduce repositories, I strongly advise against using these when communicating across boundaries. Rather only the relevant `Command/QueryHandler` should know about the corresponding `Repository`.
Instead use `Commands/Queries` for communication across boundaries. This creates a very low coupling AND ensures the same contract is being used, regardless of where the communication starts.
This is also called `East-West communication` and is actually a fine way to handle boundary-crossing concerns.

In fact when using `Mediator` the same approach can be used even when crossing `Domains` (assuming you have more than one `Domain` project in your solution). One caveat with this though, is the `Command/Query` definition should be put into a `SharedKernel` project instead of residing in either of the two `Domain` projects. This way neither `Domain` will depend on the other, but rather they'll both depend on a very _slim_ `SharedKernel`. (Slim being the keyword here, a bloated `SharedKernel` is never desirable)

## Domain-Infrastructure communication

This one is pretty simple: Your `Infrastructure` _NEVER_ calls your `Domain`. Period.
If you find the need to do this, you've put business logic into your `Infrastructure`, which should instead be dealt with in your `Domain`. (where inter-domain communication is allowed as just discussed)
This is also known as `North-South communication` and it's basically meant to be a one-way street.
If you still want to do this, know that it can lead to circular loops in both your dependencies and your logic. Neither is something you want... Ever.

## Cross cutting concerns

Cross cutting concerns cover functionality that applies across your domain, usually it's a `generic domain` or `supporting domain` and as such you're likely to not implement this yourself.
Common types of cross cutting concerns include: Authentication/Authorization, Caching, Compression, Logging, Encryption, Exception handling and probably a bunch more.

If you're building an API the simple answer to dealing with this is mainly `Middleware`. This way you can inject the functionality across your entire API in one swoop.

For other types of applications or API concerns where `Middleware` doesn't cut it, I'd suggesting looking into the `Proxy`/`Decorator` design pattern.
This too enables you to implement the functionality once, and apply it broadly afterwards.

# Implementation details

## Aggregate

I tend to _not_ include the `0-n other aggregates` on an aggregate. The reason is fairly simple: It complicates persistence and few ORMs are actually well equipped to deal with this (when your persistence layer is a relational database).
As such I'd rather suggest keeping the aggregates separated, and dealing with the relationship logic through a mixture of commands, queries and notifications.
This is a pragmatic approach to dealing with lackluster persistence, and as such feel free to do things differently if your persistence handles this well or you just like a challenge ;-)

## Command/Query/Notification

I find the C# type `record` works wonderful for these, as none of these should have any logic - they're purely `Data Transfer Objects` (DTO). An example using the `primary constructor` syntax:

```
public record OrderDeleteCommand(OrderId Id) : IRequest<Order?>;
```

This `Command` returns the `Aggregate` being deleted, if it was found. Otherwise null.
Note: I won't go into `class vs. struct` memory allocation concerns here, but you could justify using `record struct` for many `Commands`/`Queries`/`Notifications`.

## Handling Commands

The `CommandHandler` should fetch the `Aggregate(s)` to issue the command onto using the `Repository`. For each `Aggregate` being commanded, the handler then invokes the corresponding method on the `Aggregate`. E.g.

```
public async Task<Order?> Handle(OrderDeleteCommand command, CancellationToken cancellationToken)
{
	var aggregate = await _repository.GetSingleAsync(command.Id, cancellationToken);
	if (aggregate is null)
		return null;

	await aggregate.DeleteAsync(command, cancellationToken); // Async to allow validation of relationships using Queries
	await _repository.SaveAggregateAsync(aggregate, cancellationToken); // Notice how the aggregate still exists after being deleted, the aggregate needs to mutate into a deleted one by itself, and only then can we persist that information
	return aggregate;
}
```

## Handling Queries

The `QueryHandler` is a lot simpler, it won't issue methods on `Aggregates` but rather purely relies on the `Repository` for fetching data based on the queries.

## Handling Notifications

The `NotificationHandler` (if you have one) should not know about a `Repository` at all, rather it should issue new `Commands`/`Queries`/`Notifications` for whatever it wants to do.
Often `Notifications` are crossing boundaries, and as such we're dealing with `Inter-domain communication`.
Even when not dealing with `Inter-domain communication` there's nothing wrong with using the already established contracts of our `Commands`/`Queries`/`Notifications` (rather, it's a good thing).

## Primitive obsession

[Primitive obsession](Primitive-Obsession.md) should be avoided in DDD where possible (it probably should be avoided in general too).
With DDD having a focus on using the same terminology across both process and code (What's known as the `Ubiquitous language`), encapsulating all concerns regarding a term is highly encouraged.
As such don't represent terms that are more than a primitive, as a primitive in code. For instance an `Order number` is "just a number", or is it? `Order numbers` are normally required to be unique, positive integers that fall within an unbroken range.
That's 3 business rules right there, which an `int` or `uint` doesn't properly capture.
As such an `OrderNumber` `Value Object` is probably the right way to represent it in your code.

This also extends to the `Id`s of your aggregates. Sure you can just use a `Guid` and call it a day, but in so doing, you lose the differentiation between the `Id` of an `Order` and a `User`, which obviously are two different things.
To simplify implementation of this I'd recommend using my `StrongTypedId` package. (Check the NuGet list at the bottom of this document for links)

## Value Objects

`Value Objects` can often benefit from being implemented using `record` as well, because a `Value Object` has no inherent identity, it instead becomes the sum of its parts. The `record` type gives you this functionality for free.
Furthermore when using `record` with only the `primary constructor` syntax they're also `immutable` - a great perk for thread safety, caching race-conditions and many other scenarios that are otherwise error-prone.
Our `Location` `Value Object` from earlier could thus be written like this instead:

```
public record struct Location(decimal Latitude, decimal Longitude);
```

Note: If the `Value Object` is subject to any form of validation rules, the `primary constructor` syntax isn't feasible. Instead either go with validation via constructor, or via setters on the properties. Regardless of the approach, the `record` type is still brilliant for comparison of `Value Objects` and strongly recommended.
Here are examples of the two approaches:

```
public record struct Location
{
	public decimal Latitude { get; }
	public decimal Longitude { get; }

	public Location(decimal latitude, decimal longitude)
	{
		ValidateLatitude(latitude);
		ValidateLongitude(longitude);
		Latitude = latitude;
		Longitude = longitude;
	}
}
```

```
public record struct Location
{
	private decimal _latitude, _longitude;

	public decimal Latitude
	{
		get => _latitude;
		init
		{
			ValidateLatitude(value);
			_latitude = value;
		}
	}

	public decimal Longitude
	{
		get => _longitude;
		init
		{
			ValidateLongitude(value);
			_longitude = value;
		}
	}
}
```

The latter approach supports `with` statements, like e.g. `Location with { Latitude = 42 }`, whereas the former doesn't. Depending on your use cases, each approach has its own merits and neither is a "wrong" way to go about it.

# NuGet package list

Here are a bunch of NuGet packages I'd recommend using in your application, when implementing DDD:

- [MediatR](https://www.nuget.org/packages/MediatR)
- [MediatR.Extensions.Microsoft.DependencyInjection](https://www.nuget.org/packages/MediatR.Extensions.Microsoft.DependencyInjection)
- [StrongTypedId](https://www.nuget.org/packages/StrongTypedId) (this one has further extensions available for e.g. NewtonSoft serialization)

If you're using a relational database for persistence, I further recommend these:

- [Dapper](https://www.nuget.org/packages/Dapper) (If using a relational database)
- [Dapper.DDD.Repository](https://www.nuget.org/packages/Dapper.DDD.Repository) (An extension framework for Dapper, created by myself - currently supports MS SQL and MariaDB/MySql)
- [Dapper.DDD.Repository.Sql](https://www.nuget.org/packages/Dapper.DDD.Repository.Sql) (If using MS SQL)
- [Dapper.DDD.Repository.MySql](https://www.nuget.org/packages/Dapper.DDD.Repository.MySql) (If using MariaDB/MySql)
- [Dapper.DDD.Repository.DependencyInjection](https://www.nuget.org/packages/Dapper.DDD.Repository.DependencyInjection)
- [StrongTypedId.Dapper.DDD.Repository](https://www.nuget.org/packages/StrongTypedId.Dapper.DDD.Repository)

# Example solution

I've created an example solution of this architecture, which can be found here: [DDD-CQRS-Template](https://github.com/steffenskov/DDD-CQRS-Template)
