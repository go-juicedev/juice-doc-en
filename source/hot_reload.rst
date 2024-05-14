Hot Update
==========
Juice provides a feature that allows SQL updates without the need to restart the service.

.. code-block:: go

    cfg, err := juice.NewXMLConfiguration("config.xml")
    if err != nil {
        panic(err)
    }
    engine, err := juice.Default(cfg)

The code above illustrates how to create an engine. Once the engine has been created, it can be updated by using `engine.SetConfiguration`.

.. code-block:: go

    func (e *Engine) SetConfiguration(cfg IConfiguration)

`SetConfiguration` is a thread-safe method. After you execute this method, the engine will automatically update the mapper configuration.

.. attention::
    After executing SetConfiguration, it will not modify the database connection configurations.