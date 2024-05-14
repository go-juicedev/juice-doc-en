缓存
=========

juice提供了一个事务级别的查询缓存机制。开启缓存，在同一个事务中对同一个action采用相同的参数进行查询，第二次则不会去查询数据库。

开启缓存
--------

juice的缓存默认是不开启的，需要显式的告诉juice当前的事务开启缓存查询。

.. code-block:: go

    tx := engine.Tx()
    defer tx.Rollback()

    juice.NewGenericManager[int](tx).Object("obj id").QueryContext(context.Background(), nil)

    juice.NewGenericManager[int](tx).Object("obj id").QueryContext(context.Background(), nil)

    tx.Commit()

以上是不开启缓存的写法, 开启缓存，我们只需要改一行代码。

.. code-block:: go

    tx := engine.CacheTx()
    defer tx.Rollbakc()

    juice.NewGenericManager[int](tx).Object("obj id").QueryContext(context.Background(), nil)

    juice.NewGenericManager[int](tx).Object("obj id").QueryContext(context.Background(), nil)

    tx.Commit()

使用CacheTx来声明开启缓存性事务。补全上述伪代码，如果你使用了 `DebugMiddleware`，你会发现上面的查询语句只查询了一次。


缓存失效
--------
默认情况下，当在同一个事务中对数据库进行了一次修改的操作的时候，juice会将当前事务的缓存清空，后面在去查的话，前面的缓存结果就会失效。

那么juice是如何知道数据库进行了修改操作呢？

当juice去调用非 `select` 的action时，juice会认为这是一次修改操作，所以要正确的使用action标签。

当然，我们也可以告诉juice当前的修改不要去清空当前事务的缓存，我们只需要在当前的action标签上面加上一个属性 `flushCache=false` 即可。

.. code-block:: xml

    <update id="id" flushCache="false"></update>  

当事务提交或者回滚的时候，当前事务的缓存会被强制清空。

自定义缓存
----------

默认情况下，juice的缓存是存在当前进程的所持有的内存里面。如果你自定义存储，请实现juice定义的cache接口。

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

    engine.SetCacheFactory(func() cache.Cache() { return  mycacheImpl{}}) // note: 这里要返回一个新的cache实现。


juice 提供了一个redis的缓存，请到 ``https://github.com/eatmoreapple/juice-cache`` 查看请使用方式。


.. attention::

    注意： 缓存只有跟 `NewGenericManager` 搭配使用才有效


二级缓存
----------

Does not implement yet.



