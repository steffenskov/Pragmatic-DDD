# Onion Architecture

The Onion architecture is a slightly different approach to dependencies, compared to well-known 3-layer model consisting of `Application`->`Domain`->`Infrastructure`.

The simplest form of Onion still contains these 3 layers:

- `Application` (could be e.g. a REST API)
- `Domain` (Houses all your business logic)
- `Infrastructure` (Houses implementations of stuff like persistence, I/O etc.)

However the way the 3 projects references each-other is different:

- `Domain` doesn't reference any other layer, rather it defines interfaces for the `Infrastructure` needs (e.g. a `Repository` interface for persistence)
- `Infrastructure` only references `Domain`, and implements concrete classes for all the interfaces defined in `Domain`
- `Application` references both other projects, and uses `Dependency Injection` for injection the `Infrastructure` into the `Domain`. `Infrastructure` is purely used for DI, everything else the `Application` project does is based around the `Domain` interfaces.

This architecture, like everything else, has pros and cons:

Pros:

- Low coupling: `Domain` doesn't know anything about its `Infrastructure`, as such the entire `Infrastructure` project can be replaced without touching the `Domain`.
- Testability: `Domain` can be built and unit-tested using `mocks` without even implementing the `Infrastructure`.
- Separation of Concerns: `Domain` houses only the business logic, `Infrastructure` houses only persistence, I/O implementations etc.

Cons:

- Over-exposure of inner workings: `Domain` needs to expose its interfaces as public, for `Infrastructure` to be able to implement them. This however means the `Application` can directly access said interfaces, and often this is not desired. e.g. a persistence `Repository` should not be called directly from the `Application` layer, as this circumvents all your business logic. (Remember the business logic resides in the `Domain` project, NOT the implementations within `Infrastructure`)
