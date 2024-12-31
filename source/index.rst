.. juice documentation master file, created by sphinx-quickstart on Sat Nov 5 19:15:50 2022. You can adapt this file completely to your liking, but it should at least contain the root `toctree` directive.

Introduction to Juice
==============================

Juice is a SQL mapper framework based on Golang, aiming to provide a straightforward and effort-saving SQL mapper to allow developers to focus more on business logic development.

If you are a Golang developer or searching for an easy-to-use SQL mapper framework, Juice might be your best choice.

Project Homepage
------------------------------
http://github.com/go-juicedev/juice

Features
------------------------------
- Lightweight, high performance, no third-party dependencies
- Dynamic SQL
- Support for multiple data sources
- Generic result set mapping
- Middleware
- Custom expressions
- Custom functions
- Hot updates
- Code generation

Installation
------------------------------
**Requirements**

- go 1.23+

.. code-block:: bash

    go get -u github.com/go-juicedev/juice

Quick Start
------------------------------
1. Write SQL mapper configuration file

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
            <mapper namespace="main">
                <select id="HelloWorld">
                    select "hello world" as message
                </select>
            </mapper>
        </mappers>
    </configuration>

.. attention::

    Note: The format of `dataSource` is the same as used in the `sql.Open()` function.

2. Write the code

.. code-block:: go

    package main

    import (
        "context"
        "fmt"
        "github.com/go-juicedev/juice"
        _ "github.com/go-sql-driver/mysql"
    )

    func HelloWorld() {}

    func main() {
        cfg, err := juice.NewXMLConfiguration("config.xml")
        if err != nil {
            fmt.Println(err)
            return
        }

        engine, err := juice.Default(cfg)
        if err != nil {
            fmt.Println(err)
            return
        }

        message, err := juice.NewGenericManager[string](engine).Object(HelloWorld).QueryContext(context.Background(), nil)
        if err != nil {
            fmt.Println(err)
            return
        }

        fmt.Println(message)
    }

.. attention::

    Note: Although `juice` does not rely on third-party libraries, it does require a database driver. For example, if you want to use MySQL, you need to install `github.com/go-sql-driver/mysql` first. If you encounter errors, it might be because your database is not running, or there is a mistake in your database configuration. Check your database configuration is correct.

3. Run the code

.. code-block:: bash

    go run main.go

4. Output

.. code-block:: bash

    [juice] 2022/11/05 19:56:49 [main.HelloWorld] select "hello world" as message 5.3138ms hello world

If you see the above output result, congratulations, you have successfully run Juice.

Detailed Documentation
------------------------------
.. toctree::
   :maxdepth: 2
   :caption: Introduction to Juice

   configuration
   mappers
   result_mapping
   tx
   cache
   dynamic_sql
   expr
   hot_reload
   middleware
   code_generate
   extension
   eatmoreapple