Transaction Management
======================

Overview
--------

Transactions ensure the atomicity, consistency, isolation, and durability of a group of database operations. Juice provides two transaction styles:

1. Manual transactions: ``engine.Tx()`` and ``engine.ContextTx(...)``
2. Functional transactions: ``juice.Transaction(...)`` and ``juice.NestedTransaction(...)``

Transaction Interfaces
----------------------

The core transaction-related interfaces in Juice are:

.. code-block:: go

    type Manager interface {
        Object(v any) SQLRowsExecutor
    }

    type TxManager interface {
        Manager
        Begin() error
        Commit() error
        Rollback() error
    }

Manual Transactions
-------------------

Use manual transactions when you need explicit control over ``Begin``, ``Commit``, and ``Rollback``:

.. code-block:: go

    tx := engine.Tx()
    if err := tx.Begin(); err != nil {
        return err
    }
    defer tx.Rollback()

    if _, err := tx.Object(Repo{}.CreateUser).ExecContext(ctx, user); err != nil {
        return err
    }

    if _, err := tx.Object(Repo{}.CreateOrder).ExecContext(ctx, order); err != nil {
        return err
    }

    return tx.Commit()

.. note::

    It is recommended to always use ``defer tx.Rollback()`` as a fallback, and then call ``tx.Commit()`` only on the success path.

Functional Transactions
-----------------------

Functional transactions automatically inject the transaction manager into the callback context, which makes them a good fit for service-layer transaction boundaries:

.. code-block:: go

    baseCtx := juice.ContextWithManager(context.Background(), engine)

    err := juice.Transaction(baseCtx, func(ctx context.Context) error {
        repo := NewUserRepository()
        _, err := repo.CreateUser(ctx, user)
        return err
    },
        tx.WithIsolationLevel(sql.LevelReadCommitted),
        tx.WithReadOnly(false),
    )

    if err != nil {
        return
    }

``Transaction`` commits when the callback returns ``nil`` and rolls back when the callback returns a non-``nil`` error.

Nested Call Semantics
---------------------

The semantics of ``NestedTransaction`` are: reuse the current transaction if one already exists, otherwise create a new transaction.

.. code-block:: go

    err := juice.Transaction(baseCtx, func(ctx context.Context) error {
        if err := serviceA(ctx); err != nil {
            return err
        }

        return juice.NestedTransaction(ctx, func(ctx context.Context) error {
            return serviceB(ctx)
        })
    })

.. attention::

    ``NestedTransaction`` is not the same as a database savepoint. It does not automatically create an independent inner commit point.

Isolation Level and Read-Only Options
-------------------------------------

You can specify the isolation level and read-only flag when opening a transaction:

.. code-block:: go

    err := juice.Transaction(baseCtx, handler,
        tx.WithIsolationLevel(sql.LevelSerializable),
        tx.WithReadOnly(true),
    )

These options are aligned with ``database/sql`` and ``sql.TxOptions``.

Further Reading
---------------

- :doc:`tx_semantics`
- :doc:`multi_source_tx`
