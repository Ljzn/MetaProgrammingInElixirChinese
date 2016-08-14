#宏

  1. 前言
  2. 我们的第一个宏
  3. 宏的隔离
  4. 环境
  5. 私有宏
  6. 负责任地编写宏

#前言

尽管Elixir已竭力为宏提供一个安全的环境,用宏编写干净代码的责任仍然落在了开发者身上.宏比传统的Elixir函数更难编写,而且在不必要的场合使用宏是不好的.所以请负责任地编写宏.

Elixir已经提供了许多数据结构和函数,能够让你以简单可读的风格编写日常代码.宏应当是最后的选择.记住,**明显胜过含蓄.清晰的代码胜过简洁的代码.**

#我们的第一个宏

ELixir中使用`defmacro/2`来定义宏.

> 本章,我们将使用文件来代替在IEx中运行样本代码.这是因为代码样本将跨越许多行,将它们全部输入IEx会适得其反.你应当将代码样本保存进`macro.exs`文件,并使用`elixir macros.exs`或`iex macro.exs`来运行.

为了更好地理解宏是如何运作的,让我们创建一个新的模块,在其中实现`unless`,它的作用与`if`相反.分别以函数和宏的形式:

```
defmodule Unless do
  def fun_unless(clause, expression) do
    if(!clause, do: expression)
  end

  defmacro macro_unless(clause, expression) do
    quote do
      if(!unquote(clause), do: unquote(expression))
    end
  end
end
```

函数接收了参数,并传送给`if`.然而正如我们在前一章所学过的,宏会接收引用表达式,将它们注入引用,最后返回另一个引用表达式.

让我们用`iex`运行上面的模块:

```
$ iex macros.exs
```

调戏一下那些定义:

```
iex> require Unless
iex> Unless.macro_unless true, IO.puts "this should never be printed"
nil
iex> Unless.fun_unless true, IO.puts "this should never be printed"
"this should never be printed"
nil
```

注意,在宏的实现中,句子没有被打印,然而在函数的实现中,句子被打印了.这是因为函数的参数会在调用函数之前被执行.而宏不会执行它们的参数.它们以引用表达式的形式接收参数,之后又将其变形为其它引用表达式.本例中,我们实际上是将`unless`宏重写成了一个`if`.

换句话说,当被这样调用时:

```
Unless.macro_unless true, IO.puts "this should never be printed"
```

我们的`macro_unless`宏接收到了:

```
macro_unless(true, {{:., [], [{:aliases, [], [:IO]}, :puts]}, [], ["this should never be printed"]})
```

然后返回了一个引用表达式:

```Elixir
{:if, [],
 [{:!, [], [true]},
  [do: {{:., [],
     [{:__aliases__,
       [], [:IO]},
      :puts]}, [], ["this should never be printed"]}]]}
```

我们可以使用`Macro.expand_once/2`来验证它:

```
iex> expr = quote do: Unless.macro_unless(true, IO.puts "this should never be printed")
iex> res  = Macro.expand_once(expr, __ENV__)
iex> IO.puts Macro.to_string(res)
if(!true) do
  IO.puts("this should never be printed")
end
:ok
```

`Macro.expand_once/2`接收了引用表达式,并根据当前环境扩展了它.本例中,它扩展/调用了`Unless.macro_unless/2`宏,并返回了结果.之后我们将返回的引用表达式转换成一个字符串并打印出来(我们将在本章稍后的位置讨论`__ENV__`).

这就是宏.它们接收引用表达式并将其变形为别的东西.事实上,Elixir中的`unless/2`是作为宏来实现的:

```
defmacro unless(clause, options) do
  quote do
    if(!unquote(clause), do: unquote(options))
  end
end
```

本教程中用到的许多纯Elixir实现的结构都是宏,例如`unless/2`,`defmacro/2`,`def/2`,`defprotocol/2`等等.这意味着,开发者可以用构建语言的结构来将语言扩展到它们工作的领域.

我们可以定义任何函数和宏,甚至覆盖Elixir中的原本定义.唯一的例外是Elixir特殊形式,它们不是由Elixir实现的,因此不能被覆盖,特殊形式的完整列表可以在`Kernel.SpecialForms`中找到.

#宏的隔离(Macros hygiene)

Elixir的宏有着低决定权.这保证了引用中的变量定义不会与宏被扩展到的语境中的变量定义相冲突.例如:

```
defmodule Hygiene do
  defmacro no_interference do
    quote do: a = 1
  end
end

defmodule HygieneTest do
  def go do
    require Hygiene
    a = 13
    Hygiene.no_interference
    a
  end
end

HygieneTest.go
# => 13
```

上述例子中,即使宏注入了`a = 1`,却没有影响到变量`a`在函数`go`中的定义.如果宏想要明确地影响语境,可以使用`var!`:

```
defmodule Hygiene do
  defmacro interference do
    quote do: var!(a) = 1
  end
end

defmodule HygieneTest do
  def go do
    require Hygiene
    a = 13
    Hygiene.interference
    a
  end
end

HygieneTest.go
# => 1
```

因为Elixir使用变量的语境来注解它,所以能够实现变量隔离.例如,一个模块的第三行定义的变量`x`可以被表示成:

```
{:x, [line: 3], nil}
```

然而一个引用变量是这样表示的:

```
defmodule Sample do
  def quoted do
    quote do: x
  end
end

Sample.quoted #=> {:x, [line: 3], Sample}
```

注意引用变量的第三个元素是原子`Sample`,而不是`nil`,它标记了变量是来自`Sample`模块的.因此,Elixir认为这两个变量来自不同语境,会分别处理它们.

Elixir也为进口(imports)和别名(aliases)提供了相似的机制.这保证了宏的行为会与它源模块中的定义相同,而不是与宏所扩展到的目标模块相冲突.使用类似`var!/2`和`alias!/2`之类的宏可以突破隔离,但是它们必须小心使用,因为这直接改变了用户环境.

有时,变量名会被动态地创建.`Macro.var/2`可用于定义新变量:

```
defmodule Sample do
  defmacro initialize_to_char_count(variables) do
    Enum.map variables, fn(name) ->
      var = Macro.var(name, nil)
      length = name |> Atom.to_string |> String.length
      quote do
        unquote(var) = unquote(length)
      end
    end
  end

  def run do
    initialize_to_char_count [:red, :green, :yellow]
    [red, green, yellow]
  end
end

> Sample.run #=> [3, 5, 6]
```

注意`Macro.var/2`的第二个变量.在下一节中我们将知道它是所使用的语境,而且能定义隔离.

#环境

本章早些时候,我们调用`Macro.expand_once/2`时,使用了特殊形式`__ENV__`.

`__ENV__`返回了一个`Macro.Env`结构的实例,它包含了编译环境的有用信息,包括当前模块,文件和行,所有定义在当前作用域中的变量,还有imports,requires等等.

```
iex> __ENV__.module
nil
iex> __ENV__.file
"iex"
iex> __ENV__.requires
[IEx.Helpers, Kernel, Kernel.Typespec]
iex> require Integer
nil
iex> __ENV__.requires
[IEx.Helpers, Integer, Kernel, Kernel.Typespec]
```

`Macro`模块中的许多函数都期望一个环境.你可以在`Macro`模块中找到关于这些函数,以及在`Macro.Env`的文档中找到关于编译环境的更多信息.

#私有宏

Elixir也支持私有宏,使用`defmacrop`来定义.和私有函数一样,这些宏只能在它的定义模块中使用,而且只在编译时.

很重要的一点是,宏在使用之前定义.没有在调用一个宏之前定义它,将会在运行时抛出一个错误,因为宏不会被扩展,而且将会被转化成函数调用:

```
iex> defmodule Sample do
...>  def four, do: two + two
...>  defmacrop two, do: 2
...> end
** (CompileError) iex:2: function two/0 undefined
```

#负责任地编写宏

宏是很强大的结构,Elixir提供了许多机制来确保它们被负责任地使用.

  \- 宏是隔离的: 定义在宏内的变量默认是不会影响用户代码的.而且,宏语境中的函数调用和别名是不会泄露到用户语境中的.

  \- 宏具有词典性质: 不可能全局地注入代码或宏.为了使用宏,你需要明确地`require`或`import`定义了宏的模块.

  \- 宏是明确的: 宏不可能在没有明确被导入的情况下运行.例如,一些语言允许开发者在内部完全重写函数,通常是通过语义转换或一些反射机制.在Elixir中,编译时,宏必须在调用者中被明确导入.

  \- 宏的语言是清晰的: 许多语言为`quote`和`unquote`提供了语法捷径.在Elixir中,我们更愿意它们被明确地拼写出来,以便清楚地划出宏定义与它的引用表达式间的界限.

即使有这些保障,开发者仍在负责任地编写宏这件事中扮演重要角色.如果你确信你需要使用宏,记住宏不是你的API.你的宏定义要保持简短,包括它们的引用内容.例如,与其像这样编写宏:

```
defmodule MyModule do
  defmacro my_macro(a, b, c) do
    quote do
      do_this(unquote(a))
      ...
      do_that(unquote(b))
      ...
      and_that(unquote(c))
    end
  end
end
```

不如这样:

```
defmodule MyModule do
  defmacro my_macro(a, b, c) do
    quote do
      # Keep what you need to do here to a minimum
      # and move everything else to a function
      do_this_that_and_that(unquote(a), unquote(b), unquote(c))
    end
  end

  def do_this_that_and_that(a, b, c) do
    do_this(a)
    ...
    do_that(b)
    ...
    and_that(c)
  end
end
```

这使得你的代码更清晰,也更容易测试和维护,因为你可以直接调用和测试`do_this_that_and_that/3`.这也有助于你为那些不愿意依赖宏的开发者来设计一个实际的API.

现在,我们结束了对宏的介绍.下一章我们将简短得讨论DSL,展示如何混合宏和模块属性,来注释和扩展模块与函数.
