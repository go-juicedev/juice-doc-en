Expression System
=================

Overview
--------

As you can see from the dynamic SQL examples, Juice supports expressions inside ``if`` and ``when`` tags. The syntax is broadly similar to Go expressions.

Usage Restrictions
------------------

1. Juice supports only single-line expressions.

2. When writing a string inside an expression, wrap the ``test`` attribute value in single quotes:

   .. code-block:: xml

        <if test='name == "eatmoreapple"'>
        </if>

3. Keep expressions as simple as possible.

Basic Operators
---------------

Comparison Operators
~~~~~~~~~~~~~~~~~~~~

Juice supports these comparison operators:

1. ``==``: equal to
2. ``!=``: not equal to
3. ``>``: greater than
4. ``>=``: greater than or equal to

.. attention::

    ``<`` and ``<=`` conflict with XML syntax. Use ``&lt;`` and ``&lt;=`` instead.

Logical Operators
~~~~~~~~~~~~~~~~~

Juice supports these logical operators:

1. ``and``: logical AND
2. ``or``: logical OR
3. ``!``: logical NOT

Arithmetic Operators
~~~~~~~~~~~~~~~~~~~~

Juice supports these arithmetic operators:

1. ``+``: addition
2. ``-``: subtraction
3. ``*``: multiplication
4. ``/``: division
5. ``%``: remainder

Reserved Keywords
~~~~~~~~~~~~~~~~~

The following keywords are reserved and cannot be used as variable names:

1. ``true``
2. ``false``
3. ``nil``
4. ``and``
5. ``or``

Built-In Functions
------------------

Juice includes several built-in functions:

1. ``length``: returns the length of a string, array, slice, or map

   .. code-block::

      func length[T slice|map|string](T) int

2. ``substr``: returns a substring

   .. code-block::

      func substr(string, int, int) string

3. ``join``: joins a string array or slice with a separator

   .. code-block::

      func join[T slice](T, string) string

4. ``contains``: checks whether a value contains a given element

   .. code-block::

      func contains[T slice|map|string](T, interface{}) bool

5. ``slice``: performs slicing

   .. code-block::

      func slice[T slice](T, int, int) T

6. ``title``: capitalizes the first letter

   .. code-block::

      func title(string) string

7. ``lower``: converts to lowercase

   .. code-block::

      func lower(string) string

8. ``upper``: converts to uppercase

   .. code-block::

      func upper(string) string

Custom Functions
----------------

Registration requirements:

1. It must be a function.
2. It must return two values. The first can be any type and the second must be ``error``.

Example:

.. code-block:: go

    func add(x, y int) (int, error) {
        return x + y, nil
    }

    func main() {
        if err := juice.RegisterEvalFunc("add", add); err != nil {
            panic(err)
        }
    }

Usage:

.. code-block:: xml

    <if test='add(1, 2) == 3'>
    </if>

Function Calls
--------------

Basic function call:

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

Function with parameters:

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

Referencing parameters:

.. code-block:: xml

    <if test='MyFunc(eatmoreapple) == "eatmoreapple"'>
    </if>

.. code-block:: go

    param := juice.H{
        "MyFunc": MyFunc,
        "eatmoreapple": "eatmoreapple",
    }

Multiple parameters:

.. code-block:: xml

    <if test='MyFunc(eatmoreapple, 1, 2) == "eatmoreapple"'>
    </if>

.. code-block:: go

    func MyFunc(str string, x, y int) (string, error) {
        return str, nil
    }

Methods on Custom Types
-----------------------

Method calls:

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

Attribute Access
----------------

Struct fields:

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

Map Operations
--------------

Indexed access:

.. code-block:: go

    param := juice.H{
        "a": juice.H{
            "Name": "eatmoreapple",
        },
    }

.. code-block:: xml

    <if test='a["Name"] == "eatmoreapple"'>
    </if>

    <if test='a.Name == "eatmoreapple"'>
    </if>

.. attention::

    Difference between the two forms:

    1. Indexed access such as ``a["Name"]`` returns a default value when the key is missing.
    2. Dot access such as ``a.Name`` throws an error when the property is missing.
    3. Dot access also supports method calls.

Array Operations
----------------

.. code-block:: xml

    <if test='a[0] == "eatmoreapple"'>
    </if>

.. code-block:: go

    param := juice.H{
        "a": []string{"eatmoreapple"},
    }

.. tip::

    Best practices:

    1. Keep expressions simple and readable.
    2. Move complex logic into Go code when possible.
    3. Use built-in functions appropriately.
    4. Pay attention to XML escaping rules.
    5. Test every condition branch.
