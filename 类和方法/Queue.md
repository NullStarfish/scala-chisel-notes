``` scala
test(new Module {
    // Example circuit using a Queue
    val io = IO(new Bundle {
      val in = Flipped(Decoupled(UInt(8.W)))
      val out = Decoupled(UInt(8.W))
    })
    val queue = Queue(io.in, 2)  // 2-element queue
    io.out <> queue
  }) { c =>
    c.io.out.ready.poke(false.B)
    c.io.in.valid.poke(true.B)  // Enqueue an element
    c.io.in.bits.poke(42.U)
    println(s"Starting:")
    println(s"\tio.in: ready=${c.io.in.ready.peek().litValue}")
    println(s"\tio.out: valid=${c.io.out.valid.peek().litValue}, bits=${c.io.out.bits.peek().litValue}")
    c.clock.step(1)

    c.io.in.valid.poke(true.B)  // Enqueue another element
    c.io.in.bits.poke(43.U)
    // What do you think io.out.valid and io.out.bits will be?
    println(s"After first enqueue:")
    println(s"\tio.in: ready=${c.io.in.ready.peek().litValue}")
    println(s"\tio.out: valid=${c.io.out.valid.peek().litValue}, bits=${c.io.out.bits.peek().litValue}")
    c.clock.step(1)

    c.io.in.valid.poke(true.B)  // Read a element, attempt to enqueue
    c.io.in.bits.poke(44.U)
    c.io.out.ready.poke(true.B)
    // What do you think io.in.ready will be, and will this enqueue succeed, and what will be read?
    println(s"On first read:")
    println(s"\tio.in: ready=${c.io.in.ready.peek().litValue}")
    println(s"\tio.out: valid=${c.io.out.valid.peek().litValue}, bits=${c.io.out.bits.peek().litValue}")
    c.clock.step(1)

    c.io.in.valid.poke(false.B)  // Read elements out
    c.io.out.ready.poke(true.B)
    // What do you think will be read here?
    println(s"On second read:")
    println(s"\tio.in: ready=${c.io.in.ready.peek().litValue}")
    println(s"\tio.out: valid=${c.io.out.valid.peek().litValue}, bits=${c.io.out.bits.peek().litValue}")
    c.clock.step(1)

    // Will a third read produce anything?
    println(s"On third read:")
    println(s"\tio.in: ready=${c.io.in.ready.peek().litValue}")
    println(s"\tio.out: valid=${c.io.out.valid.peek().litValue}, bits=${c.io.out.bits.peek().litValue}")
    c.clock.step(1)
}
```