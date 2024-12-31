Cache Mechanism
================

`juice` provides a transaction-level query caching mechanism. When caching is enabled, querying the same action with identical parameters within the same transaction will bypass the database lookup on subsequent queries.

Enable Caching
--------------
By default, `juice` does not enable caching; it must be explicitly enabled for the current transaction. Below is an example of how it works without caching enabled:

.. code-block:: go

    tx := engine.Tx()
    defer tx.Rollback()
    juice.NewGenericManager[int](tx).Object("obj id").QueryContext(context.Background(), nil)
    juice.NewGenericManager[int](tx).Object("obj id").QueryContext(context.Background(), nil)
    tx.Commit()

To enable caching, only a single line needs to be changed:

.. code-block:: go

    tx := engine.CacheTx()
    defer tx.Rollback()
    juice.NewGenericManager[int](tx).Object("obj id").QueryContext(context.Background(), nil)
    juice.NewGenericManager[int](tx).Object("obj id").QueryContext(context.Background(), nil)
    tx.Commit()

Using `CacheTx` declares a transaction with caching enabled. If you use `DebugMiddleware`, you will notice that the query statement above is only executed once.

Cache Invalidation
------------------
By default, if a modification is made to the database within the same transaction, `juice` will clear the cache for that transaction. Any subsequent queries would result in cache invalidation.

How does `juice` detect a modification? When `juice` executes a non-`select` action, it is assumed to be a modification. Therefore, it is crucial to use action tags correctly.

However, you can instruct `juice` not to clear the cache after modifications by adding an attribute `flushCache=false` to your action tag.

.. code-block:: xml

    <update id="id" flushCache="false"></update>

When a transaction is committed or rolled back, the cache for that transaction is forcibly cleared.

Custom Caching
--------------
By default, `juice`’s cache resides in the memory of the current process. To use custom storage, implement the cache interface defined by `juice`.

.. code-block:: go

    // ErrCacheNotFound is the error that cache not found.
    var ErrCacheNotFound = errors.New("juice: cache not found")

    type Cache interface {
        // Set sets the value for the key.
        Set(ctx context.Context, key string, value any) error
        // Get gets the value for the key.
        // If the value does not exist, it returns ErrCacheNotFound.
        Get(ctx context.Context, key string, dst any) error
        // Flush flushes all the cache.
        Flush(ctx context.Context) error
    }
    _ Cache = (*mycacheImpl)(nil)

    engine.SetCacheFactory(func() cache.Cache {
        return mycacheImpl{}
    }) // note: returns a new cache implementation.

`juice` also provides a Redis cache implementation. Please visit ``https://github.com/go-juicedev/juice-cache`` for usage details.

.. attention::

    Note: The cache is only effective when used in conjunction with `NewGenericManager`.

Secondary Cache
---------------
Does not implement yet.