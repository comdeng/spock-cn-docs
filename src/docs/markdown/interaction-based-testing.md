---
title: "基于交互的测试"
---



# 基于交互的测试

交互式测试是一种设计和测试技术，于2000年代早期在极限编程（Extreme Programming，XP）社区中出现。它专注于对象的行为而非状态，通过方法调用来探索对象（们）如何与其协作者进行交互。

例如，假设我们有一个名为`Publisher`的对象，它向其`Subscriber`（订阅者）发送消息：

<pre><code class="language-groovy">
class Publisher {
  List<Subscriber> subscribers = []
  int messageCount = 0
  void send(String message){
    subscribers*.receive(message)
    messageCount++
  }
}
interface Subscriber {
  void receive(String message)
}

class PublisherSpec extends Specification {
  Publisher publisher = new Publisher()
} 
</code></pre>

re我们要如何测试Publisher呢？使用基于状态的测试，我们可以验证Publisher是否跟踪其订阅者。然而，更有趣的问题是，由Publisher发送的消息是否被Subscriber接收到。为了回答这个问题，我们需要一个特殊的Subscriber实现，它可以监听Publisher和订阅者之间的交流。这样的实现被称为模拟对象。

虽然我们可以手动创建一个Subscriber的模拟实现，但随着方法数量和交互复杂性的增加，编写和维护这段代码可能会变得不愉快。这就是模拟框架的用武之地：它们提供了一种描述对象和其协作者之间预期交互的方式，并且可以生成验证这些期望的模拟协作者的实现。

>  #### 模拟实现是如何生成的呢？
> 与大多数Java模拟框架类似，Spock在运行时使用[JDK动态代理](https://docs.oracle.com/javase/7/docs/api/java/lang/reflect/Proxy.html)（用于模拟接口）以及[Byte Buddy](https://bytebuddy.net/)或[CGLIB](https://github.com/cglib/cglib)代理（用于模拟类）来生成模拟实现。与基于Groovy元编程的实现相比，这种方法的优点在于它也适用于测试Java代码。

Java世界上有许多受欢迎和成熟的模拟框架：[JMock](https://www.jmock.org/)、[EasyMock](https://www.easymock.org/)、[Mockito](https://mockito.org/)等等。虽然这些工具都可以与Spock一起使用，但我们决定自己开发一个与Spock规范语言紧密集成的模拟框架。我们做这个决定是因为我们希望利用Groovy的所有功能，使基于交互的测试更易写、更可读，且最终会更有趣。希望在本章结束时，你会认可我们已经通过Spock实现了这些目标。

除非另有说明，Spock模拟框架的所有功能都适用于测试Java和Groovy代码。



<a name="_creating_mock_objects"></a>

## 创建模拟对象

模拟对象是通过`MockingApi.Mock()`方法创建的。我们先来创建两个模拟的Subscriber。

<pre><code class="language-groovy">
def subscriber = Mock(Subscriber)
def subscriber2 = Mock(Subscriber)
</doce></pre>

另外，还支持以下类似Java的语法，这样可以获得更好的IDE支持：
<pre><code class="language-groovy">
Subscriber subscriber = Mock()
Subscriber subscriber2 = Mock()
</doce></pre>

在这里，模拟对象的类型是从赋值语句左侧的变量类型推断出来的。

> 如果在赋值语句的左侧给出了mock的类型，可以（但不是必须）在右侧省略它。


Mock对象实际上实现（或者在类的情况下扩展）了它所代表的类型。换句话说，在我们的例子中，`subscriber`是一个`Subscriber`。因此，它可以传递给需要这个类型的静态类型（Java）代码。



<a name="_default_behavior_of_mock_objects"></a>

## 模拟对象的默认行为



> #### 宽松/严格模拟框架的对比
>
> 与`Mockito`的理念一样，我们坚信一个模拟框架应该默认是宽松的。这意味着在模拟对象上发生的意外方法调用（或者换句话说，对于当前测试来说不相关的交互）是被允许的，并且会返回一个默认的响应。相反，像`EasyMock`和`JMock`这样的模拟框架默认是严格的，对于每个意外的方法调用都会抛出异常。尽管严格性可以保证执行的严谨性，但它也可能导致过度规范化，使得每次内部代码更改时都会造成测试失败。Spock的模拟框架使得描述交互的相关部分变得简单，避免了过度规范化的陷阱。



初始状态下，模拟对象没有任何行为。它们的方法都可以被调用，除了把方法的返回类型的默认值（`false`、`0`或`null`）返回之外，没有其他影响。比较特殊的是`Object.equals`、`Object.hashCode`和`Object.toString`方法，它们具有以下默认行为：模拟对象只等于它自己，具有唯一的哈希码，并且包含所表示类型名称的字符串表示形式。这种默认行为可以通过对方法进行存根来实现覆盖，我们将在[存根](/docs/interaction_based_testing#_stubbing)部分学习。



<a name="_injecting_mock_objects_into_code_under_specification"></a>

## 将模拟对象注入到被测试代码中

在创建Publisher和Subscriber之后，我们需要让Subscriber对Publisher进行引用：

<pre><code class="language-groovy">
class PublisherSpec extends Specification {
  Publisher publisher = new Publisher()
  Subscriber subscriber = Mock()
  Subscriber subscriber2 = Mock()
  def setup() {
    publisher.subscribers << subscriber // << is a Groovy shorthand for List.add()
    publisher.subscribers << subscriber2
  }
</pre></code>

现在我们可以描述两方之间预期的交互。





<a name="_mocking"></a>

## 模拟（Mocking）

模拟是描述被测试对象与其协作者之间（强制性的）交互的行为。下面是一个例子：
<pre><code class="language-groovy">
def "should send messages to all subscribers"() {
  when:
  publisher.send("hello")

  then:
  1 * subscriber.receive("hello")
  1 * subscriber2.receive("hello")
}
</pre></code>

大声读出："当发布者发送一条 'hello' 消息时，两个订阅者都应该准确地接收到该消息一次。"

当运行此特性方法时，执行`when:`块期间对模拟对象的所有调用将与`then:`块中描述的交互进行匹配。如果其中一个交互未满足，将抛出一个（子类）`InteractionNotSatisfiedError`异常。此验证自动进行，无需额外的代码。





<a name="_interactions_2"></a>

### 交互

> #### 一个交互只是一个普通的方法调用吗？
> 
> 不完全是。虽然一个交互看起来类似于一个普通的方法调用，但它实际上是表达了哪些方法调用被期望发生的方式。可以将一个交互视为对所有传入的模拟对象调用进行匹配的正则表达式。根据情况，该交互可能匹配零次、一次或多次调用。



让我们更仔细地看一下 `then:` 块。它包含两个交互，每个交互有四个不同的部分：基数（cardinality）、目标约束（target constraint）、方法约束（method constraint）和参数约束（argument constraint）。

<pre><code class="language-bash">
1 * subscriber.receive("hello")
|   |             |          |
|   |             |          参数约束
|   |             方法约束
|   目标约束
基数
</code></pre>


<a name="_cardinality"></a>

### 基数

交互的基数描述了方法调用的期望次数。它可以是一个固定的数字或一个范围：

<pre><code class="language-groovy">
1 * subscriber.receive("hello")        // 只调用1次
0 * subscriber.receive("hello")        // 0次调用
(1..3) * subscriber.receive("hello") // 1-3次（包含）调用
(1.._) * subscriber.receive("hello") // 至少1次调用
(_..3) * subscriber.receive("hello") // 最多3次调用
_ * subscriber.receive("hello")        // 任何次数的调用，包含0次
                                                // (很少需要；请参阅“Strict Mocking”。)
</code></pre>


<a name="_target_constraint"></a>

### 目标约束

目标约束描述了预期接收方法调用的模拟对象是哪个：
<pre><code class="language-groovy">
1 * subscriber.receive("hello")  // 对'subscriber'的1次调用
1 * _.receive("hello")                   // 对任意模拟对象的1次调用
</code></pre>


<a name="_method_constraint"></a>

### 方法约束

方法约束描述了预期调用的方法是哪个：

<pre><code class="language-groovy">
1 * subscriber.receive("hello")    // 一个名为'receive'的方法
1 * subscriber./r.*e/("hello")     // 一个与给定的正则表达式匹配的方法名称
                                            // （在这个示例中，方法名以'r'开头且以'e'结尾）
</code></pre>

当期望调用一个getter方法时，可以使用Groovy属性语法而不是方法语法：

<pre><code class="language-groovy">
1 * subscriber.status             // 类似于： 1 * subscriber.getStatus()
</code></pre>

当期望调用一个setter方法时，只能使用方法语法：

<pre><code class="language-groovy">
1 * subscriber.setStatus("ok")  // 不能这样写： 1 * subscriber.status = "ok"
</code></pre>




<a name="_argument_constraints"></a>

### 参数约束

参数约束用于描述预期的方法参数：

<pre><code class="language-groovy">
1 * subscriber.receive("hello")        // 参数等于字符串 "hello"
1 * subscriber.receive(!"hello")       // 参数不等于字符串 "hello"
1 * subscriber.receive()               // 空参数列表（在我们的示例中永远不会匹配）
1 * subscriber.receive(_)              // 任意单个参数（包括 null）
1 * subscriber.receive(*_)             // 任意参数列表（包括空参数列表）
1 * subscriber.receive(!null)          // 任意非空参数
1 * subscriber.receive(_ as String)    // 任意非空参数，且为字符串类型
1 * subscriber.receive(endsWith("lo")) // 参数与给定的 Hamcrest 匹配器匹配
                                       // 在这个例子中，字符串参数以 "lo" 结尾
1 * subscriber.receive({ it.size() > 3 && it.contains('a') })
// 满足给定断言的参数，即
// 代码参数约束需要返回 true 或 false
// 根据它们是否匹配
// （在这里：消息长度大于3且包含字符 a）
</code></pre>

对于具有多个参数的方法，参数约束需要这样使用：

<pre><code class="language-groovy">
1 * process.invoke("ls", "-a", _, !null, { ["abcdefghiklmnopqrstuwx1"].contains(it) })
</code></pre>

在处理可变参数方法时，也可以在相应的交互中使用可变参数语法：

<pre><code class="language-groovy">
interface VarArgSubscriber {
    void receive(String... messages)
}

...

subscriber.receive("hello", "goodbye")
</code></pre>

> #### Spock深入解析：Groovy可变参数
> 
> Groovy允许以可变参数方式调用最后一个参数为数组类型的方法。因此，可变参数语法也可以在匹配这类方法的交互中使用。



#### 等于约束

使用Groovy的等于运算符来检查参数，即`argument == constraint`。你可以使用以下方式作为等于约束：

- 任何字面值：`1 * check('string')`、`1 * check(1)`、`1 * check(null)`，
- 变量：`1 * check(var)`，
- 列表或映射字面值：`1 * check([1])`、`1 * check([foo: 'bar'])`，
- 对象：`1 * check(new Person('sam'))`，
- 方法调用的结果：`1 * check(person())`。



#### Hamcrest约束：
如果约束对象是Hamcrest匹配器（matcher），则将使用该匹配器来检查参数。

#### 通配符约束
通配符约束将匹配任何参数，无论是`null`还是其他。它的表示形式是`1 * subscriber.receive()`。还有一个扩展通配符约束`*_`，`1 * subscriber.receive(*_)`匹配任意数量的参数，包括零个。

#### 代码约束
代码约束是最灵活的约束之一。它是一个Groovy闭包，以参数的形式接收参数。该闭包被视为条件块，因此它的行为类似于`then`块，即每行都被视为隐式断言。它可以模拟除了扩展通配符约束之外的所有约束，但建议在可能的情况下使用更简单的约束。你可以进行多个断言，调用方法进行断言，或使用`with`/`verifyAll`方法。

<pre><code class="language-groovy">
1 * list.add({
  verifyAll(it, Person) {
    firstname == 'William'
    lastname == 'Kirk'
    age == 45
  }
})
</code></pre>

#### 否定约束
否定约束符号"!"是一个复合约束，即它需要与另一个约束组合使用才能起作用。它对嵌套约束的结果取反，例如`1 * subscriber.receive(!null)`是检查null的等式约束与否定约束组合的结果，将其转换为非null。

虽然它可以与任何其他约束结合使用，但并不总是有意义的，例如`1 * subscriber.receive(!_)`将不匹配任何内容。还要记住，对于不匹配的否定约束，诊断信息只会显示内部约束的匹配结果，没有其他信息。

#### 类型约束
类型约束用于检查参数的类型/类，与否定约束一样，它也是一个复合约束。通常将其表示为`_ as Type`，它是通配符约束和类型约束的组合。你也可以将其与其他约束结合使用，例如`1 * subscriber.receive({ it.contains('foo')} as String)`将在执行代码约束之前断言它是一个String，并检查其是否包含"foo"。

<a name="_matching_any_method_call"></a>
### 匹配任意方法调用
有时候匹配"任何"方法调用可能是有用的，根据特定情况：

<pre><code class="language-groovy">
1 * subscriber._(*_)     // 订阅者上的任何方法，使用任何参数列表
1 * subscriber._          // 上述形式的简写，也是首选

1 * _._                  // 任何模拟对象上的任何方法调用
1 * _                    // 上述形式的简写，也是首选
</code></pre>

> 尽管`(_.._) * _._(*_) >> _` 是有效的交互声明，但它既不是良好的风格，也没有特别的用途。



<a name="_strict_mocking"></a>

## 严格模拟

现在，匹配任意方法调用在什么情况下会有用呢？一个很好的例子是严格模拟，它是一种模拟的风格，只允许显式声明的交互，其他所有交互都是禁止的：

<pre><code class="language-groovy">
when:
publisher.publish("hello")

then:
1 * subscriber.receive("hello")    // 要求在 'subscriber' 上调用一次 'receive' 方法
_ * auditing._                                  // 允许与 'auditing' 进行任何交互
0 * _                                                 // 不允许任何其他交互
</code></pre>

`0 *` 只在 `then: `块或方法的最后一个交互中有意义。请注意使用` _ *` (任意次调用)，它允许与 `auditing` 组件进行任何交互。


> `_ *` 只在严格模拟的上下文中有意义。特别是在 [存根](/docs/interaction_based_testing#_stubbing)调用时，它完全没有存在的必要。例如，`_ * auditing.record() >> "ok"` 可以（而且应该！）简化为 `auditing.record() >> "ok"`。



<a name="_where_to_declare_interactions"></a>

## 交互声明的位置


到目前为止，我们在 `then:` 块中声明了所有的交互。这通常会使规范读起来更自然。然而，在 `when:` 块之前的任何地方声明交互也是允许的。特别是，这意味着可以在 `setup` 方法中声明交互。交互还可以在同一个规范类的任何 "辅助" 实例方法中声明。

当发生对模拟对象的调用时，它会按照声明的交互顺序进行匹配。如果一个调用匹配多个交互，那么尚未达到其上限调用次数的最早声明的交互将获胜。有一个例外：在`then:` 块中声明的交互会在任何其他交互之前进行匹配。这允许使用 `then:` 块中声明的交互来覆盖在 `setup` 方法中声明的交互，或者在其他场景下进行覆盖。

> #### Spock深入探讨：如何识别交互？
> 换句话说，是什么使一个表达式成为一个交互声明，而不是一个常规方法调用？Spock使用一个简单的语法规则来识别交互：如果一个表达式处于语句位置，并且是乘法 (*) 或右移 (>>, >>>) 操作之一，那么它被视为一个交互并相应地解析。这样的表达式在语句位置上几乎没有或根本没有价值，所以改变其含义是可以的。请注意，这些操作符对应于在模拟时声明基数 （当模拟时） 或响应生成器（当存根时）的语法。其中之一必须始终存在；单独的 `foo.bar()` 永远不会被视为交互声明。



<a name="declaring-interactions-at-creation-time"></a>

## 在创建模拟对象时声明交互



如果一个模拟对象具有一组固定的 "基本" 交互，可以在创建模拟对象的同时声明它们：

<pre><code class="language-groovy">
Subscriber subscriber = Mock {
   1 * receive("hello")
   1 * receive("goodbye")
}
</code></pre>

这个功能对于[存根](/docs/interaction_based_testing#_stubbing)（Stubbing）和具有专门的[存根](/docs/interaction_based_testing#Stubs) (Stubs) 尤其有吸引力。请注意，这些交互没有（也不能有）目标约束；从上下文中可以明确它们属于哪个模拟对象。

交互还可以在使用模拟对象初始化实例字段时声明：

<pre><code class="language-groovy">
class MySpec extends Specification {
    Subscriber subscriber = Mock {
        1 * receive("hello")
        1 * receive("goodbye")
    }
}
</code></pre>


<a name="_grouping_interactions_with_same_target"></a>

## 将具有相同目标的交互进行分组

具有相同目标的交互可以在 `Specification.with` 块中分组。与[在模拟对象创建时声明交互类似](/docs/interaction_based_testing#declaring-interactions-at-creation-time)，这样就不需要重复目标约束了：

<pre><code class="language-groovy">
with(subscriber) {
    1 * receive("hello")
    1 * receive("goodbye")
}
</code></pre>

一个 with 块也可以用于将具有相同目标的条件分组。



<a name="_mixing_interactions_and_conditions"></a>

## 混合交互和条件

一个`then:`块可以包含交互和条件。虽然不是强制要求，但习惯上在条件之前声明交互：
<pre><code class="language-groovy">
when:
publisher.send("hello")

then:
1 * subscriber.receive("hello")
publisher.messageCount == 1
</code></pre>

请朗读："当发布者发送一条'hello'消息时，订阅者应该只收到一次消息，而发布者的消息计数应为1。"



<a name="_explicit_interaction_blocks"></a>

## 显式交互块

在内部，Spock在交互发生之前必须获得有关预期交互的完整信息。那么，在`then:`块中如何声明交互呢？答案是，Spock在幕后将在`then:`块中声明的交互移动到紧接在前面的`when:`块之前。在大多数情况下，这样做没有问题，但有时可能会导致问题：

<pre><code class="language-groovy">
when:
publisher.send("hello")

then:
def message = "hello"
1 * subscriber.receive(message)
</code></pre>

在这里，我们引入了一个变量来表示预期的参数。（同样，我们也可以为基数引入一个变量。）然而，Spock并不聪明（嗯？）到能够知道交互与变量声明之间的内在关联。因此，它只会移动交互，这将导致运行时出现`MissingPropertyException`。

解决这个问题的一种方法是将（至少）变量声明移到`when:`块之前。（喜欢[数据驱动测试](/docs/data_driven_testing)的粉丝可能会将变量移动到`where:`块中。）在我们的例子中，这将带来一个额外的好处，即我们可以使用同一个变量发送消息。

另一种解决方案是明确变量声明和交互之间的关联：

<pre><code class="language-groovy">
when:
publisher.send("hello")

then:
interaction {
  def message = "hello"
  1 * subscriber.receive(message)
}
</code></pre>

由于`MockingApi.interaction`块始终完整地移动，现在代码按预期运行。



<a name="_scope_of_interactions"></a>

## 交互的作用域

在 `then:` 块中声明的交互的作用域限定在前面的 `when:` 块中：

<pre><code class="language-groovy">
when:
publisher.send("message1")

then:
1 * subscriber.receive("message1")

when:
publisher.send("message2")

then:
1 * subscriber.receive("message2")
</code></pre>

这样可以确保在第一个`when:` 块执行期间，`subscriber` 接收到 "message1"，在第二个 `when:` 块执行期间，`subscriber` 接收到 "message2"。

在 `then:` 块之外声明的交互从声明开始一直有效，直到包含的特性方法结束。

交互始终限定在特定的特性方法中。因此，它们不能在静态方法、`setupSpec`方法或`cleanupSpec` 方法中声明。同样，模拟对象不应存储在静态或 `@Shared` 字段中。



<a name="_verification_of_interactions"></a>

## 交互验证

模拟对象的测试可能会以两种主要方式失败：一个交互匹配的调用次数超过了允许的次数，或者交互匹配的调用次数少于所需的次数。前一种情况在调用发生时就会被检测到，并引发`TooManyInvocationsError`错误：

<pre><code class="language-groovy">
Too many invocations for:

2 * subscriber.receive(_) (3 invocations)
</code></pre>

为了更容易诊断为什么会匹配太多次调用，Spock会显示与所涉及交互匹配的所有调用：

<pre><code class="language-groovy">
Matching invocations (ordered by last occurrence):

2 * subscriber.receive("hello")   <-- this triggered the error
1 * subscriber.receive("goodbye")
</code></pre>

根据这个输出，其中一个`receive("hello")`调用触发了`TooManyInvocationsError`错误。请注意，由于无法区分的调用（如两次`subscriber.receive("hello")`调用）会被合并为一行输出，因此第一个`receive("hello")`可能在`receive("goodbye")`之前发生。

后一种情况（调用次数少于所需次数）只能在`when`块执行完成后才能检测到（在此之前，可能会发生进一步的调用）。它会引发`TooFewInvocationsError`错误：

<pre><code class="language-groovy">
Too few invocations for:

1 * subscriber.receive("hello") (0 invocations)
</code></pre>

请注意，无论方法根本未被调用、相同方法使用了不同的参数、相同方法在不同的模拟对象上被调用，还是另一个方法被调用"代替"了这个方法，都会导致`TooFewInvocationsError`错误发生。

为了更容易诊断未匹配调用的情况，Spock会显示所有未匹配的调用，按照与所涉及交互的相似性进行排序。特别是，与交互的参数除外，其他所有方面都匹配的调用将首先显示：

<pre><code class="language-groovy">
Unmatched invocations (ordered by similarity):

1 * subscriber.receive("goodbye")
1 * subscriber2.receive("hello")
</code></pre>



<a name="_invocation_order_2"></a>

### 调用顺序

通常情况下，确切的方法调用顺序并不重要，而且可能随时间而变化。为了避免过度规定，Spock默认允许任何调用顺序，只要满足指定的交互即可：

<pre><code class="language-groovy">
then:
2 * subscriber.receive("hello")
1 * subscriber.receive("goodbye")
</code></pre>

在这种情况下，任何满足指定交互的调用序列，如 "hello" "hello" "goodbye"，"hello" "goodbye" "hello" 和 "goodbye" "hello" "hello" 都是可以的。

在那些调用顺序很重要的情况下，你可以通过将交互拆分为多个`then:`块来强制执行顺序：

<pre><code class="language-groovy">
then:
2 * subscriber.receive("hello")

then:
1 * subscriber.receive("goodbye")
</code></pre>

现在，Spock将验证在"goodbye"之前收到了两个"hello"。换句话说，调用顺序在`then:`块之间强制执行，但在`then:`块内部不强制执行。

> 使用`and:`拆分`then:`块不会强制任何顺序，因为`and:`仅用于文档目的，不具有任何语义。



<a name="_mocking_classes"></a>

## 模拟类

除了接口之外，Spock还支持对类进行模拟。模拟类的工作方式与模拟接口类似；唯一的额外要求是将`byte-buddy` 1.9+或`cglib-nodep` 3.2.0+放置在类路径上。

当使用以下情况时：
- 普通的模拟或存根(Mock或Stub)；
- 配置了`useObjenesis`: true的`Spy`；
- 对具体实例进行监视，如`Spy(myInstance)`。

此外，还需要将`objenesis` 3.0+放置在类路径上，除非具有可访问的无参构造函数或已配置的`constructorArgs`，除非不应执行构造函数调用，例如为了避免不需要的副作用。

如果在类路径上缺少这些库中的任何一个，Spock会友好地通知你。



<a name="_stubbing"></a>

## 存根


存根是使协作对象以某种方式响应方法调用的行为。在存根方法时，你不关心方法是否以及被调用多少次；你只希望在每次调用时返回某个值或执行某些副作用。

为了说明以下示例，请修改`Subscriber`的`receive`方法以返回指示订阅者是否能够处理消息的状态码：

<pre><code class="language-groovy">
interface Subscriber {
    String receive(String message)
}
</code></pre>

现在，让`receive`方法在每次调用时返回"ok"：

<pre><code class="language-groovy">
subscriber.receive(_) >> "ok"
</code></pre>

大声朗读：“每当订阅者接收到一条消息，让它回复'ok'。”

与模拟交互相比，存根交互在左端没有基数，并在右端添加响应生成器：

<pre><code class="language-groovy">
subscriber.receive(_) >> "ok"
|             |          |      |
|             |          |      响应生成器
|             |          参数约束
|             方法约束
目标约束
</code></pre>

存根交互可以在通常的位置声明：要么在`then:`块内，要么在`when:`块之前。（查看[交互声明的位置](/docs/interaction_based_testing#_where_to_declare_interactions)了解更多细节。）如果模拟对象仅用于存根，通常会在[创建模拟时](/docs/interaction_based_testing#declaring-interactions-at-creation-time)或在`given:`块中声明交互。



<a name="_returning_fixed_values"></a>

### 返回固定值

我们已经看到了使用右移（`>>`）运算符返回固定值的用法：

<pre><code class="language-groovy">
subscriber.receive(_) >> "ok"
</code></pre>

要为不同的调用返回不同的值，请使用多个交互：

<pre><code class="language-groovy">
subscriber.receive("message1") >> "ok"
subscriber.receive("message2") >> "fail"
</code></pre>

这将在接收到"message1"时返回"ok"，在接收到"message2"时返回"fail"。可以返回的值没有限制，只要它们与方法声明的返回类型兼容即可。



<a name="_returning_sequences_of_values"></a>

### 返回值序列

要在连续的调用中返回不同的值，请使用三重右移（`>>>`）运算符：

<pre><code class="language-groovy">
subscriber.receive(_) >>> ["ok", "error", "error", "ok"]
</code></pre>

这将在第一次调用时返回"ok"，在第二次和第三次调用时返回"error"，并在所有剩余的调用中返回"ok"。右侧必须是Groovy知道如何迭代的值；在此示例中，我们使用了一个普通列表。



<a name="_computing_return_values"></a>

### 计算返回值

要根据方法的参数计算返回值，请使用右移（>>）运算符与闭包结合使用。如果闭包声明了一个未经类型声明的参数，它将被传递方法的参数列表：

<pre><code class="language-groovy">
subscriber.receive(_) >> { args -> args[0].size() > 3 ? "ok" : "fail" }
</code></pre>

如果消息的长度超过三个字符，将返回"ok"，否则返回"fail"。

在大多数情况下，直接访问方法的参数会更方便。如果闭包声明了多个参数或一个已类型声明的参数，方法参数将一一映射到闭包参数：

<pre><code class="language-groovy">
subscriber.receive(_) >> { String message -> message.size() > 3 ? "ok" : "fail" }
</code></pre>

该响应生成器的行为与前一个相同，但可读性更好。

如果你需要有关方法调用的更多信息而不仅仅是其参数，请查看`org.spockframework.mock.IMockInvocation`。在闭包中，此接口中声明的所有方法都可用，无需前缀。 （在Groovy术语中，闭包委托给`IMockInvocation`的实例。）



<a name="_performing_side_effects"></a>

### 执行副作用

有时，你可能希望执行更多操作而不仅仅计算返回值。一个典型的例子是抛出异常。同样，闭包可以解决这个问题：

<pre><code class="language-groovy">
subscriber.receive(_) >> { throw new InternalError("ouch") }
</code></pre>

当每次传入的调用与交互匹配时，闭包中的代码会执行。



<a name="_chaining_method_responses"></a>

### 链接方法响应

方法响应可以链接起来：

<pre><code class="language-groovy">
subscriber.receive(_) >>> ["ok", "fail", "ok"] >> { throw new InternalError() } >> "ok"
</code></pre>

这将为前三个调用返回"ok"、"fail"、"ok"，对于第四个调用抛出InternalError，并对任何进一步的调用返回ok。



<a name="_returning_a_default_response"></a>

### 返回默认响应

如果你不关心返回什么，但必须返回非空值，可以使用`_`。它将使用与存根（参见[存根](/docs/interaction_based_testing#Stubs)）相同的逻辑计算响应，因此它支队 `Mock` 和 `Spy` 实例有用。

<pre><code class="language-groovy">
subscriber.receive(_) >> _
</code></pre>

当然，你也可以与链接一起使用。这对于存根实例可能也很有用。

<pre><code class="language-groovy">
subscriber.receive(_) >>> ["ok", "fail"] >> _ >> "ok"
</code></pre>

此用法是将`Mock`行为像`Stub`一样，但仍然能够进行断言。如果方法的返回类型可从模拟类型（但不包括对象）进行分配，则默认响应将返回模拟本身。这在处理流畅的API（例如构建器）时非常有用，否则很难对其进行模拟。

<pre><code class="language-groovy">
given:
ThingBuilder builder = Mock() {
  _ >> _
}

when:
Thing thing = builder
  .id("id-42")
  .name("spock")
  .weight(100)
  .build()

then:
1 * builder.build() >> new Thing(id: 'id-1337')
thing.id == 'id-1337'
</code></pre>

`_ >> _`指示模拟对象对所有交互返回默认响应。然而，在`then`块中定义的交互将优先于`given`块中定义的交互，这样我们就可以覆盖和断言我们真正关心的交互。



<a name="_combining_mocking_and_stubbing"></a>

## 组合模拟和存根

模拟和存根是密切相关的：

<pre><code class="language-groovy">
1 * subscriber.receive("message1") >> "ok"
1 * subscriber.receive("message2") >> "fail"
</code></pre>

当对同一个方法调用进行模拟和存根时，它们必须在同一个交互中发生。特别地，以下类似Mockito风格的将存根和模拟分开为两个单独语句的做法是行不通的：

<pre><code class="language-groovy">
given:
subscriber.receive("message1") >> "ok"

when:
publisher.send("message1")

then:
1 * subscriber.receive("message1")
</code></pre>

如在"[声明交互的位置](/docs/interaction_based_testing#_where_to_declare_interactions)"中所解释的，`receive`调用首先会与`then`块中的交互进行匹配。由于该交互没有指定响应，方法的返回类型的默认值（在这种情况下为`null`）将被返回。 （这只是Spock对模拟的宽松处理的另一个方面。）因此，在`given`块中的交互将永远没有机会匹配。

> 模拟和存根的同一方法调用必须在同一个交互中进行。



<a name="OtherKindsOfMockObjects"></a>

## 其他类型的模拟对象

到目前为止，我们使用`MockingApi.Mock`方法创建了模拟对象。除此方法之外，`MockingApi`类还提供了几个其他的工厂方法，用于创建更专门的模拟对象。



<a name="Stubs"></a>

### 存根

可以使用`MockingApi.Stub`工厂方法创建存根：

<pre><code class="language-groovy">
Subscriber subscriber = Stub()
</code></pre>

与模拟对象不同，存根只能用于存根。将协作者限定为存根将其角色传达给规范的读者。

> 如果存根调用与必需的交互（如1 * foo.bar()）匹配，则会引发InvalidSpecException异常。

与模拟对象类似，存根允许出现意外的调用。然而在这种情况下，存根返回的值更加宽泛：

- 对于基本类型，将返回基本类型的默认值。
- 对于非基本数值类型（如`BigDecimal`），将返回零。
- 如果值可以赋值给存根实例，则返回该实例（例如构建器模式）。
- 对于非数值类型，将返回一个"空"或"虚拟"对象。这可能是一个空字符串、一个空集合、通过其默认构造函数构造的对象，或者返回默认值的另一个存根。有关详细信息，请参见`org.spockframework.mock.EmptyOrDummyResponse`类。

> 如果方法的响应类型是一个final类或者它需要一个类模拟库，而cglib或ByteBuddy不可用，则无法创建"虚拟"对象，并会引发`CannotCreateMockException`异常。

存根通常具有固定的一组交互，这使得在模拟创建时声明交互特别有吸引力：

<pre><code class="language-groovy">
Subscriber subscriber = Stub {
    receive("message1") >> "ok"
    receive("message2") >> "fail"
}
</code></pre>


<a href="Spies"></a>

### 间谍（Spies）



（慎重使用此功能。更改规范代码的设计可能会更好。）

使用`MockingApi.Spy`工厂方法创建一个Spy：

<pre><code class="language-groovy">
SubscriberImpl subscriber = Spy(constructorArgs: ["Fred"])
</code></pre>

Spy总是基于一个真实对象。因此，你必须提供一个类类型而不是接口类型，以及类型的任何构造函数参数。如果未提供构造函数参数，将使用该类型的无参数构造函数。

如果给定的构造函数参数导致歧义，你可以像通常一样使用`as`或Java风格的强制类型转换。例如，如果被测试对象（testee）有一个带有字符串参数的构造函数和一个带有Pattern参数的构造函数，并且你想将`constructorArg`设置为`null`：

<pre><code class="language-groovy">
SubscriberImpl subscriber = Spy(constructorArgs: [null as String])
SubscriberImpl subscriber2 = Spy(constructorArgs: [(Pattern) null])
</code></pre>

你还可以从一个已实例化的对象创建一个Spy。这在你无法完全控制所感兴趣的类型的实例化的情况下非常有用（例如，在依赖注入框架（如Spring或Guice）中进行测试时）。

Spy上的方法调用将自动委托给真实对象。同样，真实对象方法返回的值将通过Spy传递给调用者。

创建了一个Spy之后，你可以监听调用者与Spy下面的真实对象之间的对话：

<pre><code class="language-groovy">
1 * subscriber.receive(_)
</code></pre>

除了确保`receive`只被调用一次之外，发布者和Spy下面的`SubscriberImpl`实例之间的对话保持不变。

在Spy上存根方法时，不再调用真实方法：

<pre><code class="language-groovy">
subscriber.receive(_) >> "ok"
</code></pre>

现在，`receive`方法将简单地返回"ok"，而不是调用`SubscriberImpl.receive`。

有时，同时执行一些代码并委托给真实方法是可取的：

<pre><code class="language-groovy">
subscriber.receive(_) >> { String message -> callRealMethod(); message.size() > 3 ? "ok" : "fail" }
</code></pre>

在这里，我们使用`callRealMethod()`将方法调用委托给真实对象。请注意，我们不需要手动传递消息参数；这会自动处理。`callRealMethod()`返回真实调用的结果，但在此示例中，我们选择返回自己的结果。如果我们想将不同的消息传递给真实方法，我们可以使用`callRealMethodWithArgs("changed message")`。

请注意，虽然从语义上来说，`callRealMethod()`和`callRealMethodWithArgs(…​)`只在Spy上有意义，但从技术上讲，你也可以在模拟对象或存根对象上调用这些方法，从而将它们变成（伪）Spy对象。唯一的前提是模拟/存根对象实际上具有真实方法的实现，即对于接口模拟

对象，必须有一个默认方法；对于类模拟对象，必须有一个（非抽象的）原始方法。



<a name="PartialMocks"></a>

### 部分Mock

（慎重使用此功能。更改规范代码的设计可能会更好。）

Spy也可以用作部分Mock：

<pre><code class="language-groovy">
// 现在，这是规范对象，而不是协作者
MessagePersister persister = Spy {
  // 在同一个对象上进行存根调用
  isPersistable(_) >> true
}

when:
persister.receive("msg")

then:
// 要求在同一个对象上调用
1 * persister.persist("msg")
</code></pre>



<a name="GroovyMocks"></a>

## Groovy模拟（Groovy Mocks）

到目前为止，我们看到的所有模拟功能都不管调用代码是用Java还是Groovy编写的都是一样的。通过利用Groovy的动态能力，Groovy模拟提供了一些专门用于测试Groovy代码的额外功能。它们使用`MockingApi.GroovyMock()`、`MockingApi.GroovyStub()`和`MockingApi.GroovySpy()`工厂方法来创建。

> 何时应该使用Groovy模拟而不是普通模拟？在代码规范中使用Groovy编写，并且需要一些独特的Groovy模拟功能时，应该使用Groovy模拟。当从Java代码调用时，Groovy模拟将表现得像普通模拟一样。请注意，仅仅因为规范的代码和/或模拟的类型是用Groovy编写的，并不意味着必须使用Groovy模拟。除非你有具体的理由使用Groovy模拟，否则请使用普通模拟。





<a name="_mocking_dynamic_methods"></a>

### 模拟动态方法

所有的Groovy模拟对象都实现了`GroovyObject`接口。它们支持对动态方法进行模拟和存根操作，就好像它们是实际声明的方法一样：

<pre><code class="language-groovy">
Subscriber subscriber = GroovyMock()

1 * subscriber.someDynamicMethod("hello")
</code></pre>



<a name="MockingAllInstancesOfAType"></a>

### 模拟类型的所有实例

（在使用此功能之前要三思。更改规范代码的设计可能会更好。）

通常，Groovy模拟对象需要像普通模拟对象一样被注入到规范代码中。然而，当一个Groovy模拟对象被创建为全局对象时，它会在整个feature方法的执行期间自动替换掉被模拟类型的所有真实实例：

<pre><code class="language-groovy">
def publisher = new Publisher()
publisher << new RealSubscriber() << new RealSubscriber()

RealSubscriber anySubscriber = GroovyMock(global: true)

when:
publisher.publish("message")

then:
2 * anySubscriber.receive("message")
</code></pre>

在这里，我们设置了一个具有两个真实订阅者实现的发布者。然后我们创建了一个相同类型的全局模拟对象。这将把所有对真实订阅者的方法调用重新定向到模拟对象。模拟对象的实例不会传递给发布者；它只用于描述交互。

> 全局模拟对象只能针对类类型进行创建。它会在整个feature方法的执行期间有效地替换掉该类型的所有实例。

由于全局模拟对象具有某种程度上的全局影响，因此通常与GroovySpy一起使用会更方便。这样可以在匹配交互时执行真实代码，并允许你选择性地监听对象并在需要时更改它们的行为。

> #### 全局Groovy模拟对象是如何实现的？
全局Groovy模拟对象通过Groovy元编程获得其超能力。更准确地说，每个全局模拟类型在特性方法的执行期间被分配了一个自定义元类。由于全局Groovy模拟对象仍然基于CGLIB代理，因此在从Java代码中调用时，它将保留其一般的模拟功能（但不包括超能力）。



<a name="MockingConstructors"></a>

### 模拟构造函数

（在使用此功能之前要三思。更改规范代码的设计可能会更好。）

全局模拟对象支持模拟构造函数：

<pre><code class="language-groovy">
RealSubscriber anySubscriber = GroovySpy(global: true)

1 * new RealSubscriber("Fred")
</code></pre>

由于我们使用了一个Spy对象，从构造函数调用返回的对象保持不变。要更改要构造的对象，可以对构造函数进行存根操作：

<pre><code class="language-groovy">
new RealSubscriber("Fred") >> new RealSubscriber("Barney")
</code></pre>

现在，无论何时有代码尝试构造一个名为Fred的订阅者，我们将构造一个名为Barney的订阅者。



<a name="_mocking_static_methods"></a>

### 模拟静态方法

（在使用此功能之前要三思。更改规范代码的设计可能会更好。）

全局模拟对象支持模拟和存根静态方法：

<pre><code class="language-groovy">
RealSubscriber anySubscriber = GroovySpy(global: true)

1 * RealSubscriber.someStaticMethod("hello") >> 42
</code></pre>

对于动态的静态方法也是一样适用。

当全局模拟对象仅用于模拟构造函数和静态方法时，实际上并不需要模拟对象的实例。在这种情况下，可以简写为：

<pre><code class="language-groovy">
GroovySpy(RealSubscriber, global: true)
</code></pre>


<a name="_advanced_features"></a>

## 高级特性

大多数情况下，你不应该需要这些特性。但如果你需要，你会很高兴拥有它们。



<a name="ALaCarteMocks"></a>

### 可选模拟对象

归根结底，`Mock()`、`Stub()`和`Spy()`工厂方法只是一种创建具有特定配置的模拟对象的预设方式。如果你希望更精细地控制模拟对象的配置，可以查看`org.spockframework.mock.IMockConfiguration`接口。该接口的所有属性都可以作为命名参数传递给`Mock()`方法。例如：

<pre><code class="language-groovy">
def person = Mock(name: "Fred", type: Person, defaultResponse: ZeroOrNullResponse.INSTANCE, verified: false)
</code></pre>

在这里，我们创建了一个模拟对象，其默认返回值与`Mock()`的返回值匹配，但其调用不被验证（类似于`Stub()`）。我们可以通过传递`ZeroOrNullResponse`的自定义`org.spockframework.mock.IDefaultResponse`来响应意外的方法调用。



<a name="DetectingMockObjects"></a>

### 检测模拟对象

要确定特定对象是否为Spock模拟对象，可以使用`org.spockframework.mock.MockUtil`：

<pre><code class="language-groovy">
MockUtil mockUtil = new MockUtil()
List list1 = []
List list2 = Mock()

expect:
!mockUtil.isMock(list1)
mockUtil.isMock(list2)
</code></pre>

还可以使用MockUtil来获取有关模拟对象的更多信息：

<pre><code class="language-groovy">
IMockObject mock = mockUtil.asMock(list2)

expect:
mock.name == "list2"
mock.type == List
mock.nature == MockNature.MOCK
</code></pre>



<a name="_further_reading"></a>

## 深度阅读

如果你希望更深入地了解基于交互的测试，我们推荐以下资源：

- [Endo-Testing: Unit Testing with Mock Objects](https://www.ccs.neu.edu/research/demeter/related-work/extreme-programming/MockObjectsFinal.PDF)

这是来自XP2000会议的论文，介绍了模拟对象的概念。

- [Mock Roles, not Objects](https://www.jmock.org/oopsla2004.pdf)

这是来自OOPSLA2004会议的论文，解释了如何正确地进行模拟。

- [Mocks Aren't Stubs](https://martinfowler.com/articles/mocksArentStubs.html)

Martin Fowler对模拟的看法。

- [Growing Object-Oriented Software Guided by Tests](https://www.growing-object-oriented-software.com/)

TDD先驱Steve Freeman和Nat Pryce详细解释了测试驱动开发和模拟在现实世界中的工作原理。