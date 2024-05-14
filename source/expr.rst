Expressions
===========

From the dynamic SQL examples above, we can see that Juice supports the use of expressions within the ``if`` and ``when`` tags. The syntax for these expressions aligns mostly with that of Go language. You can write most of the expressions that you would in Go, but there are a few special considerations:

1. Juice only supports single-line expressions, not multi-line expressions.
2. When writing a string within an expression, you need to enclose the value of the test attribute in single quotes, for example:

.. code:: xml

   <if test='name == "eatmoreapple"'>
   </if>

3. Try to write simpler expressions.

Expressions
-----------

Comparison Operators
~~~~~~~~~~~~~~~~~~~~

Juice supports the following comparison operators:

1. ``==``: Checks if two values are equal.
2. ``!=``: Checks if two values are not equal.
3. ``>``: Checks if the left value is greater than the right.
4. ``>=``: Checks if the left value is greater than or equal to the right.

.. attention:: ``<`` and ``<=`` are not supported as they clash with XML syntax. Instead, ``&lt;`` and ``&lt;=`` can be used as substitutes.

Logical Operators
~~~~~~~~~~~~~~~~~

Juice supports the following logical operators:

1. ``and``: Logical AND.
2. ``or``: Logical OR.
3. ``!``: Logical NOT.

Arithmetic Operators
~~~~~~~~~~~~~~~~~~~~

Juice supports the following arithmetic operators:

1. ``+``: Addition.
2. ``-``: Subtraction.
3. ``*``: Multiplication.
4. ``/``: Division.
5. ``%``: Modulus.

Reserved Keywords
~~~~~~~~~~~~~~~~~

Reserved keywords in Juice are predefined and cannot be used as variable names:

1. ``true``: Boolean true.
2. ``false``: Boolean false.
3. ``nil``: Represents a null value.
4. ``and``: Logical AND.
5. ``or``: Logical OR.

Functions
---------

To enrich the functionality of expressions, Juice supports the invocation of functions within expressions. Juice has built-in functions that you can use within your expressions:

1. **length**: Returns the length of a string, array, slice, or map.

.. code:: func

   length[T slice|map|string](T) int

2. **substr**: Returns a substring, with the first parameter as the string, the second as the start position, and the third as the end position.

.. code:: func

   substr(string, int, int) string

3. **join**: Converts an array or slice of strings into a single string, with the first parameter being the array or slice and the second being the delimiter.

.. code:: func

   join[T slice](T, string) string

4. **contains**: Checks if a string, slice, or map contains a specified element.

.. code:: func

   contains[T slice|map|string](T, interface{}) bool

5. **slice**: Slices an array or a slice, with the first parameter as the array or slice, the second as the start position, and the third as the end position.

.. code:: func

   slice[T slice](T, int, int) T

6. **title**: Capitalizes the first letter of a string.

.. code:: func

   title(string) string

7. **lower**: Converts a string to lowercase.

.. code:: func

   lower(string) string

8. **upper**: Converts a string to uppercase.

.. code:: func

   upper(string) string

Custom Global Function Registration
-----------------------------------

When the built-in functions are insufficient, you can register custom functions, which must meet the following criteria:

1. It must be a function.
2. It must have two return values, the first of any type and the second of error type.

When these conditions are met, you can register the function in Juice:

.. code:: go

   func add(x, y int) (int, error) {
       return x + y, nil
   }

   func main() {
       if err := juice.RegisterEvalFunc("add", add); err != nil {
           panic(err)
       }
   }

In Juice, the function can be used like this:

.. code:: xml

   <if test='add(1, 2) == 3'>
   </if>

Passing Function Calls
----------------------

Juice supports passing function calls as parameters, for example:

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

* With parameters function:

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

* Passing function parameters:

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

* Multiple parameters:

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

Custom Type Method Calls
------------------------

Juice supports the invocation of custom type methods when passing parameters:

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

.. attention:: When passing function calls, the function must return two values, the first of any type and the second of error type.

Property Access
---------------

Juice supports accessing properties of custom types when passing parameters:

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

Map Index Access
----------------

.. code-block:: go

   param := juice.H{
       "a": juice.H{
           "Name": "eatmoreapple",
       },
   }

.. code-block:: xml

   <if test='a["Name"] == "eatmoreapple"'>
   </if>

This XML can also be written as follows:

.. code-block:: xml

   <if test='a.Name == "eatmoreapple"'>
   </if>

What is the difference between these two methods?

Index access returns the default value of the map when the corresponding key does not exist, while the `a.Name` format will throw an exception if `Name` does not exist. Using the `a.Name` format supports method calls, for example:

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

Array Index Access
------------------

.. code-block:: xml

   <if test='a[0] == "eatmoreapple"'>
   </if>

.. code-block:: go

   param := juice.H{
       "a": []string{"eatmoreapple"},
   }