SQL Mappers
=============

After configuring the data source information, we can use SQL Mappers to access the database. First, we need to tell juice where to find our SQL statements.

mappers Tag
----------------
The `mappers` tag is the parent tag for the `mapper` tags; it is a collection tag used to store multiple `mapper` tags.

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
        </mappers>
    </configuration>

mapper Tag
----------------
The `mapper` tag is a collection tag used to store SQL statements. Here’s a simple example:

.. code-block:: xml

    <mappers>
        <mapper namespace="main">
            <select id="HelloWorld">
                select "hello world" as message
            </select>
        </mapper>
        <mapper resource="path_to_another_mapper.xml"/>
        <mapper url="http(s)://domain:port/path"/>
        <mapper url="file://path to your mapper"/>
    </mappers>

- .. class:: namespace: Used to specify the namespace of the mapper, which helps to differentiate between different mappers. The value must be unique.
- .. class:: resource: Used to reference another mapper file. Note: the referenced mapper file must have a namespace attribute if it does not refer to another file.
- .. class:: url: Used to reference a mapper file through a URL. Currently supports http and file protocols. If the referenced mapper file does not refer to another file, then its namespace attribute is mandatory.

Using referenced mapper files allows us to distribute SQL statements across different files, making our structure clearer.

.. attention::
   The `namespace`, `resource`, and `url` attributes are mutually exclusive; only one can be used within a single mapper tag.

select, insert, update, delete Tags
-----------------------------------

The `select` tag is used to store select statements and must be used within a mapper tag.

.. code-block:: xml

    <mapper namespace="main">
        <select id="HelloWorld">
            select * from user
        </select>
        <insert id="insertUser">
            insert into user (name, age) values ("eatmoreapple", 18))
        </insert>
        <update id="updateUser">
            update user set age = 19 where name = "eatmoreapple"
        </update>
        <delete id="deleteUser">
            delete from user where name = "eatmoreapple"
        </delete>
    </mapper>

The `select`, `insert`, `update`, and `delete` tags are collection tags for SQL statements. Each of them must have an `id` attribute, which is used to identify the SQL statement and must be unique within the same mapper.

*Question: Can you write a delete statement inside the select tag?*

*Answer: You can, but it is not recommended as each tag should have its own semantic meaning.*

Using Parameters
----------------

We can use parameters in our SQL statements, which can be passed in from external sources. We just need to use specific syntax to reference these parameters.

Parameter Example
~~~~~~~~~~~~~~~~~~~

.. code-block:: xml

    <mapper namespace="main">
        <select id="CountUserByName">
            select count(*) from user where name = #{name}
        </select>
    </mapper>

In the above SQL statement, we use ``#{name}`` to reference a parameter. This parameter’s value will be passed in when executing the SQL statement. The ``#{}`` syntax is replaced by placeholders at runtime to prevent SQL injection. However, if we need to construct the SQL statement by concatenating strings, we'd use ``${}`` to reference the parameters.

.. code-block:: xml

    <mapper namespace="main">
        <select id="CountUserByName">
            select count(*) from user where name = ${name}
        </select>
    </mapper>

In this SQL statement, we use ``${name}`` to reference a parameter. The value of this parameter will be passed during the execution of the SQL statement. However, the ``${}`` syntax is not replaced by a placeholder, which can lead to SQL injection issues. Therefore, when using ``${}``, ensure the parameter value is secure.

Parameter Passing
~~~~~~~~~~~~~~~~~~~

.. code-block:: go

    package main

    import (
        "fmt"
        "github.com/go-juicedev/juice"
        _ "github.com/go-sql-driver/mysql"
    )

    func CountUserByName() {}

    func main() {
        cfg, err := juice.NewXMLConfiguration("config.xml")
        if err != nil {
            fmt.Println(err)
            return
        }

        engine, err := juice.DefaultEngine(cfg)
        if err != nil {
            fmt.Println(err)
            return
        }

        count, err := juice.NewGenericManager[int64](engine).Object(CountUserByName).Query(map[string]interface{}{
            "name": "eatmoreapple",
        })
        if err != nil {
            fmt.Println(err)
            return
        }

        fmt.Println(count)
    }

As shown above, after creating the `engine`, we use `NewGenericManager` to create a `GenericManager`. This method accepts a generic parameter specifying the return type, which in this case is `int64`. Then, we use the `Object` method to specify the SQL statement we want to execute. This method accepts a parameter, in this instance, we pass the `CountUserByName`, which is a function under the main package and does not belong to any custom structure, so its full name is `main.CountUserByName`. In the XML configuration file, it searches for the `CountUserByName` id under the main namespace. We can also directly pass the SQL statement id we want to execute, like `main.CountUserByName`, when calling the `Object` method. Lastly, we use the `Query` method to execute the SQL statement, which accepts a parameter for the arguments to pass to the SQL statement.

Map-Struct Parameters
"""""""""""""""""""""""

As shown, we passed a `map` where the map's key is the parameter name used in the SQL statement, and its value is the argument to pass to the SQL statement. Alternatively, we can pass a struct, and the struct’s field names would be the parameter names used in the SQL statement, with the field values being the argument values. If we want to customize so the struct field names do not match the SQL parameter names directly, we can use juice's tag to specify:

.. code-block:: go

    type User struct {
        Name string `param:"name"`
    }

By specifying the struct field’s tag as `param`, that field will be treated as the SQL parameter name, not the field name.

.. attention::
   When passing a map as an argument, the map's key must be a string type.

Non-Map-Struct Parameter Passing
"""""""""""""""""""""""""""""""""

Since both maps and structs can be converted into a key-value structure, what is the key used in the XML if we pass a non-struct or non-map parameter? Juice will then wrap this parameter in a `map`, where the `map`'s key is ``param``, and its value is our passed parameter.

.. code-block:: go

    count, err := juice.NewGenericManager[int64](engine).Object(CountUserByName).Query("eatmoreapple")

.. code-block:: xml

    <mapper namespace="main">
        <select id="CountUserByName">
            select count(*) from user where name = #{param}
        </select>
    </mapper>

The wrapped `map`'s key can also be customized. We can specify the `paramName` attribute in the corresponding action tag, as shown:

.. code-block:: xml

    <mapper namespace="main">
        <select id="CountUserByName" paramName="name">
            select count(*) from user where name = #{name}
        </select>
    </mapper>

or via the environment variable ``JUICE_PARAM_NAME``.

H
"""""

``juice.H`` is an alias for ``map[string]interface{}``. It's designed to help developers pass parameters conveniently.

.. attention::
   Please ensure the arguments you pass are serializable, or it could cause some features to malfunction, such as caching.