---
title: "清晰架构（Clean Architecture）的Go微服务: 事物管理"
shortTitle: "事物管理"
date: "2019-07-24"
draft: false
tags: ["Golang","Database Transaction", "Clean Architecture"]
categories: ["Go Microservice"]

tags: ["Golang", "清晰架构", "事物管理"]
categories: ["Go微服务"]

description: "Go中的业务级事务管理系统，它完成了声明式事务的大部分功能。"

---

为了支持业务层中的事务，我试图在Go中查找类似Spring的声明式事务管理，但是没找到，所以我决定自己写一个。 事务很容易在Go中实现，但很难做到正确地实现。

##### **需求:**

1. 将业务逻辑与事务代码分开。
在编写业务用例时，开发者应该只需考虑业务逻辑，不需要同时考虑怎样给业务逻辑加事务管理。如果以后需要添加事务支持，你可以在现有业务逻辑的基础上进行简单封装，而无需更改任何其他代码。事务实现细节应该对业务逻辑透明。

1. 事务逻辑应该作用于用例层（业务逻辑）
不在持久层上。

1. 数据服务（数据持久性）层应对事务逻辑透明。
这意味着持久性代码应该是相同的，无论它是否支持事务

1. 你可以选择延迟支持事物。
你可以先编写没有事务的用例，稍后可以在不修改现有代码的情况下给该用例加上事务。你只需添加新代码。

我最终的解决方案还不是声明式事务管理，但它非常接近。创建一个真正的声明式事务管理需要付出很多努力，因此我构建了一个可以实现声明式事务的大多数功能的事务管理，同时又没花很多精力。

##### **方案:**

最终解决方案涉及本程序的所有层级，我将逐一解释它们。

**数据库链接封装**


在Go的“sql”lib中，有两个数据库链接sql.DB和sql.Tx. 不需要事务时，使用sql.DB访问数据库; 当需要事务时，你使用sql.Tx. 为了共享代码，持久层需要同时支持两者。 因此需要对数据库链接进行封装，然后把它作为数据库访问方法的接收器。 我从[这里](https://stackoverflow.com/questions/26593867/db-transaction-in-golang)¹得到了粗略的想法。

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

数据库实现类型SqlDBTx和sqlConnTx都需要实现SqlGdbc接口（包括“Transactioner”）接口才行。 需要为每个数据库（例如MySQL， CouchDB）实现“Transactioner”接口以支持事务。

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

**数据库存储层（datastore layer）的事物管理代码**

以下是“Transactioner”接口的实现代码，其中只有TxBegin（）是在SqlDBTx（sql.DB）上实现，因为事务从sql.DB开始，然后所有事务的其他操作都在SqlConnTx（sql.Tx）上。 我从[这里](https://stackoverflow.com/questions/16184238/database-sql-tx-detecting-commit-or-rollback/23502629#23502629)²得到了这个想法。

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

**用例层的事物接口**

在用例层中，你可以拥有相同业务功能的一个函数的两个版本，一个支持事务，一个不支持，并且它们的名称可以共享相同的前缀，而事务可以添加“withTx”作为后缀。 例如，在以下代码中，“ModifyAndUnregister”是不支持事务的那个，“ModifyAndUnregisterWithTx”是支持事务的那个。 “EnableTxer”是用例层上唯一的事务支持接口，任何支持事务的“用例”都需要它。 这里的所有代码都在是用例层级（包括“EnableTxer”）代码，不涉及数据库内容。

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

以下是不包含事务的业务逻辑代码的示例。 “modifyAndUnregister（ruc，user）”是事务和非事务用例函数共享的业务功能。 你需要使用TxBegin（）和TxEnd（）（在TxDataInterface中）来包装业务功能以支持事务，这些是数据服务层接口，并且与数据库访问层无关。 该用例还实现了“EnableTx（）”接口，该接口实际上将底层数据库链接从sql.DB切换到sql.Tx.

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

为什么我需要在“TxDataInterface”中调用函数“EnbaleTx”来替换底层数据库链接而不是直接在用例中执行？ 因为sql.DB和sql.Tx层级要比用例层低几个级别，直接调用会搞砸依赖关系。 保持合理依赖关系的诀窍是在每一层上都有TxBegin（）和TxEnd（）并逐层调用它们以维持合理的依赖关系。

**数据服务层的事物接口**

我们讨论了用例层和数据存储层上的事务功能，我们还需要数据服务层中的事务功能将这两者连接在一起。 以下代码是数据服务层的事务接口（“TxDataInterface”）。 “TxDataInterface”是仅为事物管理而创建的数据服务层接口。 每个数据库只需要实现一次。 还有一个“EnableTxer”接口（这是一个数据服务层接口，不要与用例层中的“EnableTxer”接口混淆），实现“EnableTxer”接口将开启数据服务类型对事务的支持，例如， 如果想要“UserDataInterface”支持事物，就需要它实现“EnableTxer”接口。

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

以下代码是“TxDataInterface”的实现。 “TxDataSql”是“TxDataInterface”的具体类型。 它调用底层数据库链接的开始和结束函数来执行真正的事务操作。

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

**事物策略:**

你可能会问为什么我在上面的代码中需要“TxDataSql”？ 确实可以在没有它的情况下实现事务，实际上最开的程序里就没有它。 但是我还是要在某些数据服务中实现“TxDataInterface”来开始和结束事务。 由于这是在用例层中完成的，用例层不知道哪个数据服务类型实现了接口，因此必须在每个数据服务接口上实现“TxDataInterface”（例如，“UserDataInterface”和“CourseDataInterface”）以保证 “用例层”不会选择没有接口的“数据服务（data service）”。 在创建“TxDataSql”之后，我只需要在“TxDataSql”中实现一次“TxDataInterface”，然后每个数据服务类型只需要实现“EnableTx（）”就行了。

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

上面的代码是“UserDataService”接口的实现程序。 “EnableTx（）”方法从“TxDataInterface”获得sql.Tx并将“UserDataSql”中的sql.DB替换为sql.Tx.

数据访问方法（例如，FindByName（））在事务代码和非事务代码之间共享，并且不需要知道“UserDataSql.DB”是sql.DB还是sql.Tx.

**依赖关系漏洞:**

上面的代码实现中存在一个缺陷，这会破坏我的设计并使其不完美。它是“TxDataInterface”中的函数“GetTx（）”，它是一个数据服务层接口，因此它不应该依赖于gdbc.SqlGdbc（数据库接口）。你可能认为数据服务层的实现代码无论如何都需要访问数据库，当前这是正确的。但是，你可以在将来更改实现去调用gRPC微服务（而不是数据库）。如果接口不依赖于SQL接口的话，则可以自由更改实现，但如果不是，则即使你的接口实现已更改，该接口也会永久保留对SQL的依赖。

为什么它是本程序中打破依赖关系的唯一地方？因为对于其他接口，容器负责创建具体类型，而程序的其余部分仅使用接口。但是对于事务，在创建具体类型之后，需要将底层数据库处理程序从sql.DB替换为sql.Tx，这破坏了设计。

它有解决方法吗？是的，容器可以为需要事务的函数创建sql.Tx而不是sql.DB，这样我就不需要在以后的用例级别中替换它。但是，配置文件中需要一个标志来指示函数是否需要事务， 而且这个标志需要配备给用例中的每个函数。这是一个太大的改动，所以我决定现在先这样，以后再重新审视它。

##### **好处:**

通过这个实现，事务代码对业务逻辑几乎是透明的（除了我上面提到的缺陷）。业务逻辑中没有数据存储（datastore）级事务代码，如Tx.Begin，Tx.Commit和Tx.Rollback（但你确实需要业务级别事物函数Tx.Begin和Tx.End），不仅如此，你的持久性代码中也几乎没有数据存储级事务代码。 如需在用例层上启用事务，你只需要在用例上实现EnableTx（）并将业务函数封装在“TxBegin（）”，EnableTx（）和“TxEnd（）”中，如上例所示。 在持久层上，大多数事务代码已经由“txDataService.go”实现，你只需要为特定的数据服务（例如UserDataService）实现“EnableTx”。 事务支持的真正操作是在“transaction.go”文件中实现的，它实现了“Transactioner”接口，它有四个函数，“Rollback”, “Commit”, “TxBegin” 和 “TxEnd”。

##### **对用例增加事物支持的步骤:**

假设我们需要在用例“listCourse”中为一个函数添加事务支持，以下是步骤

1. 在列表课程用例（“listCourse.go”）中实现“EnableTxer”界面

1. 在域模型（“course”）数据服务层（courseDataMysql.go）中实现“EnableTxer”接口

1. 创建一个新的事务启用函数并将现有业务函数包装在“TxBegin（）”，EnableTx（）和“TxEnd（）”中

##### **缺陷:**

首先，它仍然不是声明​​式事物管理;第二，它没有完全达到需求中的＃4。要将用例函数从非事务更改为事务，你可以创建一个支持事务的新函数，它需要更改调用函数; 或者你修改现有函数并将其包装到事务中，这也需要代码更改。为了实现＃4，需要添加许多代码，因此我将其推迟到以后。第三，它不支持嵌套事务（Nested Transaction），因此你需要手动确保代码中没有发生嵌套事务。如果代码库不是太复杂，这很容易做到。如果你有一个非常复杂的代码库，有很多事务和非事务函数混在一起，那么手工做起来会比较困难，这是需要在程序中实现嵌套事务或找到已经支持它的方案。我没有花时间研究添加嵌套事务所需的工作量，但这可能并不容易。如果你对它感兴趣，[这里](https://github.com/golang/go/issues/7898)³是一些讨论。到目前为止，对于大多数情况而言，当前的解决方案可能是在代价不大的情况下的最佳方案。

##### **应用范围:**

首先，它只支持SQL数据库的事务。 如果你有NoSql数据库，它将无法工作（大多数NoSql数据库无论如何都不支持事务）。 其次，如果事务跨越了数据库的边界（例如在不同的微服务器之间），那么它将无法工作。 在这种情况下，你需要使用[Saga](https://www.youtube.com/watch?v=xDuwrtwYHu8)⁴。它的原理是为事物中的每个操作写一个补偿操作，然后在回滚阶段挨个执行每一个补偿操作。 在当前框架中添加Sage解决方案应该不难。

##### **其他数据库相关问题:**
<br/>

**关闭数据库链接（Close connection）**

我从来没有为数据库链接调用Close（）函数，因为没有必要这样做。 你可以传入sql.DB或sql.Tx作为持久性函数的接收器（receiver）。 对于sql.DB，数据库将自动创建链接池并为你管理链接。 链接完成后，它将返回到链接池，无需关闭。 对于sql.Tx，在事务结束时，你可以提交或回滚，之后链接将返回到连接池，而无需关闭。 请参阅[此处](https://www.vividcortex.com/blog/2015/09/22/common-pitfalls-go/)⁵ 和 [此处](http://go-database-sql.org/connection-pool.html)⁶ .

**对象关系映射（O/R mapping）**

我简要地查看了几个“O/R”映射库，但它们没有提供我所需要的功能。 我认为“O/R映射”只适合两种情况。 首先，你的应用程序主要是CRUD，没有太多的查询或搜索; 第二，开发人员不熟悉SQL。 如果不是这种情况，则O/R映射不会提供太多帮助。 我想从扩展数据库模块中获得两个功能，一个是将sql.row加载到我的域模型结构（包括处理NULL值）中（例如“User”），另一个是自动关闭sql类型，如sql.statement或sql.rows。 有一些sql扩展库似乎提供了至少部分这样的功能。 我还没有尝试，但似乎值得一试。

**延迟（Defer）:**

在进行数据库访问时，你将进行大量重复调用以关闭数据库类型（例如statements, rows）。例如以下代码中的“defer row.close（）”。 你想要记住这一点，要在错误处理函数之后调用“defer row.close（）”，因为如果不是这样，当出现错误时，“rows”将为nil，这将导致恐慌并且不会执行错误处理代码。

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

**恐慌（panic）:**

我看到很多Go数据库代码在出现数据库错误时抛出了恐慌（panic）而不是错误（error），这可能会导致微服务出现问题，因为在微服务环境中你通常希望服务一直运行。 假设当更新语句中出现SQL错误时，用户将无法访问该功能，这很糟糕。 但如果因为这个，整个微服务或网站被关闭，那就更糟了。 因此，正确的方法是将错误传播到上一级并让它决定要做什么。 因此正确的做法是不在你的程序中抛出panic，但如果第三方库抛出恐慌呢？ 这时你需要捕获恐慌并从中恢复以保持你的服务正常运行。 我在另一篇文章“[日志管理](https://jfeng45.github.io/posts/go_logging_and_error_handling/)”⁸中有具体示例.

##### **源程序:**

完整的源程序链接 [github](https://github.com/jfeng45/servicetmpl): https://github.com/jfeng45/servicetmpl

##### **其他文章:**

请在这里阅读本系列的其他文章 “[清晰架构（Clean Architecture）的Go微服务](https://jfeng45.github.io/posts/clean_architecture_with_go/)”.

##### **索引:**

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
