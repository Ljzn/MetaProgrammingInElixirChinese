#领域特定语言

  1. 前言
  2. 构建我们的测试案例
  3. `test`宏
  4. 用属性存储信息

#前言

领域特定语言(DSL)允许开发者修改他们的应用以适应特定的领域.制作DSL可以不需要用到宏:你在你的模块中定义的每个数据结构和每个函数都是你的领域特定语言的一部分.

比如,假设你想要实现一个提供了数据验证领域特定语言的验证模块.你可以使用数据结构,函数或宏来实现它.让我们来看看这些不同的DSL:

```
# 1. data structures
import Validator
validate user, name: [length: 1..100],
               email: [matches: ~r/@/]

# 2. functions
import Validator
user
|> validate_length(:name, 1..100)
|> validate_matches(:email, ~r/@/)

# 3. macros + modules
defmodule MyValidator do
  use Validator
  validate_length :name, 1..100
  validate_matches :email, ~r/@/
end

MyValidator.validate(user)
```

上述方法中,第一个是最有弹性的.如果我们的领域规则可以使用数据结构来编码,它们组成和实现起来会非常简单,因为Elixir的标准库中有很多用于操作不同数据类型的函数.

第二个方法使用函数调用,它能更好地适应更复杂的API(例如,需要传递很多选项)而且归功于Elixir中的管道操作符,它更易读.

第三个方法,使用宏,无疑是最复杂的.需要更多行的代码来实现,测试它是很难且很昂贵的(与测试简单的函数相比),而且它也限制了用户使用库的方法,因为所有验证需要在一个模块中定义.

为了更好地理解,假设你想要只在某种情况下验证某个特定属性.我们可以简单地使用第一个方法操作相应的数据结构来实现,或者使用第二个方法在调用函数前使用条件语句(if/else).然而不可能使用宏来完成,除非该DSL已经增强过了.

也就是说:

```
data > functions > macros
```

仍然有一些时候,由宏和模块来构建领域特定语言是很管用的.因为我们已经在入门教程中探索过了数据结构和函数定义,本章将会探索如何使用宏和模块属性来处理更复杂的DSL.

#构建我们自己的测试案例

本章的目标是构建一个名为`TestCase`的模块,它允许我们这样编写:

```
defmodule MyTest do
  use TestCase

  test "arithmetic operations" do
    4 = 2 + 2
  end

  test "list operations" do
    [1, 2, 3] = [1, 2] ++ [3]
  end
end

MyTest.run
```

在上面的例子中,使用`TestCase`,我们可以用`test`宏来编写测试,它定义了一个名为`run`的函数来自动为我们运行所有测试.我们的原型将简单地依赖匹配操作符(`=`)作为一个断言机制.

#`test`宏

让我们创建一个模块,它在被使用时简单地定义并进口了`test`宏:

```
defmodule TestCase do
  # Callback invoked by `use`.
  #
  # For now it simply returns a quoted expression that
  # imports the module itself into the user code.
  @doc false
  defmacro __using__(_opts) do
    quote do
      import TestCase
    end
  end

  @doc """
  Defines a test case with the given description.

  ## Examples

      test "arithmetic operations" do
        4 = 2 + 2
      end

  """
  defmacro test(description, do: block) do
    function_name = String.to_atom("test " <> description)
    quote do
      def unquote(function_name)(), do: unquote(block)
    end
  end
end
```

假设我们在一个名为`tests.exs`的文件中定义了`TestCase`,我们可以通过运行`iex tests.exs`打开它,并定义我们的第一个测试:

```
iex> defmodule MyTest do
...>   use TestCase
...>
...>   test "hello" do
...>     "hello" = "world"
...>   end
...> end
```

现在我们还没有一个运行测试的机制,但是我们知道有一个名为`test hello`的函数已经在幕后被定义了.当调用它时,他应该会失败:

```
iex> MyTest."test hello"()
** (MatchError) no match of right hand side value: "world"
```

#使用属性存储信息

为了完整实现我们的`TestCase`,我们需要能够访问所有已经定义的测试案例.我们可以通过在运行时使用`__MODULE__.__info__(:functions)`来检索测试,它会返回一个包含给定模块中所有函数的列表.然而,考虑到我们可能需要存储更多关于每个测试的信息,这就要求要有一个更灵活地方法.

当在前几章中讨论模块属性时,我们提到了它们是如何被用作临时存储的.那就是我们在本节中将应用的特性.

在`__using__/1`的实现中,我们将会把一个名为`@tests`的模块属性初始化成一个空列表,然后存储每个已定义测试的名字到该属性中,这样测试就可以从`run`函数调用.

这是`TestCase`模块更新后的代码:

```
defmodule TestCase do
  @doc false
  defmacro __using__(_opts) do
    quote do
      import TestCase

      # Initialize @tests to an empty list
      @tests []

      # Invoke TestCase.__before_compile__/1 before the module is compiled
      @before_compile TestCase
    end
  end

  @doc """
  Defines a test case with the given description.

  ## Examples

      test "arithmetic operations" do
        4 = 2 + 2
      end

  """
  defmacro test(description, do: block) do
    function_name = String.to_atom("test " <> description)
    quote do
      # Prepend the newly defined test to the list of tests
      @tests [unquote(function_name) | @tests]
      def unquote(function_name)(), do: unquote(block)
    end
  end

  # This will be invoked right before the target module is compiled
  # giving us the perfect opportunity to inject the `run/0` function
  @doc false
  defmacro __before_compile__(env) do
    quote do
      def run do
        Enum.each @tests, fn name ->
          IO.puts "Running #{name}"
          apply(__MODULE__, name, [])
        end
      end
    end
  end
end
```

通过启动一个新的IEx会话,我们现在可以定义并运行我们的测试:

```
iex> defmodule MyTest do
...>   use TestCase
...>
...>   test "hello" do
...>     "hello" = "world"
...>   end
...> end
iex> MyTest.run
Running test hello
** (MatchError) no match of right hand side value: "world"
```

尽管我们跳过了一些细节,但这就是在Elixir中创建领域特定模块的主要思想.宏使得我们能够返回在调用者中执行了的引用表达式,我们可以用它来变形代码并通过模块属性来在目标模块中存储相关信息.最后,`@before_compile`这样的回调能让我们将代码注入到已定义完成的模块中.

除了`@before_compile`,还有其它有用的模块属性,例如`@on_definition`和`@after_compile`,你可以在`Module`模块的文档中获取有关它们的信息.你也可以在`Macro`模块和`Macro.Env`的文档中找到关于宏和编译环境的有用信息.
