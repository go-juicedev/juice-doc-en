Configuration Details
=====================

Juice uses an XML format for its default configuration files. You can create an XML file in your project directory to configure Juice-related settings.

Configuring
-----------

.. code-block:: xml

    <?xml version="1.0" encoding="UTF-8"?>
    <configuration>
    </configuration>

All the configuration information for ``juice`` should be written within the ``configuration`` tag, such as database connection information.

Environments
------------

The ``environments`` tag is used to configure different settings for various environments, like development, testing, and production environments. Developers can configure different environments according to their needs, and ``juice`` will load specific configuration information based on the ``default`` attribute of the ``environments`` tag.

.. code-block:: xml

    <?xml version="1.0" encoding="UTF-8"?>
    <configuration>
      <environments default="prod">
        <environment id="prod">
          <dataSource>root:qwe123@tcp(localhost:3306)/database</dataSource>
          <driver>mysql</driver>
        </environment>
        <environment id="dev">
          <dataSource>./foo.db</dataSource>
          <driver>sqlite3</driver>
        </environment>
      </environments>
    </configuration>

In the configuration above, we have set up two environments: ``prod`` and ``dev``, where ``prod`` is the default environment specified by ``environments``. Juice will use this configuration to load the database connection information.

.. attention::

   The default attribute inside the environments tag is mandatory; without this attribute, juice does not know which environment's configuration information to load. Each environment's configuration settings are specified within the ``environment`` tag, where the ``id`` attribute is required and unique to identify different environments. The ``dataSource`` tag is used to configure the database connection information, and the ``driver`` tag is for configuring the database driver.

Once the environment is configured, we can use the database connection provided by ``juice`` in our code.

.. code-block:: go

    package main

    import (
      "fmt"
      "github.com/go-juicedev/juice"
      _ "github.com/go-sql-driver/mysql"
    )

    func main() {
      // Parsing the configuration file
      cfg, err := juice.NewXMLConfiguration("config.xml") // config.xml is the configuration file we just wrote.
      if err != nil {
        fmt.Println(err)
        return
      }

      // Building engine from configuration file
      engine, err := juice.DefaultEngine(cfg)
      if err != nil {
        fmt.Println(err)
        return
      }

      if err = engine.DB().Ping(); err != nil {
        fmt.Println(err)
        return
      }

      fmt.Println(" connected to database")
    }

.. attention::

   By default, juice only connects to the ``environment`` specified by the ``default`` attribute within ``environments``.


Database Environment Switching
--------------------------------

By default, juice only connects to the environment specified by the ``default`` attribute in ``environments``.

When multiple data sources are configured, developers need to manually switch between them.

.. code-block:: xml

    <?xml version="1.0" encoding="UTF-8"?>
        <configuration>
            <environments default="master">

                <environment id="master">
                    <dataSource>root:qwe123@tcp(localhost:3306)/database</provider>
                    <driver>mysql</driver>
                </environment>

                <environment id="slave1">
                    <dataSource>root:qwe123@tcp(localhost:3307)/database</provider>
                    <driver>mysql</driver>
                </environment>

                <environment id="slave2">
                    <dataSource>root:qwe123@tcp(localhost:3308)/database</provider>
                    <driver>mysql</driver>
                </environment>

            </environments>
        </configuration>


As shown above, we have configured three environments, with ``master`` as the default environment.

When we want to switch to the ``slave1`` environment:

.. code-block:: go

    engine, _ := juice.New(cfg)

    slave1Engine, err := engine.With("slave1")


Provider
--------

Sometimes we do not want to hard code the database connection information in a configuration file but want to dynamically load it through some other means. In this case, we can use the ``provider`` tag to configure the database connection information.

.. code-block:: xml

    <?xml version="1.0" encoding="UTF-8"?>
    <configuration>
      <environments default="prod">
        <environment id="prod" provider="env">
          <dataSource>${DATA_SOURCE}</dataSource>
          <driver>mysql</driver>
        </environment>
      </environments>
    </configuration>

As shown above, we have configured a ``provider`` tag in the ``prod`` environment with a value of ``env``. ``env`` is a default provider offered by ``juice``, which fetches the database connection information from environment variables.

If you want to customize your ``provider``, you can refer to the ``provider`` implementations provided by ``juice`` and create your own.

.. code-block:: go

    // EnvValueProvider defines an environment value provider.
    type EnvValueProvider interface {
      Get(key string) (string, error)
    }

    // RegisterEnvValueProvider registers an environment value provider.
    // The key is the name of the provider.
    // The value is a provider.
    // It allows overriding the default provider.
    func RegisterEnvValueProvider(name string, provider EnvValueProvider)

As shown above, by implementing the ``EnvValueProvider`` interface, you can register your own ``provider`` using the ``RegisterEnvValueProvider`` method provided by ``juice``. The name parameter of the ``RegisterEnvValueProvider`` function corresponds to the provider value specified in the XML. When you implement your own ``EnvValueProvider``, juice will pass the configuration information read from the ``environment`` unaltered to the ``EnvValueProvider``. 

As shown above, when the value of ``provider`` is specified as ``env``, it will follow predefined rules for parsing, and here is the specific implementation.

.. code-block:: go

    var formatRegexp = regexp.MustCompile(`\$\{ *?([a-zA-Z0-9_\.]+) *?\}`)

    // OsEnvValueProvider is an environment value provider that uses os.Getenv.
    type OsEnvValueProvider struct{}

    // Get returns the value of the environment variable.
    // It uses os.Getenv.
    func (p OsEnvValueProvider) Get(key string) (string, error) {
      var err error
      key = formatRegexp.ReplaceAllStringFunc(key, func(find string) string {
        value := os.Getenv(formatRegexp.FindStringSubmatch(find)[1])
        if len(value) == 0 {
          err = fmt.Errorf("environment variable %s not found", find)
        }
        return value
      })
      return key, err
    }

By reviewing the code, we can understand that when parsing the `${}` syntax format, it attempts to find the contents through environment variables; otherwise, it returns the raw content.

Connection Pool Configuration
-----------------------------

.. code-block:: xml

    <?xml version="1.0" encoding="UTF-8"?>
    <configuration>
      <environments default="prod">
        <environment id="prod">
          <dataSource>root:qwe123@tcp(localhost:3306)/database</dataSource>
          <driver>mysql</driver>
          <maxIdleConnNum>10</maxIdleConnNum>
          <maxOpenConnNum>10</maxOpenConnNum>
          <maxConnLifetime>3600</maxConnLifetime>
          <maxIdleConnLifetime>3600</maxIdleConnLifetime>
        </environment>
      </environments>
    </configuration>

**In the configuration above, we have configured the connection pool settings.**

- The ``maxIdleConnNum`` tag is used to configure the maximum number of idle connections.
- The ``maxOpenConnNum`` tag is used to configure the maximum number of open connections.
- The ``maxConnLifetime`` tag is used to configure the maximum lifetime of connections, measured in seconds.
- The ``maxIdleConnLifetime`` tag is used to configure the maximum lifetime of idle connections, also in seconds.

Developers can configure the connection pool settings according to their needs, and ``juice`` will initialize the connection pool based on these settings.

Settings
--------

The ``settings`` tag is used to inject custom configuration information into ``juice``. The ``settings`` tag is the parent tag for multiple ``setting`` tags. Each ``setting`` tag must have a ``name`` attribute to identify the name of the configuration, and the ``value`` attribute is optional, for setting the value of the configuration. The ``settings`` tag is optional and can be omitted if not needed.

The specific use depends on the developer's requirements. For instance, in the ``DebugMiddleware`` middleware provided by ``juice``, it decides whether to enable debug mode based on the configuration information. By default, it is enabled, but you can disable it in the configuration file.

To disable debug mode, use the following configuration:

.. code-block:: xml

    <?xml version="1.0" encoding="UTF-8"?>
    <configuration>
      <settings>
        <setting name="debug" value="false"/>
      </settings>
    </configuration>