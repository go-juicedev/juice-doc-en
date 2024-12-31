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

Juice does not currently support this feature but it may in the future. Here is a concept for developers interested in implementing read-write separation using middleware:

.. code-block:: go

    // Middleware is a wrapper of QueryHandler and ExecHandler.
    type Middleware interface {
        // QueryContext wraps the QueryHandler.
        QueryContext(stmt *Statement, next QueryHandler) QueryHandler
        // ExecContext wraps the ExecHandler.
        ExecContext(stmt *Statement, next ExecHandler) ExecHandler
    }

Pseudocode for Read-Write Middleware:

.. code-block:: go

    type ReadWriteMiddleware struct {
        slaves []*sql.DB
        master *sql.DB
    }

    func (r ReadWriteMiddleware) QueryContext(_ *juice.Statement, next juice.QueryHandler) juice.QueryHandler {
        return func(ctx context.Context, query string, args ...any) (*sql.Rows, error) {
            index := rand.Intn(len(r.slaves))
            db := r.slaves[index]
            ctx = juice.SessionWithContext(ctx, db)
            return next(ctx, query, args...)
        }
    }

    func (r ReadWriteMiddleware) ExecContext(_ *juice.Statement, next juice.ExecHandler) juice.ExecHandler {
        return func(ctx context.Context, query string, args ...any) (sql.Result, error) {
            ctx = juice.SessionWithContext(ctx, r.master)
            return next(ctx, query, args...)
        }
    }

.. attention::
   Note: Although database read-write separation can improve application performance, it can also introduce transaction management issues, as the middleware implementation may override all sessions. If the current session involves a transaction, this might invalidate transaction operations. Specific business logic must be implemented by developers.

Tracing Middleware
------------------

Similarly to the read-write separation, tracing functionality can be added non-intrusively using middleware. Here is a pseudocode example:

.. code-block:: go

    type TraceMiddleware struct{}

    func (r TraceMiddleware) QueryContext(_ *juice.Statement, next juice.QueryHandler) juice.QueryHandler {
        return func(ctx context.Context, query string, args ...any) (*sql.Rows, error) {
            trace.Log(ctx, "query", query)  // your own trace implementation
            return next(ctx, query, args...)
        }
    }

    func (r TraceMiddleware) ExecContext(stmt *juice.Statement, next juice.ExecHandler) juice.ExecHandler {
        return func(ctx context.Context, query string, args ...any) (sql.Result, error) {
            trace.Log(ctx, "exec", query)  // your own trace implementation
            return next(ctx, query, args...)
        }
    }

XML Document Constraint
-----------------------

XML Document Type Definition (DTD) is a language used to define the structure and rules of an XML document. By using a DTD, you can constrain an XML document to include specific elements, attributes, relationships, and sequences.

In practical applications, it is common to associate a DTD file with an XML document to validate it during parsing. In the XML document, you can specify the DTD file and its location using the <!DOCTYPE> element.

For example, in the Juice configuration files like `config.xml` or `mapper.xml`, you can associate a DTD file by specifying the PUBLIC attribute and URI in the <!DOCTYPE> element. This allows editors or other tools to check the XML document against the defined DTD rules and identify potential errors and issues.

config xml:

.. code-block:: xml

    <?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE configuration PUBLIC "-//juice.org//DTD Config 1.0//EN"
    "https://raw.githubusercontent.com/eatmoreapple/juice/main/config.dtd">

mapper xml:

.. code-block:: xml

    <?xml version="1.0" encoding="utf-8" ?>
    <!DOCTYPE mapper PUBLIC "-//juice.org//DTD Config 1.0//EN"
    "https://raw.githubusercontent.com/eatmoreapple/juice/main/mapper.dtd">