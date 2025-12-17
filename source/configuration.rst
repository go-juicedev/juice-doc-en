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

Connection Pool Tuning Guide
----------------------------

The connection pool is a key factor in database application performance. Reasonable connection pool configuration can significantly improve application performance and stability.

Tuning Principles
~~~~~~~~~~~~~~~~~

1. **Adjust Based on Actual Load**
   
   Different application scenarios require different connection pool configurations:
   
   - Web Applications: High concurrency requiring a larger connection pool.
   - Batch Tasks: Low concurrency, a small connection pool suffices.
   - Microservices: Adjust based on service scale and call frequency.

2. **Avoid Over-Configuration**
   
   A connection pool that is too large can cause problems:
   
   - Consuming too many database resources.
   - Increasing the burden on the database server.
   - Wasting application server memory.

3. **Monitor and Adjust**
   
   Continuously monitor the following metrics:
   
   - Connection pool usage rate.
   - Wait time for connections.
   - Frequency of connection creation and destruction.
   - Database server load.

Recommended Configurations for Different Scenarios
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Web Applications (High Concurrency)**

.. code-block:: xml

    <environment id="web-prod">
        <dataSource>root:password@tcp(localhost:3306)/database</dataSource>
        <driver>mysql</driver>
        <!-- Max open connections: Set based on concurrency -->
        <maxOpenConnNum>200</maxOpenConnNum>
        <!-- Max idle connections: Usually set to 25%-50% of maxOpenConnNum -->
        <maxIdleConnNum>50</maxIdleConnNum>
        <!-- Max connection lifetime: 30 minutes -->
        <maxConnLifetime>1800</maxConnLifetime>
        <!-- Max idle connection lifetime: 10 minutes -->
        <maxIdleConnLifetime>600</maxIdleConnLifetime>
    </environment>

**Batch Tasks (Low Concurrency, Long Running)**

.. code-block:: xml

    <environment id="batch">
        <dataSource>root:password@tcp(localhost:3306)/database</dataSource>
        <driver>mysql</driver>
        <!-- Batch processing usually has lower concurrency -->
        <maxOpenConnNum>20</maxOpenConnNum>
        <maxIdleConnNum>5</maxIdleConnNum>
        <!-- Long-running tasks can have longer connection lifetimes -->
        <maxConnLifetime>7200</maxConnLifetime>
        <maxIdleConnLifetime>3600</maxIdleConnLifetime>
    </environment>

**Microservices (Medium Concurrency)**

.. code-block:: xml

    <environment id="microservice">
        <dataSource>root:password@tcp(localhost:3306)/database</dataSource>
        <driver>mysql</driver>
        <!-- Microservices usually require fast response -->
        <maxOpenConnNum>50</maxOpenConnNum>
        <maxIdleConnNum>10</maxIdleConnNum>
        <!-- Shorter lifetime to release resources quickly -->
        <maxConnLifetime>900</maxConnLifetime>
        <maxIdleConnLifetime>300</maxIdleConnLifetime>
    </environment>

**Development Environment**

.. code-block:: xml

    <environment id="dev">
        <dataSource>./dev.db</dataSource>
        <driver>sqlite3</driver>
        <!-- Keep simple configuration for development -->
        <maxOpenConnNum>10</maxOpenConnNum>
        <maxIdleConnNum>2</maxIdleConnNum>
        <maxConnLifetime>3600</maxConnLifetime>
    </environment>

Configuration Parameters Detail
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. list-table::
   :header-rows: 1
   :widths: 25 15 60

   * - Parameter Name
     - Default Value
     - Description & Suggestions
   * - maxOpenConnNum
     - Unlimited
     - **Maximum Open Connections**
       
       - Limits the total number of simultaneously open database connections.
       - Suggestion: Set based on database server performance and application concurrency.
       - Web Apps: 50-200, Batch: 10-50.
       - Setting it too large will consume excessive database resources.
   * - maxIdleConnNum
     - 2
     - **Maximum Idle Connections**
       
       - The number of idle connections kept in the pool.
       - Suggestion: 25%-50% of maxOpenConnNum.
       - Setting it too small causes frequent creation/destruction of connections.
       - Setting it too large wastes resources.
   * - maxConnLifetime
     - Permanent
     - **Maximum Connection Lifetime (Seconds)**
       
       - Maximum time from creation to forced closure.
       - Suggestion: 1800-7200 (30 minutes - 2 hours).
       - Prevents issues caused by long-lived connections.
       - 0 means never expire (not recommended).
   * - maxIdleConnLifetime
     - Permanent
     - **Maximum Idle Connection Lifetime (Seconds)**
       
       - Maximum time an idle connection is kept.
       - Suggestion: 300-3600 (5 minutes - 1 hour).
       - Should be less than maxConnLifetime.
       - Release inactive connections in time.

Calculating Connection Pool Size
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Basic Formula**

.. code-block:: text

    maxOpenConnNum = (Core Count × 2) + Effective Spindle Count

    OR

    maxOpenConnNum = Concurrent Requests × Average Connection Hold Time / Request Interval

**Example Calculation**

Assuming your application:

- Deployed on a 4-core CPU server
- Using SSD storage (counts as 1 effective spindle)
- Expected concurrency: 100 QPS
- Average connection hold time per request: 50ms
- Request interval: 10ms

Method 1: ``maxOpenConnNum = (4 × 2) + 1 = 9`` (Conservative estimate)

Method 2: ``maxOpenConnNum = 100 × 0.05 / 0.01 = 500`` (Theoretical maximum)

**Practical Suggestion**: Start with a smaller value (e.g., 50) and gradually adjust to the optimal value through monitoring.

Performance Monitoring
~~~~~~~~~~~~~~~~~~~~~~

**Monitoring Metrics**

.. code-block:: go

    // Get connection pool statistics
    stats := engine.DB().Stats()

    fmt.Printf("Max Open Connections: %d\n", stats.MaxOpenConnections)
    fmt.Printf("Open Connections: %d\n", stats.OpenConnections)
    fmt.Printf("In Use: %d\n", stats.InUse)
    fmt.Printf("Idle: %d\n", stats.Idle)
    fmt.Printf("Wait Count: %d\n", stats.WaitCount)
    fmt.Printf("Wait Duration: %v\n", stats.WaitDuration)
    fmt.Printf("Max Idle Closed: %d\n", stats.MaxIdleClosed)
    fmt.Printf("Max Lifetime Closed: %d\n", stats.MaxLifetimeClosed)

Common Issue Troubleshooting
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Issue 1: Connection Pool Exhaustion**

Symptoms:
  - Application response slows down.
  - Large number of requests waiting for database connections.
  - ``stats.WaitCount`` keeps growing.

Solutions:
  1. Increase ``maxOpenConnNum``.
  2. Check for connection leaks (not closed properly).
  3. Optimize slow queries to reduce connection holding time.
  4. Consider using connection pool monitoring.

**Issue 2: Frequent Connection Creation/Destruction**

Symptoms:
  - ``stats.MaxIdleClosed`` grows rapidly.
  - CPU usage fluctuates.
  - Database connection count fluctuates greatly.

Solutions:
  1. Increase ``maxIdleConnNum``.
  2. Extend ``maxIdleConnLifetime``.
  3. Evaluate if connection pool warmup is needed.

**Issue 3: Database Connection Timeout**

Symptoms:
  - "connection timeout" errors occur.
  - Connection failures after long running times.

Solutions:
  1. Set a reasonable ``maxConnLifetime``.
  2. Ensure it is less than the database server's timeout setting.
  3. Implement connection health checks.

Best Practices
~~~~~~~~~~~~~~

**1. Connection Pool Warmup**

.. code-block:: go

    func warmupConnectionPool(engine *juice.Engine, size int) error {
        db := engine.DB()
        
        // Create multiple connections
        var conns []*sql.Conn
        for i := 0; i < size; i++ {
            conn, err := db.Conn(context.Background())
            if err != nil {
                return err
            }
            conns = append(conns, conn)
        }
        
        // Execute simple query to ensure connection availability
        for _, conn := range conns {
            if err := conn.PingContext(context.Background()); err != nil {
                return err
            }
        }
        
        // Release connections back to the pool
        for _, conn := range conns {
            conn.Close()
        }
        
        return nil
    }

**2. Graceful Shutdown**

.. code-block:: go

    func gracefulShutdown(engine *juice.Engine) {
        // Stop accepting new requests
        // ...
        
        // Wait for existing requests to complete
        time.Sleep(5 * time.Second)
        
        // Close connection pool
        if err := engine.Close(); err != nil {
            log.Printf("Failed to close connection pool: %v", err)
        }
    }

**3. Configuration Checklist**

Before deploying to production, please check:

.. code-block:: text

    ☐ Is maxOpenConnNum set according to actual load?
    ☐ Is maxIdleConnNum 25%-50% of maxOpenConnNum?
    ☐ Is maxConnLifetime less than database server timeout?
    ☐ Is maxIdleConnLifetime less than maxConnLifetime?
    ☐ Is connection pool monitoring implemented?
    ☐ Are alert thresholds set?
    ☐ Has stress testing been performed?
    ☐ Is connection pool warmup mechanism in place?
    ☐ Is graceful shutdown implemented?
    ☐ Are configurations differentiated for different environments?

.. tip::
    **Tuning Advice**:
    
    1. Start with conservative configuration (small connection pool).
    2. Collect actual data through monitoring.
    3. Gradually adjust to the optimal value.
    4. Regularly review and adjust configuration.
    5. Record the reason and effect of each adjustment.