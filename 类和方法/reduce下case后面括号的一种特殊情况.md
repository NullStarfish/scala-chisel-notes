  
```scala
val exList = List(1, 5, 7, 100)

// write a custom function to add two numbers, then use reduce to find the sum of all values in exList  
def add(a: Int, b: Int): Int = a + b  
val sum = exList.reduce(add)

// find the sum of exList using an anonymous function (hint: you've seen this before!)  
val anon_sum = exList.reduce( (a: Int, b: Int) => a + b)

```


这个代码是正确的。但是当我把(a: Int, b: Int) => a + b改为case (a, b) => a + b的时候，就错误了。  
这是因为case后面的括号被代码认为是tuple。  
我该怎么让reduce接受case的类型匹配？


### 注意
在scala3中，case (a, b) => a + b的写法也合法了



### 一种解决方式
```scala
// 你的写法，非常适合复杂逻辑
List(1, 5, 7, 100).reduce { (x, y) =>
  (x, y) match {
    case (a, b) if a > b  => a + b       // 复杂逻辑 A
    case (0, b)           => b           // 复杂逻辑 B (模式匹配擅长捕捉特定值)
    case (a, b)           => a + b       // 默认逻辑
  }
}
```
















你的理解非常准确。问题的核心在于 **函数的参数数量（Arity）** 和 **参数类型** 不匹配。

简单来说：
1.  `reduce` 需要一个接受 **两个独立参数** 的函数：`(Int, Int) => Int`。
2.  `{ case (a, b) => ... }` 定义的是一个接受 **单个参数（且该参数必须是元组）** 的函数：`((Int, Int)) => Int`。

Scala 不会自动把 `reduce` 传进来的两个参数“打包”成一个元组塞给 `case` 语句。

如果你非要用 `case` 模式匹配来写（虽然在这个场景下不推荐），你需要把这个“接受元组的函数”转换成“接受两个参数的函数”。

### 解决方法：使用 `Function.untupled`

Scala 标准库提供了一个工具 `Function.untupled`，它可以把一个接受元组的函数，“解包”成接受多个参数的函数。

```scala
val exList = List(1, 5, 7, 100)

// 使用 Function.untupled 包裹住 case 语句块
val anon_sum_case = exList.reduce(Function.untupled { 
  case (a, b) => a + b 
})

println(anon_sum_case) // 输出 113
```

### 为什么会这样？（深度解析）

让我们看看类型的区别：

1.  **reduce 的期望：**
    `reduce` 在 `List[Int]` 上的签名类似于：
    ```scala
    def reduce(op: (Int, Int) => Int): Int
    ```
    它会这样调用你的函数：`op(1, 5)`。它传递了 **2个** 整数。

2.  **case 语句的本质：**
    当你写 `{ case (a, b) => ... }` 时，你实际上定义了一个匿名函数，它大概长这样：
    ```scala
    // 这是一个接受 1 个参数（x）的函数
    val myFunc = (x: (Int, Int)) => x match {
      case (a, b) => a + b
    }
    ```
    它期望被这样调用：`myFunc((1, 5))`。注意这里的双层括号，它需要 **1个** 元组对象。

**冲突点：**
`reduce` 拿着两个 `Int` 想往里塞，但你的 `case` 函数只开了一个口子（虽然这个口子形状是元组），所以编译器报错。

### 什么时候可以直接用 `case`？

通常在处理集合中的元素**本身就是元组**时，我们会在 `map` 或 `foreach` 中直接用 `case`，因为此时集合迭代出来的每一个元素就是一个元组（单参数）。

```scala
val pairs = List((1, 2), (3, 4), (5, 6))

// 这里可以直接用 case，因为 map 每次拿出一个元素，这个元素本身就是 (Int, Int) 类型的元组
val sums = pairs.map { case (a, b) => a + b } 
```

### 总结

对于 `reduce` 这种明确要求 `(A, A) => A` （两个参数）的操作：
1.  **推荐写法：** `(a, b) => a + b` （最简洁，最符合直觉）。
2.  **折腾写法：** `Function.untupled { case (a, b) => a + b }` （仅作教学演示，实际开发不要这么写，多此一举）。

















**是的，完全合法，而且是非常地道的写法。**

这正是解决“既想要 `reduce` 接收两个参数，又想要使用 `match/case` 的强大模式匹配功能”的标准方案。

### 为什么这样是合法的？

让我们拆解一下你的代码：

1.  **外层：`(x, y) => ...`**
    *   这定义了一个匿名函数，它显式接收 **两个参数** `x` 和 `y`。
    *   这完美满足了 `reduce` 对函数签名的要求：`(Int, Int) => Int`。编译器看到这里就放心了。

2.  **内层：`(x, y) match { ... }`**
    *   在函数体内部，你手动将这两个参数组合成了一个 **元组** `(x, y)`。
    *   然后你对这个临时构造的元组使用了 `match` 表达式。
    *   `match` 表达式的结果（`Int`）成为了整个函数的返回值。

### 这种写法的优势

既然你提到了“任意复杂逻辑”，这种写法在处理复杂分支时非常清晰。

如果不用 `match`，你可能需要写很丑的嵌套 `if-else`：

```scala
// 不使用 match 的写法，逻辑复杂时很难看
List(1, 5, 7, 100).reduce { (x, y) =>
  if (x > y) {
    x + y // 复杂逻辑 A
  } else if (x == 0) {
    y     // 复杂逻辑 B
  } else {
    x + y // 默认逻辑
  }
}
```

使用你刚才提出的 `match` 写法，结构会扁平化，可读性更强：

```scala
// 你的写法，非常适合复杂逻辑
List(1, 5, 7, 100).reduce { (x, y) =>
  (x, y) match {
    case (a, b) if a > b  => a + b       // 复杂逻辑 A
    case (0, b)           => b           // 复杂逻辑 B (模式匹配擅长捕捉特定值)
    case (a, b)           => a + b       // 默认逻辑
  }
}
```

### 总结

*   **之前的错误写法**：`reduce { case (a, b) => ... }`
    *   错误原因：试图直接把一个“接收元组的函数”传给“接收两个参数的接口”。

*   **现在的正确写法**：`reduce { (x, y) => (x, y) match { ... } }`
    *   正确原因：外层负责适配接口（接收两个参数），内层负责逻辑处理（构造元组并匹配）。

这是一种非常常见的 **"Adapter"（适配）** 手法：先接收参数，再在内部转换形式以利用模式匹配。完全可以放心使用。