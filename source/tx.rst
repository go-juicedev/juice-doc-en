Transactions
============

When working with databases, we often encounter transactions, such as when we need to manipulate multiple tables simultaneously. In such cases, we use transactions to ensure data consistency.

How do we use transactions in ``juice``? Let's review our previous usage first:

.. code-block:: go
   cfg, err := juice.NewXMLConfiguration("config.xml")
   if err != nil {
       panic(err)
   }
   engine, err := juice.DefaultEngine(cfg)
   if err != nil {
       panic(err)
   }

   engine.Object("your object")..QueryContext(context.Background(), nil)
   juice.NewGenericManager[User](engine).Object("your object").QueryContext(context.Background(), nil)

In the above code, we see that we operate through the ``engine``, which obtains a connection from the database connection pool, performs operations, and then returns the connection back to the pool.

So, how do we handle transactions? Let’s find out.

Manager
-------

From inspecting the source code of ``NewGenericManager``, we see that it accepts a parameter named ``Manager``. The definition of the Manager interface is as follows:

.. code-block:: go

   // Manager is an interface for managing database operations.
   type Manager interface {
       Object(v any) Executor
   }

The ``engine`` we used above is a structure that implements the ``Manager`` interface.

Initiating Transactions
-----------------------

We can initiate a transaction using ``engine.Tx()``, which returns an implementation of the ``Manager`` interface that we can use to perform transaction operations. Note that the transaction starts as soon as you call ``engine.Tx()``.

We can conduct transactional operations using the transaction object returned by ``engine.Tx()``. Here’s how we might write it:

.. code-block:: go

   cfg, err := juice.NewXMLConfiguration("config.xml")
   if err != nil {
       panic(err)
   }
   engine, err := juice.DefaultEngine(cfg)
   if err != nil {
       panic(err)
   }

   tx := engine.Tx()
   defer tx.Rollback()

   {
       tx.Object("your object").QueryContext(context.Background(), nil)
   }
   {
       juice.NewGenericManager[User](tx).Object("your object").QueryContext(context.Background(), nil)
   }

   tx.Commit()

In the example above, we switch the ``Manager`` from ``engine`` to ``tx``, and all operations happen within a transaction. We use ``tx`` to manage the transaction. When we need to commit the transaction, we call ``tx.Commit()``, and if we need to roll it back, we call ``tx.Rollback()``.

Note, once we call either ``tx.Commit()`` or ``tx.Rollback()``, the transaction ends, and we must not attempt to perform any more operations on it.

If nested transactions are required, we can initiate them by making multiple calls to ``engine.Tx()``, but ensure after each call you terminate the transaction using either ``tx.Rollback()`` or ``tx.Commit()``, or your transaction will remain open.

Isolation Levels
----------------

The official Go package database/sql provides controls for transaction isolation levels:

.. code-block:: go

   // IsolationLevel is the transaction isolation level used in TxOptions.
   type IsolationLevel int

   // Various isolation levels that drivers may support in BeginTx.
   // If a driver does not support a given isolation level an error may be returned.
   //
   // See https://en.wikipedia.org/wiki/Isolation_(database_systems)#Isolation_levels.
   const (
       LevelDefault IsolationLevel = iota
       LevelReadUncommitted
       LevelReadCommitted
       LevelWriteCommitted
       LevelRepeatableRead
       LevelSnapshot
       LevelSerializable
       LevelLinearizable
   )

   // TxOptions holds the transaction options to be used in DB.BeginTx.
   type TxOptions struct {
       // Isolation is the transaction isolation level.
       // If zero, the driver or database's default level is used.
       Isolation IsolationLevel
       ReadOnly  bool
   }

   func (db *DB) BeginTx(ctx context.Context, opts *TxOptions) (*Tx, error)

``juice`` also provides similar functionality:

.. code-block:: go

   func (e *Engine) ContextTx(ctx context.Context, opt *sql.TxOptions) TxManager

This also allows for configuration of transactions in ``juice``, providing similar capabilities and controls for transaction management as seen in the Go standard library.
