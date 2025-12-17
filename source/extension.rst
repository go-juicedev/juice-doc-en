Extensions
==========

Embed Configuration Files into Executable
-----------------------------------------

In the beginning, we used the following code to load configuration files:

.. code-block:: go

    cfg, err := juice.NewXMLConfiguration("config.xml")

This method has a problem; after compiling the Go program, it depends on this configuration file to run, and they are not a single cohesive unit.

With the addition of the `embed` package in the Go 1.16 standard library, developers can now package static files directly into the executable. Juice also supports this capability:

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

Note: If your mappers refer to other mapper files, all referred mapper files also need to be embedded:

Example:

.. code-block:: xml

    <mappers>
        <mapper resource="mappers.xml"/>
    </mappers>

The best practice is to place configuration files under one folder:

.. code-block:: shell

    ├── config
        ├── config.xml
        └── mappers.xml

As shown above, the file structure should be organized accordingly. You only need to adjust your code slightly:

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

This modification addresses the issue mentioned above.

Read-Write Separation
---------------------

Juice provides a powerful implementation for read-write separation to optimize database performance and scalability.

Configuring Multiple Data Sources
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

First, configure multiple data sources in the configuration file:

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

By default, Juice only connects to the data source specified by the ``default`` attribute in ``environments``.
It is recommended to set ``master`` as the default data source to handle write operations.

Enabling Read-Write Separation
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To enable read-write separation, you need to use the ``TxSensitiveDataSourceSwitchMiddleware`` middleware:

.. code-block:: go

    var engine *juice.Engine
    ... // initialization
    engine.Use(&juice.TxSensitiveDataSourceSwitchMiddleware{})

Routing Strategy
~~~~~~~~~~~~~~~~

Juice supports multiple read operation routing strategies, which can be configured at the statement level or global level:

Global Configuration
^^^^^^^^^^^^^^^^^^^^

Configure the default routing strategy in the project's ``settings``:

.. code-block:: xml

    <settings>
        <setting name="selectDataSource" value="?"/>
    </settings>

Statement Level Configuration
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

1. **Specify Data Source**

   Explicitly specify a slave database for reading:

   .. code-block:: xml

        <select id="GetUserByID" dataSource="slave1">
            select * from user where id = #{id}
        </select>

2. **Random Routing**

   Use ``?`` to randomly select from all available data sources (including the master):

   .. code-block:: xml

        <select id="GetUserByID" dataSource="?">
            select * from user where id = #{id}
        </select>

3. **Slave-Only Random Routing**

   Use ``?!`` to randomly select from slave databases (excluding the master):

   .. code-block:: xml

        <select id="GetUserByID" dataSource="?!">
            select * from user where id = #{id}
        </select>

Note: Statement-level configuration has higher priority than global configuration. If the ``dataSource`` attribute is not configured for a statement, the ``selectDataSource`` value in the global configuration is used.


Transaction Safety
~~~~~~~~~~~~~~~~~~

The middleware is transaction-aware. When it detects that the current operation is within a transaction, it will not switch data sources and will directly use the data source of the current transaction to ensure transaction integrity and data consistency.

Best Practices
~~~~~~~~~~~~~~

1. Use the master database for all write operations.
2. Use ``?!`` for read-intensive operations to distribute load among slave databases.
3. Use explicit routing (e.g., ``slave1``, ``slave2``) when specific slave features are needed.
4. Use ``?`` to include the master database in load balancing when read consistency requirements are not high.

Use Cases
~~~~~~~~~

Read-write separation is particularly suitable for:

- Read-intensive scenarios
- Scenarios requiring expanded read capacity
- Scenarios needing to reduce master database load
- Improving overall application performance


Tracing Middleware
------------------

Similar to read-write separation, tracing functionality can be added non-intrusively using middleware. Here is a pseudocode example:

.. code-block:: go

    type TraceMiddleware struct{}

    func (r TraceMiddleware) QueryContext(_ *juice.Statement, next juice.QueryHandler) juice.QueryHandler {
        return func(ctx context.Context, query string, args ...any) (*sql.Rows, error) {
            trace.Log(ctx, "query", query) // your own trace
            return next(ctx, query, args...)
        }
    }

    func (r TraceMiddleware) ExecContext(stmt *juice.Statement, next juice.ExecHandler) juice.ExecHandler {
        return func(ctx context.Context, query string, args ...any) (sql.Result, error) {
            trace.Log(ctx, "exec", query) // your own trace
            return next(ctx, query, args...)
        }
    }


XML Document Constraint
-----------------------

DTD
~~~

XML Document Type Definition (DTD) is a language used to define the structure and rules of an XML document. By using a DTD, you can constrain an XML document to include specific elements, attributes, relationships, and sequences.

In practical applications, it is common to associate a DTD file with an XML document to validate it during parsing. In the XML document, you can specify the DTD file and its location using the <!DOCTYPE> element.

For example, in the Juice configuration files like `config.xml` or `mapper.xml`, you can associate a DTD file by specifying the PUBLIC attribute and URI in the <!DOCTYPE> element. This allows editors or other tools to check the XML document against the defined DTD rules and identify potential errors and issues.

config xml

.. code-block:: xml

    <?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE configuration PUBLIC "-//juice.org//DTD Config 1.0//EN"
            "https://raw.githubusercontent.com/go-juicedev/juice/refs/heads/main/config.dtd">

mapper xml

.. code-block:: xml

    <?xml version="1.0" encoding="utf-8" ?>
    <!DOCTYPE mapper PUBLIC "-//juice.org//DTD Config 1.0//EN"
            "https://raw.githubusercontent.com/go-juicedev/juice/refs/heads/main/mapper.dtd">


XSD
~~~

XML Schema Definition (XSD) is the successor to DTD, providing a more powerful and flexible mechanism for XML document validation. Compared to DTD, XSD offers the following advantages:

1. **Type System**
   
   - Supports richer data types
   - Allows custom complex types
   - Supports type inheritance and extension

2. **Namespace Support**
   
   - Better modularity support
   - Avoids naming conflicts
   - Easier to manage large XML structures

3. **Readability**
   
   - Written in XML syntax
   - Easier to understand and maintain
   - Better tool support

Using XSD in Juice:

.. code-block:: xml

    <?xml version="1.0" encoding="UTF-8"?>
    <configuration
        xmlns="http://github.com/go-juciedev/juice/schema"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://github.com/go-juciedev/juice/schema
                  https://raw.githubusercontent.com/go-juicedev/juice/refs/heads/main/juice-mapper.xsd">