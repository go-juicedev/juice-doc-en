SQL Mappers
================

After configuring the data source information, we can use SQL Mappers to access the database. First, we need to tell Juice where to find our SQL statements.

mappers Tag
----------------

The ``mappers`` tag is the parent tag for the ``mapper`` tags; it is a collection tag used to store multiple ``mapper`` tags.

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
----------------

The ``mapper`` tag is a collection tag used to store SQL statements.

Basic Usage Examples:

.. code-block:: xml

    <mappers>
        <!-- Inline mapper, define SQL directly in configuration file -->
        <mapper namespace="main">
            <select id="HelloWorld">
                select "hello world" as message
            </select>
        </mapper>

        <!-- Reference external mapper file -->
        <mapper resource="path_to_another_mapper.xml"/>
        
        <!-- Reference mapper file via URL -->
        <mapper url="http(s)://domain:port/path"/>
        <mapper url="file://path to your mapper"/>
    </mappers>

**Attribute Descriptions:**

- ``namespace``: Specifies the namespace of the mapper, which helps to differentiate between different mappers. The value must be unique.
- ``resource``: References another mapper file. Note: if the referenced mapper file does not reference other files, then its ``namespace`` attribute is required.
- ``url``: References a mapper file through a URL. Currently supports ``http`` and ``file`` protocols. If the referenced mapper file does not reference other files, then its ``namespace`` attribute is required.

Using referenced mapper files allows us to distribute SQL statements across different files, making our structure clearer.

.. attention::
   The ``namespace``, ``resource``, and ``url`` attributes are mutually exclusive; only one can be used within a single ``mapper`` tag.

SQL Statement Tags
-----------------------------------

Juice supports four types of SQL statement tags: ``select``, ``insert``, ``update``, ``delete``.

.. code-block:: xml

    <mapper namespace="main">
        <!-- Query statement -->
        <select id="HelloWorld">
            select * from user
        </select>

        <!-- Insert statement -->
        <insert id="insertUser">
            insert into user (name, age) values (#{name}, #{age})
        </insert>

        <!-- Update statement -->
        <update id="updateUser">
            update user set age = #{age} where name = #{name}
        </update>

        <!-- Delete statement -->
        <delete id="deleteUser">
            delete from user where name = #{name}
        </delete>
    </mapper>

The ``select``, ``insert``, ``update``, and ``delete`` tags all have an ``id`` attribute, which is used to identify the SQL statement and must be unique within the same mapper.

**Common Questions:**

*Question: Can you write a delete statement inside the select tag?*

*Answer: Technically yes, but it is strongly not recommended as each tag should have its own semantic meaning.*

Parameter Handling
----------------

Using Parameters in SQL Statements
~~~~~~~~~~~~~~~~~~~~

We can use parameters in our SQL statements, which can be passed in from external sources. We just need to use specific syntax to reference these parameters.

**Parameter Definition Example:**

.. code-block:: xml

    <mapper namespace="main">
        <select id="CountUserByName">
            select count(*) from user where name = #{name}
        </select>
    </mapper>

In the above SQL statement, we use ``#{name}`` to reference a parameter. This parameter's value will be passed in when executing the SQL statement.

**Parameter Syntax Comparison:**

- ``#{name}``: Prepared parameter, replaced with placeholders (``?``), prevents SQL injection, **recommended**
- ``${name}``: Direct string replacement, not replaced with placeholders, **has SQL injection risk, use with caution**

.. code-block:: xml

    <mapper namespace="main">
        <!-- Recommended: Using prepared parameters -->
        <select id="GetUserByName">
            select * from user where name = #{name}
        </select>

        <!-- Use with caution: Direct string replacement -->
        <select id="GetUserByDynamicColumn">
            select * from user order by ${columnName}
        </select>
    </mapper>

.. warning::
    When using ``${}`` syntax, ensure the parameter value is safe as it does not provide SQL injection protection.

Parameter Passing Methods
~~~~~~~~~~~~~~~~~~~~

**1. Map Parameter Passing**

.. code-block:: go

    userMap := map[string]interface{}{
        "name": "eatmoreapple",
        "age":  25,
    }

    engine.Object("main.CountUserByName").QueryContext(context.TODO(), userMap)

**2. Struct Parameter Passing**

.. code-block:: go

    type User struct {
        Name string `param:"name"`  // Use param tag to customize parameter name
        Age  int    `param:"age"`
    }

    user := User{
        Name: "eatmoreapple",
        Age:  25,
    }

    engine.Object("main.CountUserByName").QueryContext(context.TODO(), user)

**3. Array/Slice Parameter Passing**

Since both map and struct can be converted to key-value structures, if we pass a slice or array parameter, we can access the passed parameters using index access:

.. code-block:: xml

     <mapper namespace="main">
        <select id="CountUserByName">
            select count(*) from user where name = #{0} and age = #{1}
        </select>
    </mapper>

.. code-block:: go

     engine.Object("main.CountUserByName").QueryContext(context.TODO(), []interface{}{"eatmoreapple", 25})

**4. Single Parameter Passing**

If we pass a parameter that is not a struct, map, or slice/array, Juice will wrap this parameter into a map where the key is ``param`` and the value is our passed parameter.

.. code-block:: xml

    <mapper namespace="main">
        <select id="CountUserByName">
            select count(*) from user where name = #{param}
        </select>
    </mapper>

.. code-block:: go

    engine.Object("main.CountUserByName").QueryContext(context.TODO(), "eatmoreapple")

**Custom Parameter Names:**

You can customize the single parameter name using the ``paramName`` attribute:

.. code-block:: xml

    <mapper namespace="main">
        <select id="CountUserByName" paramName="name">
            select count(*) from user where name = #{name}
        </select>
    </mapper>

Or set globally via the environment variable ``JUICE_PARAM_NAME``.

**Convenience Type:**

``juice.H`` is an alias for ``map[string]interface{}``, designed to help developers pass parameters conveniently.

.. code-block:: go

    params := juice.H{
        "name": "eatmoreapple",
        "age":  25,
    }

    engine.Object("main.GetUser").QueryContext(context.TODO(), params)

.. attention::
    When the parameter is a map type, the map's key must be a string type.

Advanced Features
--------

Statement Attributes
~~~~~~~~

SQL statement tags support various attributes to control execution behavior:

.. code-block:: xml

    <mapper namespace="main">
        <select id="GetUser" 
                timeout="5000" 
                debug="false"
                paramName="userId">
            select * from user where id = #{userId}
        </select>
        
        <insert id="CreateUser" 
                useGeneratedKeys="true" 
                keyProperty="id">
            insert into user (name, email) values (#{name}, #{email})
        </insert>
    </mapper>

**Common Attribute Descriptions:**

- ``timeout``: Set SQL execution timeout (milliseconds)
- ``debug``: Whether to enable debug mode
- ``paramName``: Customize single parameter name
- ``useGeneratedKeys``: Whether to use auto-generated keys
- ``keyProperty``: Specify the property name to receive auto-generated keys

Best Practices
--------

1. **Naming Conventions**
   - Use meaningful namespaces
   - SQL statement IDs should clearly express their functionality
   - Parameter names should be descriptive

2. **File Organization**
   - Divide mapper files by functional modules
   - Don't make each mapper file too large
   - Use reasonable directory structure

3. **Security**
   - Prefer using ``#{}`` parameter syntax
   - Avoid direct SQL string concatenation
   - Validate input parameters

4. **Performance Optimization**
   - Use indexes appropriately
   - Avoid SELECT *
   - Use appropriate data types

Example: Complete Mapper Configuration
~~~~~~~~~~~~~~~~~~~~~~~

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

Through the above configuration and usage methods, you can fully utilize Juice's SQL Mapper functionality to build an efficient and secure data access layer.