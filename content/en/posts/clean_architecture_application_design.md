---
title: "Go Microservice with Clean Architecture: Application Design"
shortTitle: "Clean Architecture Design"
date: 2019-07-22
draft: false
tags: ["Golang", "Clean Architecture", "gRPC"]
categories: ["Go Microservice"]

canonicalUrl: "https://medium.com/@jfeng45/go-microservice-with-clean-architecture-application-design-68f48802c8f"

description: "Described Clean Architecture design of this Go Microservice gRPC project, and the three business layers 
of the project. It also talked about two deviations that this project is different from Clean Architecture"

---

I created a Microservice with Go and gRPC and tried to apply best practice of application design and programming to this project. I wrote a series of articles about decisions and trade-offs I made when working on the project. This one is about application design.

The design of the application followed [Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)¹. There are three layers in business logic code: use case, domain model and data service.

There are three top level packages “usecase”, “model” and “dataservice”, and one for each layer. There is only one file, named after the package, in each top level package ( except model). The file defines the interface to the outside world for each package. The hierarchy of dependency from top level down is: “usecase”, “dataservice” and “model”. The upper level package depends on the lower level package and the dependency never goes reverse direction.

## **Use Case:**

“usecase” is the entry point of the application and has most of the business logic. I got some of the business logic ideas from this [article](https://medium.com/@hatajoe/clean-architecture-in-go-4030f11ec1b1)². There are three use cases “registration”, “listUser” and “listCourse”. each use case implements a business feature. The use cases may not resemble real world use case, and they are created to illustrate the design concepts. The following is the interface for registration use case:

{{< highlight go "linenos=table,linenostart=1" >}}

// RegistrationUseCaseInterface is for users to register themselves to an application. It has registration related functions.
// ModifyAndUnregisterWithTx() is the one supporting transaction, the other are not.
type RegistrationUseCaseInterface interface {
	// RegisterUser register a user to an application, basically save it to a database. The returned resultUser that has
	// a Id ( auto generated by database) after persisted
	RegisterUser(user *model.User) (resultUser *model.User, err error)
	// UnregisterUser unregister a user from an application by user name, basically removing it from a database.
	UnregisterUser(username string) error
	// ModifyUser change user information based on the User.Id passed in.
	ModifyUser(user *model.User) error
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
{{< / highlight >}}
<br/>

The "main” function will call “use case” through this interface, which only depends on the model layer.

The following is a partial code for “registration.go”, which implements the functions in “RegistrationUseCaseInterface”. “RegistrationUseCase” is the concrete struct. It has two members “UserDataInterface” and “TxDataInterface”. “UserDataInterface” can be used to call methods in data service layer (for example “UserDataInterface.Insert(user)”). “TxDataInterface” is used to implement transaction. The concrete types of them are created by the application container and injected into each function through dependency injection. Any use case code only depends on the data service interface and there is no dependency on database related code ( for example, sql.DB or sql.Stmt). Any database access code is executed through the data service interface.

{{< highlight go "linenos=table,linenostart=1" >}}
// RegistrationUseCase implements RegistrationUseCaseInterface.
// It has UserDataInterface, which can be used to access persistence layer
// TxDataInterface is needed to support transaction
type RegistrationUseCase struct {
	UserDataInterface dataservice.UserDataInterface
	TxDataInterface   dataservice.TxDataInterface
}

func (ruc *RegistrationUseCase) RegisterUser(user *model.User) (*model.User, error) {
	err := user.Validate()
	if err != nil {
		return nil, errors.Wrap(err, "user validation failed")
	}
	isDup, err := ruc.isDuplicate(user.Name)
	if err != nil {
		return nil, errors.Wrap(err, "")
	}
	if isDup {
		return nil, errors.New("duplicate user for " + user.Name)
	}
	resultUser, err := ruc.UserDataInterface.Insert(user)

	if err != nil {
		return nil, errors.Wrap(err, "")
	}
	return resultUser, nil
}
{{< / highlight >}}
<br/>

Usually one use case can have one or more functions. The above code shows the “RegisterUser” function. It first checks that the passed in parameter “user” is valid, then it checks that the user is not already registered, and finally it calls the data service layer to register the user.

## **Data service:**

Code in this layer handles direct database access. Here is the interface of data persistence layer for domain model “User”.

{{< highlight go "linenos=table,linenostart=1" >}}
// UserDataInterface represents interface for user data access through database
type UserDataInterface interface {
	// Remove deletes a user by user name from database.
	Remove(username string) (rowsAffected int64, err error)
	// Find retrieves a user from database based on a user's id
	Find(id int) (*model.User, error)
	// FindByName retrieves a user from database by User.Name
	FindByName(name string) (user *model.User, err error)
	// FindAll retrieves all users from database as an array of user
	FindAll() ([]model.User, error)
	// Update changes user information on the User.Id passed in.
	Update(user *model.User) (rowsAffected int64, err error)
	// Insert adds a user to a database. The returned resultUser has a Id, which is auto generated by database
	Insert(user *model.User) (resultUser *model.User, err error)
	// Need to add this for transaction support
	EnableTxer
}
{{< / highlight >}}
<br/>


The following is the code for MySql implementation of “insert” function in “UserDataInterface”. Here I use “gdbc.SqlGdbc” interface as a wrapper for database handler in order to support transaction. The “gdbc.SqlGdbc” can be sql.DB (which doesn’t support transaction) or sql.Tx ( which supports transaction) at run-time. By passing in “gdbc.SqlGdbc” as the receiver through UserDataSql struct, the “Insert()” function became transparent to transaction. In “insert” function, It first gets database handler from UserDataSql, then it creates the prepared statement and executes it; at last it retrieves the inserted id and returns it to the calling function.

{{< highlight go "linenos=table,linenostart=1" >}}
// UserDataSql is the SQL implementation of UserDataInterface
type UserDataSql struct {
	DB gdbc.SqlGdbc
}

func (uds *UserDataSql) Insert(user *model.User) (*model.User, error) {

	stmt, err := uds.DB.Prepare(INSERT_USER)
	if err != nil {
		return nil, errors.Wrap(err, "")
	}
	defer stmt.Close()
	res, err := stmt.Exec(user.Name, user.Department, user.Created)
	if err != nil {
		return nil, errors.Wrap(err, "")
	}
	id, err := res.LastInsertId()
	if err != nil {
		return nil, errors.Wrap(err, "")
	}
	user.Id = int(id)
	logger.Log.Debug("user inserted:", user)
	return user, nil
}
{{< / highlight >}}
<br/>

If you need to support different databases, you will have one separate implementation for each of them. I will explain it in detail in another article “[Transaction Support](https://jfeng45.github.io/en/posts/transaction_support/)”³.

## **Model:**

Model is the only layer that doesn’t have interfaces. In Clean Architecture, it is called “entity”. It is where I deviated from Clean Architecture. The model layer in this application doesn’t have much business logic, it only defines the data. Most business logic is in “use case” layer. From my experience, because of lazy loading or other reasons, when executing a use case, most of the time the data in a domain model is not loaded or at least not fully loaded, so the “use case” needs to call data service to load data from database. Since a domain model can’t call a data service, it has to be the “use case” layer, which makes it the perfect place to put business logic in.

## Validation:
{{< highlight go "linenos=table,linenostart=1" >}}
import (
	"github.com/go-ozzo/ozzo-validation"
	"time"
)

// User has a name, department and created date. Name and created are required, department is optional.
// Id is auto-generated by database after the user is persisted.
// json is for couchdb
type User struct {
	Id         int       `json:"uid"`
	Name       string    `json:"username"`
	Department string    `json:"department"`
	Created    time.Time `json:"created"`
}

// Validate validates a newly created user, which has not persisted to database yet, so Id is empty
func (u User) Validate() error {
	return validation.ValidateStruct(&u,
		validation.Field(&u.Name, validation.Required),
		validation.Field(&u.Created, validation.Required))
}

//ValidatePersisted validate a user that has been persisted to database, basically Id is not empty
func (u User) ValidatePersisted() error {
	return validation.ValidateStruct(&u,
		validation.Field(&u.Id, validation.Required),
		validation.Field(&u.Name, validation.Required),
		validation.Field(&u.Created, validation.Required))
}
{{< / highlight >}}
<br/>


The above is the code for domain model “User”. I also have simple validation in it. It is natural to put the validation logic in the model layer, which should be the lowest layer in the application because every other layer depends on it. Validation rules usually only involve low level actions, so it shouldn’t cause any problem. The validation library used in this application is “[ozzo-validation](https://github.com/go-ozzo/ozzo-validation)”⁴. It is interface-based, which makes it less intrusive to the code, please see “[Input validation in GoLang](https://medium.com/@apzuk3/input-validation-in-golang-bc24cdec1835)”⁵ for comparisons of different validation libraries. One concern is that “ozzo” depends on “database/sql” package because of SQL validation support, which messed up the dependency. In the future, if there is a problem, we may need to switch to a different library or remove the “sql” dependency in the library.

You may ask why you put the validation in the domain model layer but business logic in the “use case” layer? Because the business logic usually involves multiple domain models or multiple instances of one model. For example, the calculation of a product price depends on the purchase quantity and whether the item is on sale or not, so it has to be in “use case” layer. Validation on the other hand, is usually rely on one instance of a model, so it can be put in the model. If a validation spreads multiple models or multiple instances of a model ( for example, checking a user is duplicate), than put it in “use case” layer.

## Data Transfer objects

This is another part that I didn’t follow Clean Architecture. According to [Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)¹, “ Typically the data that crosses the boundaries is simple data structures. You can use basic structs or simple Data Transfer objects if you like.” DTO (Data Transfer Object) is not used in this application, instead the domain model is shared among different layers. If the business logic is very complex, there may be some benefit to have a separate DTO, at which time I don’t mind to create them, but not now.

### Format translation

When across a service boundary, I do see the need to have different domain models. For example, this application is also published as a gRPC Microservice. On the server side, we use the application domain model; on the client side, we use gRPC model, and they have different types, so format translation is necessary between them.

{{< highlight go "linenos=table,linenostart=1" >}}
// GrpcToUser converts from grpc User type to domain Model user type
func GrpcToUser(user *uspb.User) (*model.User, error) {
	if user == nil {
		return nil, nil
	}
	resultUser := model.User{}

	resultUser.Id = int(user.Id)
	resultUser.Name = user.Name
	resultUser.Department = user.Department
	created, err := ptypes.Timestamp(user.Created)
	if err != nil {
		return nil, errors.Wrap(err, "")
	}
	resultUser.Created = created
	return &resultUser, nil
}

// UserToGrpc converts from domain Model User type to grpc user type
func UserToGrpc(user *model.User) (*uspb.User, error) {
	if user == nil {
		return nil, nil
	}
	resultUser := uspb.User{}
	resultUser.Id = int32(user.Id)
	resultUser.Name = user.Name
	resultUser.Department = user.Department
	created, err := ptypes.TimestampProto(user.Created)
	if err != nil {
		return nil, errors.Wrap(err, "")
	}
	resultUser.Created = created
	return &resultUser, nil
}

// UserListToGrpc converts from array of domain Model User type to array of grpc user type
func UserListToGrpc(ul []model.User) ([]*uspb.User, error) {
	var gul []*uspb.User
	for _, user := range ul {
		gu, err := UserToGrpc(&user)
		if err != nil {
			return nil, errors.Wrap(err, "")
		}
		gul = append(gul, gu)
	}
	return gul, nil
}
	
{{< / highlight >}}
<br/>


The above data translation code is in “adapter/userclient” package. At first glance, it seems to sense to make the domain model “User” having a method “toGrpc()” and the validation method will be executed like this-“user.toGrpc(user *uspb.User)”, but this will create a dependency from the domain model to gRPC. So, it is better to create a separate function and put it under “adapter/userclient” package. This package will depend on both domain model and gRPC model. However, because of that, both domain model and gRPC model are clean and they don’t depend on each other.

## Conclusion:

The design of the application followed Clean Architecture. There are three layers in business logic code: “use case”, “domain model” and “data service”. However, I deviated from Clean Architecture on two parts. One is that I have most business logic code in “use case” layer; the other is that I don’t have DTO, instead I use domain models to share data among different layers.

## Source Code:

The complete code is in [github](https://github.com/jfeng45/servicetmpl): https://github.com/jfeng45/servicetmpl

## Other articles:

Please read the rest of the articles in this series in “[Go Microservice with Clean Architecture](https://jfeng45.github.io/en/posts/clean_architecture_with_go/)”.

## Reference:

[1][The Clean Code Blog]
(https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)

[2][Clean Architecture in Go]
(https://medium.com/@hatajoe/clean-architecture-in-go-4030f11ec1b1)

[3] [Go Microservice with Clean Architecture: Transaction Support]
(https://jfeng45.github.io/en/posts/transaction_support/)

[4][ozzo-validation]
(https://github.com/go-ozzo/ozzo-validation)

[5] [Input validation in GoLang]
(https://medium.com/@apzuk3/input-validation-in-golang-bc24cdec1835)
