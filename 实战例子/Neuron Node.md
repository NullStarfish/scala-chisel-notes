
![[Pasted image 20251119222005.png]]


纯串行，C语言逻辑写法：

```scala
class Neuron(inputs: Int, act: FixedPoint => FixedPoint) extends Module {
  val io = IO(new Bundle {
    val in      = Input(Vec(inputs, FixedPoint(16.W, 8.BP)))
    val weights = Input(Vec(inputs, FixedPoint(16.W, 8.BP)))
    val out     = Output(FixedPoint(16.W, 8.BP))
  })

  // 1. 定义一个 Scala 的 var 变量作为“累加器”
  // 初始化为 0 (注意要指定好类型和位宽)
  var accumulator = 0.F(16.W, 8.BP) 

  // 2. 使用常规的 Scala for 循环
  for (i <- 0 until inputs) {
    // 3. 核心逻辑：不断更新 accumulator 变量所指向的电路节点
    // 注意这里用的是 = (Scala 变量赋值)，而不是 := (Chisel 信号连接)
    accumulator = accumulator + (io.in(i) * io.weights(i))
  }

  // 4. 最后将累加结果连接到输出并激活
  io.out := act(accumulator)
}
```

并行逻辑
这个zip很关键
```scala
import chisel3._
import chisel3.experimental.FixedPoint

class Neuron(inputs: Int, act: FixedPoint => FixedPoint) extends Module {
  val io = IO(new Bundle {
    // 建议给 FixedPoint 明确宽度，比如 16.W, 8.BP
    val in      = Input(Vec(inputs, FixedPoint(16.W, 8.BP)))
    val weights = Input(Vec(inputs, FixedPoint(16.W, 8.BP)))
    val out     = Output(FixedPoint(16.W, 8.BP))
  })

  // 步骤 1: 乘法 (并行计算所有乘积)
  // (io.in zip io.weights) 会把两个 Vec 打包成 pairs: ((in0, w0), (in1, w1), ...)
  val products = (io.in zip io.weights).map { case (i, w) => 
    i * w 
  }

  // 步骤 2: 求和 (使用 reduce 构建加法树)
  // reduce 会生成一个平衡的加法器树，性能比串行相加更好
  val sum = products.reduce(_ + _)

  // 步骤 3: 激活 (最后应用一次激活函数)
  io.out := act(sum)
}
```



示例act函数：
```scala
val Step: FixedPoint => FixedPoint = x => Mux(x <= 0.F(8.BP), 0.F(8.BP), 1.F(8.BP))
val ReLU: FixedPoint => FixedPoint = x => Mux(x <= 0.F(8.BP), 0.F(8.BP), x)
```
