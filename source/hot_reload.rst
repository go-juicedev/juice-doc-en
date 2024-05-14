热更新
========

juice 提供了一个能够在服务不重启的情况下更新sql的功能。

.. code-block:: go

    cfg, err := juice.NewXMLConfiguration("config.xml")
	
    if err != nil {
        panic(err)
    }

    engine, err := juice.DefaultEngine(cfg)

上面是我们创建一个engine的代码。当我们的engine创建完成之后，我们可以通过 `engine.SetConfiguration` 来对 engine 的配置进行更新。

.. code-block:: go

    func (e *Engine) SetConfiguration(cfg *Configuration)

`SetConfiguration` 是一个线程安全的方法。

当你执行完这个方法之后，engine会自动的更新mapper的配置。

.. attention::
    SetConfiguration 执行完毕之后不会修改数据库连接相关配置。