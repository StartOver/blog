Title: 【译】Python中如何创建mock？
Date: 2015-06-06 11:10
Category: Python
Tags: Python, mock
Slug: python-mock-introduction
Authors: startover

原文地址：[http://engineroom.trackmaven.com/blog/making-a-mockery-of-python/](http://engineroom.trackmaven.com/blog/making-a-mockery-of-python/)

今天我们来谈论下mock的使用。当然，请不要误会，这里的mock可不是嘲弄的意思。mock是一门技术，通过伪造部分实际代码，从而让我们能够验证剩余代码的正确性。现在我们将通过几个简单的示例演示mock在Python测试代码中的使用，以及这项极其有用的技术是如何帮助我们改善测试代码的。

## 为什么我们需要mock？
当我们进行单元测试的时候，我们的目标往往是为了测试非常小的代码块，例如一个独立存在的函数或类方法。换句话说，我们只需要针对那个函数内部的代码进行测试。如果测试代码依赖于其他的代码片段，即使被测试的函数没有变化，我们会发现在某种不幸的情形下，这部分内嵌代码的修改可能会破坏原有的测试。看看下面的例子，你将豁然开朗：

```python
# function.py
def add_and_multiply(x, y):

    addition = x + y
    multiple = multiply(x, y)

    return (addition, multiple)


def multiply(x, y):

    return x * y

# test.py
import unittest
from function import add_and_multiply


class MyTestCase(unittest.TestCase):
    def test_add_and_multiply(self):

        x = 3
        y = 5

        addition, multiple = add_and_multiply(x, y)

        self.assertEqual(8, addition)
        self.assertEqual(15, multiple)

if __name__ == "__main__":
    unittest.main()
```

```python
$ python test.py
.
----------------------------------------------------------------------
Ran 1 test in 0.001s

OK
```

在上面的例子中，`add_and_multiply`计算两个数的和与乘积并返回。`add_and_multiply`调用了另一个函数`multiply`进行乘积计算。

假设我们想要摒弃“传统“的数学，并重新定义`multiply`函数，在原有的乘积结果上加3。

新的`multiply`函数如下：

```python
def multiply(x, y):

    return x * y + 3
```

现在我们遇到一个问题。我们的测试代码没有变化，我们想要测试的函数也没有变化，然而，`test_add_and_multiply`却会执行失败：

```python
$ python test.py
F
======================================================================
FAIL: test_add_and_multiply (__main__.MyTestCase)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "test.py", line 13, in test_add_and_multiply
    self.assertEqual(15, multiple)
AssertionError: 15 != 18

----------------------------------------------------------------------
Ran 1 test in 0.001s

FAILED (failures=1)
```

这个问题之所以会发生，是因为我们的原始测试代码并非真正的单元测试。尽管我们想要测试的是外部函数，但我们隐性的将内部函数也包含进来，因为我们期望的结果是依赖于这个内部函数的行为的。虽然在上面简单的示例中呈现的差异显得毫无意义，但某些场景下，我们需要测试一个复杂的逻辑代码块 - 例如，一个Django视图函数基于某些特定条件调用各种不同的内部功能，从函数调用结果中分离出视图逻辑的测试就显得尤为重要了。

解决这个问题有两种方案。我们要么忽略它，像集成测试那样去进行单元测试，要么求助于mock。第一种方案的缺点是，集成测试仅仅告诉我们函数调用时哪一行代码出问题了，这样更难找到问题根源所在。这并不是说，集成测试没有用处，因为在某些情况下它确实非常有用。不管怎样，单元测试和集成测试用于解决不同的问题，它们应该被同时使用。因此，如果我们想要成为一个好的测试人员，我们会选择另一种方案：mock。

## mock是什么？

mock是一个极其优秀的Python包，Python 3已将其纳入标准库。对于我们这些还在UnicodeError遍布的Python 2.x中挣扎的苦逼码农，可以通过pip进行安装：

```python
pip install mock==1.0.1
```

mock有多种不同的用法。我们可以用它提供猴子补丁功能，创建伪造的对象，甚至可以作为一个上下文管理器。所有这些都是基于一个共同目标的，用副本替换部分代码来收集信息并返回伪造的响应。

mock的[文档](http://www.voidspace.org.uk/python/mock/)非常密集，寻找特定的用例信息可能会非常棘手。这里，我们就来看看一个常见的场景 - 替换一个内嵌函数并检查它的输入和输出。

## 开始mock之旅

让我们用mock来重新编写单元测试。接下来，我们将讨论发生了什么，以及为什么从测试的角度来看它是非常有用的：

```python
# test.py
import mock
import unittest
from function import add_and_multiply


class MyTestCase(unittest.TestCase):

    @mock.patch('function.multiply')
    def test_add_and_multiply(self, mock_multiply):

        x = 3
        y = 5

        mock_multiply.return_value = 15

        addition, multiple = add_and_multiply(x, y)

        mock_multiply.assert_called_once_with(3, 5)

        self.assertEqual(8, addition)
        self.assertEqual(15, multiple)

if __name__ == "__main__":
    unittest.main()
```

至此，我们可以改变`multiply`函数来做任何我们想做的 - 它可能返回加3后的乘积，返回None，或返回[favourite line from Monty Python and the Holy Grail](https://www.youtube.com/watch?v=q-yxOFIkgxU&t=1m15s) - 你会发现，我们上面的测试仍然可以通过。这是因为我们mock了`multiply`函数。在真正的单元测试场景下，我们并不关心`multiply`函数内部发生了什么，从测试`add_and_multiply`的角度来看，我们只关心`multiply`被正确的参数调用了。这里我们假定有另一个单元测试会针对`multiply`的内部逻辑进行测试。

## 刚才我们做了什么？

咋一看，上面的语法可能不好理解。让我们逐行分析：

```python
@mock.patch('function.multiply')
def test_add_and_multiply(self, mock_multiply):
```

我们使用`mock.patch`装饰器来用mock对象替换`multiply`。然后，我们将它作为一个参数`mock_multiply`插入到我们的测试代码中。在这个测试的上下文中，任何对`multiply`的调用都会被重定向到`mock_multiply`对象。

有人会质疑 - “怎么能用对象替换函数！？“别担心！在Python的世界，函数也是对象。通常情况下，当我们调用`multiply()`，我们实际执行的是`multiply`函数的`__call__`方法。然而，恰当的使用mock，对`multiply()`的调用将执行我们的mock对象而不是`__call__`方法。

```python
mock_multiply.return_value = 15
```

为了使mock函数可以返回任何东西，我们需要定义其`return_value`属性。实际上，当mock函数被调用时，它用于定义mock对象的返回值。

```python
addition, multiple = add_and_multiply(x, y)

mock_multiply.assert_called_once_with(3, 5)
```

在测试代码中，我们调用了外部函数`add_and_multiply`。它会调用内嵌的`multiply`函数，如果我们正确的进行了mock，调用将会被我们定义的mock对象取代。为了验证这一点，我们可以用到mock对象的高级特性 - 当它们被调用时，传给它们的任何参数将被储存起来。顾名思义，mock对象的`assert_called_once_with`方法就是一个不错的捷径来验证某个对象是否被一组特定的参数调用过。如果被调用了，测试通过。反之，`assert_called_once_with`会抛出`AssertionError`的异常。


## 我们从中学到了什么？

好吧，我们遇到了很多实际问题。首先，我们通过mock将`multiply`函数从`add_and_multiply`中分离出来。这就意味着我们的单元测试只针对`add_and_multiply`的内部逻辑。只有针对`add_and_multiply`的代码修改将影响测试的成功与否。

其次，我们现在可以控制内嵌函数的输出，以确保外部函数处理了不同的情况。例如，`add_and_multiply`可能有逻辑条件依赖于`multiply`的返回值：比如说，我们只想在乘积大于10的条件下返回一个值。通过人为设定`multiply`的返回值，我们可以模拟乘积小于10的情况以及乘积大于10的情况，从而可以很容易测试我们的逻辑正确性。

最后，我们现在可以验证被mock的函数被调用的次数，并传入了正确的参数。由于我们的mock对象取代了`multiply`函数的位置，我们知道任何针对`multiply`函数的调用都会被重定向到该mock对象。当测试一个复杂的功能时，确保每一步都被正确调用将是一件非常令人欣慰的事情。
