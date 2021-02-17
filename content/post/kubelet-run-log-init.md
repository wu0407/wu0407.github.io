---
title: "kubelet启动--日志初始化"
date: 2021-02-17T21:14:20+08:00
draft: false
description: "kubelet启动的时候进行日志klog相关的初始化"
keywords:  [ "kubernetes", "kubelet", "klog","源码解析"]
tags:  ["kubernetes", "源码解析", "kubelet"]
categories:  ["kubernetes"]
---

kubernetes 默认使用的日志库是klog，kubernetes的`k8s.io/component-base/logs`库，专门做日志初始化相关操作。

klog是glog的fork版本，由于glog不在开发、在容器中运行有一系列问题、不容易测试等问题，所以kubenetes自己维护了一个klog。<!--more--> 

在1.18版本中使用v1.0.0版本的klog，这个版本的`--add_dir_header`不生效，在https://github.com/kubernetes/klog/pull/101修复了，但是在1.18中一直存在这个问题，小版本更新一直没有更新klog版本。

## 入口文件

kubelet在main函数中直接调用`k8s.io/component-base/logs`的InitLogs()初始化日志，并在程序退出时候执行flushlog。

cmd\kubelet\kubelet.go

```go
import (
	"math/rand"
	"os"
	"time"

	"k8s.io/component-base/logs"
	_ "k8s.io/component-base/metrics/prometheus/restclient"
	_ "k8s.io/component-base/metrics/prometheus/version" // for version metric registration
	"k8s.io/kubernetes/cmd/kubelet/app"
)

func main() {
	rand.Seed(time.Now().UnixNano())

	command := app.NewKubeletCommand()
	logs.InitLogs()
	defer logs.FlushLogs()

	if err := command.Execute(); err != nil {
		os.Exit(1)
	}
}
```

## 模块初始化

### 加载`k8s.io/component-base/logs`模块

这个模块还会加载klog模块，而klog模块也有init()。

全局变量logFlushFreq--注册命令行选项`--log-flush-frequency`，init()--初始化klog。

```go
const logFlushFreqFlagName = "log-flush-frequency"

var logFlushFreq = pflag.Duration(logFlushFreqFlagName, 5*time.Second, "Maximum number of seconds between log flushes")

func init() {
	klog.InitFlags(flag.CommandLine)
}
```



### 加载klog模块

设置命令行选项的默认值和goroutine来周期性刷写日志。

```go
// init sets up the defaults and runs flushDaemon.
func init() {
	logging.stderrThreshold = errorLog // Default stderrThreshold is ERROR.
	logging.setVState(0, nil, false)
	logging.logDir = ""
	logging.logFile = ""
	logging.logFileMaxSizeMB = 1800
	logging.toStderr = true
	logging.alsoToStderr = false
	logging.skipHeaders = false
	logging.addDirHeader = false
	logging.skipLogHeaders = false
	go logging.flushDaemon()
}
```

flushDaemon()--每隔5秒执行lockAndFlushAll()

```go
const flushInterval = 5 * time.Second

// flushDaemon periodically flushes the log file buffers.
func (l *loggingT) flushDaemon() {
	for range time.NewTicker(flushInterval).C {
		l.lockAndFlushAll()
	}
}

// lockAndFlushAll is like flushAll but locks l.mu first.
func (l *loggingT) lockAndFlushAll() {
	l.mu.Lock()
	l.flushAll()
	l.mu.Unlock()
}

// flushAll flushes all the logs and attempts to "sync" their data to disk.
// l.mu is held.
func (l *loggingT) flushAll() {
	// Flush from fatal down, in case there's trouble flushing.
	for s := fatalLog; s >= infoLog; s-- {
		file := l.file[s]
		if file != nil {
			file.Flush() // ignore error
			file.Sync()  // ignore error
		}
	}
}
```

### 执行klog.InitFlag

注册klog命令行选项

```go
// InitFlags is for explicitly initializing the flags.
func InitFlags(flagset *flag.FlagSet) {
	if flagset == nil {
		flagset = flag.CommandLine
	}

	flagset.StringVar(&logging.logDir, "log_dir", logging.logDir, "If non-empty, write log files in this directory")
	flagset.StringVar(&logging.logFile, "log_file", logging.logFile, "If non-empty, use this log file")
	flagset.Uint64Var(&logging.logFileMaxSizeMB, "log_file_max_size", logging.logFileMaxSizeMB,
		"Defines the maximum size a log file can grow to. Unit is megabytes. "+
			"If the value is 0, the maximum file size is unlimited.")
	flagset.BoolVar(&logging.toStderr, "logtostderr", logging.toStderr, "log to standard error instead of files")
	flagset.BoolVar(&logging.alsoToStderr, "alsologtostderr", logging.alsoToStderr, "log to standard error as well as files")
	flagset.Var(&logging.verbosity, "v", "number for the log level verbosity")
	flagset.BoolVar(&logging.skipHeaders, "add_dir_header", logging.addDirHeader, "If true, adds the file directory to the header")
	flagset.BoolVar(&logging.skipHeaders, "skip_headers", logging.skipHeaders, "If true, avoid header prefixes in the log messages")
	flagset.BoolVar(&logging.skipLogHeaders, "skip_log_headers", logging.skipLogHeaders, "If true, avoid headers when opening log files")
	flagset.Var(&logging.stderrThreshold, "stderrthreshold", "logs at or above this threshold go to stderr")
	flagset.Var(&logging.vmodule, "vmodule", "comma-separated list of pattern=N settings for file-filtered logging")
	flagset.Var(&logging.traceLocation, "log_backtrace_at", "when logging hits line file:N, emit a stack trace")
}
```



## 执行component-base/logs的InitLogs

兼容系统log库--如果使用系统的log库来写日志，最终还是使用klog来记录。然后设置log库将原始的日志传给klog，不做任何处理。

比如使用log.Print来记录日志，它最终会调用KlogWriter的Write方法来记录。

为什么要兼容系统log库记录？

可能是历史原因一些使用外部库的日志使用的是系统的log库。

```go
// Write implements the io.Writer interface.
func (writer KlogWriter) Write(data []byte) (n int, err error) {
	klog.InfoDepth(1, string(data))
	return len(data), nil
}

// InitLogs initializes logs the way we want for kubernetes.
func InitLogs() {
	// 兼容使用log来记录，实际还是使用klog
	log.SetOutput(KlogWriter{})
	// 设置log输出格式为空，日志格式由klog决定
	log.SetFlags(0)
	// The default glog flush interval is 5 seconds.
	go wait.Forever(klog.Flush, *logFlushFreq)
}
```

klog.Flush()--执行日志的刷写，这里与上面的klog init起的goroutine执行一样的方法lockAndFlushAll()，kubelet起了两个goroutine来刷写日志。疑惑？

```go
// Flush flushes all pending log I/O.
func Flush() {
	logging.lockAndFlushAll()
}
```



## 再次注册klog命令行选项

执行到addKlogFlags(fs)时候，再次执行klog.InitFlag。

**为什么再次执行klog.InitFlag?**

仔细会发现再次执行时候的参数是一个新建的flagset--`flag.NewFlagSet(os.Args[0], flag.ExitOnError)`(不是`flag.CommandLine`)，也就是说在这个flagset中重复注册命令行选项，然后保存到kubelet的pflagset中，而`flag.CommandLine`的放者不用，这个极大浪费。

从commit提交信息发现，addKlogFlags是klog升级到0.3.0版本添加的--[相关pull request](https://github.com/kubernetes/kubernetes/pull/76474)，我猜测是klog升级到1.0.0版本，但是kubelet代码没有做相应变化。

cmd\kubelet\app\options\globalflags.go

```go
// AddGlobalFlags explicitly registers flags that libraries (glog, verflag, etc.) register
// against the global flagsets from "flag" and "github.com/spf13/pflag".
// We do this in order to prevent unwanted flags from leaking into the Kubelet's flagset.
func AddGlobalFlags(fs *pflag.FlagSet) {
	addKlogFlags(fs)
	addCadvisorFlags(fs)
	addCredentialProviderFlags(fs)
	verflag.AddFlags(fs)
	logs.AddFlags(fs)
}

// addKlogFlags adds flags from k8s.io/klog
func addKlogFlags(fs *pflag.FlagSet) {
	local := flag.NewFlagSet(os.Args[0], flag.ExitOnError)
	klog.InitFlags(local)
	fs.AddGoFlagSet(local)
}
```



## 程序退出flushlog

程序退出flush日志，保证日志不会丢失。

```go
// FlushLogs flushes logs immediately.
func FlushLogs() {
	klog.Flush()
}
```

## 总结

感觉日志这块一直在演进，难免会存在历史账，但是最终是work的😂。在kubernetes 1.20版本中， klog升级到2.4.0版本，更好的支持结构化的日志。

这里没有分析klog源码，后面有时间再分析klog。kubernetes真的太大了，能不能在它衰退前看完，时间真不够。