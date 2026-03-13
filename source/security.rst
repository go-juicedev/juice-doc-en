Security Guide
================

Security Baseline
--------------------------------

Juice does not replace application security controls. It is best to think about security in three layers:

1. SQL construction layer, including parameter binding and dynamic fragments.
2. Application layer, including input validation, authorization, and auditing.
3. Database layer, including account permissions and connection policies.

Parameter Binding Rules (Required)
---------------------------------------

Prefer ``#{}`` in mappers:

- ``#{name}``: prepared parameter binding through placeholders. Recommended.
- ``${name}``: direct string interpolation. This carries injection risk and should only be used in controlled scenarios.

Safe example:

.. code-block:: text

    <select id="GetUsers">
      SELECT id, name
      FROM users
      WHERE name = #{name}
    </select>

High-risk example to avoid:

.. code-block:: text

    <select id="DangerousSearch">
      SELECT * FROM users WHERE ${rawWhere}
    </select>

Safe Patterns for Dynamic SQL
--------------------------------

**1) Whitelist dynamic sorting**

.. code-block:: go

    var orderByWhitelist = map[string]string{
        "created_at": "created_at",
        "name":       "name",
        "id":         "id",
    }

    func SafeOrderBy(input string) string {
        if v, ok := orderByWhitelist[input]; ok {
            return v
        }
        return "created_at"
    }

Then pass only the sanitized value to the mapper. Even if ``${}`` is used, the value still comes from a whitelist.

**2) Whitelist dynamic table names**

In sharding scenarios, do not trust request parameters directly. Map them to known table names first.

**3) Limit IN-list length**

Whether you use ``foreach`` or handwritten SQL, cap list sizes to avoid oversized SQL statements and resource exhaustion.

Logging and Auditing
--------------------------------

- Avoid logging sensitive parameters in production, such as phone numbers, ID numbers, tokens, and passwords.
- Add a masking middleware before the debug middleware when possible.
- Record audit fields for critical write operations, such as operator, request ID, timestamp, and source.

Principle of Least Privilege for Database Access
------------------------------------------------

- Use separate accounts for reads and writes. Read-only accounts should have ``SELECT`` only, and write accounts should have only the necessary ``DML`` permissions.
- Do not grant dangerous permissions such as ``DROP`` or ``ALTER`` to application accounts.
- Split database accounts by service to avoid sharing high-privilege credentials.

Transactions and Security
--------------------------------

- Do not treat cross-datasource writes as a single local transaction. See :doc:`multi_source_tx`.
- For critical paths such as payments and inventory, combine idempotency keys with unique constraints to prevent replay and concurrent double execution.

Security Checklist Before Release
---------------------------------

- Check whether any ``${}`` usage in ``mapper`` files is not protected by a whitelist.
- Check whether debug logs leak sensitive parameters.
- Check whether database account permissions are minimized.
- Check whether high-risk endpoints have authorization, rate limiting, and auditing.
- Check whether dynamic queries have proper limits for pagination, IN-list length, and timeouts.
