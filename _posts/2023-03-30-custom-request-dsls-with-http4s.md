---
title: Custom Request DSLs with Http4s
layout: post
created: 2023-03-14T19:51:58+00:00
modified: 2023-03-20T21:20:26+00:00
---

> This post focuses on Http4s but as the techniques are entirely native to Scala, so really only familiarity with Scala pattern matching is a must-have.

I've recently been playing around with Http4s' routing DSL and adding some custom routing mechanics of my own. Sometimes we want to access request information generically in a way that isn't strictly relevant to the route's normal purpose. A great example of this is Http4s' `AuthedRoutes`.

```scala
val normal: HttpRoutes[IO] = HttpRoutes.of {
  case GET -> / "user" / UserId(user) / "resource" 
}

val authed: AuthedRoutes[UserId, IO] = AuthedRoutes.of {
  case GET -> / "user" / "resource" as user
}
```

The difference is subtle but that small `as user` at the end of the second route definition is doing a lot of heavy lifting around user authentication. Even better as the route type itself is modified we can't forget to add authentication to a route (which we might if were adding auth route by route).

For my change I'm going to focus primarily on a relatively neutral change that works with the normal Http4s request and response types. Let's work with a simple requirement.

> Modify the route's behaviour based on the `User-Agent` of the caller.

How might that look? Let's sketch out a possible DSL.

```scala
HttpRoutes.of[IO] {
  case Service.Distributor using GET -> Root / "resource" / IntVar(id) =>
}
```

To understand how we might add this we need to explore how Http4s cleverly uses Scala's pattern match system. When we pattern match on a case class Scala is actually invoking a method called `unapply` to decide if the given value “matches”. Typically, that method might look something like this.

```scala
object Foo {
  def unapply(foo: Foo): Option[Int]
}
```

In this case the implementation knows how to decompose a `Foo` into a `Int`, by using the class' `foo` field. In some cases we might not be able to access a field like that (for example, an ADT) and so the optional return type allows us to indicate the match failed. This method doesn't have to be synthetic and Scala allows us to define our own custom `unapply` matchers, which Scala refers to as [extractor objects](https://docs.scala-lang.org/tour/extractor-objects.html).

Http4s itself has some great examples of when we might want to use extractors. The status matcher `Succesful` matches response statuses within the 2xx range, with similar matchers for 4xx and 5xx too. This showcases the first type of matcher — a way of grouping similar terms in a match.

The other useful property of Scala's matchers is that they can be nested the same way that our data can be nested. For example `case Right(Some(3)) => ` first applies the `Either` matcher before applying the `Option` matcher to the result. Surprisingly this same nesting is how Scala achieves another familiar pattern match; the list matcher. This pattern `case 1 :: 2 :: 3 :: Nil => ` is actually a `::` matcher which unpacks the elements of the list one by one. This special syntax eliminates the bracket syntax we're used to seeing in favour of something more readable. Understanding this you might begin to see how the Http4s DSL takes shape.

Through a series of DSL objects made available via the `Htp4sDSL` trait we can string a series of matchers along to define our expected request structure. Jumping to implementation (or checking the source code) gives us an idea on how to structure our own custom matcher.

{% raw %}
```scala
object -> {

  /** HttpMethod extractor:
    * {{{
    *   (request.method, Path(request.path)) match {
    *     case Method.GET -> Root / "test.json" => ...
    * }}}
    */
  def unapply[F[_]](req: Request[F]): Some[(Method, Path)] = ...
}
```
{% endraw %}

This is the first component of a Http4s route. The input is the inbound request itself. The output is two chunks that have been split from the request; the method and the path. Through the same syntax as `::` we saw earlier, Scala can match the request as a method on the left and a path on the right. Scala also allows us to match against specific values, so we can further refine our match with an exact method. The path however is then fed to another `Path` matcher which decomposes the value further.

```scala
object / {
  def unapply(path: Path): Option[(Path, String)] = ...
}
```

This path matcher consumes a `Path` and splits to a `String` component on its right-hand side. There's no limit to how many times we can repeat this pattern on the left-hand side as a `Path` is always returned. Hopefully you can start to see a pattern emerging, and we can start implementing our own. Let's revisit our desired syntax.

```scala
HttpRoutes.of[IO] {
  case Service.Distributor using GET -> Root / "resource" / IntVar(id) =>
}
```

What are we actually looking at then? Our matcher wants to look something like `Request[IO] => (Service , Request[IO])` because we are consuming an inbound request, and returning a `Service` and the request itself. This allows us to match on the identified user while still allowing the next matchers to consume the request. Implementing this is fairly simple.

<script src="https://scastie.scala-lang.org/s5swyEcORHGz0wYU67AZ4w.js"></script>

> You might notice the `unapply` method actually has a generic type parameter. Pattern matches do not support type parameters so if you include them they must be inferred from the matcher input.

The logic here could become more complex if we wanted it to, for example we could extract a specific user-agent version too. Like `AuthedRoutes` the best syntax additions are obvious and limited. By changing the order of the outputs we could move our syntax the end of the match rather than the front. Experimenting with the values you extract in your match brings up some interesting possibilities. Take Http4s' `->>` matcher as an example.

{% raw %}
```scala
  /** Extractor to match an http resource and then enumerate all supported methods:
    * {{{
    *   (request.method, Path(request.path)) match {
    *     case withMethod ->> Root / "test.json" => withMethod {
    *       case Method.GET => ...
    *       case Method.POST => ...
    * }}}
    *
    * Returns an error response if the method is not matched, in accordance with [[https://datatracker.ietf.org/doc/html/rfc7231#section-4.1 RFC7231]]
    */
  def unapply[F[_]: Applicative](
      req: Request[F]
  ): Some[(PartialFunction[Method, F[Response[F]]] => F[Response[F]], Path)] = ...
```
{% endraw %}

By returning a partial function the DSL enables some more interesting features, like being able to affect the routes response and delegate more control back to the route itself. Taking this idea further you could even have dynamic dependency injection depending on the requests values. 

The big question is… _should you?_ How much you want to implement here is up to you, the technique itself is fairly simple and yet powerful at the same time. However, a custom DSL can sometimes hide too much information, or introduce confusing syntax that is problematic for newcomers. These techniques don't only apply to Http4s and could be useful in any situation where you want to encode complex event matching to your domain. Even just a simple group matcher can turn `case Left(Some(event)) if event.timestamp > cutOffTime =>` into `case LiveEvent(event) =>`. 
