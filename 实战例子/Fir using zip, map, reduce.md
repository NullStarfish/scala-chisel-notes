```scala
val muls = Wire(Vec(length, UInt(8.W)))
for(i <- 0 until length) {
  if(i == 0) muls(i) := io.in * io.consts(i)
  else       muls(i) := regs(i - 1) * io.consts(i)
}

val scan = Wire(Vec(length, UInt(8.W)))
for(i <- 0 until length) {
  if(i == 0) scan(i) := muls(i)
  else scan(i) := muls(i) + scan(i - 1)
}

io.out := scan(length - 1
```



```scala
io.out := (taps zip io.consts).map { case (a, b) => a * b }.reduce(_ + _)
```


case (a, b)
[[匿名函数]]

- assume `taps` is the list of all samples, with `taps(0) = io.in`, `taps(1) = regs(0)`, etc.
- `(taps zip io.consts)` takes two lists, `taps` and `io.consts`, and combines them into one list where each element is a tuple of the elements at the inputs at the corresponding position. Concretely, its value would be `[(taps(0), io.consts(0)), (taps(1), io.consts(1)), ..., (taps(n), io.consts(n))]`. Remember that periods are optional, so this is equivalent to `(taps.zip(io.consts))`.
- `.map { case (a, b) => a * b }` applies the anonymous function (takes a tuple of two elements returns their product) to the elements of the list, and returns the result. In this case, the result is equivalent to `muls` in the verbose example, and has the value `[taps(0) * io.consts(0), taps(1) * io.consts(1), ..., taps(n) * io.consts(n)]`. You'll revisit anonymous functions in the next module. For now, just learn this syntax.
- Finally, `.reduce(_ + _)` also applies the function (addition of elements) to elements of the list. However, it takes two arguments: the first is the current accumulation, and the second is the list element (in the first iteration, it just adds the first two elements). These are given by the two underscores in the parentheses. The result would then be, assuming left-to-right traversal, `(((muls(0) + muls(1)) + muls(2)) + ...) + muls(n)`, with the result of deeper-nested parentheses evaluated first. This is the output of the convolution.