Raw SQL Execution
=================

Overview
--------

In some scenarios, you may want to execute SQL directly instead of going through XML configuration. Juice provides a straightforward API for that use case.

Basic Usage
-----------

The simplest way is to use ``engine.Raw()``:

.. code-block:: go

    var engine *juice.Engine

    rows, err := engine.Raw("SELECT * FROM user WHERE id = #{id}").
        Select(context.TODO(), juice.H{"id": 1})

Result Set Mapping
------------------

Juice provides several ways to map query results into Go structs:

.. code-block:: go

    type User struct {
        ID   int64  `column:"id"`
        Name string `column:"name"`
    }

    runner := engine.Raw("SELECT id, name FROM user WHERE id = #{id}")

    users, err := juice.NewGenericRunner[[]User](runner).
        Bind(context.TODO(), juice.H{"id": 1})
    // users is []User

    users, err = juice.NewGenericRunner[User](runner).
        List(context.TODO(), juice.H{"id": 1})
    // users is []User

    users2, err := juice.NewGenericRunner[User](runner).
        List2(context.TODO(), juice.H{"id": 1})
    // users2 is []*User

Transaction Support
-------------------

Executing raw SQL inside a transaction is just as simple:

.. code-block:: go

    var engine *juice.Engine

    tx := engine.Tx()
    if err := tx.Begin(); err != nil {
        // handle error
    }

    runner := tx.Raw("SELECT id, name FROM user WHERE id = #{id}")
    // Subsequent usage is the same as in the non-transactional case.

    if err := tx.Commit(); err != nil {
        tx.Rollback()
    }

Notes
-----

- Parameter binding uses the ``#{paramName}`` syntax.
- Make sure you handle transaction commit and rollback correctly.

sql.DB Execution
----------------

Once the engine has been initialized successfully, you can access the underlying ``sql.DB`` object through ``engine.DB()``.

You can then use ``DB.Exec()`` or ``DB.Query()`` directly:

.. code-block:: go

    engine.DB().Query("SELECT id, name FROM user WHERE id = ?", 1)

.. code-block:: go

    engine.Raw("SELECT id, name FROM user WHERE id = #{id}").Select(context.TODO(), juice.H{"id": 1})

The main differences are:

- ``engine.Raw()`` abstracts away placeholder differences between drivers, while ``DB.Exec()`` and ``DB.Query()`` require you to handle placeholders manually.
- ``engine.Raw()`` goes through Juice middleware, while ``DB.Exec()`` and ``DB.Query()`` do not.
