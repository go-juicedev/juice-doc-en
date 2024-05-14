Middleware
==========

What is Middleware
------------------

In juice, middleware is an interface which is described as follows:

.. code:: go

    // QueryHandler defines the handler of the query.
    type QueryHandler func(ctx context.Context, query string, args ...any) (*sql.Rows, error)

    // ExecHandler defines the handler of the exec.
    type ExecHandler func(ctx context.Context, query string, args ...any) (sql.Result, error)

    // Middleware is a wrapper of QueryHandler and ExecHandler.
    type Middleware interface {
        // QueryContext wraps the QueryHandler.
        QueryContext(stmt Statement, next QueryHandler) QueryHandler

        // ExecContext wraps the ExecHandler.
        ExecContext(stmt Statement, next ExecHandler) ExecHandler
    }

The role of middleware is to process SQL statements before execution, such as logging SQL statements, caching SQL queries, etc. When a query operation is performed, ``QueryContext`` is executed. Otherwise, ``ExecContext`` is used.

Juice comes with some built-in middleware, such as:

- :class:`juice/middleware/DebugMiddleware`: A middleware for printing SQL statements.

.. code:: go

    // logger is a default logger for debug.
    var logger = log.New(log.Writer(), "[juice] ", log.Flags())

    // DebugMiddleware is a middleware that prints the SQL statement and the execution time.
    type DebugMiddleware struct{}

    // QueryContext implements Middleware.
    // QueryContext will print the SQL statement and the execution time.
    func (m *DebugMiddleware) QueryContext(stmt Statement, next QueryHandler) QueryHandler {
        if !m.isBugMode(stmt) {
            return next
        }
        return func(ctx context.Context, query string, args ...any) (*sql.Rows, error) {
            start := time.Now()
            rows, err := next(ctx, query, args...)
            spent := time.Since(start)
            logger.Printf("\x1b[33m[%s]\x1b[0m \x1b[32m %s\x1b[0m \x1b[34m %v\x1b[0m \x1b[31m %v\x1b[0m\n", stmt.Name(), query, args, spent)
            return rows, err
        }
    }

    // ExecContext implements Middleware.
    // ExecContext will print the SQL statement and the execution time.
    func (m *DebugMiddleware) ExecContext(stmt Statement, next ExecHandler) ExecHandler {
        if !m.isBugMode(stmt) {
            return next
        }
        return func(ctx context.Context, query string, args ...any) (sql.Result, error) {
            start := time.Now()
            rows, err := next(ctx, query, args...)
            spent := time.Since(start)
            logger.Printf("\x1b[33m[%s]\x1b[0m \x1b[32m %s\x1b[0m \x1b[34m %v\x1b[0m \x1b[31m %v\x1b[0m\n", stmt.Name(), query, args, spent)
            return rows, err
        }
    }

    // isBugMode returns true if the debug mode is on.
    // Default debug mode is on.
    // You can turn off the debug mode by setting the debug tag to false in the mapper statement attribute or the configuration.
    func (m *DebugMiddleware) isBugMode(stmt Statement) bool {
        debug := stmt.Attribute("debug")
        if debug == "false" {
            return false
        }
        if cfg := stmt.Configuration(); cfg.Settings.Get("debug") == "false" {
            return false
        }
        return true
    }

When you enable this middleware, juice logs each executed SQL statement and its parameters to the default logger of the log package (usually the console) and also logs the execution time.

You can disable this middleware globally by setting debug to false in the settings as shown below:

.. code:: xml

    <settings>
        <setting name="debug" value="false"/>
    </settings>

If you want to disable this feature locally, configure it on the corresponding action as follows:

.. code:: xml

    <insert id="xxx" debug="false"></insert>

- :class:`juice/middleware/TimeoutMiddleware`: A middleware for controlling SQL execution timeout.

.. code-block:: go

    // TimeoutMiddleware is a middleware that sets the timeout for the SQL statement.
    type TimeoutMiddleware struct{}

    // QueryContext implements Middleware.
    // QueryContext will set the timeout for the SQL statement.
    func (t TimeoutMiddleware) QueryContext(stmt Statement, next QueryHandler) QueryHandler {
        timeout := t.getTimeout(stmt)
        if timeout <= 0 {
            return next
        }
        return func(ctx context.Context, query string, args ...any) (*sql.Rows, error) {
            ctx, cancel := context.WithTimeout(ctx, time.Duration(timeout)*time.Millisecond)
            defer cancel()
            return next(ctx, query, args...)
        }
    }

    // ExecContext implements Middleware.
    // ExecContext will set the timeout for the SQL statement.
    func (t TimeoutMiddleware) ExecContext(stmt Statement, next ExecHandler) ExecHandler {
        timeout := t.getTimeout(stmt)
        if timeout <= 0 {
            return next
        }
        return func(ctx context.Context, query string, args ...any) (sql.Result, error) {
            ctx, cancel := context.WithTimeout(ctx, time.Duration(timeout)*time.Millisecond)
            defer cancel()
            return next(ctx, query, args...)
        }
    }

    // getTimeout returns the timeout from the Statement.
    func (t TimeoutMiddleware) getTimeout(stmt Statement) (timeout int64) {
        timeoutAttr := stmt.Attribute("timeout")
        if timeoutAttr == "" {
            return
        }
        timeout, _ = strconv.ParseInt(timeoutAttr, 10, 64)
        return
    }

To enable this feature, add the timeout attribute to the corresponding action tag. The time unit is milliseconds, as follows:

.. code-block:: xml

    <insert id="xxx" timeout="1000"></insert>

.. attention::

    Note: The TimeoutMiddleware implements timeout at the Go language level, not at the database level.

Creating Custom Middleware
--------------------------

To create custom middleware, you just need to implement the ``Middleware`` interface, and then register it with the corresponding engine as shown below:

.. code-block:: go

    func main() {
        var mymiddleware Middleware = yourMiddlewareImpl{}
        cfg, err := juice.NewXMLConfiguration("config.xml")
        if err != nil {
            panic(err)
        }
        engine, err := juice.DefaultEngine(cfg)
        if err != nil {
            panic(err)
        }
        engine.Use(mymiddleware)
    }

