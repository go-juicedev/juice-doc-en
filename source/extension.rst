Extensions
==========

Embedding Configuration Files into the Executable
-------------------------------------------------

The initial way to load configuration is usually:

.. code-block:: go

    cfg, err := juice.NewXMLConfiguration("config.xml")

This works, but it means the compiled Go program still depends on an external configuration file at runtime.

Since Go 1.16, the standard library includes ``embed``, which allows static files to be bundled into the executable. Juice supports this as well.

.. code-block:: go

    package main

    import (
        "embed"
        "fmt"

        "github.com/go-juicedev/juice"
    )

    //go:embed config.xml
    var fs embed.FS

    func main() {
        cfg, err := juice.NewXMLConfigurationWithFS(fs, "config.xml")
        if err != nil {
            panic(err)
        }
        fmt.Println(cfg)
    }

If your ``mappers`` section references other mapper files, those files must also be embedded.

For example:

.. code-block:: xml

    <mappers>
        <mapper resource="mappers.xml"/>
    </mappers>

In that case, the best approach is to put all related configuration files into one directory:

.. code-block:: text

    config/
    ├── config.xml
    └── mappers.xml

Then update the code like this:

.. code-block:: go

    package main

    import (
        "embed"
        "fmt"

        "github.com/go-juicedev/juice"
    )

    //go:embed config
    var fs embed.FS

    func main() {
        cfg, err := juice.NewXMLConfigurationWithFS(fs, "config/config.xml")
        if err != nil {
            panic(err)
        }
        fmt.Println(cfg)
    }

That solves the problem of keeping referenced mapper files together with the executable.

Read-Write Splitting
--------------------

Juice provides a read-write splitting mechanism to improve database performance and scalability.

Configuring Multiple Datasources
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Start by defining multiple datasources in your configuration:

.. code-block:: xml

    <environments default="master">
        <environment id="master">
            <dataSource>root:qwe123@tcp(localhost:3306)/database</dataSource>
            <driver>mysql</driver>
        </environment>

        <environment id="slave1">
            <dataSource>root:qwe123@tcp(localhost:3307)/database</dataSource>
            <driver>mysql</driver>
        </environment>

        <environment id="slave2">
            <dataSource>root:qwe123@tcp(localhost:3308)/database</dataSource>
            <driver>mysql</driver>
        </environment>
    </environments>

By default, Juice connects only to the datasource selected by the ``default`` attribute of ``environments``. It is recommended to set ``master`` as the default for write operations.

Enabling Read-Write Splitting
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To enable read-write splitting, register ``TxSensitiveDataSourceSwitchMiddleware``:

.. code-block:: go

    var engine *juice.Engine
    // ...
    engine.Use(&juice.TxSensitiveDataSourceSwitchMiddleware{})

Routing Strategies
~~~~~~~~~~~~~~~~~~

Juice supports multiple routing strategies for read operations. They can be configured either globally or per statement.

Global Configuration
^^^^^^^^^^^^^^^^^^^^

Set the default routing strategy in ``settings``:

.. code-block:: xml

    <settings>
        <setting name="selectDataSource" value="?"/>
    </settings>

Statement-Level Configuration
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

1. **Explicit datasource**

   Read from a specific replica:

   .. code-block:: xml

        <select id="GetUserByID" dataSource="slave1">
            select * from user where id = #{id}
        </select>

2. **Random routing across all datasources**

   Use ``?`` to choose randomly from all available datasources, including the primary:

   .. code-block:: xml

        <select id="GetUserByID" dataSource="?">
            select * from user where id = #{id}
        </select>

3. **Random routing across replicas only**

   Use ``?!`` to choose randomly from replicas only:

   .. code-block:: xml

        <select id="GetUserByID" dataSource="?!">
            select * from user where id = #{id}
        </select>

Statement-level configuration has higher priority than global configuration. If a statement does not specify ``dataSource``, Juice falls back to the global ``selectDataSource`` setting.

Transaction Safety
~~~~~~~~~~~~~~~~~~

This middleware is transaction-aware. If the current operation is already running inside a transaction, Juice will not switch datasources and will keep using the transaction's datasource to preserve consistency.

Best Practices
~~~~~~~~~~~~~~

1. Send all write operations to the primary datasource.
2. Use ``?!`` for read-heavy workloads to spread traffic across replicas.
3. Use explicit routing such as ``slave1`` or ``slave2`` when a specific replica is required.
4. Use ``?`` when read consistency requirements are relaxed and the primary can join load balancing.

Typical Use Cases
~~~~~~~~~~~~~~~~~

Read-write splitting is especially useful when:

- read traffic is significantly higher than write traffic
- you need to scale read capacity
- you want to reduce primary database load
- you want to improve overall application performance

Tracing
-------

Just like read-write splitting, tracing can be added in a non-invasive way with middleware.

Pseudo-code example:

.. code-block:: go

    type TraceMiddleware struct{}

    func (r TraceMiddleware) QueryContext(_ *juice.Statement, next juice.QueryHandler) juice.QueryHandler {
        return func(ctx context.Context, query string, args ...any) (*sql.Rows, error) {
            trace.Log(ctx, "query", query)
            return next(ctx, query, args...)
        }
    }

    func (r TraceMiddleware) ExecContext(stmt *juice.Statement, next juice.ExecHandler) juice.ExecHandler {
        return func(ctx context.Context, query string, args ...any) (sql.Result, error) {
            trace.Log(ctx, "exec", query)
            return next(ctx, query, args...)
        }
    }

XML Document Constraints
------------------------

DTD
~~~

XML Document Type Definition, or DTD, is a language for defining the structure and rules of XML documents. With DTD, you can constrain which elements, attributes, relationships, and ordering are allowed in an XML document.

In practice, the DTD file is associated with an XML document so that editors and tooling can validate the document automatically. In XML, this is done through the ``<!DOCTYPE>`` declaration.

For Juice configuration and mapper files, you can reference the DTD like this.

Configuration XML:

.. code-block:: xml

    <?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE configuration PUBLIC "-//juice.org//DTD Config 1.0//EN"
            "https://raw.githubusercontent.com/go-juicedev/juice/refs/heads/main/config.dtd">

Mapper XML:

.. code-block:: xml

    <?xml version="1.0" encoding="utf-8" ?>
    <!DOCTYPE mapper PUBLIC "-//juice.org//DTD Config 1.0//EN"
            "https://raw.githubusercontent.com/go-juicedev/juice/refs/heads/main/mapper.dtd">

XSD
~~~

XML Schema Definition, or XSD, is the successor to DTD and provides a more powerful and flexible validation mechanism. Compared with DTD, XSD offers:

1. **A richer type system**

   - richer built-in data types
   - support for custom complex types
   - support for inheritance and extension

2. **Namespace support**

   - better modularity
   - fewer naming conflicts
   - easier management of large XML structures

3. **Better readability**

   - it is itself written in XML syntax
   - easier to understand and maintain
   - better tooling support

Using XSD in Juice:

.. code-block:: xml

    <?xml version="1.0" encoding="UTF-8"?>
    <configuration
        xmlns="http://github.com/go-juciedev/juice/schema"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://github.com/go-juciedev/juice/schema
                  https://raw.githubusercontent.com/go-juicedev/juice/refs/heads/main/juice-mapper.xsd">
