Result Mapping
==============

Result mapping is a core feature of Juice. It converts database query results into structured Go values. Juice supports multiple mapping styles, from simple single-row cases to more advanced collection-oriented patterns.

Native sql.Rows Support
-----------------------

Juice is fully compatible with the ``Rows`` interface from ``database/sql``:

.. code-block:: go

    type Rows interface {
        Columns() ([]string, error)
        Close() error
        Next() bool
        Scan(dest ...interface{}) error
    }

Basic example:

.. code-block:: go

    rows, err := engine.Object("main.SelectUser").QueryContext(context.TODO(), nil)
    if err != nil {
        panic(err)
    }
    defer rows.Close()

    for rows.Next() {
        var user User
        if err := rows.Scan(&user.Id, &user.Name, &user.Age); err != nil {
            panic(err)
        }
        fmt.Println(user)
    }

    if err = rows.Err(); err != nil {
        panic(err)
    }

About Object
""""""""""""

``Object`` supports multiple kinds of identifiers:

1. **String**

   .. code-block:: go

       engine.Object("main.SelectUser")

2. **Function**

   Use the function's location in code as the statement ID.

.. note::

    ``Object`` ID generation rules:

    - ``package.FunctionName`` for plain functions
    - ``package.TypeName.MethodName`` for struct methods
    - Keep interface methods and struct methods distinct

Generic Result Mapping
----------------------

Juice provides strong support for generics, making result mapping more type-safe.

Mapping Scenarios
"""""""""""""""""

1. **Single column, single row**

   .. code-block:: go

       count, err := juice.NewGenericManager[int](engine).
           Object("CountUsers").QueryContext(context.TODO(), nil)

2. **Multiple columns, single row**

   .. code-block:: go

       type User struct {
           ID   int64  `column:"id"`
           Name string `column:"name"`
       }

       user, err := juice.NewGenericManager[User](engine).
           Object("GetUser").QueryContext(context.TODO(), nil)

3. **Single column, multiple rows**

   .. code-block:: go

       ids, err := juice.NewGenericManager[[]int64](engine).
           Object("GetUserIDs").QueryContext(context.TODO(), nil)

4. **Multiple columns, multiple rows**

   .. code-block:: go

       users, err := juice.NewGenericManager[[]User](engine).
           Object("GetUsers").QueryContext(context.TODO(), nil)

.. attention::

    - Structs should use the ``column`` tag to map database columns.
    - Multi-row results should be received with slice types.
    - Mapping directly into ``map`` is not supported by design.

Custom Result Mapping
---------------------

Juice provides four main result-mapping helpers: ``Bind``, ``List``, ``List2``, and ``Iter``.

Bind
""""

``Bind`` is the most flexible option and can map many result shapes:

.. code-block:: go

    func Bind[T any](rows *sql.Rows) (result T, err error)

Characteristics:

- supports arbitrary mapping targets such as structs, slices, and scalar types
- offers the most flexibility
- can handle both single-row and multi-row data

Example:

.. code-block:: go

    type User struct {
        ID   int    `column:"id"`
        Name string `column:"name"`
    }

    rows, _ := db.Query("SELECT id, name FROM users")
    defer rows.Close()

    users, err := Bind[[]User](rows)
    user, err := juice.Bind[User](rows)

List
""""

``List`` is specialized for mapping into slice types:

.. code-block:: go

    func List[T any](rows *sql.Rows) (result []T, err error)

Characteristics:

- always returns ``[]T``
- performs better than ``Bind`` for slice-oriented use cases
- returns an empty slice rather than ``nil`` for empty results
- includes optimizations for non-pointer element types

Example:

.. code-block:: go

    rows, _ := db.Query("SELECT id, name FROM users")
    defer rows.Close()

    users, err := juice.List[User](rows)

List2
"""""

``List2`` is a ``List`` variant that returns a slice of pointers:

.. code-block:: go

    func List2[T any](rows *sql.Rows) ([]*T, error)

It is mainly intended for pointer-oriented generic result sets.

Characteristics:

- returns ``[]*T``
- useful when elements need to be modified later
- useful for large structs
- avoids value-copy overhead

Example:

.. code-block:: go

    rows, _ := db.Query("SELECT id, name FROM users")
    defer rows.Close()

    users, err := juice.List2[User](rows)
    // users is []*User

Iter
""""

``Iter`` converts a result set into an iterator so that you do not need to load everything into memory at once.

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
"""""""""""""""

1. Use ``Bind`` when:

   - you need maximum flexibility
   - you are not sure about the final shape yet
   - you need to handle single-row results

2. Use ``List`` when:

   - you know the result is a slice
   - you want better performance for value slices
   - you prefer empty slices over ``nil``

3. Use ``List2`` when:

   - you need to modify elements after mapping
   - you are dealing with large structs
   - you want to avoid value copying

4. Use ``Iter`` when:

   - you need to process large result sets incrementally

.. note::

    Performance notes:

    - ``List`` is specially optimized for non-pointer element types.
    - ``List2`` adds another choice for pointer-oriented code generation and usage.
    - ``Bind`` is the most flexible, but not always the fastest.
    - ``Iter`` is the best fit for large streaming-style workloads, but it holds a connection while you are iterating.

Auto-Increment Primary Keys
---------------------------

Juice supports automatically retrieving generated primary key values:

.. code-block:: xml

    <insert id="CreateUser" useGeneratedKeys="true" keyProperty="ID">
        INSERT INTO users (name, age) VALUES (#{name}, #{age})
    </insert>

Requirements:

1. The database driver must support ``LastInsertId``.
2. The parameter must be a struct pointer.
3. ``useGeneratedKeys="true"`` must be enabled.
4. You must specify ``keyProperty`` or use ``autoincr:"true"``.
5. The key field type must support integer assignment.

Batch Insert Optimization
-------------------------

Juice supports efficient batch inserts:

.. code-block:: xml

    <insert id="BatchInsertUsers" batchSize="100">
        INSERT INTO users (name, age) VALUES
        <foreach collection="users" item="user" open="(" separator="," close=")">
            (#{user.name}, #{user.age})
        </foreach>
    </insert>

For batch inserts, the parameter type must be a slice, an array, or a map with exactly one key whose value is a slice or array.

Optimization features:

1. **Smart batching**

   - automatically splits large datasets into batches
   - batch size is configurable via ``batchSize``
   - single execution is the default behavior

2. **Prepared-statement optimization**

   - prepared statements are reused
   - at most two prepared statements are generated
   - database pressure is reduced effectively

3. **Performance suggestions**

   - recommended batch size: 50-1000
   - tune according to data volume and database capacity
   - avoid oversized batches that overload the database

.. tip::

    Batch insert best practices:

    1. Set batch sizes carefully.
    2. Monitor database performance.
    3. Consider transaction boundaries.
    4. Handle errors thoroughly.
