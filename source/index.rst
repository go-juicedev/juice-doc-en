.. juice documentation master file, created by sphinx-quickstart on Sat Nov 5 19:15:50 2022. You can adapt this file completely to your liking, but it should at least contain the root `toctree` directive.

Introduction to Juice
==============================

Juice is a SQL mapper framework based on Golang, aiming to provide a straightforward and effort-saving SQL mapper to allow developers to focus more on business logic development.

If you are a Golang developer or searching for an easy-to-use SQL mapper framework, Juice might be your best choice.

Project Homepage
------------------------------

https://github.com/go-juicedev/juice

**Documentation Links**

- English: https://juice-doc.readthedocs.io/projects/juice-doc-en/en/latest/
- 简体中文: https://juice-doc.readthedocs.io/en/latest/

Features
------------------------------

- 🚀 **Lightweight, High Performance, No Third-party Dependencies**
  
  Depends only on the Go standard library, small binary size, fast startup.

- 🔧 **Dynamic SQL Support for Flexible Complex Query Construction**
  
  Full support for dynamic tags like if, where, set, foreach, choose, etc.

- 🗄️ **Multi-datasource Support with Master-Slave Switching**
  
  Easily implement read-write separation and multi-datasource management.

- 🎯 **Generic Result Set Mapping, Type Safe**
  
  Fully utilizing Go 1.18+ generics, providing compile-time type checking.

- 🔗 **Middleware Mechanism, Extensible Architecture**
  
  Built-in debug and timeout middleware, supports custom extensions.

- 📝 **Custom Expressions and Functions**
  
  Support for custom functions and expressions in dynamic SQL.

- 🛠️ **Code Generation Tools**
  
  Optional code generation tools to improve development efficiency.

- 🔒 **Transaction Management**
  
  Concise transaction API, supporting nesting and propagation.

- 🔍 **SQL Debugging and Performance Monitoring**
  
  Built-in SQL logging and execution time statistics.

- 📊 **Multiple Database Support**
  
  MySQL, PostgreSQL, SQLite, Oracle, and other database/sql compatible databases.

- 🔧 **Connection Pool Management**
  
  Flexible connection pool configuration to optimize resource usage.


Why Choose Juice
------------------------------

Design Philosophy
~~~~~~~~~~~~~~~~~~

Juice follows these core design philosophies:

- **Simplicity Over Complexity**: API design is concise and intuitive, with a gentle learning curve.
- **Explicit Over Implicit**: SQL statements are clearly visible, ensuring predictable behavior.
- **Performance First**: Optimizations like zero dependencies, object pooling, and pre-allocation.
- **Type Safety**: Fully taking advantage of Go generics for compile-time type checks.
- **Extensibility**: Middleware mechanism supports flexible extensions.

Comparison with Other Frameworks
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. list-table::
   :header-rows: 1
   :widths: 20 15 15 15 15 20

   * - Feature
     - Juice
     - GORM
     - sqlx
     - ent
     - Note
   * - Zero Dependency
     - ✅
     - ❌
     - ❌
     - ❌
     - Depends only on Go stdlib
   * - Dynamic SQL
     - ✅
     - Partial
     - ❌
     - ❌
     - Complete MyBatis-style dynamic SQL
   * - Generics Support
     - ✅
     - ❌
     - ❌
     - ✅
     - Go 1.18+ Generics
   * - SQL Visibility
     - ✅
     - ❌
     - ✅
     - ❌
     - Centralized SQL management
   * - Learning Curve
     - Low
     - Medium
     - Low
     - High
     - MyBatis-like design
   * - Performance
     - 🚀 Excellent
     - Good
     - 🚀 Excellent
     - Good
     - Close to native database/sql
   * - Code Gen
     - ✅
     - ✅
     - ❌
     - ✅
     - Optional tools
   * - Middleware
     - ✅
     - ✅
     - ❌
     - ✅
     - Flexible interceptor mechanism

Applicable Scenarios
~~~~~~~~~~~~~~~~~~~~

Juice is particularly suitable for:

✅ **Complex Query Scenarios**
   - Complex SQL queries required
   - Dynamic query condition construction
   - Precise control over SQL execution

✅ **High Performance Requirements**
   - Performance-critical applications
   - High concurrency scenarios
   - Scenarios requiring near-native performance

✅ **Team Collaboration**
   - Clear division between DBA and developers
   - Centralized SQL management and view
   - SQL version control

✅ **Microservices Architecture**
   - Lightweight, suitable for containerized deployment
   - Zero dependencies, reducing supply chain risks
   - Fast startup, low resource footprint

❌ **Less Suitable Scenarios**
   - Simple CRUD-heavy applications (GORM might be better)
   - Scenarios requiring automatic migrations (ent might be better)
   - Teams unfamiliar with SQL

Performance Features
~~~~~~~~~~~~~~~~~~~~

Juice focuses heavily on performance optimization:

**Zero Dependency Design**
   - Only standard library dependencies
   - Reduced dependency chain, lower risk
   - Smaller binary size

**Object Pool Optimization**
   - Uses sync.Pool to reuse strings.Builder
   - Reduces memory allocation and GC pressure
   - Improves performance in high concurrency

**Pre-allocation Strategy**
   - Slice capacity pre-allocation
   - String builder capacity estimation
   - Reduces memory reallocation

**Smart Caching**
   - Prepared statement caching
   - Reflection result caching
   - Expression compilation caching

**Batch Optimization**
   - Smart batch processing
   - Prepared statement reuse
   - Reduces database round-trips


Installation
------------------------------

**Requirements**

- go 1.18+

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
        defer engine.Close()

        message, err := juice.NewGenericManager[string](engine).Object(HelloWorld).QueryContext(context.Background(), nil)
        if err != nil {
            fmt.Println(err)
            return
        }
        fmt.Println(message)
    }

.. attention::

    Note: Although `juice` does not rely on third-party libraries, it does require a database driver. For example, if you want to use MySQL, you need to install `github.com/go-sql-driver/mysql` first. 
    
    If you encounter errors, it might be because your database is not running, or there is a mistake in your database configuration. Check your database configuration is correct.

3. Run the code

.. code-block:: bash

    go run main.go

4. Output

.. code-block:: bash

    [juice] 2022/11/05 19:56:49 [main.HelloWorld]  select "hello world" as message  []  5.3138ms
    hello world

If you see the above output result, congratulations, you have successfully run Juice.


More Examples
------------------------------

User Repository Example
~~~~~~~~~~~~~~~~~~~~~~~

**Configuration (user_mapper.xml)**

.. code-block:: xml

   <?xml version="1.0" encoding="UTF-8"?>
   <mapper namespace="repository.UserRepository">
       <!-- Query user by ID -->
       <select id="GetUserByID">
           SELECT id, name, email, age, created_at
           FROM users
           WHERE id = #{id}
       </select>

       <!-- Dynamic condition search -->
       <select id="SearchUsers">
           SELECT id, name, email, age
           FROM users
           <where>
               <if test='name != ""'>
                   AND name LIKE concat('%', #{name}, '%')
               </if>
               <if test='minAge > 0'>
                   AND age >= #{minAge}
               </if>
               <if test='maxAge > 0'>
                   AND age <= #{maxAge}
               </if>
           </where>
           ORDER BY created_at DESC
       </select>

       <!-- Batch Insert -->
       <insert id="BatchInsertUsers" batchSize="100">
           INSERT INTO users (name, email, age)
           VALUES
           <foreach collection="users" item="user" separator=",">
               (#{user.name}, #{user.email}, #{user.age})
           </foreach>
       </insert>

       <!-- Dynamic Update -->
       <update id="UpdateUser">
           UPDATE users
           <set>
               <if test='name != ""'>
                   name = #{name},
               </if>
               <if test='email != ""'>
                   email = #{email},
               </if>
               <if test='age > 0'>
                   age = #{age},
               </if>
           </set>
           WHERE id = #{id}
       </update>
   </mapper>

**Go Code**

.. code-block:: go

   package repository

   import (
       "context"
       "github.com/go-juicedev/juice"
   )

   type User struct {
       ID        int64  `column:"id"`
       Name      string `column:"name"`
       Email     string `column:"email"`
       Age       int    `column:"age"`
       CreatedAt string `column:"created_at"`
   }

   type UserRepository interface {
       GetUserByID(ctx context.Context, id int64) (*User, error)
       SearchUsers(ctx context.Context, name string, minAge, maxAge int) ([]*User, error)
       BatchInsertUsers(ctx context.Context, users []*User) error
       UpdateUser(ctx context.Context, id int64, name, email string, age int) error
   }

   type userRepositoryImpl struct {
       engine juice.Manager
   }

   func NewUserRepository(engine juice.Manager) UserRepository {
       return &userRepositoryImpl{engine: engine}
   }

   func (r *userRepositoryImpl) GetUserByID(ctx context.Context, id int64) (*User, error) {
       params := juice.H{"id": id}
       return juice.NewGenericManager[*User](r.engine).
           Object(r.GetUserByID).
           QueryContext(ctx, params)
   }

   func (r *userRepositoryImpl) SearchUsers(ctx context.Context, name string, minAge, maxAge int) ([]*User, error) {
       params := juice.H{
           "name":   name,
           "minAge": minAge,
           "maxAge": maxAge,
       }
       return juice.NewGenericManager[[]*User](r.engine).
           Object(r.SearchUsers).
           QueryContext(ctx, params)
   }

   func (r *userRepositoryImpl) BatchInsertUsers(ctx context.Context, users []*User) error {
       params := juice.H{"users": users}
       _, err := r.engine.Object(r.BatchInsertUsers).ExecContext(ctx, params)
       return err
   }

   func (r *userRepositoryImpl) UpdateUser(ctx context.Context, id int64, name, email string, age int) error {
       params := juice.H{
           "id":    id,
           "name":  name,
           "email": email,
           "age":   age,
       }
       _, err := r.engine.Object(r.UpdateUser).ExecContext(ctx, params)
       return err
   }


FAQ
------------------------------

**Q: What is the difference between Juice and MyBatis?**

A: Juice borrows design concepts from MyBatis but is optimized for Go:
- Fully utilizes Go generics for type-safe APIs
- Zero dependencies, lightweight
- More idiomatic to Go

**Q: Why use XML instead of annotations?**

A: Go does not support annotations. XML configuration offers:
- Centralized SQL management for easier review and version control
- Independent SQL optimization by DBAs
- Support for complex dynamic SQL
- IDE plugin support (syntax highlighting, auto-completion)

**Q: How is Juice's performance?**

A: Juice's performance is close to native database/sql:
- Zero dependencies, no extra overhead
- Object pooling to reduce GC pressure
- Pre-allocation strategies to reduce memory allocation

**Q: How to debug SQL statements?**

A: Juice provides multiple ways to debug:
- Use DebugMiddleware to print SQL and parameters
- Enable debug mode in configuration
- Use IDEA plugin to view SQL
- Check generated SQL logs

**Q: Which databases are supported?**

A: Juice supports all database/sql compatible databases:
- MySQL / MariaDB
- PostgreSQL
- SQLite
- Oracle
- SQL Server
- Others supported by database/sql


Community & Support
------------------------------

**Get Help**

- GitHub Issues: https://github.com/go-juicedev/juice/issues
- Documentation: https://juice-doc.readthedocs.io/
- Examples: https://github.com/go-juicedev/juice/tree/main/examples

**Contribute**

Pull Requests and Issues are welcome!

**IDEA Plugin**

Install the Juice plugin for a better development experience:

- Syntax highlighting
- Auto-completion
- SQL navigation
- Error checking

Plugin URL: https://plugins.jetbrains.com/plugin/26401-juice


Detailed Documentation
------------------------------

.. toctree::
   :maxdepth: 2
   :caption: Core Features

   configuration
   mappers
   result_mapping
   tx
   dynamic_sql
   expr
   raw_sql
   middleware

.. toctree::
   :maxdepth: 2
   :caption: Advanced Features

   code_generate
   extension
   idea-plugin

.. toctree::
   :maxdepth: 2
   :caption: Others

   eatmoreapple