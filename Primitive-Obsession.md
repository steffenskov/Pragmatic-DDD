# Primitive obsession

This describes a situation where most of your codebase is using primitives. A simple (maybe slightly oversimplified) example:

```
public class User
{
	public Guid Id { get; set; }
	public string Username { get; set; }
	public string Password { get; set; }
}
```

Whilst this works, it's prone to quite a few issues:

- Swapping ids between different types of `Entities`, resulting in working on the wrong dataset.
- Needing to implement validation rules in multiple places
- It conveys little information to the caller, about what's expected for a given property

Here's a less-primitive version of the same class:

```
public class User
{
	public UserId Id { get; set; }
	public string Username { get; set; }
	public PbKdf2Hash Password { get; set; }
}
```

One could even go as far as to encapsulate `Username` as well, since there are surely business constraints around that one as well.

The benefits of this approach boils down to basically solving the issues listed above:

- A `UserId` cannot be swapped out with e.g. an `OrderId` in a method that expects the former.
- Validation rules can be encapsulated exactly once: Within the `UserId` class.
- Information about what to expect is conveyed more clearly, e.g. here we see that `Password` is indeed hashed in a fashion suited for dealing with passwords. Whereas in the first example, it could be in clear-text for all we know.
