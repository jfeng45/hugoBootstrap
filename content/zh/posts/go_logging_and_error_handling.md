---
title: "清晰架构（Clean Architecture）的Go微服务: 日志管理"
shortTitle: "日志管理和错误处理"
date: 2019-07-25
draft: false
tags: ["Golang", "日志管理", "错误处理", “微服务”]
categories: ["Go微服务"]

# canonicalUrl: "https://medium.com/@jfeng45/go-microservice-with-clean-architecture-application-logging-b43dc5839bce"
description: “描述了Go Microservice 和gRPC项目中日志管理和错误处理的最佳实践，并比较了两种不同的
             记录库ZAP和Logrus。“

---

<!--more-->
良好的日志记录可以提供丰富的日志数据，便于在调试时发现问题，从而大大提高编码效率。 记录器提供的自动化信息越多越好，日志信息也需要以简洁的方式呈现，便于找到重要的数据。
##### **日志需求:**

1. 无需修改业务代码即可切换到其他日志库

1. 不需直接依赖任何日志库

1. 整个应用程序只有一个日志库的全局实例，因此您可以在一个位置更改日志配置并将其应用于整个程序。

1. 可以在不修改代码的情况下轻松更改日志记录选项，例如，日志级别

1. 能够在程序运行时动态更改日志级别

##### **资源句柄：为什么日志记录与数据库不同**
<br/>
   
当应用程序需要处理外部资源时，例如数据库，文件系统，网络连接， SMTP服务器时，它通常需要一个资源句柄（Resource Handler）。在依赖注入中，容器创建一个资源句柄并将其注入每个业务函数，因此它可以使用资源句柄来访问底层资源。在此应用程序中，资源句柄是一个接口，因此业务层不会直接依赖于资源句柄的任何具体实现。数据库和gRPC链接都以这种方式处理。
   
但是，日志记录器稍有不同，因为几乎每个函数都需要它，但数据库不是。在Java中，我们为每个Java类初始化一个记录器（Logger）实例。 Java日志记录框架使用层次关系来管理不同的记录器，因此它们从父日志记录器继承相同的日志配置。在Go中，不同的记录器之间没有层次关系，因此您要么创建一个记录器，要么具有许多彼此不相关的不同记录器。为了获得一致的日志记录配置，最好创建一个全局记录器并将其注入每个函数。但者将需要做很多工作，所以我决定在一个中心位置创建一个全局记录器，每个函数可以直接引用它。
   
为了不将应用程序紧密绑定到特定的记录器，我创建了一个通用的记录器接口，因此应用程序对于具体的记录器透明的。以下是记录器（Logger）接口。
   
{{< highlight go "linenos=table,linenostart=1" >}}
// Log is a package level variable, every program should access logging function through "Log"
var Log Logger

// Logger represent common interface for logging function
type Logger interface {
	Errorf(format string, args ...interface{})
	Fatalf(format string, args ...interface{})
	Fatal(args ...interface{})
	Infof(format string, args ...interface{})
	Info( args ...interface{})
	Warnf(format string, args ...interface{})
	Debugf(format string, args ...interface{})
	Debug(args ...interface{})
}
{{< / highlight >}}

因为每个文件都依赖于日志记录，很容易产生循环依赖，所以我在“容器”包里面创建了一个单独的子包“logger”来避免这个问题。 它只有一个“Log”变量和“Logger”接口。 每个文件都通过这个变量和接口访问日志功能。

##### **记录器封装**：
<br/>

支持一个日志库的标准方法（例如[ZAP](https://github.com/uber-go/zap)¹或[Logrus](https://github.com/sirupsen/logrus)²) 是创建一个封装来实现已经创建的记录器接口。 这很简单，以下是代码。

{{< highlight go "linenos=table,linenostart=1" >}}
type loggerWrapper struct {
	lw *zap.SugaredLogger
}
func (logger *loggerWrapper) Errorf(format string, args ...interface{}) {
	logger.lw.Errorf(format, args)
}
func (logger *loggerWrapper) Fatalf(format string, args ...interface{}) {
	logger.lw.Fatalf(format, args)
}
func (logger *loggerWrapper) Fatal(args ...interface{}) {
	logger.lw.Fatal(args)
}
func (logger *loggerWrapper) Infof(format string, args ...interface{}) {
	logger.lw.Infof(format, args)
}
func (logger *loggerWrapper) Warnf(format string, args ...interface{}) {
	logger.lw.Warnf(format, args)
}
func (logger *loggerWrapper) Debugf(format string, args ...interface{}) {
	logger.lw.Debugf(format, args)
}
func (logger *loggerWrapper) Printf(format string, args ...interface{}) {
	logger.lw.Infof(format, args)
}
func (logger *loggerWrapper) Println(args ...interface{}) {
	logger.lw.Info(args, "\n")
}
{{< / highlight >}}
<br/>

但是日志记录存在一个问题。日志记录的一个功能是在日志消息中打印记录者名字。在对接口封装之后，方法的调用者不是打印日志的程序，而是封装程序。要解决该问题，您可以直接更改日志库的源代码，但在升级日志库时会导致兼容性问题。最终的解决方案是要求日志记录库创建一个新功能，该功能可以根据方法是否使用封装来返回合适的调用方。

为了让代码现在能正常工作，我走了捷径。因为ZAP和Logrus之间的大多数函数签名是相似的，所以我提取了常用的签名并创建了一个共享接口，因为两个日志库都已经有了这些函数，它们自动实现这些接口。 Go接口设计的优点在于，您可以先创建具体实现，然后再创建接口，如果函数签名相互匹配，则自动实现接口。这有点作弊，但非常有效。如果要用的记录器不支持公共的接口，则还是要对它进行封装， 这样就只能暂时先牺牲调用者功能或修改源代码。

##### **日志库比较:**
<br/>

不同的日志库提供不同的功能，其中一些功能对于调试很重要。

需要记录的重要信息（需要以下数据）：

1. 文件名和行号

1. 方法名称和调用文件名

1. 消息记录级别

1. 时间戳

1. 错误堆栈跟踪

1. 自动记录每个函数调用包括参数和结果

我希望日志库自动提供这些数据，例如调用方法名称，而不编写显式代码来实现。对于上述6个功能，目前没有日志库提供＃6，但它们都提供1到5个中的部分或全部。我尝试了两个非常流行的日志库Logrus和ZAP。 Logrus提供了所有功能，但是我的控制台上的格式不正确（它在我的Windows控制台上显示“\ n \ t”而不是新行）并且输出格式不像ZAP那样干净。 ZAP不提供＃2，但其他一切看起来都不错，所以我决定暂时使用它。

令人惊讶的是，本程序被证明是一个非常好的工具来测试不同的日志库，因为您可以切换到不同的日志库来比较输出结果，而只需要更改配置文件中的一行。这不是本程序的功能，而是一个好的副作用。

实际上，我最需要的功能是自动记录每个函数调用包括参数和结果（#6），但是还没有日志库提供该功能提供。我希望将来能够得到它。

##### **错误（error）处理:**
<br/>
错误处理与日志记录直接相关，所以我也在这里讨论一下。以下是我在处理错误时遵循的规则。

1.使用堆栈跟踪创建错误<br/>
    错误消息本身需要包含堆栈跟踪信息。如果错误源自您的程序，您可以导入“github.com/pkg/errors”库来创建错误以包含堆栈跟踪。但是如果它是从另一个库生成的并且该库没有使用“pkg/errors”，您需要用“errors.Wrap（err，message）”语句包装该错误，以获取堆栈跟踪信息。由于我们无法控制第三方库，因此最好的解决方案是在我们的程序中对所有错误进行包装。详情请见[这里](https://dave.cheney.net/2016/06/12/stack-traces-and-the-errors-package)³。

2.使用堆栈跟踪打印错误<br/>
您需要使用“logger.Log.Errorf（”％+v\n“，err）”或“fmt.Printf（”％+v\n“，err）”以便打印堆栈跟踪信息，关键是“+v”选项（当然你必须已经使用＃1）。

3.只有顶级函数才能处理错误<br/>
“处理”表示记录错误并将错误返回给调用者。因为只有顶级函数处理错误，所以错误只在程序中记录一次。顶层的调用者通常是面向用户的程序，它是用户界面程序（UI）或另一个微服务。您希望记录错误消息（因此您的程序中具有记录），然后将消息返回到UI或其他微服务，以便他们可以重试或对错误执行某些操作。

4.所有其他级别函数应只是将错误传播到较高级别<br/>
底层或中间层函数不要记录或处理错误，也不要丢弃错误。您可以向错误中添加更多数据，然后传播它。当出现错误时，您不希望停止整个应用程序。

##### **恐慌（Panic）:**
<br/>

除了在本地的“main.go”之外，我从未使用过恐慌（Panic）。它更像是一个bug而不是一个功能。在[让我们谈谈日志](https://dave.cheney.net/2015/11/05/lets-talk-about-logging)⁴中，Dave Cheney写道“人们普遍认为应用库不应该使用恐慌”。另一个错误是log.Fatal，它具有与恐慌相同的效果，也应该被禁止。 “log.Fatal”更糟糕，它看起来像一个日志，但是在输出日志后它“恐慌”，这违反了单一责任规则。

恐慌有两个问题。首先，它与错误的处理方式不同，但它实际上是一个错误，一个错误的子类型。现在，错误处理代码需要处理错误和恐慌，例如[事务处理代码](https://stackoverflow.com/questions/16184238/database-sql-tx-detecting-commit-or-rollback/23502629#23502629)⁵中的错误处理代码。其次，它会停止应用程序，这非常糟糕。只有顶级主控制程序才能决定如何处理错误，所有其他被调用的函数应该只将错误传播到上层。特别是现在，服务网格层（Service Mesh）可以提供重试等功能，恐慌使其更加复杂。

如果你正在调用第三方库并且它在代码中产生恐慌，那么为了防止代码停止，你需要截获恐慌并从中恢复。以下是代码示例，您需要为每个可能发生恐慌的顶级函数执行此操作（在每个函数中放置“defer catchPanic（）”）。在下面的代码中，我们有一个函数“catchPanic”来捕获并从恐慌中恢复。函数“RegisterUser”在代码的第一行调用“defer catchPanic（）”。有关恐慌的详细讨论，请参阅[此处](https://eli.thegreenplace.net/2018/on-the-uses-and-misuses-of-panics-in-go/)⁶。

{{< highlight go "linenos=table,linenostart=1" >}}
func catchPanic() {
	if p := recover(); p != nil {
		logger.Log.Errorf("%+v\n", p)
	}
}

func (uss *UserService) RegisterUser(ctx context.Context, req *uspb.RegisterUserReq)
    (*uspb.RegisterUserResp, error) {
	
 	defer catchPanic()
	ruci, err := getRegistrationUseCase(uss.container)
	if err != nil {
		logger.Log.Errorf("%+v\n", err)
		return nil, errors.Wrap(err, "")
	}
	mu, err := userclient.GrpcToUser(req.User)
...
}
{{< / highlight >}}
<br/>

##### **结论：**
   
良好的日志记录可以使程序员更有效。您希望使用堆栈跟踪记录错误。 只有顶级函数才能处理错误，所有其他级别函数只应将错误传播到上一级。 不要使用恐慌。

##### **源程序:**

完整的源程序链接 [github](https://github.com/jfeng45/servicetmpl): https://github.com/jfeng45/servicetmpl

##### **其他文章:**

请在这里阅读本系列的其他文章 “[清晰架构（Clean Architecture）的Go微服务](https://jfeng45.github.io/posts/clean_architecture_with_go/)”.

##### **索引:**

[1] [zap]
(https://github.com/uber-go/zap)

[2] [Logrus]
(https://github.com/sirupsen/logrus)

[3][Stack traces and the errors package]
(https://dave.cheney.net/2016/06/12/stack-traces-and-the-errors-package)

[4][Let’s talk about logging]
(https://dave.cheney.net/2015/11/05/lets-talk-about-logging)

[5][database/sql Tx — detecting Commit or Rollback]
(https://stackoverflow.com/questions/16184238/database-sql-tx-detecting-commit-or-rollback/23502629#23502629)

[6][On the uses and misuses of panics in Go]
(https://eli.thegreenplace.net/2018/on-the-uses-and-misuses-of-panics-in-go/)
