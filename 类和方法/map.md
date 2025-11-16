`List[A].map` has type signature `map[B](f: (A) ⇒ B): List[B]`. You'll learn more about types in a later module. For now, think of types A and B as `Int`s or `UInt`s, meaning they could be software or hardware types.

In plain English, it takes an argument of type `(f: (A) ⇒ B)`, or a function that takes one argument of type `A` (the same type as the element of the input List) and returns a value of type `B` (which can be anything). `map` then returns a new list of type `B` (the return type of the argument function).

As we've already explained the behavior of List in the FIR example, let's get straight into the examples and exercises:

``` scala
println(List(1, 2, 3, 4).map(x => x + 1))  // explicit argument list in function
println(List(1, 2, 3, 4).map(_ + 1))  // equivalent to the above, but implicit arguments
println(List(1, 2, 3, 4).map(_.toString + "a"))  // the output element type can be different from the input element type

println(List((1, 5), (2, 6), (3, 7), (4, 8)).map { case (x, y) => x*y })  // this unpacks a tuple, note use of curly braces

// Related: Scala has a syntax for constructing lists of sequential numbers
println(0 to 10)  // to is inclusive , the end point is part of the result
println(0 until 10)  // until is exclusive at the end, the end point is not part of the result

// Those largely behave like lists, and can be useful for generating indices:
val myList = List("a", "b", "c", "d")
println((0 until 4).map(myList(_)))

```

