
``` scala


/**
  * A naive implementation of an FIR filter with an arbitrary number of taps.
  */
class ScalaFirFilter(taps: Seq[Int]) {
  var pseudoRegisters = List.fill(taps.length)(0)

  def poke(value: Int): Int = {
    pseudoRegisters = value :: pseudoRegisters.take(taps.length - 1)
    var accumulator = 0
    for(i <- taps.indices) {
      accumulator += taps(i) * pseudoRegisters(i)
    }
    accumulator
  }
}


```

```
::    List class方法，将value放到List的最前面

take   List class方法，取前n个值，返回值为List



indice   Seq trait 方法，返回taps的index range
```


``` scala
class MyManyElementFir(consts: Seq[Int], bitWidth: Int) extends Module {
  val io = IO(new Bundle {
    val in = Input(UInt(bitWidth.W))
    val out = Output(UInt(bitWidth.W))
  })

  val regs = mutable.ArrayBuffer[UInt]()
  for(i <- 0 until consts.length) {
      if(i == 0) regs += io.in
      else       regs += RegNext(regs(i - 1), 0.U)
  }
  
  val muls = mutable.ArrayBuffer[UInt]()
  for(i <- 0 until consts.length) {
      muls += regs(i) * consts(i).U
  }

  val scan = mutable.ArrayBuffer[UInt]()
  for(i <- 0 until consts.length) {
      if(i == 0) scan += muls(i)
      else scan += muls(i) + scan(i - 1)
  }

  io.out := scan.last
}
```



**Example: Add run-time configurable taps to our FIR**  
The following code adds an additional `consts` vector to the IO of our FIR generator which allows the coefficients to be changed externally after circuit generation. This is done with the Chisel collection type `Vec`. `Vec` supports many of the scala collection methods but it can only contain Chisel hardware elements. `Vec` should only be used in situations where ordinary Scala collections won't work.  
Basically this is in one of two situations.

1. You need a collection of elements in a Bundle, typically a Bundle that will be used as IO.
2. You need to access the collection via an index that is part of the hardware (think Register File).



```scala
class MyManyDynamicElementVecFir(length: Int) extends Module {
  val io = IO(new Bundle {
    val in = Input(UInt(8.W))
    val out = Output(UInt(8.W))
    val consts = Input(Vec(length, UInt(8.W)))
  })

  // Reference solution
  val regs = RegInit(VecInit(Seq.fill(length - 1)(0.U(8.W))))
  for(i <- 0 until length - 1) {
      if(i == 0) regs(i) := io.in
      else       regs(i) := regs(i - 1)
  }
  
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

  io.out := scan(length - 1)
}
```