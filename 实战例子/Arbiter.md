

我自己写的：
```scala
//> using repository https://central.sonatype.com/repository/maven-snapshots
//> using scala 2.13.17
//> using dep org.chipsalliance::chisel:7.3.0
//> using plugin org.chipsalliance:::chisel-plugin:7.3.0
//> using options -unchecked -deprecation -language:reflectiveCalls -feature -Xcheckinit
//> using options -Xfatal-warnings -Ywarn-dead-code -Ywarn-unused -Ymacro-annotations



import chisel3._
import chisel3.util._
import _root_.circt.stage.ChiselStage


class MyRoutingArbiter(numChannels: Int) extends Module {
  val io = IO(new Bundle {
    val in = Vec(numChannels, Flipped(Decoupled(UInt(8.W))))
    val out = Decoupled(UInt(8.W))
  } )

  // YOUR CODE BELOW
    val index = Wire(UInt(2.W))
    index := 0.U
    for (i <- 0 until numChannels) {
        when ( io.in(i).valid === 1.U ) 
            index := i.U
    }
    
    io.out.bits := io.in(index).bits
    io.out.valid := io.in(index).valid

    io.in.map(_.ready).foreach(ready => ready := io.out.ready)
}



object Main extends App {
  println(
    ChiselStage.emitSystemVerilog(
      gen = new MyRoutingArbiter(4),
      firtoolOpts = Array("-disable-all-randomization", "-strip-debug-info", "-default-layer-specialization=enable")
    )
  )
}

```


### 1. for循环中的When逻辑在展开后，变成纯并行逻辑，导致实际上优先级固定为最后一级
### 2. in.ready不能全部由out.ready来决定， 应该仅仅只让selected的channel的ready由out.ready决定，然后让

### 3. 所有输出的值都应先赋默认值，比如io.in.ready和io.out.valid


针对第一点
1. 其实直接让i从高位向低位迭代即可
```scala
   val index = Wire(UInt(2.W)) index := 0.U // 默认值，或者任何一个不可能的索引 // 迭代顺序: i = 3, 2, 1, 0 
   for (i <- (numChannels - 1) to 0 by -1) { when (io.in(i).valid) { index := i.U } }
   ```
2. 使用[[PriorityMux]]
```scala
val chosen = PriorityMux(io.in.map(_.valid), io.in)
```


针对第二点
我们需要有一个chosen
```scala
// 3. 将选中的通道连接到输出 
io.out.valid := chosen.valid io.out.bits := chosen.bits 
// 4. 将输出的 ready 信号连接到被选中的输入的 ready 信号 
chosen.ready := io.out.ready
```


完整代码：
```scala
class MyRoutingArbiter(numChannels: Int) extends Module { val io = IO(new Bundle { val in = Vec(numChannels, Flipped(Decoupled(UInt(8.W)))) 
  val out = Decoupled(UInt(8.W)) } ) 
  // ======== YOUR CODE BELOW (CORRECTED) ======== 
  // 1. 默认情况下，所有输入通道的 ready 都为 false 
  io.in.foreach(_.ready := false.B) 
  // 2. 使用 PriorityMux 来选择第一个有效的输入通道 
  // PriorityMux(select, sequence) 会从 sequence 中选择第一个 select 位为 true 的元素 
  // in.map(_.valid) 会生成一个 Bool 向量，例如 [false, true, false, true] 
  // in.map(d => (d.valid, d.bits)) 会生成一个 (Bool, UInt) 的元组向量 
  val chosen = PriorityMux(io.in.map(_.valid), io.in) 
  // 3. 将选中的通道连接到输出 
  io.out.valid := chosen.valid io.out.bits := chosen.bits 
  // 4. 将输出的 ready 信号连接到被选中的输入的 ready 信号 
  chosen.ready := io.out.ready }
```
这个chosen.ready := io.out.ready依然是错的：
PriorityMux的返回值是一个输出值。
firrtltool会不让你驱动这个硬件的输出端口。

改进方法是仅仅选出一个index，然后用index驱动端口：



```scala


class MyRoutingArbiter(numChannels: Int) extends Module { 
  val io = IO(new Bundle { val in = Vec(numChannels, Flipped(Decoupled(UInt(8.W)))) 
  val out = Decoupled(UInt(8.W)) } 
  ) 
  // ======== YOUR CODE BELOW (CORRECTED) ======== 
  // 1. 默认情况下，所有输入通道的 ready 都为 false 
  io.in.foreach(_.ready := false.B) 
  // 2. 使用 PriorityMux 来选择第一个有效的输入通道 
  // PriorityMux(select, sequence) 会从 sequence 中选择第一个 select 位为 true 的元素 
  // in.map(_.valid) 会生成一个 Bool 向量，例如 [false, true, false, true] 
  // in.map(d => (d.valid, d.bits)) 会生成一个 (Bool, UInt) 的元组向量 
  val chosenIndex = PriorityMux(io.in.map(_.valid), (0 until numChannels).map(_.U)) 
  // 3. 将选中的通道连接到输出 
  io.out.valid := io.in(chosenIndex).valid 
  io.out.bits := io.in(chosenIndex).bits 
  // 4. 将输出的 ready 信号连接到被选中的输入的 ready 信号 
  io.in(chosenIndex).ready := io.out.ready 
}
  
  

```



官方bootcamp炫技版：
```scala
class MyRoutingArbiter(numChannels: Int) extends Module {
  val io = IO(new Bundle {
    val in = Vec(numChannels, Flipped(Decoupled(UInt(8.W))))
    val out = Decoupled(UInt(8.W))
  } )

  // YOUR CODE BELOW
  io.out.valid := io.in.map(_.valid).reduce(_ || _)
  val channel = PriorityMux(
    io.in.map(_.valid).zipWithIndex.map { case (valid, index) => (valid, index.U) }
  )
  
  
  
  io.out.bits := io.in(channel).bits
  io.in.map(_.ready).zipWithIndex.foreach { case (ready, index) =>
    ready := io.out.ready && channel === index.U
  }
}
```