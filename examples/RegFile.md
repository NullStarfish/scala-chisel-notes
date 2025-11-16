


```scala
import chisel3._
import chisel3.util._

class RegisterFile(readPorts: Int) extends Module {
    require(readPorts >= 0)
    val io = IO(new Bundle {
        // 写端口
        val wen   = Input(Bool())
        val waddr = Input(UInt(5.W))
        val wdata = Input(UInt(32.W))
        // 读端口 (向量化)
        val raddr = Input(Vec(readPorts, UInt(5.W)))
        val rdata = Output(Vec(readPorts, UInt(32.W)))
    })
    
    // 一个由32个32位寄存器组成的向量寄存器，复位值为0
    val reg = RegInit(VecInit(Seq.fill(32)(0.U(32.W))))
    
    // --- 写逻辑 ---
    // 使用 when 来生成硬件条件逻辑
    // 当 wen 为高，且 waddr 不为0时，在时钟上升沿更新对应地址的寄存器
    when(io.wen && io.waddr =/= 0.U) {
        reg(io.waddr) := io.wdata
    }
    
    // --- 读逻辑 ---
    // 读操作是组合逻辑，它应该总是发生的，不受 wen 影响
    // 使用 for 循环来生成多个读端口的电路
    for (i <- 0 until readPorts) {
        // 对于每一个读端口 i...
        when (io.raddr(i) === 0.U) {
            // 如果读地址为0，总是读出0
            io.rdata(i) := 0.U
        } .otherwise {
            // 否则，读出对应地址寄存器的值
            // reg(io.raddr(i)) 会生成一个 MUX，这是组合逻辑读
            io.rdata(i) := reg(io.raddr(i))
        }
    }
}
```


