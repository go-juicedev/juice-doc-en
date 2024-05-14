SQL mappers
================

当我们将数据源信息配置好之后，我们就可以使用SQL Mapper来访问数据库了。 首先，我们得需要告诉juice去哪里找到我们的sql语句。

mappers标签
----------------

mappers 是mapper标签的父标签，它是一个集合标签，用来存放mapper标签。

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


mapper标签
----------------

mapper标签是用来存储sql语句的集合标签。

简单的例子：

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

- .. class:: namespace: 用来指定mapper的命名空间，这个命名空间是用来区分不同mapper的，它的值必须是一个唯一的。
- .. class:: resource: 用来引用另外一个mapper文件，注意：引用的mapper文件如果没有再次引用别的文件，那么它的namespace属性是必须的。
- .. class:: url: 通过url来引用mapper文件。目前支持http和file协议。如果引用的mapper文件没有再次引用别的文件，那么它的namespace属性是必须的。

通过引用mapper文件，我们可以将sql语句分散到不同的文件中，这样可以使得我们的结构更加清晰。

.. attention::
   namespace、resource、url三个属性是互斥的，一个mapper标签只能使用其中的一个。


select，insert，update，delete标签
-----------------------------------

select标签用来存储select语句。 select标签必须在mapper标签中才能使用。

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


上述的 `select、insert、update、delete` 标签都是sql语句的集合标签，它们都有一个id属性，这个属性是用来标识sql语句的，它的值在同一个mapper中必须是唯一的。

*问：可不可以在 select 标签里面写 delete 语句呢？*

*答：可以，但不推荐，每个标签都要有自己的语义。*

接受参数
----------------

我们可以在我们的sql语句中使用参数，这些参数可以通过外部传递进来，我们只需要通过特定的语法来引用这些参数即可。

定义参数实例
~~~~~~~~~~~~~~~~

.. code-block:: xml

   <mapper namespace="main">
        <select id="CountUserByName">
            select count(*) from user where name = #{name}
        </select>
    </mapper>

上述的sql语句中，我们使用了 ``#{name}`` 来引用参数，这个参数的值将会在执行sql语句的时候传递进来。

``#{}`` 的语法会在运行时被替换成占位符，这样可以防止sql注入。但是，如果我们需要使用字符串拼接的方式来构造sql语句，那么我们就需要使用 ``${}`` 来引用参数了。

.. code-block:: xml

   <mapper namespace="main">
        <select id="CountUserByName">
            select count(*) from user where name = ${name}
        </select>
    </mapper>

上述的sql语句中，我们使用了 ``${name}`` 来引用参数，这个参数的值将会在执行sql语句的时候传递进来。

但是，``${}`` 的语法不会被替换成占位符，这样就会导致sql注入的问题。所以，我们在使用 ``${}`` 的时候，必须要保证参数的值是安全的。


参数传递
~~~~~~~~~~~~~~~~

.. code-block:: go

    package main

    import (
        "fmt"
        "github.com/eatmoreapple/juice"
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



如上所示，我们在创建完 `engine` 之后, 使用 `NewGenericManager` 来创建一个 `GenericManager` , 这个方法接受一个泛型参数, 这个参数是用来指定返回值的类型的, 这里我们指定的是 ``int64`` 。然后，我们使用 `Object` 方法来指定我们要执行的sql语句，这个方法接受一个参数，这里我们传入了 `CountUserByName` 这个函数，因为 `CountUserByName` 这个函数在main包下，并且它不属于任何自定义结构，所以它的全名就是 `main.CountUserByName` 。

对应到xml配置文件中，它就会去找main这个命名空间下的 `CountUserByName` 这个id。当然，我们也可以在直接调用 `Object` 方法的时候，传入一个字符串，这个字符串就是我们要执行的sql语句的id，如 `main.CountUserByName` 。

最后，我们使用 `Query` 方法来执行sql语句，这个方法接受一个参数，这个参数就是我们要传递给sql语句的参数。

map-struct参数
"""""""""""""""

如上所示，我们传递了一个 `map`，这个 `map` 的key就是我们在sql语句中使用的参数名，这个map的value就是我们要传递给sql语句的参数值。当然我们也可以传递一个struct，这个struct的字段名就是我们在sql语句中使用的参数名，这个struct的字段值就是我们要传递给sql语句的参数值。

如果我们想自定义struct的字段名和sql语句中的参数名不一致，那么我们可以使用juice的tag来指定，如下所示：

.. code-block:: go

    type User struct {
        Name string `param:"name"`
    }

指定结构体字段的tag为param，那么这个字段就会被当作sql语句中的参数名，而不是字段名。


.. attention::
    当你的参数是一个map的时候，这个map的key必须是string类型的。

非map-struct的参数传递
"""""""""""""""""""""""

既然map和struct都可以转换成key-value结构，那么如果我们传递一个非struct的参数或者非map的参数，那么这个参数传递到xml中的key是什么呢？

这个时候，juice就会将这个参数包装成一个 `map`，这个 `map` 的key就是 ``param`` ，这个 `map` 的value就是我们传递的参数。


如下所示：

.. code-block:: go

    count, err := juice.NewGenericManager[int64](engine).Object(CountUserByName).Query("eatmoreapple")

.. code-block:: xml

    <mapper namespace="main">
        <select id="CountUserByName">
            select count(*) from user where name = #{param}
        </select>
    </mapper>

包装的 `map` 的key也是可以自定义的，我们可以在对应的action标签上，使用 ``paramName`` 属性来指定，如下所示：

.. code-block:: xml

    <mapper namespace="main">
        <select id="CountUserByName" paramName="name">
            select count(*) from user where name = #{name}
        </select>
    </mapper>


或者通过环境变量 ``JUICE_PARAM_NAME`` 来设置。


H
"""""

``juice.H`` 是一个 ``map[string]interface{}`` 的别名。用来方便开发者传递参数。


.. attention::
    请确保你传递的参数是可被序列化的，否则会导致部分功能异常，如缓存。




