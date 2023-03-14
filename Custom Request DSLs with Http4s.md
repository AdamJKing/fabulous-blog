---
created: 2023-03-14T19:51:58+00:00
modified: 2023-03-14T20:08:47+00:00
---

# Custom Request DSLs with Http4s

Recently I've been playing around with Http4s' DSL it provides to define our service's endpoints. The technique Http4s uses is a powerful set of combined pattern matchers that allows us to layer endpoint information into Scala's pattern match mechanic. (The cost of this unfortunately is a loss of inspectability in our endpoint definitions.)

// Using Auth routes as an example?

In some use-cases we might want to capture some information from the request as part of our endpoint definition beyond the common components. For the sake of example let's say we want to change endpoint behaviour based on the `User-Agent` header.

How might that look? Let's sketch out our desired DSL.

```scala
HttpRoutes.of[IO] {
  case User(Bob) ~ Get / "resource" / IntVar(id) =>
}
```

To understand how to achieve this we need to dig into how the existing DSL matchers work.
