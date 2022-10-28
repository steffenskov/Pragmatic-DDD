# Pragmatic-DDD

A pragmatic to-the-point guide for implementing DDD in C#.

# Introduction

Domain-Driven Design (also known as DDD) is a lot more than just writing code. This guide expects you to already know about the DDD process, and have reached the point of starting to implement software using DDD.
If you don't yet know about the process I highly recommend relying on the books on the topic, rather than a brief guide like this. It's a rather large topic after all.

# Vocabulary

## Aggregate

An `Aggregate` in DDD consists of an `Aggregate root` as well as `0-n other aggregates`. For simplicity I usually make the `Aggregate` itself its root, rather than nesting the root into it. e.g.

```
class Order // Is its own root
{
	public OrderId Id { get; private set; } // Part of the root
	public OrderDetails Details { get; private set; } // Part of the root
	public IList<OrderLine> OrderLines { get; private set; } // 0-n other aggregates
}
```

You could split the root from the `Aggregate` like this instead, but I don't recommend it as you IMHO gain little for the added complexicity:

```
class OrderAggregate // Has nested root
{
	public Order Root { get; private set; } // Root as a whole
	public IList<OrderLine> OrderLines { get; private set; } // 0-n other aggregates
}
```

Furthermore an `Aggregate` is responsible for maintaining its own business rules and state. As such all data should be privately set, so mutations only can occur via methods on the `Aggregate` that in turn maintains business rules and a valid state.
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

To remember: Use fine granularity with `Value Objects` to gain the most benefit from them.
