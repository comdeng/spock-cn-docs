---
title: "数据驱动测试"
---

# 数据驱动测试

通常情况下，通过多次运行相同的测试代码，使用不同的输入和预期结果是很有用的。Spock的数据驱动测试支持使得这成为一项一流的特性。



<a name="introduction-2"></a>

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



<a name="data-tables"></a>

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


<a name="isolated-execution-of-iterations"></a>

## 迭代的隔离执行

每个迭代都在独立的执行环境中进行，就像是独立的特性方法一样。每个迭代都会获得自己的规范类实例，并在执行之前和之后分别调用设置（setup）和清理（cleanup）方法。这样确保了每个迭代的执行相互独立，不会相互影响。



<a name="sharing-of-objects-between-iterations"></a>

## 迭代间的对象共享

为了在迭代之间共享对象，需要将它保存在`@Shared`或`static`字段中。

> 只有`@Shared`和`static`变量可以从`where:`块内部访问。

请注意，这样的对象也将与其他方法共享。目前没有很好的方法可以在同一方法的迭代之间共享对象。如果你认为这是一个问题，请考虑将每个方法放入单独的规范中，所有规范可以保存在同一个文件中。这样可以实现更好的隔离，代价则是一些模板代码。



<a name="syntactic-variations"></a>

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



<a name="reporting-of-failures"></a>

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



<a name="method-uprolling-and-unrolling"></a>

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



<a name="data-pipes"></a>

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



<a name="multi-variable-data-pipes"></a>

## 多变量数据管道

如果数据提供者在每次迭代中返回多个值（作为Groovy可迭代对象），它可以同时连接到多个数据变量。这种语法与Groovy的多重赋值类似，但在左侧使用方括号而不是圆括号：

<pre><code class="language-groovy">
@Shared sql = Sql.newInstance("jdbc:h2:mem:", "org.h2.Driver")

def "maximum of two numbers"() {
  expect:
  Math.max(a, b) == c

  where:
  [a, b, c] << sql.rows("select a, b, c from maxdata")
}
</code></pre>

不感兴趣的数据值可以使用下划线（`_`）来忽略：
<pre><code class="language-groovy">
...
where:
[a, b, _, c] << sql.rows("select * from maxdata")
</code></pre>

多重赋值甚至可以嵌套。以下示例将生成这些迭代：

以下是转换成Markdown表格的形式：

| a        | b   | c   |
|----------|-----|-----|
| ['a1', 'a2'] | 'b1' | 'c1' |
| ['a2', 'a1'] | 'b1' | 'c1' |
| ['a1', 'a2'] | 'b2' | 'c2' |
| ['a2', 'a1'] | 'b2' | 'c2' |



<pre><code class="language-groovy">
...
where:
[a, [b, _, c]] << [
  ['a1', 'a2'].permutations(),
  [
    ['b1', 'd1', 'c1'],
    ['b2', 'd2', 'c2']
  ]
].combinations()
</code></pre>


<a name="multi-data-pipe-named"></a>

### 命名解构数据管道

自Spock 2.2版本以来，多变量数据管道也可以从映射中进行解构。当数据提供程序返回带有命名键的映射时，这非常有用。或者，如果您有较长的值不适合使用数据表，那么使用映射使得阅读更加容易。



```groovy
...
where:
[a, b, c] << [
  [
    a: 1,
    b: 3,
    c: 5
  ],
  [
    a: 2,
    b: 4,
    c: 6
  ]
]
```

你可以在嵌套的数据管道中使用命名解构，但只能在最内层的嵌套级别上进行。

```gr
...
where:
[a, [b, c]] << [
  [1, [b: 3, c: 5]],
  [2, [c: 6, b: 4]]
]
```





<a name="data-variable-assignment"></a>

## 数据变量赋值

一个数据变量可以直接被赋值：

```groovy
...
where:
a = 3
b = Math.random() * 100
c = a > b ? a : b
```

赋值语句在每次迭代中重新计算。正如上面已经展示的，赋值语句的右侧可以引用其他数据变量：

```groovy
...
where:
row << sql.rows("select * from maxdata")
// pick apart columns
a = row.a
b = row.b
c = row.c
```



<a name="accessing-other-data-variables"></a>

## 访问其他数据变量

有两种可能性可以从另一个数据变量的计算中访问一个数据变量。

第一种可能性是像上一节所示的派生数据变量。通过直接赋值定义的每个数据变量都可以访问先前定义的所有数据变量，包括通过数据表或数据管道定义的变量：

```groovy
...
where:
a = 3
b = Math.random() * 100
c = a > b ? a : b
```



第二种可能性是在数据表中访问先前的列：

```groovy
...
where:
a | b
3 | a + 1
7 | a + 2
0 | a + 3
```

这也包括在同一`where`块中之前的数据表中的列：

```groovy
...
where:
a | b
3 | a + 1
7 | a + 2
0 | a + 3

and:
c = 1

and:
d     | e
a * 2 | b * 2
a * 3 | b * 3
a * 4 | b * 4
```



<a name="multi-variable-assignment"></a>

## 多变量赋值

与数据管道类似，如果你有一些 Groovy 可以迭代的对象，你也可以在一个表达式中对多个变量进行赋值。与数据管道不同，这里的语法与标准的 Groovy 多重赋值语法相同：



```groovy
@Shared sql = Sql.newInstance("jdbc:h2:mem:", "org.h2.Driver")

def "maximum of two numbers multi-assignment"() {
  expect:
  Math.max(a, b) == c

  where:
  row << sql.rows("select a, b, c from maxdata")
  (a, b, c) = row
}
```

不感兴趣的数据值可以使用下划线（`_`）忽略掉：

```groovy
...
where:
row << sql.rows("select * from maxdata")
(a, b, _, c) = row
```



<a name="combining-data-tables-data-pipes-and-variable-assignments"></a>

## 组合使用数据表、数据管道和变量赋值

数据表、数据管道和变量赋值可以根据需要进行组合使用：

```groovy
...
where:
a | b
1 | a + 1
7 | a + 2
0 | a + 3

c << [3, 4, 0]

d = a > c ? a : c
```



<a name="type-coercion-for-data-variable-values"></a>

## 数据变量值的类型强制转换

数据变量值通过类型强制转换被转换为声明的参数类型。因此，可以通过扩展模块或使用规范的 `@Use` 扩展来提供自定义类型转换（如果应用于特性，则对 `where` 块没有影响）。



```groovy
def "type coercion for data variable values"(Integer i) {
  expect:
  i instanceof Integer
  i == 10

  where:
  i = "10"
}
```



```groovy
@Use(CoerceBazToBar)
class Foo extends Specification {
  def foo(Bar bar) {
    expect:
    bar == Bar.FOO

    where:
    bar = Baz.FOO
  }
}
enum Bar { FOO, BAR }
enum Baz { FOO, BAR }
class CoerceBazToBar {
  static Bar asType(Baz self, Class<Bar> clazz) {
    return Bar.valueOf(self.name())
  }
}
```



<a name="number-of-iterations"></a>

## 迭代次数

迭代次数取决于可用数据的数量。对同一方法的连续执行可能产生不同数量的迭代。如果一个数据提供程序比其他提供程序更早耗尽数值，将会引发异常。变量赋值不会影响迭代次数。一个仅包含赋值的 `where:` 块将产生恰好一次迭代。



<a name="closing-of-data-providers"></a>

## 数据提供程序的关闭

在所有迭代完成后，对于所有具有零参数 `close` 方法的数据提供程序将调用该方法。



<a name="unrolled-iteration-names"></a>

## 展开的迭代名称

默认情况下，展开的迭代名称由特性的名称、数据变量和迭代索引组成。这将始终产生唯一的名称，并且应该能够轻松地识别出失败的数据变量组合。

例如，[失败的报告](/docs/data-driven-testing#reporting-of-failures) 中的示例显示了`最多两个数字[a: 7, b: 4, c: 7, #1]`，其中第二次迭代 (`#1`)失败了，其数据变量取值为 `7`、`4` 和`7` 。

通过一些改进，我们可以做得更好：

```groovy
def "maximum of #a and #b is #c"() {
...
```

这种方法名称使用占位符来表示数据变量`a`、`b` 和 `c`，占位符以 `#` 号开头。在输出中，这些占位符将被具体的值替代：

```bash
maximum of 1 and 3 is 3   PASSED
maximum of 7 and 4 is 7   FAILED

Math.max(a, b) == c
|    |   |  |  |  |
|    |   7  4  |  7
|    42        false
class java.lang.Math

maximum of 0 and 0 is 0   PASSED
```

现在我们一眼就能看出 `max` 方法在输入为 `7` 和 `4` 时失败了。

展开的方法名类似于 Groovy 的 `GString`，但有以下区别：

- 表达式用 `#` 号表示，而不是 `$` 号，并且没有相应的 `${...}`语法。

- 表达式仅支持属性访问和零参数方法调用。

假设有一个具有 `name` 和 `age` 属性的 `Person` 类，以及一个类型为 `Person` 的数据变量 `person`，以下是有效的方法名：



```groovy
def "#person is #person.age years old"() { // property access
def "#person.name.toUpperCase()"() { // zero-arg method call
```

非字符串值（例如上面的 `#person`）根据 Groovy 语义转换为字符串。

以下是无效的方法名：

```groovy
def "#person.name.split(' ')[1]" {  // cannot have method arguments
def "#person.age / 2" {  // cannot use operators
```

如果需要，可以引入额外的数据变量来保存更复杂的表达式：

```groovy
def "#lastName"() {
  ...
  where:
  person << [new Person(age: 14, name: 'Phil Cole')]
  lastName = person.name.split(' ')[1]
}
```

此外，还支持数据变量 `#featureName` 和 `#iterationIndex`。前者在实际特性名称内没有太多意义，但在定义展开模式的另外两个位置上更有用。

```groovy
def "#person is #person.age years old [#iterationIndex]"() {
```

将被报告为：



```bash
╷
└─ Spock ✔
   └─ PersonSpec ✔
      └─ #person.name is #person.age years old [#iterationIndex] ✔
         ├─ Fred is 38 years old [0] ✔
         ├─ Wilma is 36 years old [1] ✔
         └─ Pebbles is 5 years old [2] ✔
```

另外，可以将展开模式作为参数传递给 `@Unroll` 注解，而不是在方法名中指定，该注解的优先级高于方法名：



```groovy
@Unroll("#featureName[#iterationIndex] (#person.name is #person.age years old)")
def "person age should be calculated properly"() {
// ...
```

将被报告为：

```bash
╷
└─ Spock ✔
   └─ PersonSpec ✔
      └─ person age should be calculated properly ✔
         ├─ person age should be calculated properly[0] (Fred is 38 years old) ✔
         ├─ person age should be calculated properly[1] (Wilma is 36 years old) ✔
         └─ person age should be calculated properly[2] (Pebbles is 5 years old) ✔
```

优势在于，你可以为整个特性设置一个描述性的方法名，同时为每个迭代单独设置一个模板。此外，特性方法名不包含占位符，因此更易读。

如果既没有给注解传递参数，也没有在方法名中包含 `#`，则会检查配置文件中展开部分的 `defaultPattern` 设置。如果它设置为非空字符串，则将使用该值作为展开模式。例如，可以将其设置为：

- `#featureName` 以使所有迭代具有相同的名称，或

- `#featureName[#iterationIndex]` 以获得简单的索引迭代名称，或

- `#iterationName` 如果您确保在每个数据驱动的特性中还设置了一个名为 `iterationName` 的数据变量，那么该变量将用于报告。



<a name="unroll-tokens"></a>

### 特殊标记

以下是特殊标记的完整列表：

- `#featureName` 是特性的名称（在 `defaultPattern` 设置中通常很有用）

- `#iterationIndex` 是当前迭代的索引
- `#dataVariables` 列出了该迭代的所有数据变量，例如 `x: 1, y: 2, z: 3`
- `#dataVariablesWithIndex` 与 `#dataVariables` 相同，但在末尾带有索引，例如 `x: 1, y: 2, z: 3, #0`



<a name="configuration"></a>

### 配置

#### 设置默认的展开模式

```groovy
unroll {
    defaultPattern '#featureName[#iterationIndex]'
}
```



如果没有使用上述三种方式之一来设置自定义的展开模式，默认情况下将使用特性名称，后跟所有数据变量的名称和它们的值，最后是迭代索引，因此结果可能为 `my feature [x: 1, y: 2, z: 3, #0]`。

如果展开表达式中存在错误，例如变量名称拼写错误，表达式中的属性或方法在评估过程中引发异常等等，测试将失败。但如果没有以任何方式设置展开模式，则无论发生什么情况， 自动回退的数据变量呈现将永远不会导致测试失败。

可以通过将配置文件中展开部分的 `validateExpressions` 设置为 `false` 来禁用带有展开表达式错误的测试失败。如果这样做并发生错误，错误的表达式 `#foo.bar` 将被替换为 `#Error:foo.bar`。



#### 禁用展开模式表达式断言



```groovy
unroll {
    validateExpressions false
}
```



某些报告框架或集成开发环境（IDE）支持适当的基于树的报告。对于这些情况，可能希望在迭代报告中省略特性名称。



#### 禁用迭代中的特性名称

```groovy
unroll {
    includeFeatureNameForIterations false
}
```



使用 `includeFeatureNameForIterations true`

```groovy
╷
└─ Spock ✔
   └─ ASpec ✔
      └─ really long and informative test name that doesn't have to be repeated ✔
         ├─ really long and informative test name that doesn't have to be repeated [x: 1, y: a, #0] ✔
         ├─ really long and informative test name that doesn't have to be repeated [x: 2, y: b, #1] ✔
         └─ really long and informative test name that doesn't have to be repeated [x: 3, y: c, #2] ✔
```



使用`includeFeatureNameForIterations false`

```groovy
╷
└─ Spock ✔
   └─ ASpec ✔
      └─ really long and informative test name that doesn't have to be repeated ✔
         ├─ x: 1, y: a, #0 ✔
         ├─ x: 2, y: b, #1 ✔
         └─ x: 3, y: c, #2 ✔
```

> 对于单个特性，可以通过使用 `@Unroll('#dataVariablesWithIndex')` 来实现相同的效果。
