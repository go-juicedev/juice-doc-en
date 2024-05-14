表达式
========

从上面的动态sql里面我们可以看出 juice 支持在 ``if`` 和 ``when`` 标签中使用表达式，这些表达式的语法和 go 语言中的语法基本一致。

你可以在表达式里面书写大部分的 go 语言表达式，但是有一些特殊的地方需要注意：

1、 juice 只支持单行表达式，不支持多行表达式。

2、当在表达式里面写一个字符串的时候，需要将test属性的的值用单引号包裹起来，例如：

   .. code:: xml

        <if test='name == "eatmoreapple"'>

        </if>

3、 尽量写简单一点的表达式。

表达式
-------

比较运算符
~~~~~~~~~~

juice 支持以下比较运算符：

1. ``==``: 判断两个值是否相等。

2. ``!=``: 判断两个值是否不相等。

3. ``>``: 判断左边的值是否大于右边的值。

4. ``>=``: 判断左边的值是否大于等于右边的值。

.. attention::

    ``<`` 和 ``<=`` 不支持，因为它们会跟xml的语法冲突。那么怎么办呢？我们可以使用 ``&lt;`` 和 ``&lt;=`` 来代替 ``<`` 和 ``<=``。

逻辑运算符
~~~~~~~~~~

juice 支持以下逻辑运算符：

1. ``and``: 逻辑与。

2. ``or``: 逻辑或。

3. ``!``: 逻辑非。

算术运算符
~~~~~~~~~~

juice 支持以下算术运算符：

1. ``+``: 加法运算。

2. ``-``: 减法运算。

3. ``*``: 乘法运算。

4. ``/``: 除法运算。

5. ``%``: 取余运算。


保留关键字
~~~~~~~~~~

保留关键字是指 juice 里面已经定义好的关键字，不能用作变量名。

1. ``true``: 布尔类型的真。

2. ``false``: 布尔类型的假。

3. ``nil``: 空值。

4. ``and``: 逻辑与。

5. ``or``: 逻辑或。


函数
-------

为了丰富表达式的功能，juice 支持在表达式里面调用函数。

juice 内置了一些函数，你可以在表达式里面使用。

1. ``length``: 返回字符串的长度，或者数组、切片、map的长度。

    .. code::

       func length[T slice|map|string](T) int

2. ``substr``: 返回字符串的子串，第一个参数是字符串，第二个参数是开始位置，第三个参数是结束位置。

    .. code::

       func substr(string, int, int) string


3. ``join``: 将字符串数组或者字符串切片转换成字符串，第一个参数是数组或者切片，第二个参数是分隔符。

    .. code::

       func join[T slice](T, string) string

4. ``contains``: 判断字符串、切片、map中是否包含制定元素

    .. code::

       func contains[T slice|map|string](T, interface{}) bool

5. ``slice``: 将数组或者切片进行切片，第一个参数是数组或者切片，第二个参数是开始位置，第三个参数是结束位置。

    .. code::

       func slice[T slice](T, int, int) T

6. ``title``: 将字符串的首字母大写。

    .. code::

       func title(string) string

7. ``lower``: 将字符串转换成小写。

     .. code::

        func lower(string) string

8. ``upper``: 将字符串转换成大写。

      .. code::

        func upper(string) string

自定义全局函数注册
----------------

当我觉得内置的函数不够用的时候，我们注册自定义函数，自定义函数需要满足以下条件：

1、 必须是一个函数（emm，这个很好理解）。

2、必须有两个返回值，第一个返回值是任意类型，第二个必须为error类型。

ok，当满足上述两个条件之后，我们就可以往juice里面注册自定义函数了。

.. code:: go

    func add(x, y int) (int, error) {
        return x + y, nil
    }

    func main() {
        if err := juice.RegisterEvalFunc("add", add); err != nil {
            panic(err)
        }
    }

在上面的代码中，我们定义了一个 ``add`` 函数，然后我们调用 ``RegisterEvalFunc`` 方法将这个函数注册到 juice 里面。

在 juice 里面，我们可以这样使用这个函数：

.. code:: xml

    <if test='add(1, 2) == 3'>

    </if>

传入函数调用
------------

juice 支持在参数传递时，传入函数调用，例如：

.. code-block:: xml

    <if test='MyFunc() == "eatmoreapple"'>

    </if>

.. code-block:: go

    func MyFunc() (string, error) {
        return "eatmoreapple", nil
    }

    param := juice.H{
        "MyFunc": MyFunc,
    }


* 带有参数的函数

.. code-block:: xml

    <if test='MyFunc("eatmoreapple") == "eatmoreapple"'>

    </if>

.. code-block:: go

    func MyFunc(str string) (string, error) {
        return str, nil
    }

    param := juice.H{
        "MyFunc": MyFunc,
    }

* 传递函数的参数的参数（怎么有点拗口？）

.. code-block:: xml

    <if test='MyFunc(eatmoreapple) == "eatmoreapple"'>

    </if>


.. code-block:: go

    func MyFunc(str string) (string, error) {
        return str, nil
    }

    param := juice.H{
        "MyFunc": MyFunc,
        "eatmoreapple": "eatmoreapple",
    }

* 多个参数

.. code-block:: xml

    <if test='MyFunc(eatmoreapple, 1, 2) == "eatmoreapple"'>

    </if>

.. code-block:: go

    func MyFunc(str string, x, y int) (string, error) {
        return str, nil
    }

    param := juice.H{
        "MyFunc": MyFunc,
        "eatmoreapple": "eatmoreapple",
    }


自定义类型方法调用
----------------

juice 支持在参数传递时，传入自定义类型的方法调用，例如：

.. code-block:: xml

    <if test='a.MyFunc() == "eatmoreapple"'>

    </if>

.. code-block:: go

    type A struct {
        Name string
    }

    func (a *A) MyFunc() (string, error) {
        return a.Name, nil
    }

    param := juice.H{
        "a": &A{Name: "eatmoreapple"},
    }



.. attention::
    传入函数调用时，函数的返回值必须是两个，第一个返回值是任意类型，第二个返回值必须是error类型。

属性调用
--------

juice 支持在参数传递时，传入自定义类型的属性调用，例如：

.. code-block:: xml

    <if test='a.Name == "eatmoreapple"'>

    </if>

.. code-block:: go

    type A struct {
        Name string
    }

    param := juice.H{
        "a": &A{Name: "eatmoreapple"},
    }

map索引取值
------------

.. code-block:: go

    param := juice.H{
        "a": juice.H{
            "Name": "eatmoreapple",
        },
    }

.. code-block:: xml

    <if test='a["Name"] == "eatmoreapple"'>

    </if>

上面的xml可以写成下面的这种形式：

.. code-block:: xml

     <if test='a.Name == "eatmoreapple"'>

     </if>

这两种写法有啥区别呢？

索引取值，当对应的key不存在时，会返回map值的默认值。

但是 ``a.Name`` 这种形式，当 ``Name`` 不存在时，会抛出异常。

以 ``a.Name`` 这种形式写，可以支持方法调用，例如：

.. code-block:: go

    type A map[string]string

    func (a *A) MyFunc() (string, error) {
        return "eatmoreapple", nil
    }

    param := juice.H{
        "a": A{"hello": "world"},
    }

.. code-block:: xml

    <if test='a.MyFunc() == "eatmoreapple"'>

    </if>

    <if test="a.hello == 'world'">

    </if>




数组索引取值
-------------

.. code-block:: xml

    <if test='a[0] == "eatmoreapple"'>

    </if>

.. code-block:: go

    param := juice.H{
        "a": []string{"eatmoreapple"},
    }