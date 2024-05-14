Dynamic SQL
===========

.. note:: Dynamic SQL refers to the technique of dynamically generating SQL statements at runtime (rather than at compile time) based on conditions such as user input, program logic, or query parameters. Dynamic SQL helps developers construct database queries more flexibly, realizing better performance and maintainability. In dynamic SQL, applications use techniques like string concatenation or parameterized queries to build SQL statements. These statements may include one or more query parameters, provided by the application at runtime. By combining query parameters with SQL statements, different queries can be generated based on variable conditions. These queries can perform operations such as querying, inserting, updating, or deleting. The advantage of dynamic SQL is its ability to dynamically generate query statements based on different conditions, making queries more flexible and customizable. However, using dynamic SQL also carries some risks, such as SQL injection attacks, thus it should be used with caution.

Juice provides support for dynamic SQL, enabling the implementation of dynamic SQL statements. The syntax of dynamic SQL is similar to that of XML, but it is simpler and more flexible.

The main syntax used in dynamic SQL includes ``if``, ``where``, ``set``, ``foreach``, ``trim``, ``choose``, ``when``, and ``otherwise``. Let's explore how to use these syntax elements.

if
----
The if statement is used to evaluate conditions; if the condition is true, the SQL statement within the if block is executed, otherwise, it is skipped. The syntax is as follows:

.. code-block:: xml

    <if test="condition expression // e.g., 1 + 1 > 0">
        SQL statement
    </if>

.. note:: The condition expression can be any expression that must evaluate to a boolean type. If the condition expression results in true, the SQL statement within the if block is executed. If not, it is skipped.

Let's see an example:

.. code-block:: xml

    <select id="SelectByCondition">
        select * from user
        <if test='name != ""'>
            where name = #{name}
        </if>
    </select>

In the example above, if the value of `name` is not empty, `where name = #{name}` is added to the SQL statement. Otherwise, it does not add anything. If the value of `name` is empty, the generated SQL statement would be `select * from user`.

where
--------

The where statement is used to add conditions to SQL queries, similar to if but specifically for where clauses. The syntax is as follows:

.. code-block:: xml

    <where>
        <if test="condition expression">
            SQL statement
        </if>
        <if test="condition expression">
            SQL statement
        </if>
        ...
    </where>

.. note:: The where statement can contain multiple if statements. If all condition expressions in the where block evaluate to false, no SQL is executed for the where block. If they evaluate to true, the SQL with conditions is executed and redundant "and" or "or" are removed.

Set
-----

Set syntax is used in update statements to specify the fields to update, removing excessive commas in the process. The syntax is as follows:

.. code-block:: xml

    <set>
        <if test="condition expression">
            SQL statement
        </if>
        <if test="condition expression">
            SQL statement
        </if>
        ...
    </set>

.. note:: The set block can contain multiple if statements. If all condition expressions in the set block evaluate to true, the corresponding SQL statements are executed. Redundant commas are removed to ensure the syntax is correct.

Foreach
----------

The foreach statement is used to iterate over collections, passing each element as a parameter to the SQL statement. The syntax is as follows:

.. code-block:: xml

    <foreach collection="collection" item="element" index="index" open="open symbol" close="close symbol" separator="separator">
        SQL statement
    </foreach>

.. note:: In the collection attribute, you specify the collection, item specifies the elements of the collection, index specifies the index in the collection, open sets the opening symbol, close sets the closing symbol, and separator sets the delimiter.

Trim
-------

Trim is used to remove unnecessary keywords from the beginning and end of SQL statements, such as "and" and "or". The syntax is as follows:

.. code-block:: xml

    <trim prefix="prefix" prefixOverrides="prefix to override" suffix="suffix" suffixOverrides="suffix to override">
        SQL statement
    </trim>

.. note:: The prefix attribute sets a keyword at the start of the SQL, the suffix sets a keyword at the end, prefixOverrides lists starting keywords to remove, and suffixOverrides lists ending keywords to remove.

Choose, When, and Otherwise
----------------------------

These statements function similarly to a switch-case-default mechanism in programming:

.. code-block:: xml

    <choose>
        <when test="condition expression">
            SQL statement
        </when>
        <when test="condition expression">
            SQL statement
        </when>
        ...
        <otherwise>
            SQL statement
        </otherwise>
    </choose>

SQL, Include
-------------

SQL statement defines a SQL fragment, and include is used to reference it:

.. code-block:: xml

    <mapper namespace="com.example.mapper.UserMapper">
        <sql id="columns"> id, name, age </sql>
    </mapper>

.. code-block:: xml

    <select id="SelectAll">
        select <include refid="columns"/> from user
    </select>

Values, Value
-------------

Used in insert statements to specify values:

.. code-block:: xml

    <insert id="Insert">
        insert into user
        <values>
            <value column="uid" value="#{uid}"/>
            <value column="create_at" value="NOW()"/>
            <value column="name"/>
        </values>
    </insert>

Alias, Field
------------

To alias tables and fields in select statements:

.. code-block:: xml

    <select id="Select">
        select
        <alias>
            <field column="uid" alias="id"/>
            <field column="name"/>
        </alias>
        from user
    </select>

This document outlines various dynamic SQL elements, how each can be used, and provides examples for better understanding of making SQL queries more dynamic and flexible in applications.