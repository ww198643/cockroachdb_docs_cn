# 用CockroachDB做一个Python的App

了解怎样在一个使用了psycopg2 驱动程序的简单Python应用程序中使用CockroachDB

[使用psycopg2](https://www.cockroachlabs.com/docs/stable/build-a-python-app-with-cockroachdb.html) [使用SQLAlchemy](https://www.cockroachlabs.com/docs/stable/build-a-python-app-with-cockroachdb-sqlalchemy.html)

本教程将向你介绍如何使用与PostgreSQL兼容的驱动程序或ORM构建一个简单的使用CockroachDB的Python应用程序。我们已经测试过并且可以推荐使用[Python psycopg2](http://initd.org/psycopg/docs/)驱动程序和[SQLAIchemy ORM](https://docs.sqlalchemy.org/en/latest/)，所以这些在这里起重要作用

## 开始之前

确保你已经安装[CockroachDB](https://www.cockroachlabs.com/docs/stable/install-cockroachdb.html)

## 第一步：安装 pdycopg2 驱动程序

运行以下命令来安装Python psycopg2驱动程序

```sh
$ pip install psycopg2
```

安装pdycopg2的其他方式，请参照[官方文档](http://initd.org/psycopg/docs/install.html)

## 第二步：搭建一个集群

为了本教程的目的，你只需要在非安全模式下运行一个的CockroachDB命令

```sh
$ cockroach start --insecure \

--store=hello-1 \

--host=localhost
```

但是就像你在[搭建一个本地集群](https://www.cockroachlabs.com/docs/stable/start-a-local-cluster.html)的教程中看到过，如果你想要模拟一个真正的集群，搭建和加入附加节点是非常容易的。

在一个新的终端，运行命令2：

```sh
$ cockroach start --insecure \

--store=hello-2 \

--host=localhost \

--port=26258 \

--http-port=8081 \

--join=localhost:26257
```

在一个新的终端，运行命令3：

```sh
$ cockroach start --insecure \

--store=hello-3 \

--host=localhost \

--port=26259 \

--http-port=8082 \

--join=localhost:26257
```

## 第三步：创建一个用户

在一个新的终端，作为`root`用户，使用[cockroach user](https://www.cockroachlabs.com/docs/stable/create-and-manage-users.html)命令来创建一个新的用户`maxroach`

```sh
$ cockroach user set maxroach --insecure
```

## 第四步：创建一个数据库和授权

作为`root`用户，使用[内置 SQL 客户端](https://www.cockroachlabs.com/docs/stable/use-the-built-in-sql-client.html)来创建一个`bank`数据库

```sh
$ cockroach sql --insecure -e 'CREATE DATABASE bank'
```

然后[授权](https://www.cockroachlabs.com/docs/stable/grant.html)给`maxroach`用户

```sh
$ cockroach sql --insecure -e 'GRANT ALL ON DATABASE bank TO maxroach'
```

## 第五步：运行Python代码

现在你有一个数据库和一个用户，你将运行代码来创建一个表格并添加一些行，然后你将运行代码来读取并更新这些值来作为一个[原子事务](https://www.cockroachlabs.com/docs/stable/transactions.html)

### 基本声明

首先，用下面的代码作为`maxroach`用户去连接并且执行一些基本的SQL声明，创建一个表格并添加一些行，然后读取并输出这些行

下载[basic-sample.py](https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/app/basic-sample.py)文件,或者你自己创建一个文件然后把代码赋值过去

```python
# Import the driver.
import psycopg2

# Connect to the "bank" database.
conn = psycopg2.connect(database='bank', user='maxroach', host='localhost', port=26257)

# Make each statement commit immediately.
conn.set_session(autocommit=True)

# Open a cursor to perform database operations.
cur = conn.cursor()

# Create the "accounts" table.
cur.execute("CREATE TABLE IF NOT EXISTS accounts (id INT PRIMARY KEY, balance INT)")

# Insert two rows into the "accounts" table.
cur.execute("INSERT INTO accounts (id, balance) VALUES (1, 1000), (2, 250)")

# Print out the balances.
cur.execute("SELECT id, balance FROM accounts")
rows = cur.fetchall()
print('Initial balances:')
for row in rows:
    print([str(cell) for cell in row])

# Close the database connection.
cur.close()
conn.close()

```

然后运行代码

```sh
$ python basic-sample.py
```

输出的应该是

```
Initial balances:
['1', '1000']
['2', '250']
```

### 事务(重启逻辑)

接下来，使用下面的代码再次作为`maxroach`用户去连接，但是这次执行一批声明作为一个原子事务来将储存的东西从一个账户转移到另一个账户，其中所有包含的语句都被提交或者中止

下载[txn-sample.py](https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/app/txn-sample.py)文件，或者你自己创建一个文件然后把代码复制过去

> **注意：**

> 使用默认的`SERIALIZABLE`隔离层级，CockeoachDB可能会要求客户端在读写争用的情况下重启事务。CockroachDB提供了一个通用的在事务中运行的重启函数，并会在需要的时候重启。你可以在这里复制这个重启函数然后粘贴到你的代码中去。

```python
# Import the driver.
import psycopg2
import psycopg2.errorcodes

# Connect to the cluster.
conn = psycopg2.connect(database='bank', user='maxroach', host='localhost', port=26257)


def onestmt(conn, sql):
    with conn.cursor() as cur:
        cur.execute(sql)


# Wrapper for a transaction.
# This automatically re-calls "op" with the open transaction as an argument
# as long as the database server asks for the transaction to be retried.
def run_transaction(conn, op):
    with conn:
        onestmt(conn, "SAVEPOINT cockroach_restart")
        while True:
            try:
                # Attempt the work.
                op(conn)

                # If we reach this point, commit.
                onestmt(conn, "RELEASE SAVEPOINT cockroach_restart")
                break

            except psycopg2.OperationalError as e:
                if e.pgcode != psycopg2.errorcodes.SERIALIZATION_FAILURE:
                    # A non-retryable error; report this up the call stack.
                    raise e
                # Signal the database that we'll retry.
                onestmt(conn, "ROLLBACK TO SAVEPOINT cockroach_restart")


# The transaction we want to run.
def transfer_funds(txn, frm, to, amount):
    with txn.cursor() as cur:

        # Check the current balance.
        cur.execute("SELECT balance FROM accounts WHERE id = " + str(frm))
        from_balance = cur.fetchone()[0]
        if from_balance < amount:
            raise "Insufficient funds"

        # Perform the transfer.
        cur.execute("UPDATE accounts SET balance = balance - %s WHERE id = %s",
                    (amount, frm))
        cur.execute("UPDATE accounts SET balance = balance + %s WHERE id = %s",
                    (amount, to))


# Execute the transaction.
run_transaction(conn, lambda conn: transfer_funds(conn, 1, 2, 100))


with conn:
    with conn.cursor() as cur:
        # Check account balances.
        cur.execute("SELECT id, balance FROM accounts")
        rows = cur.fetchall()
        print('Balances after transfer:')
        for row in rows:
            print([str(cell) for cell in row])

# Close communication with the database.
conn.close()

```

然后运行代码

```sh
$ python txn-sample.py
```

输出应该是

```
Balances after transfer:
['1', '900']
['2', '350']
```

另外，如果你想验证储存的资源已经从一个账户转移到了另一个账户，使用[内置SQL客户端](https://www.cockroachlabs.com/docs/stable/use-the-built-in-sql-client.html)

```sh
$ cockroach sql --insecure -e 'SELECT id, balance FROM accounts' --database=bank
```

```
+----+---------+
| id | balance |
+----+---------+
|  1 |     900 |
|  2 |     350 |
+----+---------+
(2 列)
```

## 接下来做什么？

阅读更多关于使用[Python psycopg driver](http://initd.org/psycopg/docs/)

您可能也有兴趣使用本地集群来探索以下CoreroachDB核心功能:

- [数据备份](https://www.cockroachlabs.com/docs/stable/demo-data-replication.html)
- [容错 & 修复](https://www.cockroachlabs.com/docs/stable/demo-fault-tolerance-and-recovery.html)
- [自动化再平衡](https://www.cockroachlabs.com/docs/stable/demo-automatic-rebalancing.html)
