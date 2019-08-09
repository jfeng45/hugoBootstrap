---
title: "Go Microservice with Clean Architecture: Transaction Support"
shortTitle: "Transaction Support"
date: "2019-07-24"
draft: false
tags: ["Golang","Database Transaction", "Clean Architecture"]
categories: ["Go Microservice"]

canonicalUrl: "https://medium.com/@jfeng45/go-microservice-with-clean-architecture-transaction-support-61eb0f886a36"
description: "A business level transaction management system in Go that fulfilled most features of the declarative transaction ."

---


In order to support the transaction in the business layer, I tried to get a Spring like declarative transaction management in Go, but couldn’t find it, so I decided to write one. Transaction is easy to implement in Go, but it is pretty difficult to get it right.

## **Requirement:**

1. Separate business logic from transaction code.
One should only think about business logic when writing a use case, no need to be aware of transaction management. If later on, there is a need to add transaction support, you can just write a wrapper on top of existing business logic, and no need to change any other code. Transaction implementation detail should be transparent to business logic.

1. Transaction logic should be applied to the use case layer ( business logic)
Not on the persistence layer.

1. The data service (data persistence) layer should be transparent to the transaction logic.
Meaning the persistence code should be the same whether it supports transaction or not

1. You have the choice to delay the transaction supporting decision later.
You can write a use case without transaction and later on to enable the transaction on that use case without modifying existing code. You only add new code.

The solution I came up with is not a declarative transaction management even though it is a pretty close one. Creating a real declarative transaction management needs a lot of effort, so I built a simple one which can fulfill most features of the declarative transaction.

## **Solution:**

The solution spreads all different layers of the application and I will explain them one by one.

**Wrapper for database handler**

In Go’s “sql” lib, there are two database handler sql.DB and sql.Tx. When there is no transaction, you use sql.DB to access database; when there is a transaction, you use sql.Tx. In order to share code, the persistence layer needs to support both. A wrapper is created as a receiver for database access methods, so it can work with both. I got the rough idea from [here](https://stackoverflow.com/questions/26593867/db-transaction-in-golang)¹.

{{< highlight go "linenos=table,linenostart=1" >}}
// SqlGdbc (SQL Go database connection) is a wrapper for SQL database handler ( can be *sql.DB or *sql.Tx)
// It should be able to work with all SQL data that follows SQL standard.
type SqlGdbc interface {
	Exec(query string, args ...interface{}) (sql.Result, error)
	Prepare(query string) (*sql.Stmt, error)
	Query(query string, args ...interface{}) (*sql.Rows, error)
	QueryRow(query string, args ...interface{}) *sql.Row
	// If need transaction support, add this interface
	Transactioner
}

// SqlDBTx is the concrete implementation of sqlGdbc by using *sql.DB
type SqlDBTx struct {
	DB *sql.DB
}

// SqlConnTx is the concrete implementation of sqlGdbc by using *sql.Tx
type SqlConnTx struct {
	DB *sql.Tx
}

{{< / highlight >}}
<br/>

Both Database handler the SqlDBTx and the sqlConnTx need to implement the SqlGdbc interface (including “Transactioner”) interface to get it work. “Transactioner” interface needs to be implemented for each database to support the transaction.

{{< highlight go "linenos=table,linenostart=1" >}}
// Transactioner is the transaction interface for database handler
// It should only be applicable to SQL database
type Transactioner interface {
	// Rollback a transaction
	Rollback() error
	// Commit a transaction
	Commit() error
	// TxEnd commits a transaction if no errors, otherwise rollback
	// txFunc is the operations wrapped in a transaction
	TxEnd(txFunc func() error) error
	// TxBegin gets *sql.DB from receiver and return a SqlGdbc, which has a *sql.Tx
	TxBegin() (SqlGdbc, error)
}
{{< / highlight >}}

<br/>

**Transaction code for database handler**

The following are the implementation code for “Transactioner” interface, among them, only TxBegin() is implemented on SqlDBTx(sql.DB) because a transaction starts from sql.DB, all others are on SqlConnTx(sql.Tx). I got the idea from [here](https://stackoverflow.com/questions/16184238/database-sql-tx-detecting-commit-or-rollback/23502629#23502629)².

{{< highlight go "linenos=table,linenostart=1" >}}
// TransactionBegin starts a transaction
func (sdt *SqlDBTx) TxBegin() (gdbc.SqlGdbc, error) {
	tx, err := sdt.DB.Begin()
	sct := SqlConnTx{tx}
	return &sct, err
}

func (sct *SqlConnTx) TxEnd(txFunc func() error) error {
	var err error
	tx := sct.DB

	defer func() {
		if p := recover(); p != nil {
			tx.Rollback()
			panic(p) // re-throw panic after Rollback
		} else if err != nil {
			tx.Rollback() // err is non-nil; don't change it
		} else {
			err = tx.Commit() // if Commit returns error update err with commit err
		}
	}()
	err = txFunc()
	return err
}

func (sct *SqlConnTx) Rollback() error {
	return sct.DB.Rollback()
}

{{< / highlight >}}

<br/>

**The transaction interface on use case layer**

In the use case layer, you can have two versions of the same function, one with the transaction and one without, and their names can share the same prefix and the transaction one can add ”withTx” as a suffix. For example, in the following code, “ModifyAndUnregister” is the one without transaction and “ModifyAndUnregisterWithTx” is the one with transaction. “EnableTxer” is the only interface for transaction support on the use case layer, any “use case” with transaction support need to have it. All the code here is the use case level ( including “EnableTxer”) code, no database stuff involved.

{{< highlight go "linenos=table,linenostart=1" >}}
type RegistrationUseCaseInterface interface {
...
	// ModifyAndUnregister change user information and then unregister the user based on the User.Id passed in.
	// It is created to illustrate transaction, no real use.
	ModifyAndUnregister(user *model.User) error
	// ModifyAndUnregisterWithTx change user information and then unregister the user based on the User.Id passed in.
	// It supports transaction
	// It is created to illustrate transaction, no real use.
	ModifyAndUnregisterWithTx(user *model.User) error
	// EnableTx enable transaction support on use case. Need to be included for each use case needs transaction
	// It replaces the underline database handler to sql.Tx for each data service that used by this use case
	EnableTxer
}
// EnableTxer is the transaction interface for use case layer
type EnableTxer interface {
	EnableTx()
}

{{< / highlight >}}

<br/>

The followings are the example of business logic code with and without transaction. The “modifyAndUnregister(ruc, user)” is the business function shared by transaction and non-transaction use case functions. You do need to wrap the business function with TxBegin() and TxEnd() (which are in TxDataInterface) to support transaction, and those are data service layer interfaces, and has nothing to do with the database access layer. The use case also implemented the “EnableTx()” interface, which basically switched the underline database handler from sql.DB to sql.Tx.

{{< highlight go "linenos=table,linenostart=1" >}}
// The use case of ModifyAndUnregister without transaction
func (ruc *RegistrationUseCase) ModifyAndUnregister(user *model.User) error {
	return modifyAndUnregister(ruc, user)
}

// The use case of ModifyAndUnregister with transaction
func (ruc *RegistrationUseCase) ModifyAndUnregisterWithTx(user *model.User) error {
	tdi, err := ruc.TxDataInterface.TxBegin()
	if err != nil {
		return errors.Wrap(err, "")
	}
	ruc.EnableTx()
	return tdi.TxEnd(func() error {
		// wrap the business function inside the TxEnd function
		return modifyAndUnregister(ruc, user)
	})
}

// The business function will be wrapped inside a transaction and inside a non-transaction function
// It needs to be written in a way that every error will be returned so it can be catched by TxEnd() function,
// which will handle commit and rollback
func modifyAndUnregister(ruc *RegistrationUseCase, user *model.User) error {
	udi := ruc.UserDataInterface
	err := modifyUser(udi, user)
	if err != nil {
		return errors.Wrap(err, "")
	}
	err = unregisterUser(udi, user.Name)
	if err != nil {
		return errors.Wrap(err, "")
	}
	return nil
}

func (ruc *RegistrationUseCase) EnableTx() {
	// Only UserDataInterface need transaction support here. If there are other data services need it,
	// then they also need to enable transaction here
	ruc.UserDataInterface.EnableTx(ruc.TxDataInterface)
}
{{< / highlight >}}

<br/>

Why do I call the function “EnbaleTx” in “TxDataInterface” to replace the underline database handler instead of doing it directly in the use case? Because sql.DB and sql.Tx are several layers down, to do that will mess up the dependency. The trick is to have TxBegin() and TxEnd() on each layer and call them layer by layer down to maintain the appropriate dependency relationship.

**Transaction interface on the data service layer**

We talked about transaction functions on the use case layer and the data store layer, and we also need transaction functions in the data service layer to connect those two together. The following code is the transaction interface (“TxDataInterface”) for the data service layer. “TxDataInterface” is a data service layer interface created solely for facilitating transaction. The interface only needs to be implemented once for each database. There is also an “EnableTxer” interface ( This is a data service layer interface, don’t be confused with the “EnableTxer” interface in use case layer), which will enable the transaction support for a data service type, such as “UserDataInterface”.

{{< highlight go "linenos=table,linenostart=1" >}}
// TxDataInterface represents operations needed for transaction support.
// It only needs to be implemented once for each database
// For sqlGdbc, it is implemented for SqlDBTx in transaction.go
type TxDataInterface interface {
	// TxBegin starts a transaction. It gets a DB handler from the receiver and return a TxDataInterface, which has a
	// *sql.Tx inside. Any data access wrapped inside a transaction will go through the *sql.Tx
	TxBegin() (TxDataInterface, error)
	// TxEnd is called at the end of a transaction and based on whether there is an error, it commits or rollback the
	// transaction.
	// txFunc is the business function wrapped in a transaction
	TxEnd(txFunc func() error) error
	// Return the underline transaction handler, sql.Tx
	GetTx() gdbc.SqlGdbc
}

// This interface needs to be included in every data service interface that needs transaction support
type EnableTxer interface {
	// EnableTx enables transaction, basically it replaces the underling database handle sql.DB with sql.Tx
	EnableTx(dataInterface TxDataInterface)
}

// UserDataInterface represents interface for user data access through database
type UserDataInterface interface {
...
	Update(user *model.User) (rowsAffected int64, err error)
	// Insert adds a user to a database. The returned resultUser has a Id, which is auto generated by database
	Insert(user *model.User) (resultUser *model.User, err error)
	// Need to add this for transaction support
	EnableTxer
}
{{< / highlight >}}

<br/>

The following code is the implementation for “TxDataInterface”. “TxDataSql” is the concrete type for “TxDataInterface”. It calls underline database handler’s begin and end function to do the real transaction work.

{{< highlight go "linenos=table,linenostart=1" >}}
// TxDataSql is the generic implementation for transaction for SQL database
// You only need to do it once for each SQL database
type TxDataSql struct {
	DB gdbc.SqlGdbc
}

func (tds *TxDataSql) TxEnd(txFunc func() error) error {
	return tds.DB.TxEnd(txFunc)
}

func (tds *TxDataSql) TxBegin() (dataservice.TxDataInterface, error) {

	sqlTx, error := tds.DB.TxBegin()
	tdi := TxDataSql{sqlTx}
	tds.DB = tdi.DB
	return &tdi, error
}
func (tds *TxDataSql) GetTx() gdbc.SqlGdbc {
	return tds.DB
}
{{< / highlight >}}

<br/>

**Transaction strategy:**

You may ask why I need “TxDataSql” in above code? It is true that transaction can be implemented without it, actually it was working that way before. However, I sill need to implement “TxDataInterface” in some data service to begin and end a transaction. Since that is done in use case layer, which doesn’t know which data service type implemented the interface, you have to implement the “TxDataInterface” on every data service interface ( for example, “UserDataInterface”, and “CourseDataInterface” ) to guarantee that “use case” won’t choose the wrong “data service” that doesn’t have the interface. After creating “TxDataSql”, I only need to implement “TxDataInterface” once in “TxDataSql”, then each data service type only needs to implement “EnableTx()”.

{{< highlight go "linenos=table,linenostart=1" >}}
// UserDataSql is the SQL implementation of UserDataInterface
type UserDataSql struct {
	DB gdbc.SqlGdbc
}

func (uds *UserDataSql) EnableTx(tx dataservice.TxDataInterface) {
	uds.DB = tx.GetTx()
}

func (uds *UserDataSql) FindByName(name string) (*model.User, error) {
	//logger.Log.Debug("call FindByName() and name is:", name)
	rows, err := uds.DB.Query(QUERY_USER_BY_NAME, name)
	if err != nil {
		return nil, errors.Wrap(err, "")
	}
	defer rows.Close()
	return retrieveUser(rows)
}
{{< / highlight >}}

<br/>

The above code is the implementation for “UserDataService”. “EnableTx()” method retrieves sql.Tx from “TxDataInterface” and replace the sql.DB inside “UserDataSql” to sql.Tx.

Data access method (for example, FindByName() ) is shared between transaction and non-transaction code and doesn’t need to know whether the passed receiver “UserDataSql.DB” is sql.DB or sql.Tx.

**Dependency leak:**

There is one flaw in the implementation, which breaks my design and makes it imperfect. It is the function “GetTx()” in “TxDataInterface”, which is a data service layer interface, thus it shouldn’t depend on gdbc.SqlGdbc (database interface). You may think the implementation code of data service layer need to access database anyway, which is true for now. However, you can change the implementation to call a gRPC Microservice in the future. If the interface doesn’t depend on SQL interface, you are free to change the implementation, but if not, the interface sticks with SQL forever even though your implementation has changed.

Why is it the only place to break the dependency? Because for other interfaces, the container is responsible to create concrete types, and the rest part of the application only uses the interface. But for transaction, after the concrete type is created, the underline database handler needs to be replace from sql.DB to sql.Tx, which breaks the design.

Is there a fix for it? Yes, the container can create sql.Tx instead of sql.DB for functions needing transaction so I don’t need to do it in use case level later. However, a flag is needed in the configuration file to indicate whether a function needs transaction or not for every function in a use case. That is too big a change, so I decided to live with an imperfect design for now and try to revisit it later.

## Benefit:

With this implementation, the transaction code is almost transparent to business logic (except the flaw I mentioned above), you don’t have the database level transaction code such as Tx.Begin, Tx.Commit and Tx.Rollback (you do need the business level function Tx.Begin and Tx.End though) mixed with business logic code, you don’t even have those in your persistence code. To enable transaction on the use case layer, you just need to implement EnableTx() on the use case and wrap the business function among “TxBegin()”, EnableTx() and “TxEnd()” as above example showed. On persistence layer, most transaction code has already been implemented by “txDataService.go”, you just need to implement “EnableTx” for a particular data service. The real meat for transaction support is in “transaction.go” file, which implemented “Transactioner” interface, which has four methods, “Rollback”, “Commit”, “TxBegin” and “TxEnd”

## **Steps to add transaction support to a use case:**

Let say we need to add transaction support for one function in the use case “listCourse”, the following are the steps

1. Implement “EnableTxer” interface in list course use case ( “listCourse.go”)

1. Implement “EnableTxer” interface in the domain model’s ( “course”) data service layer ( courseDataMysql.go)

1. Create a new transaction enabled function and wrap the existing business function among “TxBegin()”, EnableTx() and “TxEnd()”

## **Limitations:**

First, it is still not declarative transaction yet; second it didn’t fully achieved #4 in the requirement. To change a use case function from non-transaction to transaction, you either create a new function with transaction enabled, which need to change the calling function; or you modify the existing function and wrap it into a transaction and both of them need small code change. In order to achieve #4, a lot more code need to be added, so I postpone it to a later time. Third, it doesn’t support nested transaction, so you need to manually make sure nested transaction is not happening in the code. If the code-base is not too complex, this is easy to do. If you have a really complex code-base with a lot of transaction and non-transaction code intervened, then it is time to implement nested transaction or find a solution supporting it. I didn’t take time to look into how much effort it takes to add nested transaction, but it may not be trivial. If you are interested in it, [here](https://github.com/golang/go/issues/7898)³ are some discussions. So far, for most use cases, the current solution probably strikes a good balance between the effort and the reward.

## **Applied Scope:**

First, it only supports transaction for SQL database. If you have a NoSql database, it won’t work (Most NoSql databases don’t support transaction anyway). Second, if your transaction boundary across databases (for example, among different Microservers), then it won’t work. The common idiom for that situation is to use [Saga](https://www.youtube.com/watch?v=xDuwrtwYHu8)⁴, basically you write a compensation action for each action in a transaction and execute the compensation action one by one during the rollback phase. It should not be difficult to add Sage solution in the current framework.

## Other database related issues:

**Close connection**

I never called Close() function for database connections because there is no need to do it. You either pass in a sql.DB or a sql.Tx as a receiver for a persistence function. For sql.DB, the database will automatically create a connection pool and manage connections for you. When a connection is done, it will be returned to the connection pool and no need to close. For sql.Tx, at the end of a transaction, you either commit or rollback, after that the connection is returned to the connection pool and no need to close. Please see reference [here](https://www.vividcortex.com/blog/2015/09/22/common-pitfalls-go/)⁵ and [here](http://go-database-sql.org/connection-pool.html)⁶ for detail.

**O/R mapping**

I looked at couple O/R mapping libraries briefly and didn’t find them providing the needed functionality. I think O/R mapping is a good fit for two situations. First, your application is mostly CRUD, not much query or search; second, the developers are not familiar with SQL. If those are not the case, O/R mapping doesn’t provide much help. There are two features I want from an extension database library, one is loading sql.row to my domain model structs (include handling of NULL value) , the other is automatically closing the sql types such as sql.statement or sql.rows. There are some [sql extension](https://github.com/jmoiron/sqlx)⁷ libraries seem to provide at least some of those functionalities. I haven’t got a chance to try it yet, but it seems worth a try.

**Defer:**

You will make a lot of repeated calls to close database types (such as statements, rows ) when work with a database, like the one-“defer row.close()” in the following code. You want to do it right though, which is always put “defer” after error handling function because if not, when there is an error, “rows” will be nil, which will cause panic and the error handling code will not be executed.

{{< highlight go "linenos=table,linenostart=1" >}}

func (uds *UserDataSql) Find(id int) (*model.User, error) {
	rows, err := uds.DB.Query(QUERY_USER_BY_ID, id)
	if err != nil {
		return nil, errors.Wrap(err, "")
	}
	defer rows.Close()
	return retrieveUser(rows)
}
{{< / highlight >}}
<br/>

**Panic:**

I saw a lot of Go database code generated a panic instead of an error when there is a database error, which could cause a problem in Microservice environment, where you always want to keep the service running. Let’s say when you have a SQL error in an update statement, then users won’t be able to access that function, which is bad. But if because that, the whole service or website is shutdown, then it is much worse. So the right way to do is propagating the error to upper level and let it decide what to do. What if it is a third party library, who generates the panic? Then, you need to catch and recover from the panic to keep your service running. I gave an example of it in another article “[Application Logging](https://jfeng45.github.io/posts/go_logging_and_error_handling/)”⁸.

## Source Code:

The complete code is in [github](https://github.com/jfeng45/servicetmpl): https://github.com/jfeng45/servicetmpl

## Other articles:

Please read the rest of the articles in this series in “[Go Microservice with Clean Architecture](https://jfeng45.github.io/posts/clean_architecture_with_go/)”.

## References:

[1][db transaction in golang]
(https://stackoverflow.com/questions/26593867/db-transaction-in-golang)

[2][database/sql Tx — detecting Commit or Rollback](https://stackoverflow.com/questions/16184238/database-sql-tx-detecting-commit-or-rollback/23502629#23502629)

[3][database/sql: nested transaction or save point support]
(https://github.com/golang/go/issues/7898)

[4][GOTO 2015 • Applying the Saga Pattern • Caitie McCaffrey — YouTube]
(https://www.youtube.com/watch?v=xDuwrtwYHu8)

[5][Common Pitfalls When Using database/sql in Go]
(https://www.vividcortex.com/blog/2015/09/22/common-pitfalls-go/)

[6][Go database/sql tutorial]
(http://go-database-sql.org/connection-pool.html)

[7][sqlx]
(https://github.com/jmoiron/sqlx)

[8][Go Microservice with Clean Architecture: Application Logging]
(https://jfeng45.github.io/posts/go_logging_and_error_handling/)
