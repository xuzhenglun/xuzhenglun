---
layout: title
title: Golang黑魔法(1)：使用其他包中未导出的函数
date: 2019-02-19 00:03:31
tags:
- Golang
- dark magic
- pragmas
categories: Golang
---

Go是一门语法以`Less is More`为哲学的语言，以简单易学为设计目标，致力于写出比Java还千人一面的符合工程要求的语言。GC、没有泛型、简单到没有的错误处理等等，都使得我这种不会管理内存，智商跟不上泛型，只要Try就Catch不到的小学生能够每天拼接一下字符串，勉强过得了生活的样子。但是，不信任用户的同义词就是只信任自己：“不让你用是怕你搞砸，至于我？我怎么可能搞砸？”。为了方便那些Go Author，他们早就给自己开好了后门。但是这些技巧最大的作用就是茶余饭后的装逼资本了，毕竟在项目里用了很可能就被买腿了。但是话说回来，黑魔法一般作为禁招也能够帮你绝处逢生，~~离开你地球就停转~~，从而保住饭碗。

<!-- more -->

# Pragmas （编译指示）
Go作为一门静态语言，和别的语言一样需要经过编译，链接。而编译指示就是在这个过程中，额外输入给编译器或者连接器的信息。利用这些指示，可以绕开或者做到一些特殊的功能。这些功能被只言片语的从[官方文档](https://golang.org/cmd/compile/#hdr-Compiler_Directives)中带过了，或者根本就[没写](https://dave.cheney.net/2018/01/08/gos-hidden-pragmas)。无论写没写，可以看得出来的是：官方的确不太想让普通用户都知道这些。

# Go:linkname

> 截止 Go 1.23 ，这个方法已经不 work 了，之前已经使用了的知名项目也都被挂到了 Go 源码的耻辱柱上，包括 Containerd、gVisor、tidb 等。
> 
> xRef:
> 1. https://tip.golang.org/doc/go1.23#linker
> 2. https://github.com/golang/go/issues/67401

回到主题，本文主要打算介绍一个比较安全的黑魔法：Linkname。利用这个Pragmas，可以使得编译器仅仅保留一个符号，并在链接的过程中将这个符号链接到目标上去。废话不再多说，直接上例子：
```Go
package main

import (
	_ "unsafe"
	"fmt"
)

func main() {
	fmt.Println(isSpace(' '))
}

//go:linkname isSpace fmt.isSpace
func isSpace(r rune) bool
```

在这个例子里，可以很明显的发现13行的`isSpace`这个函数只有签名没有函数体，取而代之的是上面有一行注释。代码13行的注释使用了`go:linkname`这个Pragmas，而他的意思也不复杂：将isSpace这个函数链接至`fmt`包中的未导出函数`isSpace`。但是，由于这种操作其实并不推荐，因为他会破坏包的封装，所以需要导入`unsafe`包来敲打你一下。

现在万事俱备，直接运行`go build`来编译一下验证一下结果：返回`./main.go:14:6: missing function body`错误。其实这是`go build`命令行工具没有支持这样的操作，为了使用黑魔法必须手动逐步编译和链接。示例命令如下：
```Bash
$ go tool compile main.go
$ go tool link -o main main.o
$ ./main
> true
```

值得一提的是，如果使用的是CGO或者需要链接至别的二进制库，需要在`go tool link`后使用`-L`参数指定二进制位置。


