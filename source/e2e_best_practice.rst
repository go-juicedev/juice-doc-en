End-to-End Best Practices
===========================

When to Use This Template
--------------------------------

This page provides a complete template from configuration to invocation. It is intended for small and medium-sized services that want a practical starting point.

Recommended Directory Layout
--------------------------------

.. code-block:: text

    .
    ├── cmd/server/main.go
    ├── internal/
    │   ├── app/
    │   │   └── bootstrap.go
    │   ├── repo/
    │   │   ├── user.go
    │   │   └── user_impl.go
    │   ├── service/
    │   │   └── user_service.go
    │   └── transport/http/
    │       └── handler.go
    ├── config/
    │   └── config.xml
    └── mapper/
        └── user_mapper.xml

Configuration Recommendations
--------------------------------

- Set a clear ``default`` datasource in ``config.xml``.
- For read-write splitting, use ``master`` for writes and ``slave`` for reads.
- Standardize connection pool parameters and timeout policies. See :doc:`configuration` for details.

Bootstrap and Dependency Injection
-----------------------------------

.. code-block:: go

    func Bootstrap() (*juice.Engine, error) {
        cfg, err := juice.NewXMLConfiguration("config/config.xml")
        if err != nil {
            return nil, err
        }

        engine, err := juice.Default(cfg)
        if err != nil {
            return nil, err
        }

        // Add custom middleware as needed, such as masking, tracing,
        // or slow-query alerts.
        // engine.Use(yourMiddleware)

        return engine, nil
    }

Recommended Request Flow
--------------------------------

1. Validate input and perform authorization in the HTTP layer.
2. Define the transaction boundary in the service layer.
3. Keep the repository layer focused on mapper calls and data mapping.
4. Convert errors consistently into business error codes.

.. code-block:: go

    func (s *UserService) CreateUser(ctx context.Context, req CreateUserReq) error {
        ctx = juice.ContextWithManager(ctx, s.engine)

        return juice.Transaction(ctx, func(ctx context.Context) error {
            if _, err := s.userRepo.Create(ctx, req.ToEntity()); err != nil {
                return err
            }
            return nil
        })
    }

Read-Write Splitting in Practice
--------------------------------

- Write path: inject the ``master`` engine into the context and open the transaction there.
- Read path: use ``engine.With("slave")`` for standalone queries, usually without a transaction.
- Avoid switching to another datasource for writes inside the same transaction callback.

Error Handling Conventions
--------------------------------

- Repository layer: preserve underlying errors and add SQL action context only when necessary.
- Service layer: map technical errors to domain errors.
- Transport layer: map domain errors to HTTP or gRPC responses.

Observability Recommendations
--------------------------------

- Enable ``DebugMiddleware`` in development to simplify troubleshooting.
- In production, consider:

  - masking SQL logs
  - alerting on slow-query thresholds
  - propagating a request ID through the full call chain

Pre-Launch Checklist
--------------------------------

- Verify that mapper methods and interface signatures are aligned one-to-one.
- Verify that all critical write paths are inside transaction boundaries.
- Verify that there is no uncontrolled ``${}`` interpolation.
- Verify that connection pool parameters are tuned based on load-testing results.
- Verify that the examples in the documentation still match the current API version.
