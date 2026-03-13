Dynamic SQL
===========

.. note::

    Dynamic SQL is a powerful query technique that generates SQL statements at runtime according to changing conditions. It is particularly useful for conditional search, batch operations, and flexible data access. When used well, it improves reuse and maintainability while reducing redundant SQL.

Juice provides rich support for dynamic SQL through XML configuration. The main elements are described below.

Conditional Logic with if
-------------------------

The ``if`` element includes or skips a SQL fragment according to the result of an expression:

.. code-block:: xml

    <select id="SelectByCondition">
        select * from user
        <if test='name != ""'>
            where name = #{name}
        </if>
    </select>

.. tip::

    Conditional expressions support common operators and functions such as ``&&``, ``||``, ``!``, ``>``, ``<``, ``==``, and ``!=``.

Dynamic WHERE with where
------------------------

The ``where`` element manages the ``WHERE`` clause automatically. It adds ``WHERE`` only when needed and strips leading ``AND`` or ``OR``:

.. code-block:: xml

    <select id="SelectByCondition">
        select * from user
        <where>
            <if test='name != ""'>
                and name = #{name}
            </if>
            <if test='age > 0'>
                and age > #{age}
            </if>
        </where>
    </select>

.. note::

    ``where`` automatically handles the following cases:

    - no ``WHERE`` keyword is emitted when no condition matches
    - leading ``AND`` and ``OR`` are removed
    - the generated SQL remains syntactically valid

Dynamic SET with set
--------------------

The ``set`` element is designed for ``UPDATE`` statements. It manages the ``SET`` clause and removes trailing commas:

.. code-block:: xml

    <update id="UpdateUser">
        update user
        <set>
            <if test='name != ""'>
                name = #{name},
            </if>
            <if test='age > 0'>
                age = #{age},
            </if>
            <if test='email != null'>
                email = #{email},
            </if>
        </set>
        where id = #{id}
    </update>

.. tip::

    ``set`` automatically:

    - adds the ``SET`` keyword
    - removes the final extra comma
    - keeps the generated SQL valid

Iterating Collections with foreach
----------------------------------

The ``foreach`` element iterates over a collection. It is commonly used for ``IN`` conditions and batch operations:

.. code-block:: xml

    <select id="SelectByIds">
        select * from user where id in
        <foreach collection="ids" item="id" open="(" close=")" separator=",">
            #{id}
        </foreach>
    </select>

    <insert id="BatchInsert">
        insert into user (name, age) values
        <foreach collection="users" item="user" separator=",">
            (#{user.name}, #{user.age})
        </foreach>
    </insert>

.. note::

    ``foreach`` supports these attributes:

    - ``collection``: the collection to iterate
    - ``item``: the current element
    - ``index``: the current index
    - ``open``: opening text
    - ``close``: closing text
    - ``separator``: separator between items

Custom Wrapping with trim
-------------------------

The ``trim`` element lets you define custom prefix and suffix handling:

.. code-block:: xml

    <select id="SelectWithTrim">
        select * from user
        <trim prefix="where" prefixOverrides="and |or ">
            <if test='name != null'>
                and name like #{name}
            </if>
            <if test='age > 0'>
                or age > #{age}
            </if>
        </trim>
    </select>

.. tip::

    ``trim`` attributes:

    - ``prefix``: text to prepend
    - ``prefixOverrides``: prefixes to remove
    - ``suffix``: text to append
    - ``suffixOverrides``: suffixes to remove

Conditional Branching with choose
---------------------------------

The ``choose`` element works like a switch statement:

.. code-block:: xml

    <select id="SelectByChoice">
        select * from user
        <where>
            <choose>
                <when test='searchType == "name"'>
                    and name like concat('%', #{keyword}, '%')
                </when>
                <when test='searchType == "email"'>
                    and email = #{keyword}
                </when>
                <otherwise>
                    and id = #{keyword}
                </otherwise>
            </choose>
        </where>
    </select>

Reusing SQL Fragments with sql and include
------------------------------------------

The ``sql`` and ``include`` elements define reusable SQL fragments:

.. code-block:: xml

    <mapper namespace="user">
        <sql id="userColumns">
            id, name, age, email, create_time
        </sql>

        <select id="GetUsers">
            select
            <include refid="userColumns"/>
            from user
            where status = 1
        </select>
    </mapper>

Cross-Namespace References
~~~~~~~~~~~~~~~~~~~~~~~~~~

You can reference SQL fragments from another namespace:

.. code-block:: xml

    <mapper namespace="user">
        <sql id="userColumns">
            id, name, age, email, create_time
        </sql>
    </mapper>

    <mapper namespace="admin">
        <select id="GetUsers">
            select
            <include refid="user.userColumns"/>
            from user
            where status = 1
        </select>
    </mapper>

.. tip::

    The ``id`` attribute on ``sql`` is required and must be a valid identifier.

Parameterized SQL Fragments
~~~~~~~~~~~~~~~~~~~~~~~~~~~

``sql`` fragments can also be parameterized with ``bind``:

.. code-block:: xml

    <mapper namespace="user">
        <sql id="selectByField">
            select * from user where ${field} = #{value}
        </sql>

        <select id="GetUsersByDynamicField">
            <bind name="field" value='name'/>
            <bind name="value" value='"eatmoreapple"'/>
            <include refid="selectByField"/>
        </select>
    </mapper>

In this example, ``field`` and ``value`` are defined with ``bind`` and then used inside the SQL fragment.

Parameter Scope and Precedence
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Scope rules:

- ``bind`` can be declared at the top level of a statement or nested inside dynamic tags such as ``if``, ``where``, and ``foreach``.
- Nested ``bind`` variables have local scope and are visible only within the current tag and its children.
- Variables defined in a parent scope are visible to child tags.
- Variables defined in an inner scope override variables with the same name from an outer scope.

Parameter lookup precedence:

1. Parameters defined by ``bind``
2. User-provided parameters
3. Built-in system parameters such as ``_databaseId`` and ``_parameter``

That means a ``bind`` variable can override a user-provided parameter with the same name.

Advanced Examples
~~~~~~~~~~~~~~~~~

**1. String processing**

.. code-block:: xml

    <select id="SearchUsers">
        <bind name="searchPattern" value='"%" + name + "%"'/>
        <bind name="upperName" value='name.toUpperCase()'/>
        SELECT * FROM users
        WHERE name LIKE #{searchPattern} OR UPPER(name) = #{upperName}
    </select>

**2. Numeric calculation**

.. code-block:: xml

    <select id="GetPageUsers">
        <bind name="offset" value='pageNum * pageSize'/>
        <bind name="limit" value='pageSize'/>
        SELECT * FROM users ORDER BY id LIMIT #{limit} OFFSET #{offset}
    </select>

**3. Complex object handling**

.. code-block:: xml

    <select id="ComplexSearch">
        <bind name="userAge" value='user.age'/>
        <bind name="userName" value='user.name'/>
        <bind name="isActive" value='user.active'/>
        SELECT * FROM users
        WHERE age >= #{userAge} AND name = #{userName} AND active = #{isActive}
    </select>

**4. Scope override example**

.. code-block:: xml

    <select id="ScopedBindExample">
        <bind name="pattern" value='"%"'/>
        SELECT * FROM users
        <where>
            <if test='name != ""'>
                <bind name="pattern" value='"%" + name + "%"'/>
                AND name LIKE #{pattern}
            </if>
        </where>
        OR description LIKE #{pattern}
    </select>

.. tip::

    Dynamic SQL best practices:

    1. Reuse SQL fragments where it improves maintainability.
    2. Be aware of the runtime cost of complex conditions.
    3. Prefer parameterized queries to avoid SQL injection.
    4. Keep SQL readable.
    5. Add comments for complex dynamic SQL logic when needed.

Real-World Query Patterns
=========================

Multi-Condition Search
----------------------

E-commerce Product Search Example
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: xml

    <select id="SearchProducts">
        SELECT
            p.id, p.name, p.price, p.stock,
            c.name as category_name,
            b.name as brand_name
        FROM products p
        LEFT JOIN categories c ON p.category_id = c.id
        LEFT JOIN brands b ON p.brand_id = b.id
        <where>
            <if test='status != nil'>
                AND p.status = #{status}
            </if>

            <if test='keyword != ""'>
                AND (
                    p.name LIKE concat('%', #{keyword}, '%')
                    OR p.description LIKE concat('%', #{keyword}, '%')
                )
            </if>

            <if test='minPrice > 0'>
                AND p.price >= #{minPrice}
            </if>
            <if test='maxPrice > 0'>
                AND p.price &lt;= #{maxPrice}
            </if>

            <if test='categoryIds != nil and length(categoryIds) > 0'>
                AND p.category_id IN
                <foreach collection="categoryIds" item="id" open="(" close=")" separator=",">
                    #{id}
                </foreach>
            </if>

            <if test='brandIds != nil and length(brandIds) > 0'>
                AND p.brand_id IN
                <foreach collection="brandIds" item="id" open="(" close=")" separator=",">
                    #{id}
                </foreach>
            </if>

            <if test='inStock == true'>
                AND p.stock > 0
            </if>

            <if test='tags != nil and length(tags) > 0'>
                AND EXISTS (
                    SELECT 1 FROM product_tags pt
                    WHERE pt.product_id = p.id
                    AND pt.tag_id IN
                    <foreach collection="tags" item="tag" open="(" close=")" separator=",">
                        #{tag}
                    </foreach>
                )
            </if>
        </where>

        ORDER BY
        <choose>
            <when test='sortBy == "price_asc"'>
                p.price ASC
            </when>
            <when test='sortBy == "price_desc"'>
                p.price DESC
            </when>
            <when test='sortBy == "sales"'>
                p.sales_count DESC
            </when>
            <when test='sortBy == "newest"'>
                p.created_at DESC
            </when>
            <otherwise>
                p.id DESC
            </otherwise>
        </choose>

        <if test='limit > 0'>
            LIMIT #{limit} OFFSET #{offset}
        </if>
    </select>

User Permission Query Example
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: xml

    <select id="GetUserPermissions">
        SELECT DISTINCT
            p.id, p.code, p.name, p.resource_type
        FROM permissions p
        <where>
            <if test='userId > 0'>
                AND EXISTS (
                    SELECT 1 FROM user_roles ur
                    INNER JOIN role_permissions rp ON ur.role_id = rp.role_id
                    WHERE ur.user_id = #{userId}
                    AND rp.permission_id = p.id
                    AND ur.status = 1
                    AND rp.status = 1
                )
            </if>

            <if test='resourceTypes != nil and length(resourceTypes) > 0'>
                AND p.resource_type IN
                <foreach collection="resourceTypes" item="type" open="(" close=")" separator=",">
                    #{type}
                </foreach>
            </if>

            AND p.status = 1
        </where>
        ORDER BY p.sort_order ASC
    </select>

Complex Statistical Queries
---------------------------

Sales Statistics Example
~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: xml

    <select id="GetSalesStatistics">
        SELECT
            <choose>
                <when test='groupBy == "day"'>
                    DATE(o.created_at) as date_key,
                </when>
                <when test='groupBy == "month"'>
                    DATE_FORMAT(o.created_at, '%Y-%m') as date_key,
                </when>
                <when test='groupBy == "year"'>
                    YEAR(o.created_at) as date_key,
                </when>
            </choose>
            COUNT(DISTINCT o.id) as order_count,
            COUNT(DISTINCT o.user_id) as customer_count,
            SUM(o.total_amount) as total_sales,
            AVG(o.total_amount) as avg_order_value,
            SUM(o.item_count) as total_items
        FROM orders o
        <where>
            <if test='startDate != ""'>
                AND o.created_at >= #{startDate}
            </if>
            <if test='endDate != ""'>
                AND o.created_at &lt; #{endDate}
            </if>

            <if test='statuses != nil and length(statuses) > 0'>
                AND o.status IN
                <foreach collection="statuses" item="status" open="(" close=")" separator=",">
                    #{status}
                </foreach>
            </if>

            <if test='categoryId > 0'>
                AND EXISTS (
                    SELECT 1 FROM order_items oi
                    INNER JOIN products p ON oi.product_id = p.id
                    WHERE oi.order_id = o.id
                    AND p.category_id = #{categoryId}
                )
            </if>
        </where>
        GROUP BY date_key
        ORDER BY date_key DESC
    </select>

Dynamic Batch Operations
------------------------

Batch Update Example
~~~~~~~~~~~~~~~~~~~~

.. code-block:: xml

    <update id="BatchUpdateProductPrices">
        <foreach collection="products" item="product" separator=";">
            UPDATE products
            SET
                price = #{product.price},
                updated_at = NOW()
            WHERE id = #{product.id}
        </foreach>
    </update>

Conditional Batch Delete
~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: xml

    <delete id="BatchDeleteInactiveUsers">
        DELETE FROM users
        <where>
            <if test='status != nil'>
                AND status = #{status}
            </if>

            <if test='createdBefore != ""'>
                AND created_at &lt; #{createdBefore}
            </if>

            <if test='lastLoginBefore != ""'>
                AND last_login_at &lt; #{lastLoginBefore}
            </if>

            <if test='excludeIds != nil and length(excludeIds) > 0'>
                AND id NOT IN
                <foreach collection="excludeIds" item="id" open="(" close=")" separator=",">
                    #{id}
                </foreach>
            </if>
        </where>
        LIMIT 1000
    </delete>

Performance Optimization Tips
=============================

Avoid Common Performance Pitfalls
---------------------------------

**1. Avoid unnecessary conditions**

Not recommended:

.. code-block:: xml

    <select id="GetUsers">
        SELECT * FROM users
        <if test='true'>
            WHERE status = 1
        </if>
    </select>

Recommended:

.. code-block:: xml

    <select id="GetUsers">
        SELECT * FROM users
        WHERE status = 1
    </select>

**2. Use foreach carefully**

Not recommended when the ``IN`` list is too large:

.. code-block:: xml

    <select id="GetUsersByIds">
        SELECT * FROM users
        WHERE id IN
        <foreach collection="ids" item="id" open="(" close=")" separator=",">
            #{id}
        </foreach>
    </select>

Recommended: split into batches or use a temporary table.

.. code-block:: go

    func GetUsersByIds(ctx context.Context, ids []int64) ([]*User, error) {
        const batchSize = 1000
        var allUsers []*User

        for i := 0; i < len(ids); i += batchSize {
            end := i + batchSize
            if end > len(ids) {
                end = len(ids)
            }

            batch := ids[i:end]
            users, err := queryBatch(ctx, batch)
            if err != nil {
                return nil, err
            }
            allUsers = append(allUsers, users...)
        }

        return allUsers, nil
    }

**3. Avoid N+1 queries**

Not recommended:

.. code-block:: go

    orders, _ := GetOrders(ctx)
    for _, order := range orders {
        items, _ := GetOrderItems(ctx, order.ID)
        order.Items = items
    }

Recommended: use ``JOIN`` or batched ``IN`` queries.

.. code-block:: xml

    <select id="GetOrdersWithItems">
        SELECT
            o.id as order_id,
            o.total_amount,
            oi.id as item_id,
            oi.product_id,
            oi.quantity,
            oi.price
        FROM orders o
        LEFT JOIN order_items oi ON o.id = oi.order_id
        WHERE o.status = 1
    </select>

Index Optimization Suggestions
------------------------------

**1. Ensure WHERE conditions use indexes**

.. code-block:: xml

    <select id="SearchUsers">
        SELECT * FROM users
        <where>
            <if test='email != ""'>
                AND email = #{email}
            </if>

            <if test='status > 0'>
                AND status = #{status}
            </if>
            <if test='createdAfter != ""'>
                AND created_at > #{createdAfter}
            </if>
        </where>
    </select>

**2. Avoid functions on indexed columns**

Not recommended:

.. code-block:: xml

    <select id="GetUsersByDate">
        SELECT * FROM users
        WHERE DATE(created_at) = #{date}
    </select>

Recommended:

.. code-block:: xml

    <select id="GetUsersByDate">
        SELECT * FROM users
        WHERE created_at >= #{date}
        AND created_at &lt; DATE_ADD(#{date}, INTERVAL 1 DAY)
    </select>

Query Optimization
------------------

**1. Select only the fields you need**

Not recommended:

.. code-block:: xml

    <select id="GetUserNames">
        SELECT * FROM users
    </select>

Recommended:

.. code-block:: xml

    <select id="GetUserNames">
        SELECT id, name FROM users
    </select>

**2. Use LIMIT to bound result sets**

.. code-block:: xml

    <select id="GetRecentOrders">
        SELECT * FROM orders
        ORDER BY created_at DESC
        LIMIT 100
    </select>

**3. Use subqueries carefully**

Not recommended:

.. code-block:: xml

    <select id="GetUsersWithOrderCount">
        SELECT
            u.*,
            (SELECT COUNT(*) FROM orders WHERE user_id = u.id) as order_count
        FROM users u
    </select>

Recommended:

.. code-block:: xml

    <select id="GetUsersWithOrderCount">
        SELECT
            u.*,
            COALESCE(o.order_count, 0) as order_count
        FROM users u
        LEFT JOIN (
            SELECT user_id, COUNT(*) as order_count
            FROM orders
            GROUP BY user_id
        ) o ON u.id = o.user_id
    </select>

Batch Operation Optimization
----------------------------

**1. Control batch size with batchSize**

.. code-block:: xml

    <insert id="BatchInsertProducts" batchSize="500">
        INSERT INTO products (name, price, stock)
        VALUES
        <foreach collection="products" item="p" separator=",">
            (#{p.name}, #{p.price}, #{p.stock})
        </foreach>
    </insert>

**2. Optimize bulk updates**

Use ``CASE WHEN`` for batch updates:

.. code-block:: xml

    <update id="BatchUpdatePrices">
        UPDATE products
        SET price = CASE id
            <foreach collection="products" item="p">
                WHEN #{p.id} THEN #{p.price}
            </foreach>
        END
        WHERE id IN
        <foreach collection="products" item="p" open="(" close=")" separator=",">
            #{p.id}
        </foreach>
    </update>

Debugging and Monitoring
========================

SQL Debugging Tips
------------------

**1. Enable debug mode**

Globally:

.. code-block:: xml

    <settings>
        <setting name="debug" value="true"/>
    </settings>

For a single statement:

.. code-block:: xml

    <select id="GetUsers" debug="true">
        SELECT * FROM users
    </select>

**2. Add a custom logging middleware**

.. code-block:: go

    type SQLLogger struct {
        logger *log.Logger
    }

    func (m *SQLLogger) QueryContext(stmt juice.Statement, cfg juice.Configuration, next juice.QueryHandler) juice.QueryHandler {
        return func(ctx context.Context, query string, args ...any) (*sql.Rows, error) {
            start := time.Now()

            m.logger.Printf("[SQL] %s", query)
            m.logger.Printf("[ARGS] %v", args)

            rows, err := next(ctx, query, args...)
            duration := time.Since(start)

            if err != nil {
                m.logger.Printf("[ERROR] %v (took %v)", err, duration)
            } else {
                m.logger.Printf("[SUCCESS] took %v", duration)
            }

            return rows, err
        }
    }

Performance Analysis
--------------------

**1. Monitor slow queries**

.. code-block:: go

    type SlowQueryMonitor struct {
        threshold time.Duration
        reporter  func(query string, duration time.Duration, args []any)
    }

    func (m *SlowQueryMonitor) QueryContext(stmt juice.Statement, cfg juice.Configuration, next juice.QueryHandler) juice.QueryHandler {
        return func(ctx context.Context, query string, args ...any) (*sql.Rows, error) {
            start := time.Now()
            rows, err := next(ctx, query, args...)
            duration := time.Since(start)

            if duration > m.threshold {
                m.reporter(query, duration, args)
            }

            return rows, err
        }
    }

**2. Collect query statistics**

.. code-block:: go

    type QueryStats struct {
        mu         sync.RWMutex
        queryCount map[string]int64
        totalTime  map[string]time.Duration
        avgTime    map[string]time.Duration
    }

    func (s *QueryStats) Record(stmtID string, duration time.Duration) {
        s.mu.Lock()
        defer s.mu.Unlock()

        s.queryCount[stmtID]++
        s.totalTime[stmtID] += duration
        s.avgTime[stmtID] = s.totalTime[stmtID] / time.Duration(s.queryCount[stmtID])
    }

    func (s *QueryStats) Report() {
        s.mu.RLock()
        defer s.mu.RUnlock()

        for stmtID, count := range s.queryCount {
            fmt.Printf("Statement: %s\n", stmtID)
            fmt.Printf("  Count: %d\n", count)
            fmt.Printf("  Total Time: %v\n", s.totalTime[stmtID])
            fmt.Printf("  Avg Time: %v\n", s.avgTime[stmtID])
        }
    }

Best Practices Summary
======================

Dynamic SQL Design Principles
-----------------------------

1. **Keep it simple**

   - Avoid deeply nested conditions where possible.
   - Split very complex logic into multiple statements.
   - Prefer straightforward condition checks.

2. **Prioritize performance**

   - Make sure generated SQL can use indexes.
   - Avoid full table scans.
   - Use ``LIMIT`` where appropriate.

3. **Keep it maintainable**

   - Add clear comments when needed.
   - Use meaningful variable names.
   - Keep formatting consistent.

4. **Keep it safe**

   - Always prefer parameter binding such as ``#{param}``.
   - Avoid string replacement such as ``${param}`` unless absolutely necessary.
   - Validate user input.

Checklist
---------

Before writing dynamic SQL, confirm the following:

.. code-block:: text

    [ ] Do you really need dynamic SQL here?
    [ ] Are the conditions reasonable?
    [ ] Could this introduce an N+1 query pattern?
    [ ] Are the relevant indexes available?
    [ ] Is the result set size bounded?
    [ ] Are parameters safely bound?
    [ ] Have you added comments where the logic is complex?
    [ ] Have you performance-tested the query?

.. tip::

    Recommended tools:

    - use ``EXPLAIN`` to analyze the query plan
    - use the Juice IDEA plugin to check SQL syntax
    - use slow-query logs to monitor performance
    - review and optimize SQL regularly

.. warning::

    Common mistakes:

    - executing queries inside loops
    - putting too many values into an ``IN`` clause
    - applying functions to indexed columns
    - ignoring ``NULL`` handling
    - forgetting to use ``LIMIT`` where appropriate
