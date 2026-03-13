SQL Mappers
===========

After configuring your datasource information, you can use SQL Mappers to access the database. The first step is to tell Juice where to find your SQL statements.

mappers Tag
-----------

``mappers`` is the parent tag of ``mapper``. It is a collection tag used to hold multiple ``mapper`` definitions.

.. code-block:: xml

    <?xml version="1.0" encoding="UTF-8"?>
    <configuration>
        <environments default="prod">
            <environment id="prod">
                <dataSource>root:qwe123@tcp(localhost:3306)/database</dataSource>
                <driver>mysql</driver>
            </environment>
        </environments>

        <mappers>
            <!-- Define mapper tags here -->
        </mappers>
    </configuration>

mapper Tag
----------

The ``mapper`` tag is a container for SQL statements.

Basic usage examples:

.. code-block:: xml

    <mappers>
        <!-- Inline mapper defined directly in the configuration file -->
        <mapper namespace="main">
            <select id="HelloWorld">
                select "hello world" as message
            </select>
        </mapper>

        <!-- Reference an external mapper file -->
        <mapper resource="path_to_another_mapper.xml"/>

        <!-- Reference a mapper file through a URL -->
        <mapper url="http(s)://domain:port/path"/>
        <mapper url="file://path to your mapper"/>
    </mappers>

**Attribute descriptions**

- ``namespace``: specifies the mapper namespace. It must be unique.
- ``resource``: references another mapper file. If the referenced file does not reference another file, its ``namespace`` attribute is required.
- ``url``: references a mapper file through a URL. ``http`` and ``file`` are currently supported. If the referenced file does not reference another file, its ``namespace`` attribute is required.

By referencing mapper files, you can distribute SQL statements across multiple files and keep the structure clearer.

.. attention::

    ``namespace``, ``resource``, and ``url`` are mutually exclusive. A single ``mapper`` tag can use only one of them.

SQL Statement Tags
------------------

Juice supports four kinds of SQL statement tags: ``select``, ``insert``, ``update``, and ``delete``.

.. code-block:: xml

    <mapper namespace="main">
        <!-- Query statement -->
        <select id="HelloWorld">
            select * from user
        </select>

        <!-- Insert statement -->
        <insert id="InsertUser">
            insert into user (name, age) values (#{name}, #{age})
        </insert>

        <!-- Update statement -->
        <update id="UpdateUser">
            update user set age = #{age} where name = #{name}
        </update>

        <!-- Delete statement -->
        <delete id="DeleteUser">
            delete from user where name = #{name}
        </delete>
    </mapper>

Each of the ``select``, ``insert``, ``update``, and ``delete`` tags has an ``id`` attribute. It identifies the SQL statement and must be unique within the same mapper.

**Common question**

*Question: Can I put a delete statement inside a select tag?*

*Answer: Technically yes, but it is strongly discouraged. Each tag should preserve its own semantics.*

Parameter Handling
------------------

Using Parameters in SQL Statements
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

You can use parameters in SQL statements and pass values from the outside. You only need to reference them with the proper syntax.

**Parameter definition example**

.. code-block:: xml

    <mapper namespace="main">
        <select id="CountUserByName">
            select count(*) from user where name = #{name}
        </select>
    </mapper>

In the example above, ``#{name}`` references a parameter whose value is passed in when the SQL is executed.

**Parameter syntax comparison**

- ``#{name}``: a prepared parameter that is replaced with a placeholder such as ``?``. It helps prevent SQL injection and is recommended.
- ``${name}``: direct string interpolation. It is not replaced with a placeholder and carries SQL injection risk, so use it carefully.

.. code-block:: xml

    <mapper namespace="main">
        <!-- Recommended: prepared parameter -->
        <select id="GetUserByName">
            select * from user where name = #{name}
        </select>

        <!-- Use carefully: direct string interpolation -->
        <select id="GetUserByDynamicColumn">
            select * from user order by ${columnName}
        </select>
    </mapper>

.. warning::

    When using ``${}``, make sure the parameter value is safe because Juice will not protect it from SQL injection automatically.

Parameter Passing Methods
~~~~~~~~~~~~~~~~~~~~~~~~~

**1. Passing a map**

.. code-block:: go

    userMap := map[string]any{
        "name": "eatmoreapple",
    }
    engine.Object("main.CountUserByName").QueryContext(context.TODO(), userMap)

**2. Passing a struct**

.. code-block:: go

    type User struct {
        Name string `param:"name"`
        Age  int    `param:"age"`
    }

    user := User{
        Name: "eatmoreapple",
        Age:  25,
    }
    engine.Object("main.CountUserByName").QueryContext(context.TODO(), user)

**3. Passing an array or slice**

Since map and struct values can both be converted into key-value form, a slice or array can be accessed by index:

.. code-block:: xml

    <mapper namespace="main">
        <select id="CountUserByName">
            select count(*) from user where name = #{0} and age = #{1}
        </select>
    </mapper>

.. code-block:: go

    engine.Object("main.CountUserByName").QueryContext(context.TODO(), []any{"eatmoreapple", 25})

**4. Passing a single parameter**

If the parameter is not a struct, map, slice, or array, Juice wraps it into a map where the key is ``param`` and the value is the argument you passed.

.. code-block:: xml

    <mapper namespace="main">
        <select id="CountUserByName">
            select count(*) from user where name = #{param}
        </select>
    </mapper>

.. code-block:: go

    engine.Object("main.CountUserByName").QueryContext(context.TODO(), "eatmoreapple")

**Custom parameter names**

You can customize the name of a single parameter with the ``paramName`` attribute:

.. code-block:: xml

    <mapper namespace="main">
        <select id="CountUserByName" paramName="name">
            select count(*) from user where name = #{name}
        </select>
    </mapper>

You can also set it globally through the ``JUICE_PARAM_NAME`` environment variable.

**Convenience type**

``juice.H`` is an alias for ``map[string]any`` and is provided to make parameter passing more convenient.

.. code-block:: go

    params := juice.H{
        "name": "eatmoreapple",
        "age":  25,
    }
    engine.Object("main.CountUserByName").QueryContext(context.TODO(), params)

.. attention::

    If the parameter is a map, its key type must be ``string``.

Advanced Features
-----------------

Statement Attributes
~~~~~~~~~~~~~~~~~~~~

SQL statement tags support a range of attributes to control execution behavior:

.. code-block:: xml

    <mapper namespace="main">
        <select id="GetUser" timeout="1000" debug="true">
            select * from user where id = #{id}
        </select>

        <insert id="CreateUser" useGeneratedKeys="true" keyProperty="id">
            insert into user (name, age) values (#{name}, #{age})
        </insert>

        <select id="CountUserByName" paramName="name">
            select count(*) from user where name = #{name}
        </select>
    </mapper>

**Common attributes**

- ``timeout``: SQL execution timeout in milliseconds
- ``debug``: whether to enable debug mode
- ``paramName``: custom name for a single parameter
- ``useGeneratedKeys``: whether to use auto-generated keys
- ``keyProperty``: the property that receives the generated key

Best Practices
--------------

1. **Naming conventions**

   - Use meaningful namespaces.
   - SQL statement IDs should clearly describe what they do.
   - Parameter names should be descriptive.

2. **File organization**

   - Split mapper files by feature or module.
   - Avoid letting a single mapper file grow too large.
   - Use a clear and consistent directory structure.

3. **Security**

   - Prefer ``#{}`` over ``${}``.
   - Avoid direct SQL string concatenation.
   - Validate user input before sending it to SQL.

4. **Performance optimization**

   - Use indexes appropriately.
   - Avoid ``SELECT *`` where possible.
   - Choose appropriate data types.

Example: Complete Mapper Configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: xml

    <?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE mapper PUBLIC "-//juice.org//DTD Config 1.0//EN"
            "https://raw.githubusercontent.com/go-juicedev/juice/main/mapper.dtd">

    <mapper namespace="user.UserRepository">

        <!-- Basic query -->
        <select id="GetById">
            select id, name, email, age, created_at
            from users
            where id = #{id}
        </select>

        <!-- Pagination query -->
        <select id="GetByPage">
            select id, name, email, age, created_at
            from users
            order by created_at desc
            limit #{limit} offset #{offset}
        </select>

        <!-- Conditional query -->
        <select id="GetByCondition">
            select id, name, email, age, created_at
            from users
            where 1=1
            <if test="name != nil and name != ''">
                and name like concat('%', #{name}, '%')
            </if>
            <if test="minAge != nil">
                and age >= #{minAge}
            </if>
            <if test="maxAge != nil">
                and age <= #{maxAge}
            </if>
            order by created_at desc
        </select>

        <!-- Create user -->
        <insert id="Create" useGeneratedKeys="true" keyProperty="id">
            insert into users (name, email, age, created_at)
            values (#{name}, #{email}, #{age}, now())
        </insert>

        <!-- Update user -->
        <update id="Update">
            update users
            set name = #{name},
                email = #{email},
                age = #{age},
                updated_at = now()
            where id = #{id}
        </update>

        <!-- Delete user -->
        <delete id="Delete">
            delete from users where id = #{id}
        </delete>

        <!-- Batch operations -->
        <insert id="BatchInsert">
            insert into users (name, email, age, created_at) values
            <foreach collection="users" item="user" separator=",">
                (#{user.name}, #{user.email}, #{user.age}, now())
            </foreach>
        </insert>

    </mapper>

With the configuration and usage patterns above, you can use Juice SQL Mappers to build an efficient and secure data access layer.
