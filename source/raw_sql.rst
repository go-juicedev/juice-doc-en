Raw SQL Execution
==================

Overview
--------
In certain scenarios, we may need to execute SQL statements directly rather than through XML configuration. Juice provides a simple API to support this requirement.

Basic Usage
-----------
The simplest way is to use the ``engine.Raw()`` method:

.. code-block:: go

    var engine *juice.Engine

    // Execute query and get results
    rows, err := engine.Raw("SELECT * FROM user WHERE id = #{id}").
        Select(context.TODO(), juice.H{"id": 1})

Result Set Mapping
------------------
Juice provides multiple ways to map query results to Go structs:

.. code-block:: go

    // Define data structure
    type User struct {
        ID   int64  `column:"id"`
        Name string `column:"name"`
    }

    // Create query
    runner := engine.Raw("SELECT id, name FROM user WHERE id = #{id}")

    // Method 1: Bind directly to slice
    users, err := juice.NewGenericRunner[[]User](runner).
        Bind(context.TODO(), juice.H{"id": 1})
    // users type is []User

    // Method 2: Use List method
    users, err := juice.NewGenericRunner[User](runner).
        List(context.TODO(), juice.H{"id": 1})
    // users type is []User

    // Method 3: Use List2 method (returns pointer slice)
    users, err := juice.NewGenericRunner[User](runner).
        List2(context.TODO(), juice.H{"id": 1})
    // users type is []*User

Transaction Support
-------------------
Executing raw SQL within transactions is equally simple:

.. code-block:: go

    var engine *juice.Engine

    // Start transaction
    tx := engine.Tx()
    if err := tx.Begin(); err != nil {
        // Handle error
    }

    // Execute query within transaction
    runner := tx.Raw("SELECT id, name FROM user WHERE id = #{id}")
    // Subsequent usage is the same as non-transactional

    // Don't forget to commit or rollback the transaction
    if err := tx.Commit(); err != nil {
        tx.Rollback()
    }

Important Notes
---------------
- Parameter binding uses ``#{paramName}`` syntax
- Remember to properly handle transaction commits and rollbacks

sql.DB Execution
----------------

After the engine is successfully initialized, you can get the underlying ``sql.DB`` object through ``engine.DB()``.

We can execute SQL statements using ``DB.Exec()`` or ``DB.Query()`` methods.

What's the difference between this and the ``engine.Raw()`` method above?

.. code-block:: go

    engine.DB().Query("SELECT id, name FROM user WHERE id = ?", 1)

.. code-block:: go

    engine.Raw("SELECT id, name FROM user WHERE id = #{id}").Select(context.TODO(), juice.H{"id": 1})

- ``engine.Raw()`` can abstract underlying driver placeholder differences, while ``DB.Exec()`` and ``DB.Query()`` require developers to manually specify placeholders.

- ``engine.Raw()`` goes through middleware, while ``DB.Exec()`` and ``DB.Query()`` do not.