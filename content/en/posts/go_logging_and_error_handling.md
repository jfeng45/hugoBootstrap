---
title: "Go Microservice with Clean Architecture: Application Logging"
shortTitle: "Logging and Error Handling"
date: 2019-07-25
draft: false
tags: ["Golang", "Logging", "Error Handling", "Microservice"]
categories: ["Go Microservice"]

canonicalUrl: "https://medium.com/@jfeng45/go-microservice-with-clean-architecture-application-logging-b43dc5839bce"
description: "Described Best practice for logging and error handling in Go Microservice gRPC project, and compared two different 
logging libraries ZAP and Logrus."

---

<!--more-->
Good logging can provide informative log data and make it easy to find problems when doing debugging, thus greatly improve coding efficiency. The more automated information provided by the logger the better it is, also the log information needs to be presented in a concise way, so it is easy to find the important data.

## **Logging Requirement:**

1. Be able to switch to other log library without modifying the business code

1. No direct dependency on any log library

1. Need one global instance of the logger for the whole application, so you can change the log configuration in one place and apply it to the whole app.

1. Can change logging behavior easily without modifying code, for example, log level

1. Be able to change log level dynamically when the programming is running

## Resource handler: Why logging is different from database

When an application needs to deal with an outside resource, such as database, file system, network connection or SMTP server, it usually needs a handler. In dependency Injection, the container creates a handler and inject it into each business function, so it can use the handler to access the underlying resource. In this application, the handler is an interface, so the business layer won’t have a direct dependency on any concrete implementation of a handler. Databases and gRPC connections are all handled this way.

However, logging is a little different, because almost every function needs it but database is not. In Java, we initialize an instance of a logger for each Java class. Java logging framework uses a hierarchical relationship to manage different loggers, so they inherit the same log configuration from a parent log. In Go, it doesn’t work that way and there is no hierarchical relationship among different loggers, so you either create one logger or have many different loggers which are not related to each other. In order to have consistent logging configuration, it is better to create one logger and use it for every function. We can create a global logger and pass it around, but it is a lot of work to pass a logger to every function, so I decide to create a global logger in a central place and have it available to every function.

In order to not tight the application to a particular logger, I created a generic logger interface, so the application is transparent to the underline logging library. The following is the “Logger” interface.
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

Because ever file depends on logging, it is easy to have circular dependency, so I created a separate sub-package “logger” inside “container” package to avoid the problem. It only has a “Log” variable and “Logger” interface. Ever file accesses logging function through the variable and the interface.

## Logger Wrapper:

The standard way to support a log library (For example, [ZAP](https://github.com/uber-go/zap)¹ or [Logrus](https://github.com/sirupsen/logrus)²) is to create a wrapper to implement the logger interface we already created. It is pretty straight forward to do, and the following is the code.

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
However, there is one problem for logging. One feature of logging is to print caller name in the log message. After you write a wrapper, the caller of a method is not the program that print the log, but the wrapper program. To fix that problem, you can directly change the source code of the log library, but it causes a compatibility issue when the log lib is upgraded. The ultimate solution is to ask logging libraries to create a new feature that can return the caller of a method based on whether it uses a wrapper or not.

In order to get the code work now, I took a short cut. Because most of the function signatures between ZAP and Logrus are similar, I extracted the common ones and created a shared interface, since both log libraries already have those functions, they automatically implemented those interfaces. The beauty of Go’s interface is that you can create the concrete implementation first and later on, creating the interface, if the function signature matches each other, then the interface is automatically implemented. It is a bit cheating, but works for now. In case the logger doesn’t support the common interface, you may have to implement it.

## **Comparison of log library:**

Different log libraries provide different features, some are important for debugging.

Logging information that is important ( the following data are required):

1. File name and line number

1. Method name and caller name

1. Message logging level

1. Timestamp

1. Error stack trace

1. Automatically logging each function call with parameters and results

I want the logging lib provides those data automatically, for example, the calling method name, without writing explicit code asking for them. For the above 6 features, currently no library is providing#6, but they all provides 1 to 5 more or less. I tried two popular ones Logrus and ZAP. Logrus provides all features, but I the format on my console is not right (It displays “\n\t” instead of a new line on my Windows console) and the layout is not as clean as ZAP. ZAP doesn’t provide #2, but everything else looks good, so I decided to use it for now.

A surprise is that the application turned out to be a pretty good tool to test different log libraries because you can switch to different ones and comparing results by changing one line in the configuration file. That is not the intention of the project, but a side-effect, a good one though.

Actually, the feature that I needed most is automatically logging function calls with input and output value(#6), but no lib provides that yet. I hope to get it in the future.

## **Error Handing:**

Error handling is directly related to logging, so I also discuss it here. The followings are the rules I followed to handle errors.

1. Create error with stack trace<br/>
An error message needs to include the stack trace information in itself. If the error originates in your application, you can import “github.com/pkg/errors” library to create the error to have the stack trace included. But if it is generated from another library and that lib didn’t use “pkg/errors”, then you need to wrap that error with “errors.Wrap(err, message)” statement in order to obtain stack trace information. Since we don’t have control on third party libraries, the best solution is to wrap all errors in our application. Please find details [here](https://dave.cheney.net/2016/06/12/stack-traces-and-the-errors-package)³.

2. Log error with stack trace<br/>
You need to use “logger.Log.Errorf(**“%+v\n”**, err)” or “fmt.Printf(**“%+v\n”**, err)” in order to print the stack trace info, the key is the “+v” option ( I assume you already followed #1).

3. Only the top level functions handle the error<br/>
“Handle” means log the error and return the error to the caller. Because only the top level function handles the error, so it is only logged once in the application. The caller at the top level is usually a user facing program, which is either a user interface program (UI) or another Microservice. You want to log the error message (So your application have the record) and then return the message to the UI or another Microservice, so the they can retry or do something with the error.

4. All other level functions should only propagate an error to upper level<br/>
Don’t log or handle errors in lower level and don’t discard errors either. You may add more data to the error, and then propagate it. You don’t want to stop the whole application when there is an error.

## Panic:

I never used panic except in local “main.go”. It is more like a bug than a feature. In “[Let’s talk about logging](https://dave.cheney.net/2015/11/05/lets-talk-about-logging)”⁴, Dave Cheney wrote “It is commonly accepted that libraries should not use panic”. Another bug is log.Fatal, which has the same effect with panic, should also be forbidden. “log.Fatal” is even worse, it looks like a log, but then it “panic” after logging, which violates the single responsibility rule.

There are two problems with panic. First, it is treated differently from error, but it is actually an error, a sub-type of an error. Now, the error handling code needs to deal both error and panic, such as the one in the [transaction handling code](https://stackoverflow.com/questions/16184238/database-sql-tx-detecting-commit-or-rollback/23502629#23502629)⁵. Second, it stops the application, which is very bad. Only the top-level main control program should make the decision on how to handle an error, all other called functions should just propagate the error to the upper level. Especially nowadays, service mesh layer can provide retry and other functions, panic makes it more complicated.

If you are calling another library and it generates panic in its code, then to protect your code from stopping, you need to catch the panic and recover from it. The following is the example code and you need to do that for every top level function that could panic ( put “defer catchPanic()” in every one of them). In the following code, we have a function “catchPanic” to catch and recover from a panic. The function “RegisterUser” calls “defer catchPanic()” in the first line of the code. Please see [here](https://eli.thegreenplace.net/2018/on-the-uses-and-misuses-of-panics-in-go/)⁶ for more detailed discussion on panic.

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
## Conclusion:

Good logging can make a programmer more effective. You want to log error with stack trace. Only the top level functions should handle the error and all other level functions should only propagate an error to upper level. Don’t use panic.

## Source Code:

The complete code is in [github](https://github.com/jfeng45/servicetmpl): https://github.com/jfeng45/servicetmpl

## Other articles:

Please read the rest of the articles in this series in “[Go Microservice with Clean Architecture](https://jfeng45.github.io/posts/clean_architecture_with_go/)”.

## References:

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
