Result Set Mapping
===================

The process of mapping query results to objects is called result set mapping. The purpose of result set mapping is to map the results from a query to objects for easier subsequent processing.

First, we define the mapper and entity we need for querying:

.. code-block:: xml

   <mapper namespace="main">
       <select id="SelectUser">
           select id, name, age from user
       </select>
   </mapper>

.. code-block:: go

   type User struct {
       Id   int64
       Name string
       Age  int
   }

sql.Rows
--------

``sql.Rows`` is a structure in the database/sql package, which can be defined as the following ``Rows`` interface:

.. code-block:: go

   type Rows interface {
       Columns() ([]string, error)
       Close() error
       Next() bool
       Scan(dest ...interface{}) error
   }

If you are familiar with the database/sql package, then you should know that ``sql.Rows`` is an iterator, with its ``Next`` method used to traverse through the query results, and the ``Scan`` method used to map these results to objects.

We use ``engine`` for querying:

.. code-block:: go

   rows, err := engine.Object("main.SelectUser").Query(nil)
   if err != nil {
       panic(err)
   }
   defer rows.Close()

   for rows.Next() {
       var user User
       err := rows.Scan(&user.Id, &user.Name, &user.Age)
       if err != nil {
           panic(err)
       }
       fmt.Println(user)
   }

   err = rows.Err()
   if err != nil {
       panic(err)
   }

Object
------

``Object`` is used to specify the mapper we want to execute. It accepts an argument of interface{} type, which can be one of the following types:

* ``StatementIDGetter`` type, which has a ``StatementID`` method returning a string, which is the corresponding ID.

.. code-block:: go

   // StatementIDGetter is an interface for getting a statement ID.
   type StatementIDGetter interface {
       // StatementID returns a statement ID.
       StatementID() string
   }

* ``string`` type indicating the full ID for ``action``.

.. code-block:: go

   engine.Object("main.SelectUser")

* Function type; internally, juice will capture the position of this function in the code as the corresponding ID. For example, if the passed function is under the ``main`` package as ``SelectUser`` function, then the ID would be ``main.SelectUser``. If the function is a method of some custom type, then the ID would be ``pkgpath.(interface|struct).methodname`` (package name, type name, method name, distinguish between ``interface`` and ``struct``).

.. attention::

   The ``Object`` mentioned here is the ``Object`` of ``engine``; the function of ``Object`` in the following methods is actually the same, thus not introduced one by one.

Executor
--------

After calling the ``Object`` method, it returns an ``Executor`` object. The definition of ``Executor`` is as follows:

.. code-block:: go

   // Executor is an executor of SQL operations.
   type Executor interface {
       QueryContext(ctx context.Context, param interface{}) (*sql.Rows, error)
       ExecContext(ctx context.Context, param interface{}) (sql.Result, error)
       Statement() Statement
   }

   * ``QueryContext``: Accepts a ``context.Context`` and a parameter to perform a query operation, returning an ``sql.Rows`` object and an ``error``.
   * ``ExecContext``: Accepts a ``context.Context`` and a parameter to perform a non-query operation, returning a ``sql.Result`` object and an ``error``.
   * ``Statement``: Returns the current statement object.

Since we are performing a query operation, we use the ``Query`` method, and since our SQL statement doesn't have parameters, we pass in ``nil``. After obtaining the ``sql.Rows``, we can use the methods of ``sql.Rows`` to traverse through the query results, and finally close the ``sql.Rows``.

This method is similar to the use of the database/sql package, so if you're familiar with the database/sql package, you should find it easy to get started.