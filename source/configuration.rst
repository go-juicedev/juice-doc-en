Configuration Details
=====================

Configuration File Format
-------------------------

Juice uses ``XML`` as its default configuration format. The format is explicit, structured, and easy to maintain.

Basic Structure
---------------

All Juice configuration must be defined under the root ``configuration`` element:

.. code-block:: xml

    <?xml version="1.0" encoding="UTF-8"?>
    <configuration>
        <!-- Add Juice configuration here -->
    </configuration>

.. note::

    XML configuration provides good readability and maintainability. It is recommended to use a proper XML formatter to keep configuration files tidy.

environments
------------

Environment Configuration
~~~~~~~~~~~~~~~~~~~~~~~~~

The ``environments`` tag is one of the core pieces of Juice configuration. It is used to manage database connection settings for different environments such as development, testing, and production.

Juice determines the default environment through the ``default`` attribute on ``environments``.

Configuration Example
~~~~~~~~~~~~~~~~~~~~~

The following example shows a typical multi-environment configuration:

.. code-block:: xml

    <?xml version="1.0" encoding="UTF-8"?>
    <configuration>
        <environments default="prod">
            <!-- Production environment -->
            <environment id="prod">
                <dataSource>root:qwe123@tcp(localhost:3306)/database</dataSource>
                <driver>mysql</driver>
                <maxIdleConns>10</maxIdleConns>
                <maxOpenConns>100</maxOpenConns>
                <connMaxLifetime>3600</connMaxLifetime>
            </environment>

            <!-- Development environment -->
            <environment id="dev">
                <dataSource>./foo.db</dataSource>
                <driver>sqlite3</driver>
            </environment>
        </environments>
    </configuration>

Configuration Notes
~~~~~~~~~~~~~~~~~~~

Each environment contains these required elements:

- ``id``: the unique identifier of the environment
- ``dataSource``: the database connection string
- ``driver``: the database driver name

Optional items include:

- ``maxIdleConns``: maximum number of idle connections
- ``maxOpenConns``: maximum number of open connections
- ``connMaxLifetime``: maximum connection lifetime in seconds
- ``connMaxIdleTime``: maximum idle lifetime in seconds

.. attention::

    Important points:

    1. ``default`` on ``environments`` is required.
    2. Each ``environment`` must have a unique ``id``.
    3. By default, Juice connects only to the environment selected by ``default``.

Using the Configuration in Code
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The following example shows how to load and use an environment configuration in code:

.. code-block:: go

    package main

    import (
        "fmt"

        "github.com/go-juicedev/juice"
        _ "github.com/go-sql-driver/mysql"
    )

    func main() {
        cfg, err := juice.NewXMLConfiguration("config.xml")
        if err != nil {
            fmt.Printf("failed to load configuration: %v\n", err)
            return
        }

        engine, err := juice.Default(cfg)
        if err != nil {
            fmt.Printf("failed to initialize engine: %v\n", err)
            return
        }
        defer engine.Close()

        if err = engine.DB().Ping(); err != nil {
            fmt.Printf("failed to connect to database: %v\n", err)
            return
        }

        fmt.Println("database connection established")
    }

Datasource Switching
--------------------

Dynamic Switching
~~~~~~~~~~~~~~~~~

By default, Juice only connects to the datasource selected by the ``default`` attribute in ``environments``. In multi-datasource scenarios, Juice also provides a flexible switching mechanism.

Configuration Example
~~~~~~~~~~~~~~~~~~~~~

The example below shows a master-replica setup:

.. code-block:: xml

    <?xml version="1.0" encoding="UTF-8"?>
    <configuration>
        <environments default="master">
            <!-- Primary datasource -->
            <environment id="master">
                <dataSource>root:qwe123@tcp(localhost:3306)/database</dataSource>
                <driver>mysql</driver>
            </environment>

            <!-- Replica 1 -->
            <environment id="slave1">
                <dataSource>root:qwe123@tcp(localhost:3307)/database</dataSource>
                <driver>mysql</driver>
            </environment>

            <!-- Replica 2 -->
            <environment id="slave2">
                <dataSource>root:qwe123@tcp(localhost:3308)/database</dataSource>
                <driver>mysql</driver>
            </environment>
        </environments>
    </configuration>

In this configuration, ``master`` is the default datasource and ``slave1`` and ``slave2`` are replicas.

Switching Datasources Manually
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Juice provides the ``With`` method to switch the datasource at runtime:

.. code-block:: go

    engine, _ := juice.New(cfg)

    fmt.Println("default datasource:", engine.EnvID())

    slave1Engine, err := engine.With("slave1")
    if err != nil {
        return
    }

    fmt.Println("switched datasource:", slave1Engine.EnvID())

.. note::

    ``With`` returns a new ``Engine`` instance and does not mutate the original one. This keeps datasource switching isolated and safe.

Configuration Value Providers
-----------------------------

Dynamic Configuration Values
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Juice provides a configuration value provider mechanism so that database connection values do not need to be hardcoded in XML. This is particularly useful for sensitive values and for multi-environment deployments.

Environment Variable Provider
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Juice includes an ``env`` provider out of the box for reading values from environment variables:

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

Custom Providers
~~~~~~~~~~~~~~~~

You can implement your own provider by implementing the ``EnvValueProvider`` interface:

.. code-block:: go

    type EnvValueProvider interface {
        Get(key string) (string, error)
    }

    func RegisterEnvValueProvider(name string, provider EnvValueProvider)

Default Environment Variable Provider
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This is the implementation of the default environment variable provider:

.. code-block:: go

    var formatRegexp = regexp.MustCompile(`\$\{ *?([a-zA-Z0-9_\.]+) *?\}`)

    type OsEnvValueProvider struct{}

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

It allows the ``${}`` syntax to be used in configuration files to resolve environment variables.

Connection Pool Configuration
-----------------------------

Juice provides connection pool settings to help optimize database connection management:

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

Connection pool parameters:

- ``maxIdleConnNum``: maximum number of idle connections
- ``maxOpenConnNum``: maximum number of open connections
- ``maxConnLifetime``: maximum connection lifetime in seconds
- ``maxIdleConnLifetime``: maximum idle connection lifetime in seconds

Global Settings
---------------

The ``settings`` tag is used to configure Juice-wide behavior:

.. code-block:: xml

    <?xml version="1.0" encoding="UTF-8"?>
    <configuration>
        <settings>
            <setting name="debug" value="false"/>
        </settings>
    </configuration>

``settings`` characteristics:

- It is optional.
- It supports multiple ``setting`` children.
- Each ``setting`` must contain the ``name`` attribute.
- The ``value`` attribute is optional.
- Configuration values can be consumed by middleware and other components.

For example, the ``debug`` setting is used by ``DebugMiddleware`` to control whether debug mode is enabled.

Connection Pool Tuning Guide
----------------------------

Connection pools are a critical factor in database application performance. Good pool settings can significantly improve both performance and stability.

Tuning Principles
~~~~~~~~~~~~~~~~~

1. **Tune according to real load**

   Different workloads need different pool sizes:

   - Web applications: high concurrency, larger pools
   - Batch jobs: low concurrency, small pools
   - Microservices: tune according to service size and call frequency

2. **Avoid over-provisioning**

   Pools that are too large can:

   - consume too many database resources
   - put unnecessary pressure on the database server
   - waste application server memory

3. **Monitor and adjust continuously**

   Track these indicators over time:

   - pool utilization
   - connection wait time
   - connection creation and destruction frequency
   - database server load

Recommended Configurations for Different Scenarios
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Web applications with high concurrency**

.. code-block:: xml

    <environment id="web-prod">
        <dataSource>root:password@tcp(localhost:3306)/database</dataSource>
        <driver>mysql</driver>
        <maxOpenConnNum>200</maxOpenConnNum>
        <maxIdleConnNum>50</maxIdleConnNum>
        <maxConnLifetime>1800</maxConnLifetime>
        <maxIdleConnLifetime>600</maxIdleConnLifetime>
    </environment>

**Batch jobs with low concurrency and long runtimes**

.. code-block:: xml

    <environment id="batch">
        <dataSource>root:password@tcp(localhost:3306)/database</dataSource>
        <driver>mysql</driver>
        <maxOpenConnNum>20</maxOpenConnNum>
        <maxIdleConnNum>5</maxIdleConnNum>
        <maxConnLifetime>7200</maxConnLifetime>
        <maxIdleConnLifetime>3600</maxIdleConnLifetime>
    </environment>

**Microservices with medium concurrency**

.. code-block:: xml

    <environment id="microservice">
        <dataSource>root:password@tcp(localhost:3306)/database</dataSource>
        <driver>mysql</driver>
        <maxOpenConnNum>50</maxOpenConnNum>
        <maxIdleConnNum>10</maxIdleConnNum>
        <maxConnLifetime>900</maxConnLifetime>
        <maxIdleConnLifetime>300</maxIdleConnLifetime>
    </environment>

**Development environments**

.. code-block:: xml

    <environment id="dev">
        <dataSource>./dev.db</dataSource>
        <driver>sqlite3</driver>
        <maxOpenConnNum>10</maxOpenConnNum>
        <maxIdleConnNum>2</maxIdleConnNum>
        <maxConnLifetime>3600</maxConnLifetime>
    </environment>

Parameter Reference
~~~~~~~~~~~~~~~~~~~

.. list-table::
   :header-rows: 1
   :widths: 25 15 60

   * - Parameter
     - Default
     - Description and recommendation
   * - maxOpenConnNum
     - Unlimited
     - **Maximum open connections**

       - Caps the total number of simultaneously open database connections
       - Recommended value depends on server capacity and application concurrency
       - Typical range: 50-200 for web apps, 10-50 for batch jobs
       - Setting it too high consumes excessive database resources
   * - maxIdleConnNum
     - 2
     - **Maximum idle connections**

       - Controls how many idle connections remain in the pool
       - Recommended value: 25%-50% of ``maxOpenConnNum``
       - Too low causes frequent connection creation and destruction
       - Too high wastes resources
   * - maxConnLifetime
     - Permanent
     - **Maximum connection lifetime in seconds**

       - Maximum time before a connection is forcibly closed
       - Recommended range: 1800-7200
       - Helps avoid issues caused by long-lived connections
       - ``0`` means never expire, which is usually not recommended
   * - maxIdleConnLifetime
     - Permanent
     - **Maximum idle connection lifetime in seconds**

       - Controls how long an idle connection may stay in the pool
       - Recommended range: 300-3600
       - Should be less than ``maxConnLifetime``
       - Helps release inactive connections in time

Sizing the Connection Pool
~~~~~~~~~~~~~~~~~~~~~~~~~~

**Basic formulas**

.. code-block:: text

    maxOpenConnNum = (number of CPU cores x 2) + number of effective disks

    or

    maxOpenConnNum = concurrent requests x average connection hold time per request / request interval

**Worked example**

Assume your application:

- runs on a 4-core server
- uses SSD storage, treated as 1 effective disk
- expects 100 QPS
- holds each connection for 50 ms on average
- receives requests every 10 ms

Method 1: ``maxOpenConnNum = (4 x 2) + 1 = 9`` for a conservative estimate

Method 2: ``maxOpenConnNum = 100 x 0.05 / 0.01 = 500`` for a theoretical upper bound

**Practical advice**: start with a smaller value, such as 50, and adjust upward based on observed metrics.

Performance Monitoring
~~~~~~~~~~~~~~~~~~~~~~

**Metrics to watch**

.. code-block:: go

    stats := engine.DB().Stats()

    fmt.Printf("Max open connections: %d\n", stats.MaxOpenConnections)
    fmt.Printf("Open connections: %d\n", stats.OpenConnections)
    fmt.Printf("In use: %d\n", stats.InUse)
    fmt.Printf("Idle: %d\n", stats.Idle)
    fmt.Printf("Wait count: %d\n", stats.WaitCount)
    fmt.Printf("Wait duration: %v\n", stats.WaitDuration)
    fmt.Printf("Max idle closed: %d\n", stats.MaxIdleClosed)
    fmt.Printf("Max lifetime closed: %d\n", stats.MaxLifetimeClosed)

**Monitoring example**

.. code-block:: go

    type ConnectionPoolMonitor struct {
        interval time.Duration
    }

    func (m *ConnectionPoolMonitor) Start(engine *juice.Engine) {
        ticker := time.NewTicker(m.interval)
        go func() {
            for range ticker.C {
                stats := engine.DB().Stats()

                usage := float64(stats.InUse) / float64(stats.MaxOpenConnections) * 100
                if usage > 80 {
                    log.Printf("[WARNING] connection pool usage is high: %.2f%%", usage)
                }

                if stats.WaitCount > 0 {
                    avgWait := stats.WaitDuration / time.Duration(stats.WaitCount)
                    if avgWait > 100*time.Millisecond {
                        log.Printf("[WARNING] average connection wait time is too high: %v", avgWait)
                    }
                }
            }
        }()
    }

Common Troubleshooting Scenarios
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Issue 1: connection pool exhaustion**

Symptoms:

- slower application responses
- many requests waiting for connections
- steadily increasing ``stats.WaitCount``

Solutions:

1. Increase ``maxOpenConnNum``.
2. Check for connection leaks.
3. Optimize slow queries to reduce connection hold time.
4. Add pool monitoring.

**Issue 2: frequent connection creation and destruction**

Symptoms:

- rapidly increasing ``stats.MaxIdleClosed``
- unstable CPU usage
- large swings in database connection count

Solutions:

1. Increase ``maxIdleConnNum``.
2. Extend ``maxIdleConnLifetime``.
3. Evaluate whether pool warmup is necessary.

**Issue 3: database connection timeout**

Symptoms:

- recurring "connection timeout" errors
- connection failures after long runtimes

Solutions:

1. Set a reasonable ``maxConnLifetime``.
2. Ensure it is lower than the database server timeout.
3. Add connection health checks.

**Issue 4: excessive memory usage**

Symptoms:

- continuously increasing memory usage
- too many idle connections

Solutions:

1. Reduce ``maxIdleConnNum``.
2. Shorten ``maxIdleConnLifetime``.
3. Check for connection leaks.

Best Practices
~~~~~~~~~~~~~~

**1. Warm up the pool**

.. code-block:: go

    func warmupConnectionPool(engine *juice.Engine, size int) error {
        db := engine.DB()

        var conns []*sql.Conn
        for i := 0; i < size; i++ {
            conn, err := db.Conn(context.Background())
            if err != nil {
                return err
            }
            conns = append(conns, conn)
        }

        for _, conn := range conns {
            if err := conn.PingContext(context.Background()); err != nil {
                return err
            }
        }

        for _, conn := range conns {
            conn.Close()
        }

        return nil
    }

**2. Run connection health checks**

.. code-block:: go

    func healthCheck(engine *juice.Engine) error {
        ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
        defer cancel()

        if err := engine.DB().PingContext(ctx); err != nil {
            return fmt.Errorf("database health check failed: %w", err)
        }

        return nil
    }

**3. Shut down gracefully**

.. code-block:: go

    func gracefulShutdown(engine *juice.Engine) {
        // Stop accepting new requests.
        // ...

        // Wait for in-flight requests to complete.
        time.Sleep(5 * time.Second)

        if err := engine.Close(); err != nil {
            log.Printf("failed to close connection pool: %v", err)
        }
    }

**4. Isolate environments**

.. code-block:: xml

    <configuration>
        <environments default="prod">
            <environment id="prod">
                <maxOpenConnNum>200</maxOpenConnNum>
                <maxIdleConnNum>50</maxIdleConnNum>
            </environment>

            <environment id="test">
                <maxOpenConnNum>50</maxOpenConnNum>
                <maxIdleConnNum>10</maxIdleConnNum>
            </environment>

            <environment id="dev">
                <maxOpenConnNum>10</maxOpenConnNum>
                <maxIdleConnNum>2</maxIdleConnNum>
            </environment>
        </environments>
    </configuration>

**5. Add monitoring and alerts**

Recommended alert thresholds:

- connection pool usage above 80%
- average wait time above 100 ms
- connection creation failure rate above 1%
- idle connection count below 20% of the configured value

Configuration Checklist
~~~~~~~~~~~~~~~~~~~~~~~

Before deploying to production, confirm the following:

.. code-block:: text

    [ ] Is maxOpenConnNum set according to real load?
    [ ] Is maxIdleConnNum 25%-50% of maxOpenConnNum?
    [ ] Is maxConnLifetime lower than the database server timeout?
    [ ] Is maxIdleConnLifetime lower than maxConnLifetime?
    [ ] Is connection pool monitoring in place?
    [ ] Are alert thresholds configured?
    [ ] Has load testing been performed?
    [ ] Is there a pool warmup mechanism?
    [ ] Is graceful shutdown implemented?
    [ ] Are settings differentiated by environment?

.. tip::

    Tuning suggestions:

    1. Start with conservative settings.
    2. Collect real metrics through monitoring.
    3. Adjust gradually.
    4. Review and retune periodically.
    5. Record why each change was made and what effect it had.

.. warning::

    Common mistakes:

    - believing bigger connection pools are always better
    - using the same settings in every environment
    - setting values once and never revisiting them
    - ignoring monitoring and alerting
    - forgetting database server-side limits
