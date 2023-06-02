# Data Driven Testing

通常情况下，通过多次运行相同的测试代码，使用不同的输入和预期结果是很有用的。Spock的数据驱动测试支持使得这成为一项一流的特性。

## 简介

假设我们想要指定`Math.max`方法的行为：



<pre><code class="language-groovy">
class MathSpec extends Specification {
  def "maximum of two numbers"() {
    expect:
    // exercise math method for a few different inputs
    Math.max(1, 3) == 3
    Math.max(7, 4) == 7
    Math.max(0, 0) == 0
  }
}
</code></pre>


虽然这种方法在像这样简单的情况下是可以的，但它也有一些潜在的缺点：

- 代码和数据混合在一起，难以独立地进行更改；

- 数据难以自动生成或从外部源获取；

- 为了多次执行相同的代码，要么需要复制代码，要么将其提取到单独的方法中；

- 如果出现失败，可能不会立即清楚是哪些输入导致了失败；

- 多次执行相同的代码不像执行单独的方法那样具有相同的隔离性。

Spock的数据驱动测试支持旨在解决这些问题。为了开始使用数据驱动测试，让我们将上述代码重构为一个数据驱动的特性方法。首先，我们引入三个方法参数（称为数据变量），用来替换硬编码的整数值：



<pre><code class="language-groovy">
class MathSpec extends Specification {
  def "maximum of two numbers"(int a, int b, int c) {
    expect:
    Math.max(a, b) == c
    ...
  }
}
</code></pre>

我们已经完成了测试逻辑，但仍需要提供要使用的数据值。这是通过`where:`块来实现的，它始终位于方法的末尾。在最简单（也是最常见）的情况下，`where: `块包含一个数据表。



## 数据表（Data Tables）

数据表是使用一组固定数据值来执行特性方法的便捷方式：

<pre><code class="language-groovy">
class MathSpec extends Specification {
  def "maximum of two numbers"(int a, int b, int c) {
    expect:
    Math.max(a, b) == c

    where:
    a | b | c
    1 | 3 | 3
    7 | 4 | 7
    0 | 0 | 0
  }
}
</code></pre>

数据表的第一行称为__表头__（table header），声明了数据变量。随后的行称为__表行__（table rows），保存了相应的值。对于每一行，特性方法将被执行一次；我们称之为__方法的迭代__（iteration of the method）。如果一个迭代失败，剩余的迭代仍然会执行。所有的失败都将被报告。

数据表必须至少有两列。一个单列的数据表可以写成：

<pre><code class="language-groovy">
where:
a | _
1 | _
7 | _
0 | _
</code></pre>

可以使用两个或更多下划线的序列将一个宽数据表分成多个较窄的数据表。如果没有这个分隔符，也没有其他数据变量的赋值在中间，就无法在一个`where`块中有多个数据表，第二个表将只是第一个表的进一步迭代，包括看似是表头的行：

<pre><code class="language-groovy">
where:
a | _
1 | _
7 | _
0 | _
__

b | c
1 | 2
3 | 4
5 | 6
</code></pre>

从语义上讲，这完全相同，只是一个更宽的组合数据表而已：

<pre><code class="language-groovy">
where:
a | b | c
1 | 1 | 2
7 | 3 | 4
0 | 5 | 6
</code></pre>

两个或更多下划线的序列可以在`where`块的任何位置使用。它将被忽略，除非在两个数据表之间使用它来分隔这两个数据表。这意味着分隔符也可以作为样式元素以不同的方式使用。它可以像最后一个示例中所示，用作分隔线，或者也用作表格的顶部边框，从视觉上起到分隔它们的效果：

<pre><code class="language-groovy">
where:
_____
a | _
1 | _
7 | _
0 | _
_____
b | c
1 | 2
3 | 4
5 | 6
</code></pre>

## 迭代的隔离执行

每个迭代都在独立的执行环境中进行，就像是独立的特性方法一样。每个迭代都会获得自己的规范类实例，并在执行之前和之后分别调用设置（setup）和清理（cleanup）方法。这样确保了每个迭代的执行相互独立，不会相互影响。



## 迭代间的对象共享

为了在迭代之间共享对象，需要将它保存在`@Shared`或`static`字段中。

> 只有`@Shared`和`static`变量可以从`where:`块内部访问。

请注意，这样的对象也将与其他方法共享。目前没有很好的方法可以在同一方法的迭代之间共享对象。如果你认为这是一个问题，请考虑将每个方法放入单独的规范中，所有规范可以保存在同一个文件中。这样可以实现更好的隔离，代价则是一些模板代码。



## 语法变化

前面的代码可以通过几种方式进行调整。

首先，由于`where:`块已经声明了所有数据变量，方法参数可以省略。

你还可以省略一些参数并指定其他参数，例如对其进行类型化。顺序也不重要，数据变量与指定的方法参数按名称匹配。

其次，可以使用双竖线符号（||）将输入和预期输出分开，从视觉上进行区分。

使用这种方式，代码变为：

<pre><code class="language-groovy">
class MathSpec extends Specification {
  def "maximum of two numbers"() {
    expect:
    Math.max(a, b) == c

    where:
    a | b || c
    1 | 3 || 3
    7 | 4 || 7
    0 | 0 || 0
  }
}
</code></pre>

除了使用单个或双个竖线分隔符之外，你还可以使用任意数量的分号来分隔数据列。

<pre><code class="language-groovy">
class MathSpec extends Specification {
  def "maximum of two numbers"() {
    expect:
    Math.max(a, b) == c

    where:
    a ; b ;; c
    1 ; 3 ;; 3
    7 ; 4 ;; 7
    0 ; 0 ;; 0
  }
}
</code></pre>

在一个表格中不能混合使用竖线和分号作为数据列分隔符。如果列分隔符发生变化，将会开始一个新的独立数据表。

<pre><code class="language-groovy">
class MathSpec extends Specification {
  def "maximum of two numbers"() {
    expect:
    Math.max(a, b) == c
    Math.max(d, e) == f

    where:
    a | b || c
    1 | 3 || 3
    7 | 4 || 7
    0 | 0 || 0

    d ; e ;; f
    1 ; 3 ;; 3
    7 ; 4 ;; 7
    0 ; 0 ;; 0
  }
}
</code></pre>

## 失败的报告

假设我们的`max`方法的实现存在一个缺陷，并且其中一个迭代失败了：

<pre><code class="language-bash">
maximum of two numbers [a: 1, b: 3, c: 3, #0]   PASSED
maximum of two numbers [a: 7, b: 4, c: 7, #1]   FAILED

Condition not satisfied:

Math.max(a, b) == c
|    |   |  |  |  |
|    |   7  4  |  7
|    42        false
class java.lang.Math

maximum of two numbers [a: 0, b: 0, c: 0, #2]   PASSED
</code></pre>

显而易见的问题是：哪个迭代失败了，它的数据值是什么？在我们的例子中，通过丰富的条件显示，很容易发现是第二个迭代（索引为1）失败了。在其他情况下，这可能更加困难甚至不可能。无论如何，Spock清楚地指出了哪个迭代失败，而不仅仅报告失败。特征方法的迭代默认采用丰富的命名模式进行展开。可以按照[展开的迭代名称](/docs/data_driven_testing#_unrolled_iteration_names)文档中所述进行配置，或者可以像下一节所述的那样禁用展开。



## 方法的卷起（Uprolling）和展开（Unrolling）

使用`@Rollup`注解标记的方法将不会单独报告其迭代次数，而只会在特征中进行聚合。例如，如果你根据计算产生许多测试用例，或者将外部数据（如数据库内容）用作测试数据，并且不希望测试计数发生变化，那么可以使用此功能。

<pre><code class="language-groovy">
@Rollup
def "maximum of two numbers"() {
...
</code></pre>

请注意，卷起（uprolling）和展开（unrolling）不会影响方法的执行方式，它们只是一种报告方式的变化。根据执行环境的不同，输出结果可能类似于：

<pre><code class="language-bash">
maximum of two numbers   FAILED

Condition not satisfied:

Math.max(a, b) == c
|    |   |  |  |  |
|    |   7  4  |  7
|    42        false
class java.lang.Math
</code></pre>

`@Rollup`注解也可以放置在规范(spec)上。这与将其放置在规范的每个没有`@Unroll`注解的数据驱动特征方法上的效果相同。

另外，[配置文件](/docs/extensions#spock-configuration-file)中`unroll`部分的`unrollByDefault`设置可以设置为`false`，以便使所有特征使用卷起（除非它们被标注为`@Unroll`或包含在`@Unrolled`的规范中），并恢复到Spock 2.0之前的默认行为。



### 禁用默认的展开

<pre><code class="language-groovy">
unroll {
  unrollByDefault false
}
</code></pre>

在规范或特征上同时使用`@Unroll`和`@Rollup`注解是不允许的，如果检测到这种情况，将会引发异常。

----------------------------

总结如下：

以下情况__特征将被卷起（uprolled）__

- 如果方法被`@Rollup`注解标记

- 如果方法未被`@Unroll`注解标记且规范被`@Rollup`注解标记

- 如果方法和规范都未被`@Unroll`注解标记，且配置选项`unroll { unrollByDefault }`设置为`false`

以下情况__特征将被展开（unrolled）__

- 如果方法被`@Unroll`注解标记

- 如果方法未被`@Rollup`注解标记且规范被`@Unroll`注解标记

- 如果方法和规范都未被`@Rollup`注解标记，且配置选项`unroll { unrollByDefault }`设置为默认值`true`



## 数据管道

数据表不是向数据变量提供值的唯一方式。事实上，数据表只是一个或多个数据管道的语法糖：

<pre><code class="language-groovy">
...
where:
a << [1, 7, 0]
b << [3, 4, 0]
c << [3, 7, 0]
</code></pre>

使用左移操作符（<<）表示的数据管道将数据变量与数据提供者连接起来。数据提供者保存了变量的所有值，每个迭代一个值。任何Groovy可迭代对象都可以用作数据提供者。这包括`Collection`、`String`、`Iterable`类型的对象，以及实现`Iterable`接口的对象。数据提供者不一定是实际的数据（例如`Collection`的情况），它们可以从外部来源（如文本文件、数据库和电子表格）获取数据，或者随机生成数据。只有在需要时（在下一次迭代之前），才会向数据提供者查询下一个值。



