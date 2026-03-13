Middleware
==========

What Middleware Is
------------------

In Juice, middleware is defined as an interface:

.. code-block:: go

    type QueryHandler func(ctx context.Context, query string, args ...any) (*sql.Rows, error)

    type ExecHandler func(ctx context.Context, query string, args ...any) (sql.Result, error)

    type Middleware interface {
        QueryContext(stmt Statement, configuration Configuration, next QueryHandler) QueryHandler
        ExecContext(stmt Statement, configuration Configuration, next ExecHandler) ExecHandler
    }

Middleware lets you intercept SQL execution before it reaches the database, which makes it suitable for logging, caching, timeout control, datasource switching, and other cross-cutting concerns.

When the statement is a query, ``QueryContext`` is used. Otherwise, ``ExecContext`` is used.

Juice ships with some built-in middleware. For example:

- :class:`juice/middleware/DebugMiddleware` for printing SQL statements and timing information

.. code-block:: go

    var logger = log.New(log.Writer(), "[juice] ", log.Flags())

    type DebugMiddleware struct{}

    func (m *DebugMiddleware) QueryContext(stmt Statement, configuration Configuration, next QueryHandler) QueryHandler {
        if !m.isDeBugMode(stmt, configuration) {
            return next
        }
        return func(ctx context.Context, query string, args ...any) (*sql.Rows, error) {
            start := time.Now()
            rows, err := next(ctx, query, args...)
            spent := time.Since(start)
            logger.Printf("\x1b[33m[%s]\x1b[0m args: \u001B[34m%v\u001B[0m time: \u001B[31m%v\u001B[0m \x1b[32m%s\x1b[0m",
                stmt.Name(), query, args, spent, query)
            return rows, err
        }
    }

    func (m *DebugMiddleware) ExecContext(stmt Statement, configuration Configuration, next ExecHandler) ExecHandler {
        if !m.isDeBugMode(stmt, configuration) {
            return next
        }
        return func(ctx context.Context, query string, args ...any) (sql.Result, error) {
            start := time.Now()
            rows, err := next(ctx, query, args...)
            spent := time.Since(start)
            logger.Printf("\x1b[33m[%s]\x1b[0m args: \u001B[34m%v\u001B[0m time: \u001B[31m%v\u001B[0m \x1b[32m%s\x1b[0m",
                stmt.Name(), query, args, spent, query)
            return rows, err
        }
    }

    func (m *DebugMiddleware) isDeBugMode(stmt Statement, configuration Configuration) bool {
        debug := stmt.Attribute("debug")
        if debug == "false" {
            return false
        }
        if configuration.Settings().Get("debug") == "false" {
            return false
        }
        return true
    }

When this middleware is enabled, Juice writes the SQL statement, bound arguments, and elapsed time to the default writer of the standard ``log`` package.

If you want to disable it globally, set ``debug`` to ``false`` in ``settings``:

.. code-block:: xml

    <settings>
        <setting name="debug" value="false"/>
    </settings>

If you want to disable it for a single action only, configure that action directly:

.. code-block:: xml

    <insert id="xxx" debug="false">
    </insert>

- :class:`juice/middleware/TimeoutMiddleware` for controlling SQL execution timeout

.. code-block:: go

    type TimeoutMiddleware struct{}

    func (t TimeoutMiddleware) QueryContext(stmt Statement, _ Configuration, next QueryHandler) QueryHandler {
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

    func (t TimeoutMiddleware) ExecContext(stmt Statement, _ Configuration, next ExecHandler) ExecHandler {
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

    func (t TimeoutMiddleware) getTimeout(stmt Statement) (timeout int64) {
        timeoutAttr := stmt.Attribute("timeout")
        if timeoutAttr == "" {
            return
        }
        timeout, _ = strconv.ParseInt(timeoutAttr, 10, 64)
        return
    }

To enable it, add the ``timeout`` attribute to the action tag. The unit is milliseconds:

.. code-block:: xml

    <insert id="xxx" timeout="1000">
    </insert>

.. attention::

    ``TimeoutMiddleware`` applies timeouts at the Go context level, not at the database server level.

Custom Middleware
-----------------

To implement custom middleware, simply implement the ``Middleware`` interface and register it on the engine:

.. code-block:: go

    func main() {
        var mymiddleware Middleware = yourMiddlewareImpl{}

        cfg, err := juice.NewXMLConfiguration("config.xml")
        if err != nil {
            panic(err)
        }

        engine, err := juice.Default(cfg)
        if err != nil {
            panic(err)
        }

        engine.Use(mymiddleware)
    }
