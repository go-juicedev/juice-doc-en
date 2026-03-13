Transaction Semantics Reference
=================================

Overview
----------------

This page focuses on the current behavioral semantics of the Juice transaction APIs so that you can choose the right pattern in the service layer.

Terminology:

- **Manual transaction**: ``engine.Tx()`` or ``engine.ContextTx(...)`` with explicit ``Begin/Commit/Rollback``.
- **Functional transaction**: ``juice.Transaction(...)`` or ``juice.NestedTransaction(...)``.
- **Current transaction context**: the manager in ``ctx`` is already a ``TxManager``.

Semantics Matrix
----------------

.. list-table::
   :header-rows: 1
   :widths: 18 18 22 22 20

   * - API
     - Start condition
     - Behavior on success
     - Behavior on failure
     - Typical scenario
   * - ``engine.Tx()`` + ``Begin``
     - ``Begin`` must be called manually
     - Commits after explicit ``Commit``
     - Rolls back after explicit ``Rollback`` or an error
     - Infrastructure-level code that needs fine-grained control
   * - ``engine.ContextTx(ctx, opts)``
     - Same as above, but accepts ``sql.TxOptions``
     - Same as above
     - Same as above
     - Transactions with a specific isolation level or read-only flag
   * - ``juice.Transaction(ctx, handler, opts...)``
     - ``ctx`` must carry a ``*juice.Engine``
     - Commits when ``handler`` returns ``nil``
     - Rolls back when ``handler`` returns a non-``nil`` error
     - A unified transaction boundary in the service layer
   * - ``juice.NestedTransaction(ctx, handler, opts...)``
     - Reuses the current transaction if one exists, otherwise opens a new one
     - When reusing, the outer transaction commits; otherwise it follows ``Transaction``
     - When reusing, the error bubbles to the outer layer; otherwise it rolls back
     - Composed services that should not repeatedly open transactions

Behavior Details
----------------

**1) Context requirements for ``Transaction``**

``juice.Transaction`` extracts the manager from ``ctx`` and requires it to be a ``*juice.Engine``. Otherwise it returns an error.

Common mistakes:

- The manager was not injected into ``ctx`` with ``juice.ContextWithManager``.
- A context that is already inside a transaction is passed back into ``Transaction`` directly. In that case the manager is usually not a ``*juice.Engine``.

Recommendation:

- Use ``Transaction`` at the outer entry point.
- Use ``NestedTransaction`` for reuse in inner layers.

**2) ``NestedTransaction`` is not a savepoint**

The semantics of ``NestedTransaction`` are "join when a transaction exists, create one otherwise". It is not a database ``SAVEPOINT`` abstraction.

That means:

- Reusing an outer transaction does not create an independent inner commit point.
- An inner failure affects whether the outer transaction can commit, and should usually be returned upward.

**3) Special error ``tx.ErrCommitOnSpecific``**

When ``handler`` returns ``tx.ErrCommitOnSpecific``, the transaction still attempts to commit and returns that error together with any commit error.

This is intended for special cases where a specific commit path must be marked explicitly. It is usually not something normal business code should depend on.

How to Choose in Practice
--------------------------

- Choose **manual transactions** when you need full lifecycle control.
- Prefer **functional transactions** when a business function is the transaction boundary.
- When service A calls service B and both may also be called independently, use outer ``Transaction`` plus inner ``NestedTransaction``.
- If you need savepoint-level semantics, wrap them explicitly on the business side according to your database capabilities.
