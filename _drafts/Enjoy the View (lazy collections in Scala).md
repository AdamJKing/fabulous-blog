---
created: 2024-08-18T16:44:37+01:00
modified: 2024-08-18T16:46:33+01:00
---

# Enjoy the View (lazy collections in Scala)

> This is aimed at relatively new Scala developers or those unfamiliar with Scala collection performance outside of the usual `List` or `Seq` types.

Scala's `List` type is a linked list implementation that provides a series of operations for traversing the list. One of the great properties of linked lists is that we don't need to evaluate every element in the collection if we are only interested in the first few. Lazily evaluated linked lists open the door to interesting algorithmic loopholes such as infinite lists, which have no clear end condition, or avoid computing expensive elements unnecessarily.

Lists in Scala are not lazy by default and so if we want this behaviour we'll actually need to use `List`'s sister class `LazyList`. This type works a lot like a list but instead the list is stored as an element and a "thunk", a function stored on the heap which can be used to calculate the next element. 

> Gotcha, lazy lists actually retain their elements after they're calculated so an infinitely sized `LazyList` can still risk overrunning your heap memory! If you want to discard elements you'll need to work recursively (with tail call optimisation)

What if we're not working with a lazy collection? We can actually work with any collection lazily by using a `View`. You can summon a view for a collection by using `.view` and continue to use the familiar collection traversal methods. The upside is that every operation we do will be evaluated lazily just like the lazy list! 

This example shows how the order of operations differs between a strict list and a lazy view on the same collection. 

As you can see the first and second maps are evaluated after the collection is fully traversed (meaning we've actually crossed the whole list twice). In the view however the map operations are actually combined making our update more efficient. Similarly, the full list is always reversed whereas the view isn't fully realised until we convert it back to a strict collection. 

A view isn't always the right choice for the job. Views are great for traversal but don't support functions that might traverse the collection in a random order (for example, sorting). Storing unevaluated thunks can also sometimes be more costly then simply evaluating the collection to begin with, for example situations where we might build a lot of views in one go.

Having worked in a couple of different Scala codebases I've noticed developers usually stick with a couple of standard collections. Hopefully this post encourages you to think more about how you access your data as well as how you store your data.
