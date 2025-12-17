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

       // GenericExecutor is a generic executor.
       type GenericExecutor[T any] interface {
       	// QueryContext executes the query and returns the direct result.
       	// The args are for any placeholder parameters in the query.
       	QueryContext(ctx context.Context, param Param) (T, error)

       	// ExecContext executes a query without returning any rows.
       	// The args are for any placeholder parameters in the query.
       	ExecContext(ctx context.Context, param Param) (sql.Result, error)

       	// Statement returns the xmlSQLStatement of the current executor.
       	Statement() Statement

       	// Session returns the session of the current executor.
       	Session() Session

       	// Driver returns the driver of the current executor.
       	Driver() driver.Driver
       }

       // Executor defines the interface of the executor.
       type Executor GenericExecutor[*sql.Rows]


Since we are performing a query operation, we use the ``QueryContext`` method, and since our SQL statement doesn't have parameters, we pass in ``nil``. After obtaining the ``sql.Rows``, we can use the methods of ``sql.Rows`` to traverse through the query results, and finally close the ``sql.Rows``.

This method is similar to the use of the database/sql package, so if you're familiar with the database/sql package, you should find it easy to get started.

Generic Result Mapping
----------------------

Juice provides powerful generic support to make result set mapping more type-safe:

Mapping Scenarios
^^^^^^^^^^^^^^^^^

1. **Single Field Single Row**:

   .. code-block:: go

       // Query single count
       count, err := juice.NewGenericManager[int](engine).
           Object("CountUsers").QueryContext(context.TODO(), nil)

2. **Multi-Field Single Row**:

   .. code-block:: go

       type User struct {
           ID   int64  `column:"id"`
           Name string `column:"name"`
       }

       // Query single user
       user, err := juice.NewGenericManager[User](engine).
           Object("GetUser").QueryContext(context.TODO(), nil)

3. **Single Field Multi-Row**:

   .. code-block:: go

       // Query multiple IDs
       ids, err := juice.NewGenericManager[[]int64](engine).
           Object("GetUserIDs").QueryContext(context.TODO(), nil)

4. **Multi-Field Multi-Row**:

   .. code-block:: go

       // Query user list
       users, err := juice.NewGenericManager[[]User](engine).
           Object("GetUsers").QueryContext(context.TODO(), nil)

.. attention::
    - Structs must use the ``column`` tag to specify database field mapping.
    - Multi-row results must be received using a slice type.
    - Using map to receive results is not supported (design choice).

Custom Result Mapping
---------------------

Juice provides three core result set mapping functions: ``Bind``, ``List``, and ``List2``, each suitable for different scenarios.

Bind Function
^^^^^^^^^^^^^

``Bind`` is the most flexible mapping function, capable of handling arbitrary types of result set mapping:

.. code-block:: go

    func Bind[T any](rows *sql.Rows) (result T, err error)

Features:
- Supports arbitrary type mapping (structs, slices, basic types, etc.)
- Highest flexibility
- Can handle single row or multi-row data

Usage Example:

.. code-block:: go

    type User struct {
        ID   int    `column:"id"`
        Name string `column:"name"`
    }

    rows, _ := db.Query("SELECT id, name FROM users")
    defer rows.Close()

    // Map to slice
    users, err := Bind[[]User](rows)

    // Map to single struct
    user, err := juice.Bind[User](rows)

List Function
^^^^^^^^^^^^^

``List`` is specifically designed for mapping result sets to slice types:

.. code-block:: go

    func List[T any](rows *sql.Rows) (result []T, err error)

Features:
- Always returns slice type ``[]T``
- Performance superior to ``Bind`` (for slice scenarios)
- Empty result returns empty slice instead of nil
- Special optimization for non-pointer types

Usage Example:

.. code-block:: go

    rows, _ := db.Query("SELECT id, name FROM users")
    defer rows.Close()

    users, err := juice.List[User](rows)

List2 Function
^^^^^^^^^^^^^^

``List2`` is a variant of ``List``, specifically returning a slice of pointers:

.. code-block:: go

    func List2[T any](rows *sql.Rows) ([]*T, error)

Features:
- Returns pointer slice ``[]*T``
- Suitable for scenarios requiring modification of slice elements
- Suitable for handling large structs
- Avoids value copy overhead

Usage Example:

.. code-block:: go

    rows, _ := db.Query("SELECT id, name FROM users")
    defer rows.Close()

    users, err := juice.List2[User](rows)
    // users type is []*User

Iter Function
^^^^^^^^^^^^^

``Iter`` is specifically designed to convert result sets into an iterator, avoiding loading all data into memory at once.

.. code-block:: go

    rows, _ := db.Query("SELECT id, name FROM users")
    defer rows.Close()

    iterator := juice.Iter[User](rows)

    for user := range iterator.Iter() {
        fmt.Println(user)
    }

    if err := iterator.Err(); err != nil {
        fmt.Println(err)
    }

Selection Guide
^^^^^^^^^^^^^^^

1. Use ``Bind`` when:
   - Requiring maximum flexibility
   - Return type is uncertain
   - Handling single row data

2. Use ``List`` when:
   - Certain about returning slice type
   - Pursuing better performance
   - Handling value type slices

3. Use ``List2`` when:
   - Need to modify slice elements
   - Handling large structs
   - Want to avoid value copying

4. Use ``Iter`` when:
   - Need to iterate over a large amount of data

.. note::
    Performance Tips:

    - ``List`` has special optimization for non-pointer types
    - ``List2``, although involving one more conversion, offers another choice in code generation
    - ``Bind`` is the most flexible but may not be the fastest choice
    - ``Iter`` has the best performance when iterating over large amounts of data, but if processing data takes a long time, it will continuously occupy a connection.

Auto-increment Key Mapping
--------------------------

Supports automatically retrieving auto-increment primary key values:

.. code-block:: xml

    <insert id="CreateUser" useGeneratedKeys="true" keyProperty="ID">
        INSERT INTO users (name, age) VALUES (#{name}, #{age})
    </insert>

Usage Conditions:

1. Database driver supports ``LastInsertId``
2. Parameter must be a struct pointer
3. ``useGeneratedKeys="true"``
4. Specify ``keyProperty`` or use ``autoincr:"true"`` tag
5. Primary key field type must support integer assignment

Batch Insert Optimization
-------------------------

Efficient batch data insert support:

.. code-block:: xml

    <insert id="BatchInsertUsers" batchSize="100">
        INSERT INTO users (name, age) VALUES
        <foreach collection="users" item="user" open="(" separator="," close=")">
            (#{user.name}, #{user.age})
        </foreach>
    </insert>

Note: Batch insert parameter type must be slice, array, or a map with exactly one key where the value is a slice or array.

Optimization Features:

1. **Smart Batch Processing**:
   - Automatically batches large amounts of data
   - Configurable batch size (batchSize)
   - Default is single execution

2. **Pre-compilation Optimization**:
   - Pre-compiled statement reuse
   - Generates at most two pre-compiled statements
   - Effectively reduces database pressure

3. **Performance Suggestions**:
   - Recommended batch size: 50-1000
   - Adjust based on data volume and database performance
   - Avoid excessively large batches causing database pressure

.. tip::
    Batch Insert Best Practices:

    1. Set reasonable batch size
    2. Monitor database performance
    3. Consider transaction management
    4. Handle errors properly