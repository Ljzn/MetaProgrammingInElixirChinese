#引用与去引用

  1. 引用(Quoting)
  2. 去引用(Unquoting)
  3. 释放(Escaping)

一个Elixir程序可以用它自己的数据结构来表现.本章,我们将会学习这些结构体的特点和如何组成它们.本章我们要学习的概念是为宏的积木(building blocks),在下一章中我们将深入研究它.

#引用(Quoting)

Elixir程序中的积木是一个三元素元组.例如,函数`sum(1, 2, 3)`的内部表示是:

```
{:sum, [], [1, 2, 3]}
```

你可以用`quote`宏来得到任何表达式的内部表现:

```
iex> quote do: sum(1, 2, 3)
{:sum, [], [1, 2, 3]}
```

第一个元素是函数名,第二个元素是一个包含了元数据的关键词列表,第三个元素是参数列表.

操作符也可以用元组来表示:

```
iex> quote do: 1 + 2
{:+, [context: Elixir, import: Kernel], [1, 2]}
```

甚至一个映射都被表示成对`%{}`的调用:

```
iex> quote do: %{1 => 2}
{:%{}, [], [{1, 2}]}
```

变量也能这样三段式表示,只不过最后的元素换成了一个原子:

```
iex> quote do: x
{:x, [], Elixir}
```

当引用更复杂的表达式时,代码被表示成了一个树状的嵌套元组.许多语言将这种表示称为抽象语法树(AST).ELixir称其为引用表达式:

```
iex> quote do: sum(1, 2 + 3, 4)
{:sum, [], [1, {:+, [context: Elixir, import: Kernel], [2, 3]}, 4]}
```

有时,把引用表达式还原为文本代码会很有用.可以用`Macro.to_string/1`来完成:

```
iex> Macro.to_string(quote do: sum(1, 2 + 3, 4))
"sum(1, 2 + 3, 4)"
```

通常,上述元组的结构会是这种格式:

```
{atom | tuple, list, list | atom}
```

  \- 第一个元素是一个原子,或者是同样表达方式的另一个元组;
  \- 第二个元素是一个关键词列表,包含元数据,比如数字和语境;
  \- 第三个元素是函数调用的参数列表或者是一个原子.当为原子时,意味着元组表示的是一个变量.

除了上面定义的元组,有五个Elixir字面量,在被引用时,会返回它们自己(而不是一个元组).它们是:

```
:sum         #=> Atoms
1.0          #=> Numbers
[1, 2]       #=> Lists
"strings"    #=> Strings
{key, value} #=> Tuples with two elements
```

大多数Elixir代码都有一个直截了当的引用表达式.我们建议你尝试不同的代码,看看结果如何.比如,`String.upcase("foo")`会如何展开?我们已经知道`if(true, do: :this, else: :that)`等同于`if true do :this else :that end`.它要如何用引用表达式来容纳?

#去引用(Unquoting)

引用可以获得一些特定代码块的内部表达式.然而,有时我们需要注入一些特定代码块到我们想要获取的表达式中.

例如,假设我们有一个变量,包含了我们想注入一个引用表达式中的数字,变量名为`number`.

```
iex> number = 13
iex> Macro.to_string(quote do: 11 + number)
"11 + number"
```

那不是我们想要的,`number`变量被引入了表达式,但`number`变量的值没有被注入.为了注入`number`变量的值,我们需要在引用表达式中使用`unquote`:

```
iex> number = 13
iex> Macro.to_string(quote do: 11 + unquote(number))
"11 + 13"
```

`unquote`甚至可以被用于注入函数名:

```
iex> fun = :hello
iex> Macro.to_string(quote do: unquote(fun)(:world))
"hello(:world)"
```

在相同的情况下,可能需要注入一个有许多值的列表.比如,假设你有一个列表`[1, 2, 6]`,我们想把`[3, 4, 5]`注入进去.使用`unquote`却不能得到想要的结果:

```
iex> inner = [3, 4, 5]
iex> Macro.to_string(quote do: [1, 2, unquote(inner), 6])
"[1, 2, [3, 4, 5], 6]"
```

这时就轮到`unquote_splicing`出场了:

```
iex> inner = [3, 4, 5]
iex> Macro.to_string(quote do: [1, 2, unquote_splicing(inner), 6])
"[1, 2, 3, 4, 5, 6]"
```

去引用在操作宏的时候很有用.编写宏的时候,开发者可以获取代码块并将它们注入到其它代码块中,这可以被用于改变代码或者编写能在编译时生成代码的代码.

#释放(Escaping)

如本章开头所见,Elixir中只有一部分值有合法的引用表达式.比如,一个映射没有合法的引用表达式.四元素的元组也没有.然而,这些值可以被表示成一个引用表达式:

```
iex> quote do: %{1 => 2}
{:%{}, [], [{1, 2}]}
```

有时你需要将这种值注入引用表达式中.我们首先需要将这些值释放到引用表达式中,使用`Macro.escape/1`来完成:

```
iex> map = %{hello: :world}
iex> Macro.escape(map)
{:%{}, [], [hello: :world]}
```

宏接收引用表达式,并必须返回引用表达式.然而执行宏的过程中,你可能需要处理一些值,并且区分值与引用表达式.

也就是说,区分常规的Elixir值(例如列表,映射,进程,参考等等)与引用表达式,是很重要的.有一些值的引用表达式就是它们自己,例如整数,原子核字符串.另一些值,比如映射,需要被显式转换.最后,函数与参考完全不能被转化成引用表达式.

你可以在`Kernel.SpecialForms`模块中阅读更多关于`quote`和`unquote`的内容.在`Macro`模块中可以找到`Macro.escape/1`的文档和其它与引用表达式相关的函数.

在本教程中我们将编写自己的第一个宏,让我们进入下一章吧.
