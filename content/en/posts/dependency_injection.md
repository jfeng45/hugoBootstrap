---
title: "Go Microservice with Clean Architecture: Dependency Injection"
shortTitle: "Dependency Injection"
date: 2019-07-29
draft: false
tags: ["Golang", "Clean Architecture", "Application Container", "Dependency Injection"]
categories: ["Go Microservice"]

canonicalUrl: "https://medium.com/@jfeng45/go-microservice-with-clean-architecture-dependency-injection-82cbd3ecb9f3"

description: "Application container uses Dependency Injection to create concrete types and inject them into each function. Under the core, it uses the factory method pattern. "

---

In Clean Architecture, each layer of the application ( use case, data service and domain model) only depends on interface of other layers instead of concrete types. At run-time, the the [application container](https://jfeng45.github.io/en/posts/application_container/)¹ is responsible for creating concrete types and inject them into each function, the technology it used is called [Dependency Injection](https://www.martinfowler.com/articles/injection.html#FormsOfDependencyInjection)². The following is the requirements.

**Requirements for dependency relationship on container:**

1. The container package is the only package that depends on concrete types and many outside libraries because it needs to create concrete types. All other packages in the application mostly depend only on interfaces.

1. The outside libraries can include DBs and DB connections, gRPC connections, HTTP connections, SMTP servers, MQ and so on.

1. The concrete types of resource handlers mentioned in #2 only need to be created once and put in a registry, and all later requests will retrieve them from the registry.

1. Only the use case layer needs to access and depend on the container package.

At the core of dependency injection is the factory method pattern.

## Factory Method Pattern:

Implementing factory method pattern is not difficult, [here](https://stackoverflow.com/a/49714445)³ is how to do it in Go. The difficult part is to make it extensible, basically how to avoid modifying the code when you add a new factory.

There are different ways of dealing with new factories.

* (1)[ Using if-else statement](https://stackoverflow.com/questions/3434466/creating-a-factory-method-in-java-that-doesnt-rely-on-if-else)⁴

* (2) [Using a map to save different factories](https://stackoverflow.com/questions/3434466/creating-a-factory-method-in-java-that-doesnt-rely-on-if-else/3434505#3434505)⁵

* (3) Using reflection to generate new concrete types.

The #1 is not a good option because you need to modify existing code whenever you add a new type. The #3 is the best one because there is no change to existing code when adding a new factory. In Java, I will use #3 because Java has a very elegant implementation of reflection. You can do something like “(Animal)Class.forName(“className”).newInstance()”, basically you can pass in the name of a class as a string parameter in a function and create a new instance of a type from it by reflection and then convert it to the appropriate type (could be a supper type), which is very powerful. Since Go’s reflection is not as good as Java’s, #3 is not a good option. In Go, the instance created by reflection is a reflection type not the real type and you can’t convert the type between the reflection type and the real type, they are in two different world, which makes reflection in Go really difficult to use. So I choose #2, which is better than #1, but you need to make limited code changes when adding a new type.

The following is the code for data store factory. It has a “dsFbInterface”, which has a “Build” function need to be implemented by each data store. “Build” is the key component of the factory. The “dsFbMap” is the map between the code for each database (or gRPC) and the real factory. This is the part that needs to be changed when there is a database added.

{{< highlight go "linenos=table,linenostart=1" >}}

// To map "database code" to "database interface builder"
// Concreate builder is in corresponding factory file. For example, "sqlFactory" is in "sqlFactory".go
var dsFbMap = map[string]dsFbInterface{
	config.SQLDB:      &sqlFactory{},
	config.COUCHDB:    &couchdbFactory{},
	config.CACHE_GRPC: &cacheGrpcFactory{},
}

// DataStoreInterface serve as a marker to indicate the return type for Build method
type DataStoreInterface interface{}

// The builder interface for factory method pattern
// Every factory needs to implement Build method
type dsFbInterface interface {
	Build(container.Container, *config.DataStoreConfig) (DataStoreInterface, error)
}

//GetDataStoreFb is accessors for factoryBuilderMap
func GetDataStoreFb(key string) dsFbInterface {
	return dsFbMap[key]
}
{{< / highlight >}}
<br/>

The following is the code for “sqlFactory”, which implemented the “dsFbInterface” in above code. It creates the data store for MySql database. In “build” function, it first retrieves the data store (MySql) from the registry, if found, then return, otherwise it creates a new one and put it into the registry. Because the registry can store data for any type, we need to convert the return value to the appropriate type (*sql.DB) after retrieving. “databasehandler.SqlDBTx” is the concrete type implemented “SqlGdbc” interface. It is created to facilitate transaction support. “sql.Open()” is called to open the database connection, but it didn’t do any action to connect the database. So, “db.Ping()” is called to make sure the database is up running.
{{< highlight go "linenos=table,linenostart=1" >}}

// sqlFactory is receiver for Build method
type sqlFactory struct{}

// implement Build method for SQL database
func (sf *sqlFactory) Build(c container.Container, dsc *config.DataStoreConfig) (DataStoreInterface, error) {
	key := dsc.Code
	//if it is already in container, return
	if value, found := c.Get(key); found {
		sdb := value.(*sql.DB)
		sdt := databasehandler.SqlDBTx{DB: sdb}
		logger.Log.Debug("found db in container for key:", key)
		return &sdt, nil
	}

	db, err := sql.Open(dsc.DriverName, dsc.UrlAddress)
	if err != nil {
		return nil, errors.Wrap(err, "")
	}
	// check the connection
	err = db.Ping()
	if err != nil {
		return nil, errors.Wrap(err, "")
	}
	dt := databasehandler.SqlDBTx{DB: db}
	c.Put(key, db)
	return &dt, nil

}
{{< / highlight >}}
<br/>

## Data service factory

The data service layer uses factory method patter to create data service types. There are different variations on how to apply this pattern. I used three different strategies when building data service factories and each of them has its pros and cons. I will explain them in detail, so you can make a decision on which one to use in a particular situation.

**Basic factory**

The easiest one is “cacheGrpcFactory”, because there is only one underlying implementation of the data store (which is gRPC), so it is straightforward to create just one factory. Always use this one as far as you can.

**Second level factory**

For the database factory, that is not the case. Because we need to support multiple databases at the same time for each data service, another level of factory is needed, which means for each data service type, such as the “UserDataService”, we need a separate factory for each database supported. Right now, since there are two databases, we need two factories.

![userDataServiceFactory](/images/userDataServiceFactory.jpg)

You can see from the above image, we need four files to finish the “UserDataService”, among them “userDataServiceFactoryWrapper.go” is the wrapper which calls the real factory in “userdataservicefactory” folder. “couchdbUserDataServiceFactory.go” and “sqlUserDataServiceFactory.go” are the real factories for CouchDB and MySql database. “userDataServiceFactory.go” defines the interface. If you have many data services, then you will create a lot of boilerplate code.

**Simplified factory**

Is there a way to simplify it? Yes, that is the third way, but it also comes with some cost. The following is the code for “courseDataServiceFactory.go”. You can see it is much simplified with only one file instead of four as before. The code is similar to the “userDataServiceFactory” we just talked about, how does it get the code simplified?

The key is to create a unified interface for the underline database handler. In “courseDataServiceFactory.go”, I can get the unified underline database handler after calling “dataStoreFactory” and set the DB of “CourseDataServiceInterface” with correct “gdbc” ( which can be any database handler as long as it implements “gdbc” interface).
{{< highlight go "linenos=table,linenostart=1" >}}

var courseDataServiceMap = map[string]dataservice.CourseDataInterface{
	config.COUCHDB: &couchdb.CourseDataCouchdb{},
	config.SQLDB:   &sqldb.CourseDataSql{},
}

// courseDataServiceFactory is an empty receiver for Build method
type courseDataServiceFactory struct{}

// GetCourseDataServiceInterface is an accessor for factoryBuilderMap
func GetCourseDataServiceInterface(key string) dataservice.CourseDataInterface {
	return courseDataServiceMap[key]
}

func (tdsf *courseDataServiceFactory) Build(c container.Container, dataConfig *config.DataConfig) (DataServiceInterface, error) {
	dsc := dataConfig.DataStoreConfig
	dsi, err := datastorefactory.GetDataStoreFb(dsc.Code).Build(c, &dsc)
	if err != nil {
		return nil, errors.Wrap(err, "")
	}
	gdbc := dsi.(gdbc.Gdbc)
	gdi := GetCourseDataServiceInterface(dsc.Code)
	gdi.SetDB(gdbc)
	return gdi, nil
}
{{< / highlight >}}
<br/>

The only drawback is that for any database supported, I need to implement both “SqlGdbc” and “NoSqlGdbc” interface in the following code, even though it will only use one of them, the other is just empty implementation (to fulfill the interface) without being used. If you only have couple databases to support, this could be a viable solution, otherwise it becomes increasingly unmanageable.
{{< highlight go "linenos=table,linenostart=1" >}}

// SqlGdbc (SQL Go database connection) is a wrapper for SQL database handler 
type SqlGdbc interface {
	Exec(query string, args ...interface{}) (sql.Result, error)
	Prepare(query string) (*sql.Stmt, error)
	Query(query string, args ...interface{}) (*sql.Rows, error)
	QueryRow(query string, args ...interface{}) *sql.Row
	// If need transaction support, add this interface
	Transactioner
}

// NoSqlGdbc (NoSQL Go database connection) is a wrapper for NoSql database handler.
type NoSqlGdbc interface {
	// The method name of underline database was Query(), but since it conflicts with the name with Query() in SqlGdbc,
	// so have to change to a different name
	QueryNoSql(ctx context.Context, ddoc string, view string) (*kivik.Rows, error)
	Put(ctx context.Context, docID string, doc interface{}, options ...kivik.Options) (rev string, err error)
	Get(ctx context.Context, docID string, options ...kivik.Options) (*kivik.Row, error)
	Find(ctx context.Context, query interface{}) (*kivik.Rows, error)
	AllDocs(ctx context.Context, options ...kivik.Options) (*kivik.Rows, error)
}

// gdbc is an unified way to handle database connections. 
type Gdbc interface {
	SqlGdbc
	NoSqlGdbc
}
{{< / highlight >}}
<br/>

There is another side effect besides the one talked above. In the following code, the “SetDB” function in the “CourseDataInterface” breaks the dependency. Because “CourseDataInterface” is the data service layer interface, it shouldn’t depend on “gdbc” interface, which is one layer down. This is the second flaw in the dependency relationship of this application, and the first one is in the [transaction support](https://jfeng45.github.io/en/posts/transaction_support/)⁶ module. There is no good fix for it, if you don’t like it just don’t use it. Try the second level factory similar to “userFataServiceFactory”.
{{< highlight go "linenos=table,linenostart=1" >}}

import (
	"github.com/jfeng45/servicetmpl/model"
	"github.com/jfeng45/servicetmpl/tool/gdbc"
)

// CourseDataInterface represents interface for persistence service for course data
// It is created for POC of courseDataServiceFactory, no real use.
type CourseDataInterface interface {
	FindAll() ([]model.Course, error)
	SetDB(gdbc gdbc.Gdbc)
}
{{< / highlight >}}
<br/>

**Which one to use?**

What influences the decision on whether to choose the simplified factory or the second level factory ? It depends on the direction of changes. If you need to support a lot of new databases, but not many new data service ( which is decided by the number of domain model types), then the second level factory is better, because most changes will happen in the data store factory. But if the supported database won’t change much, and the number of data service could increase a lot, then choose the simplified factory. What if both of them could increase a lot? Then just use the second level factory and live with a lot of boilerplate code.

When to use the basic factory vs the second level factory? Actually, even if you need to support multiple databases, but not at the same time, you can still use the basic one. For example, you need to switch from MySQL to MongoDB, even though there are two different databases, but after switching, you only use MongoDB, then you can still use the basic factory. For the basic factory, when there are multiple types, you need to change the code to make the switch (but for the second level factory, you only change the config file), so it is acceptable to change the code if you don’t do it often.

## Dependency Injection Library

There are several dependency injection libraries in Go, why didn’t I use them? I purposely not using any libraries at the beginning so I can have a better control on the project structure, only after the whole application structure is laid out, I will consider replacing some components of the application with outside libraries.

I briefly looked at couple popular libraries, one is [Dig](https://blog.drewolson.org/dependency-injection-in-go)⁷ from [Uber](https://github.com/uber-go/dig)⁸, the other is [Wire](https://blog.drewolson.org/go-dependency-injection-with-wire)⁹ from [Google](https://github.com/google/wire)¹⁰. Dig uses reflection and Wire uses code generation. I don’t like either approach, but since Go currently doesn’t support generics, those are the only options available. Even though I don’t like their approach, I have to admit that those two libraries are more full-featured dependency injection libraries.

I did give Dig a try and found it didn’t make the code simpler, so I decided to stay with the current solution. In Dig, you create “build” function for each concrete type and then register it with the container, then the container will auto-wire them together to create the top level type. The complexity of this application is because we need to support two implementations of databases, thus two different database handlers and two different sets of data service implementations for each domain model. There is no way to make this part simpler in Dig, you still need to create all the factories and then register them with the container. Of course, you can use “if-else” approach to implement the factory, which will make the code simpler, but you need to put a lot more effort to maintain the code in the future.

My approach is simple and easy to extend and it also supports loading configuration from a file, but you need to understand how things are putting together to extend it. The additional function Dig providing is auto-wire. If your application has a lot of types and complex dependency relationship among the types, then probably you want to switch to Dig or Wire, otherwise stay with the current solution.

## Interface design

The following is the code for “userDataServiceFactoryWrapper”.
{{< highlight go "linenos=table,linenostart=1" >}}
// DataServiceInterface serves as a marker to indicate the return type for Build method
type DataServiceInterface interface{}

// userDataServiceFactory is a empty receiver for Build method
type userDataServiceFactoryWrapper struct{}

func (udsfw *userDataServiceFactoryWrapper) Build(c container.Container, dataConfig *config.DataConfig) 
    (DataServiceInterface, error) {
	key := dataConfig.DataStoreConfig.Code
	udsi, err := userdataservicefactory.GetUserDataServiceFb(key).Build(c, dataConfig)
	if err != nil {
		return nil, errors.Wrap(err, "")
	}
	return udsi, nil
}
{{< / highlight >}}
<br/>

You may notice the return type for the “Build()” function is “DataServiceInterface” which is an empty interface, why do we need an empty interface? Can we write the function like this, basically to replace “DataServiceInterface” with “interface{}”?
{{< highlight go "linenos=table,linenostart=1" >}}
// userDataServiceFactory is a empty receiver for Build method
type userDataServiceFactoryWrapper struct{}

func (udsfw *userDataServiceFactoryWrapper) Build(c container.Container, dataConfig *config.DataConfig) 
    (interface{}, error) {
...
}
{{< / highlight >}}
<br/>

If you replace the return type from “DataServiceInterface” to “interface{}”, it is essentially the same. The benefit of “DataServiceInterface” is that it can tell me the returned type of the function, which is the data service interface; actually the real return type is “dataservice.UserDataInterface”, but “DataStoreInterface” is good enough for now, another cheating solution to make life little easier.

## Conclusion:

Application container uses Dependency Injection to create concrete types and inject them into each function. Under the core, it uses the factory method pattern. There are three ways to implement it in Go and the best one is saving different factories in a map. There are also different ways to apply the factory method pattern to data service layer and they all have pros and cons. You need to choose the right one based on the change direction of your application.

## Source Code:

The complete code is in [github](https://github.com/jfeng45/servicetmpl): https://github.com/jfeng45/servicetmpl

## Other articles:

Please read the rest of the articles in this series in “[Go Microservice with Clean Architecture](https://jfeng45.github.io/en/posts/clean_architecture_with_go/)”.

## Reference:

[1][Go Microservice with Clean Architecture: Application Container]
(https://jfeng45.github.io/en/posts/application_container/)

[2] [Inversion of Control Containers and the Dependency Injection pattern]
(https://www.martinfowler.com/articles/injection.html#FormsOfDependencyInjection)

[3]][Golang Factory Method]
(https://stackoverflow.com/a/49714445)

[4][Creating a factory method in Java that doesn’t rely on if-else]
(https://stackoverflow.com/questions/3434466/creating-a-factory-method-in-java-that-doesnt-rely-on-if-else)

[5][Tom Hawtin’s answer]
(https://stackoverflow.com/questions/3434466/creating-a-factory-method-in-java-that-doesnt-rely-on-if-else/3434505#3434505)

[6][Go Microservice with Clean Architecture: Transaction Support]
(https://jfeng45.github.io/en/posts/transaction_support/)

[7][Dependency Injection in Go] 
(https://blog.drewolson.org/dependency-injection-in-go)

[8][Uber’s dig]
(https://github.com/uber-go/dig)

[9][Go Dependency Injection with Wire]
(https://blog.drewolson.org/go-dependency-injection-with-wire)

[10][Google’s Wire: Automated Initialization in Go]
(https://github.com/google/wire)
