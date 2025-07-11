Transaction Management
========

Overview
----

Transactions are the basic unit of database operations, used to ensure data consistency and integrity. Juice provides comprehensive transaction support, including basic transaction operations, nested transactions, and isolation level control.

Transaction Interfaces
-------

Juice defines two core interfaces to handle transactions:

.. code-block:: go

    // Manager interface defines basic database operations
    type Manager interface {
        Object(v any) Executor
    }

    // TxManager interface extends Manager, adding transaction control methods
    type TxManager interface {
        Manager
        Begin() error
        Commit() error
        Rollback() error
    }

Basic Transaction Operations
----------

Example code shows how to use transactions:

.. code-block:: go

    engine, err := juice.Default(cfg)
    if err != nil {
        panic(err)
    }

    // Start transaction
    tx := engine.Tx()
    if err := tx.Begin(); err != nil {
        panic(err)
    }

    // Ensure transaction will eventually be rolled back or committed
    defer tx.Rollback()

    // Operations within transaction
    {
        // Query operations
        result1, err := tx.Object("QueryUser").
            QueryContext(context.TODO(), nil)
        if err != nil {
            return err
        }

        // Use generic manager
        result2, err := juice.NewGenericManager[User](tx).
            Object("CreateUser").
            QueryContext(context.TODO(), param)
        if err != nil {
            return err
        }
    }

    // Commit transaction
    return tx.Commit()

.. note::
    Transaction usage recommendations:

    1. Always use defer tx.Rollback()
    2. Check all errors before committing
    3. Do not use the transaction object after completion

Nested Transactions
-------

Juice supports nested transactions, but proper management is required:

.. code-block:: go

    tx1 := engine.Tx()
    if err := tx1.Begin(); err != nil {
        return err
    }
    defer tx1.Rollback()

    // Nested transaction
    tx2 := engine.Tx()
    if err := tx2.Begin(); err != nil {
        return err
    }
    defer tx2.Rollback()

    // Inner transaction operations
    if err := tx2.Commit(); err != nil {
        return err
    }

    // Outer transaction operations
    return tx1.Commit()

.. attention::
    Nested transaction considerations:

    1. Each transaction object needs to be properly closed
    2. Follow the first-opened-last-closed principle
    3. Pay attention to dependencies between transactions

Isolation Level Control
----------

Juice supports complete transaction isolation level control, consistent with the ``database/sql`` package:

.. code-block:: go

    // Supported isolation levels
    const (
        LevelDefault         sql.IsolationLevel = iota
        LevelReadUncommitted
        LevelReadCommitted
        LevelWriteCommitted
        LevelRepeatableRead
        LevelSnapshot
        LevelSerializable
        LevelLinearizable
    )

Usage example:

.. code-block:: go

    // Start transaction with specific isolation level
    tx := engine.ContextTx(ctx, &sql.TxOptions{
        Isolation: sql.LevelSerializable,
        ReadOnly:  false,
    })
    if err := tx.Begin(); err != nil {
        return err
    }
    defer tx.Rollback()

    // Transaction operations...

    return tx.Commit()