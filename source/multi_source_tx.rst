Multi-Datasource and Transaction Interaction
==============================================

Goal
----------------

This page explains how datasource switching with ``engine.With(...)`` interacts with transactions such as ``Transaction`` and ``Tx`` in Juice, so that cross-database transaction misuse can be avoided.

Key Conclusions
----------------

- ``engine.With("name")`` returns a new ``Engine`` view bound to the target datasource.
- A ``juice.Transaction`` only covers the connection for the datasource associated with the current engine.
- ``NestedTransaction`` only decides whether a transaction already exists. It does not provide cross-datasource consistency.
- Juice does not provide a distributed transaction abstraction such as 2PC or XA. Cross-database consistency should be handled with business-level patterns.

Interaction Matrix
------------------

.. list-table::
   :header-rows: 1
   :widths: 24 22 30 24

   * - Scenario
     - Covered by one transaction?
     - Risk
     - Recommendation
   * - ``Transaction`` within one datasource
     - Yes
     - Low
     - Use directly
   * - Call ``With("slave")`` before ``Transaction``
     - Yes, but only for that slave datasource
     - Read/write responsibilities can become unclear
     - Use only after defining a clear read/write policy
   * - Switch to another datasource inside a transaction callback
     - No, this usually becomes two separate transaction domains
     - Inconsistency and partial commits
     - Avoid it; use saga or outbox patterns instead
   * - Switch between multiple datasources and write in one request
     - No
     - No atomic commit is possible
     - Split the steps and design compensation

Recommended Patterns
--------------------

Pattern 1: **Choose the datasource first, then open the transaction**

.. code-block:: go

    base := juice.ContextWithManager(context.Background(), engine)

    writeEngine, err := engine.With("master")
    if err != nil {
        return err
    }

    ctx := juice.ContextWithManager(base, writeEngine)
    return juice.Transaction(ctx, func(ctx context.Context) error {
        _, err := repo.CreateOrder(ctx, req)
        return err
    })

Pattern 2: **Under read-write splitting, keep write paths on master**

- Queries can go to replicas according to policy, typically without transactions or with read-only transactions.
- Write paths such as order creation, inventory deduction, and bookkeeping should consistently use a transaction on the primary datasource.

Pattern 3: **Use business patterns for cross-datasource consistency**

Possible strategies:

- outbox with asynchronous delivery
- saga with compensating transactions
- idempotency keys with retries

Anti-Pattern (Not Recommended)
------------------------------

.. code-block:: go

    // The outer layer opens a transaction on master.
    _ = juice.Transaction(ctx, func(ctx context.Context) error {
        if _, err := repoA.WriteOnMaster(ctx, a); err != nil {
            return err
        }

        // Switch to another datasource inside the same callback and write again.
        other, _ := engine.With("other")
        otherCtx := juice.ContextWithManager(ctx, other)
        _, err := repoB.WriteOnOther(otherCtx, b)
        return err
    })

The code above usually cannot guarantee atomic consistency across both writes.

Troubleshooting Tips
--------------------

- If some steps succeed and others fail, first check whether one business action writes to multiple datasources.
- If a transaction seems ineffective, check whether another ``engine.With(...)`` context was mixed into the call chain.
- Logging the EnvID in middleware, if available in your setup, helps locate datasource drift quickly.
