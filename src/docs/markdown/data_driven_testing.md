# Data Driven Testing

通常情况下，通过多次运行相同的测试代码，使用不同的输入和预期结果是很有用的。Spock的数据驱动测试支持使得这成为一项一流的特性。

## 简介

假设我们想要指定`Math.max`方法的行为：



```groovy
class MathSpec extends Specification {
  def "maximum of two numbers"() {
    expect:
    // exercise math method for a few different inputs
    Math.max(1, 3) == 3
    Math.max(7, 4) == 7
    Math.max(0, 0) == 0
  }
}
```

虽然这种方法在像这样简单的情况下是可以的，但它也有一些潜在的缺点：

- 代码和数据混合在一起，难以独立地进行更改；

- 数据难以自动生成或从外部源获取；

- 为了多次执行相同的代码，要么需要复制代码，要么将其提取到单独的方法中；

- 如果出现失败，可能不会立即清楚是哪些输入导致了失败；

- 多次执行相同的代码不像执行单独的方法那样具有相同的隔离性。

Spock的数据驱动测试支持旨在解决这些问题。为了开始使用数据驱动测试，让我们将上述代码重构为一个数据驱动的特性方法。首先，我们引入三个方法参数（称为数据变量），用来替换硬编码的整数值：



```groovy
class MathSpec extends Specification {
  def "maximum of two numbers"(int a, int b, int c) {
    expect:
    Math.max(a, b) == c
    ...
  }
}
```

我们已经完成了测试逻辑，但仍需要提供要使用的数据值。这是通过`where:`块来实现的，它始终位于方法的末尾。在最简单（也是最常见）的情况下，`where: `块包含一个数据表。



## 数据表（Data Tables）

数据表是使用一组固定数据值来执行特性方法的便捷方式：

```groovy
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
```

数据表的第一行称为__表头__（table header），声明了数据变量。随后的行称为__表行__（table rows），保存了相应的值。对于每一行，特性方法将被执行一次；我们称之为__方法的迭代__（iteration of the method）。如果一个迭代失败，剩余的迭代仍然会执行。所有的失败都将被报告。

数据表必须至少有两列。一个单列的数据表可以写成：

```groovy
where:
a | _
1 | _
7 | _
0 | _
```

可以使用两个或更多下划线的序列将一个宽数据表分成多个较窄的数据表。如果没有这个分隔符，也没有其他数据变量的赋值在中间，就无法在一个`where`块中有多个数据表，第二个表将只是第一个表的进一步迭代，包括看似是表头的行：

```groovy
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
```

从语义上讲，这完全相同，只是一个更宽的组合数据表而已：

```groovy
where:
a | b | c
1 | 1 | 2
7 | 3 | 4
0 | 5 | 6
```

两个或更多下划线的序列可以在`where`块的任何位置使用。它将被忽略，除非在两个数据表之间使用它来分隔这两个数据表。这意味着分隔符也可以作为样式元素以不同的方式使用。它可以像最后一个示例中所示，用作分隔线，或者也用作表格的顶部边框，从视觉上起到分隔它们的效果：

```groovy
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
```

