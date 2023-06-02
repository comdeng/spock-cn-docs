# Spock Primer

本章内容假定你已经具有Groovy和单元测试的基本知识。如果你是Java开发人员，但没有听说过Groovy，请不要担心 —— Groovy会让您觉得非常熟悉！实际上，Groovy的主要设计目标之一是成为Java身边的脚本语言。因此，只需跟随教程，并在需要时查阅Groovy文档即可。

本章的目标是教给你足够的Spock知识，以编写真实世界的Spock规范，并激发你进一步学习的兴趣。

要了解更多关于Groovy的信息，请访问<https://groovy-lang.org/>。

要了解更多关于单元测试的信息，请访问<https://en.wikipedia.org/wiki/Unit_testing>。



## 术语定义
我们首先来定义几个术语：Spock允许你编写[`规范`](https://en.wikipedia.org/wiki/Specification_by_example)，描述其预期的`特性`（`属性`、`方面`）。所关注的系统可以是从单个类到整个应用的任何内容，也被称为`待规范系统`（SUS）或系统。特性的描述始于对待规范系统及其协作者的特定快照；这个快照被称为特性的夹具。

以下各节将引导你了解Spock规范可能由哪些构建块组成。典型的规范只使用它们的子集。


## 导入（Imports）

<pre><code class="language-groovy">
import spock.lang.*
</code></pre>

`spock.lang`包包含了编写测试规范需要用到的关键类型。



## 规范（Specification）
<pre><code class="language-groovy">
class MyFirstSpecification extends Specification {
  // fields
  // fixture methods
  // feature methods
  // helper methods
}
</code></pre>
一个规范被表示为一个继承`spock.lang.Specification`的Groovy类。规范的名称通常与规范描述的系统或系统操作有关。例如，`CustomerSpec`、`H264VideoPlayback`和`ASpaceshipAttackedFromTwoSides`都是规范的合理名称。

Specification类包含了一些有用的方法来编写规范。此外，它还指示JUnit使用`Sputnik`（Spock的JUnit运行器）来运行规范。通过`Sputnik`，Spock规范可以在大多数现代Java IDE和构建工具中运行。

## 字段（Fields）
<pre><code class="language-groovy">
def obj = new ClassUnderSpecification()
def coll = new Collaborator()
</code></pre>

实例字段是存储规范的夹具对象的好地方。最好的做法是在声明的时候就进行初始化。（从语义上讲，这等同于在`setup()`方法的最开始进行初始化。）存储在实例字段中的对象在特性方法之间不共享。相反，每个特性方法都有自己的对象。这有助于将特性方法相互隔离，也是通常的预期目标。

<pre><code class="language-groovy">
@Shared res = new VeryExpensiveResource()
</code></pre>

有时候你需要在特性方法之间共享一个对象。例如，该对象可能创建起来的代价很大，或者你希望特性方法彼此交互。为了实现这个目的，可以声明一个@Shared字段。同样，最好在声明的时候就对字段进行初始化。（从语义上讲，这等同于在`setupSpec()`方法的最开始进行初始化。）

<pre><code class="language-groovy">
static final PI = 3.141592654
</code></pre>

静态字段应该仅用于定义常量。否则，使用共享字段是更好的办法，因为它们在共享方面的语义更加明确定义。


## 夹具方法（Fixture Methods）


<pre><code class="language-groovy">
def setupSpec() {}    // 运行一次 -  在第一个特性方法之前
def setup() {}        // 每个特性方法之前运行
def cleanup() {}      // 每个特性方法之后运行
def cleanupSpec() {}  // 运行一次 -  在最后一个特性方法之后

</code></pre>

夹具方法(Fixture method)负责设置和清理特性方法(Feture method)运行的环境。通常建议为每个特性方法使用一个新的夹具，这就是 `setup()` 和 `cleanup()` 方法的作用。

所有的夹具方法都是可选的。


某些情况下，特性方法共享夹具是有意义的。这可以通过使用共享字段和 `setupSpec()` 和 `cleanupSpec()` 方法来实现。需要注意的是，`setupSpec()` 和 `cleanupSpec()` 不能引用实例字段，除非它们被标注了 `@Shared` 注解。



> 每个规范类中每种类型的 fixture 方法只能有一个。



### 调用次序（Invocation Order）



如果在规范的子类中重写了夹具方法，那么父类的`setup()`将在子类的`setup()`之前运行。`cleanup()`则按相反的顺序工作，也就是子类的`cleanup()`将在父类的`cleanup()`之前执行。`setupSpec()`和`cleanupSpec()`的行为方式相同。不需要显式调用`super.setup()`或`super.cleanup()`，因为Spock会自动在继承层次结构中找到并执行夹具方法。


1. `super.setupSpec`
2. `sub.setupSpec`
3. `super.setup`
4. `sub.setup`
5. 特性方法
6. `sub.cleanup`
7. `super.cleanup`
8. `sub.cleanupSpec`
9. `super.cleanupSpec`



## 特性方法（Feture Methods）

特性方法是规范的核心。它们描述了你希望在待规范系统中找到的特性（属性、方面）。按照惯例，特性方法的命名使用字符串字面量。尽量为特性方法选择好的命名，并且可以自由使用任何字符！

从概念上讲，特性方法包括四个阶段：

1. 设置特性的夹具
2. 向待规范系统提供刺激
3. 描述对系统的预期响应
4. 清理特性的夹具

第一个和最后一个阶段是可选的，而刺激和响应阶段总是存在的（除了在交互式特性方法中），并且可能会发生多次。

## 块（Blocks）
Spock内置了对特性方法的每个概念阶段的支持。为此，特性方法被结构化为所谓的`块`（blocks）。块以标签开始，并延伸到下一个块的开始或方法的结束。有六种类型的块：`given`、`when`、`then`、`expect`、`cleanup`和`where`块。在方法的开始和第一个显式块之间的任何语句都属于隐式的`given`块。

特性方法必须至少有一个显式（即带有标签）的块，实际上，显式块的存在使方法成为特性方法。块将方法划分为不同的部分，不能被嵌套使用。

右侧的图片显示了块如何映射到特性方法的概念阶段。`where`块还承担了一个特殊的角色，稍后将揭示。我们先来仔细看看其他的块。

<div style="text-align:right">

![Blocks to Phases](/resources/img/Blocks2Phases.png)

</div>

### `given`块

`given`块是你为所描述的特性进行任何设置工作的地方。它不能在其他块之前出现，也不能重复出现。`given`块本身没有特殊的语义。`given:`标签是可选的，可以省略，从而形成一个隐式的`given`块。最初，`setup:`是首选的块名称，但使用`given:`通常会是更易读的特性方法描述（参见[规范即文档](#specifications_as_documentation)）。

### `when`和`then`块

<pre><code class="language-groovy">
when: // 刺激
then: // 响应
</code></pre>

`when`和`then`块总是一起出现。它们描述了一个刺激和期望的响应。`when`块可以包含任意代码，而`then`块则限制为条件（conditions）、异常条件（exception conditions）、交互（interactions）和变量定义（variable definitions）。一个特性方法可以包含多对`when-then`块。

#### 条件
条件描述了一个期望的状态，类似于JUnit的断言。然而，条件是以普通的布尔表达式形式编写的，消除了对断言API的需求。（更准确地说，条件也可以产生一个非布尔值，然后根据Groovy的真值进行评估。）让我们看一些条件的实际应用：

<pre><code class="language-groovy">
when:
stack.push(elem)

then:
!stack.empty
stack.size() == 1
stack.peek() == elem
</code></pre>

> 尽量保持每个特性方法中条件的数量较少。一个到五个条件是一个很好的指导原则。如果你有更多条件，需要确认一下是否一次性指定了多个不相关的特性。如果答案是肯定的，建议将特性方法分解为多个较小的方法。如果你的条件只是在值上有所不同，请考虑使用[数据表格](/docs/data_driven_testing#data-tables)。



如果条件不满足，Spock提供了详细的反馈。让我们尝试将第二个条件改为`stack.size() == 2`，以下是反馈信息：

<pre><code class="language-groovy">
Condition not satisfied:

stack.size() == 2
|     |      |
|     1      false
[push me]
</code></pre>

正如你所见，Spock捕获了在评估条件时产生的所有值，并以易于理解的形式呈现出来。是不是很好呢？这样的反馈信息有助于理解条件为何失败，并且方便进行故障排除。



#### 隐式和显式条件

条件是`then`块和`expect`块的重要组成部分。除了对`void`方法的调用和被分类为交互的表达式外，这些块中的所有顶级表达式都会被隐式地视为条件。要在其他地方使用条件，你需要使用Groovy的`assert`关键字来标识它们：

<pre><code class="language-groovy">
def setup() {
  stack = new Stack()
  assert stack.empty
}
</code></pre>

如果显式条件不满足，它将产生与隐式条件相同的良好诊断消息。



#### 异常条件

异常条件用于描述`when`块应该抛出异常的情况。它们使用`thrown()`方法来定义，传递预期的异常类型。例如，要描述从空栈中弹出应该抛出`EmptyStackException`异常，可以编写如下代码：

<pre><code class="language-groovy">
when:
stack.pop()

then:
thrown(EmptyStackException)
stack.empty
</code></pre>

如你所见，异常条件后面可以跟随其他条件（甚至其他块）。这对于指定异常的预期内容特别有用。要访问异常，首先将其绑定到一个变量：

<pre><code class="language-groovy">
when:
stack.pop()

then:
def e = thrown(EmptyStackException)
e.cause == null
</code></pre>

或者，你可以使用稍微变化的语法：

<pre><code class="language-groovy">
when:
stack.pop()

then:
EmptyStackException e = thrown()
e.cause == null
</code></pre>

这种语法有两个小优点：首先，异常变量是强类型的，使得IDE能够更容易地提供代码补全。其次，条件读起来更像一个句子（"then an EmptyStackException is thrown"）。请注意，如果`thrown()`方法没有传递异常类型，则会从左侧的变量类型推断异常类型。

有时我们需要表达不应该抛出异常的情况。例如，让我们尝试表达`HashMap`应该接受`null`键：

<pre><code class="language-groovy">
def "HashMap accepts null key"() {
  setup:
  def map = new HashMap()
  map.put(null, "elem")
}
</code></pre>

这段代码可以工作，但不显示代码的意图。是不是有人在实现这个方法之前就离开了？毕竟，条件在哪里？幸运的是，我们可以做得更好：

<pre><code class="language-groovy">
def "HashMap accepts null key"() {
  given:
  def map = new HashMap()

  when:
  map.put(null, "elem")

  then:
  notThrown(NullPointerException)
}
</code></pre>

通过使用`notThrown()`，我们明确指出特别不应该抛出`NullPointerException`异常。（根据`Map.put()`的约定，对于不支持`null`键的映射，这是正确的做法。）然而，如果抛出任何其他异常，该方法也会失败。



#### 交互

与条件描述对象的状态不同，交互描述对象之间如何进行通信。关于交互和基于交互的测试将在单独的[章节](/docs/interaction_based_testing#interaction-based-testing)中介绍，因此我们在这里只给出一个快速的示例。假设我们想要描述从发布者到订阅者的事件流程。下面是代码示例：

<pre><code class="language-groovy">
def "events are published to all subscribers"() {
  given:
  def subscriber1 = Mock(Subscriber)
  def subscriber2 = Mock(Subscriber)
  def publisher = new Publisher()
  publisher.add(subscriber1)
  publisher.add(subscriber2)

  when:
  publisher.fire("event")

  then:
  1 * subscriber1.receive("event")
  1 * subscriber2.receive("event")
}
</code></pre>



### `expect`块

一个`expect`块比一个`then`块更加受限，因为它只能包含条件和变量定义。它在某些情况下非常有用，例如在单个表达式中描述刺激和预期响应更加自然。具体来看，比较下面两种描述`Math.max()`方法的尝试：

<pre><code class="language-groovy">
when:
def x = Math.max(1, 2)

then:
x == 2

expect:
Math.max(1, 2) == 2
</code></pre>

虽然这两个片段在语义上是等价的，但第二个片段显然更可取。对两者的指导原则是：使用`when-then`来描述具有副作用的方法，使用`expect`来描述纯粹功能性的方法。

> 利用[Groovy JDK](https://docs.groovy-lang.org/docs/latest/html/groovy-jdk/)方法，如`any()`和`every()`，可以创建更具表达性和简洁的条件。



### `cleanup`块

<pre><code class="language-groovy">
given:
def file = new File("/some/path")
file.createNewFile()

// ...

cleanup:
file.delete()
</code></pre>



`cleanup`块只能跟随在`where`块之后，而且不能重复。类似于`cleanup`方法，它用于释放特性方法中使用的任何资源，并且即使（前面的一部分）特性方法有异常产生，它也会被执行。因此，`cleanup`块必须进行防御性编码；在最坏的情况下，它必须优雅地处理特性方法的第一条语句抛出异常的情况，与此同时，所有局部变量仍然具有它们的默认值。

> Groovy的安全解引用运算符（foo?.bar()）简化了编写防御性代码的过程。

对象级别的规范通常不需要`cleanup`方法，因为它们仅消耗内存资源，这些资源会被垃圾收集器自动回收。然而，更粗粒度的规范可能会使用`cleanup`块来清理文件系统、关闭数据库连接或关闭网络服务。

> 如果一个规范被设计成其所有的特性方法都需要相同的资源，那么使用`cleanup()`方法；否则，更倾向于使用`cleanup`块。同样的策略也适用于`setup()`方法和`given`块。

### `where`块

`where`块总是出现在方法的最后，并且不能重复。它用于编写数据驱动的特性方法。为了让你了解如何做到这一点，请看下面的例子：

<pre><code class="language-groovy">
def "computing the maximum of two numbers"() {
  expect:
  Math.max(a, b) == c

  where:
  a << [5, 3]
  b << [1, 9]
  c << [5, 9]
}
</code></pre>

这个`where`块实际上创建了特性方法的两个“版本”：一个版本中a为5，b为1，c为5，另一个版本中a为3，b为9，c为9。

尽管`where`块在声明时出现在最后，但它会在包含它的特性方法运行之前执行。

`where`块在“[数据驱动测试](/docs/data_driven_testing#data-driven-testing)”章节中进一步解释。



## 辅助方法（Helper Methods）

有时特性方法会变得很大，或者包含大量重复的代码。在这种情况下，就适合引入一个或多个辅助方法了。作为辅助方法的候选方法，则是设置/清理逻辑和复杂的条件。对于辅助方法，将其提取出来非常简单，所以让我们来看看条件：

<pre><code class="language-groovy">
def "offered PC matches preferred configuration"() {
  when:
  def pc = shop.buyPc()

  then:
  pc.vendor == "Sunny"
  pc.clockRate >= 2333
  pc.ram >= 4096
  pc.os == "Linux"
}
</code></pre>

如果你碰巧是一个电脑极客，你的首选电脑配置可能非常详细，或者你可能想要比较多个商店的报价。因此，让我们将条件提取出来：

<pre><code class="language-groovy">
def "offered PC matches preferred configuration"() {
  when:
  def pc = shop.buyPc()

  then:
  matchesPreferredConfiguration(pc)
}

def matchesPreferredConfiguration(pc) {
  pc.vendor == "Sunny"
  && pc.clockRate >= 2333
  && pc.ram >= 4096
  && pc.os == "Linux"
}
</code></pre>

新的辅助方法`matchesPreferredConfiguration()`由一个返回布尔表达式的单语句组成（Groovy中`return`关键字是可选的）。然而，呈现出来的信息不够精准：

<pre><code class="language-groovy">
Condition not satisfied:

matchesPreferredConfiguration(pc)
|                             |
false                         ...
</code></pre>

这样做用处不大。幸运的是，我们可以做得更好：

<pre><code class="language-groovy">
void matchesPreferredConfiguration(pc) {
  assert pc.vendor == "Sunny"
  assert pc.clockRate >= 2333
  assert pc.ram >= 4096
  assert pc.os == "Linux"
}
</code></pre>

当将条件提取到辅助方法中时，需要考虑两个问题：首先，需要使用`assert`关键字将隐式条件转换为显式条件。其次，辅助方法必须具有`void`返回类型。否则，Spock可能会将返回值解释为失败的条件，这并不是我们想要的结果。

诚如所愿，改进后的辅助方法明确地告诉我们具体出了什么问题：

<pre><code class="language-groovy">
Condition not satisfied:

assert pc.clockRate >= 2333
       |  |         |
       |  1666      false
       ...
</code></pre>


最后的建议是：尽管代码重用通常是件好事，但也不要过度使用。请注意，使用夹具方法或者辅助方法会增加特性方法之间的耦合。如果重用太多或错误的代码，你得到的将是脆弱且难以迭代的规范。



## 使用`with设置预期

替换辅助方法的另一种选择是使用`with(target, closure)`方法和被验证的对象进行交互。这在`then`和`expect`块中特别有用。

<pre><code class="language-groovy">
def "offered PC matches preferred configuration"() {
  when:
  def pc = shop.buyPc()

  then:
  with(pc) {
    vendor == "Sunny"
    clockRate >= 2333
    ram >= 406
    os == "Linux"
  }
}
</code></pre>

与使用辅助方法时不同，这里不需要显式的`assert`语句来进行错误报告。

在验证模拟对象时，`with`语句也可以简化冗长的验证语句。

<pre><code class="language-groovy">
def service = Mock(Service) // 具有 start()、stop() 和 doWork() 方法
def app = new Application(service) // 控制 service 的生命周期

when:
app.run()

then:
with(service) {
  1 * start()
  1 * doWork()
  1 * stop()
}
</code></pre>

有时，IDE 可能无法确定目标的类型，这时可以通过使用`with(target, type, closure)`的形式手动指定目标类型予以解决。



## 使用`verifyAll`同时断言多个预期

通常情况下，预期在第一个失败的断言时会导致测试失败。有时候，在测试失败之前收集所有的失败信息以获取更多信息是很有帮助的，这种行为也被称为软断言。

`verifyAll`方法可以像`with`一样使用，

<pre><code class="language-groovy">
def "offered PC matches preferred configuration"() {
  when:
  def pc = shop.buyPc()

  then:
  verifyAll(pc) {
    expect vendor == "Sunny"
    expect clockRate >= 2333
    expect ram >= 406
    expect os == "Linux"
  }
}
</code></pre>

或者可以在没有目标的情况下使用。

<pre><code class="language-groovy">
expect:
verifyAll {
  expect 2 == 2
  expect 4 == 4
}
</code></pre>

与`with`类似，你还可以选择为 IDE 提供类型提示。




## 规范即文档
<a name="specifications_as_documentation"></a>

编写精良的规范是有价值的信息源。尤其对于面向更广泛受众（如架构师、领域专家、客户等）的高级规范，除了规范和特性的名称，提供更多自然语言的信息也是有意义的。因此，Spock提供了一种在代码块中附加文本描述的方式：

<pre><code class="language-groovy">
given: "open a database connection"
// code goes here
</code></pre>

使用`and:`标签来描述逻辑上不同的代码块部分：

<pre><code class="language-groovy">
given: "open a database connection"
// code goes here

and: "seed the customer table"
// code goes here

and: "seed the product table"
// code goes here
</code></pre>

`and:`标签后面跟着一个描述，可以在特性方法的任何（顶层）位置插入，并不会改变方法的语义。

在行为驱动开发模式下，使用`given-when-then`的格式来描述面向客户的特性（称为故事）。Spock使用`given:`标签直接支持这种规范风格：

<pre><code class="language-groovy">
given: "an empty bank account"
// ...

when: "the account is credited \$10"
// ...

then: "the account's balance is \$10"
// ...
</code></pre>

代码块描述不仅存在于源代码中，还可供Spock运行时使用。有计划地使用代码块描述的方式，不仅能增强诊断消息，也能提供给所有利益相关者更易理解的文本报告。



## 扩展

正如我们所见，Spock为编写规范提供了许多功能。然而，总会有需要其他功能的时候。因此，Spock提供了一种基于拦截的扩展机制。扩展通过称为`指令`（directives）的注解来激活。目前，Spock附带以下指令：

- `@Timeout`：为特性或固定方法设置执行超时时间。

- `@Ignore`：忽略带有此注解的任何特性方法。

- `@IgnoreRest`：执行带有此注解的特性方法，而忽略其他所有方法。用于快速运行单个方法。

- `@FailsWith`：期望特性方法异常终止。
- `@FailsWith`： 有两种用途：首先，用于记录无法立即解决的已知错误。其次，用于替代某些特定情况下无法使用异常条件（例如指定异常条件的行为）的异常条件。在其他所有情况下，异常条件是首选的。



请转到[扩展](/docs/extensions#extensions)章节，了解如何实现自己的指令和扩展。



## 对比JUnit



尽管Spock使用了不同的术语，但其许多概念和特性都受到JUnit的启发。以下是一个大致的比较：

| Spock               | JUnit             |
| ------------------- | ----------------- |
| Specification       | Test class        |
| setup()             | @Before           |
| cleanup()           | @After            |
| setupSpec()         | @BeforeClass      |
| cleanupSpec()       | @AfterClass       |
| Feature             | Test              |
| Feature method      | Test method       |
| Data-driven feature | Theory            |
| Condition           | Assertion         |
| Exception condition | @Test(expected=…) |
| Interaction         | Mock expectation  |
