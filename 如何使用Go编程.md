# 如何使用Go编程

[TOC]

## 介绍

- 这份文档假设当前使用的 Go 版本是1.13或更新， 且GO111MODULE环境变量未设置

## 代码的组织

- Go程序会被组织成包，一个包是由同一个目录下的所有源文件编译而成。被写进一个源文件的函数、类型、变量、常数对于同一个包内的所有源文件都是可见的
- 一个仓库包含一个或多个模块，一个模块是所有相关的Go包
的集合。一个Go仓库通常包含仅一个模块，位于仓库的根目录
- `go.mod`文件声明模块的路径：即所有模块内的包导入的路径前缀。一个包含go.mod文件的模块包含的多个包都是以子目录的形式划分
- 每个模块的路径不仅是其包的import路径前缀，还表明应该从哪里下载该模块。比如：`golang.org/x/tools`表明应该从 `https://golang.org/x/tools` 下载该模块
- 一个包的导入路径是模块路径加上包所在的子目录路径。比如：`github.com/google/go-cmp` 包含一个子目录为`cmp`的包，那该包的导入路径是 `github.com/google/go-cmp/cmp`，标准库中的包不包含路径前缀

## 你的第一个程序

- To compile and run a simple program, first choose a module path (we'll use github.com/KESNERO/hello) and create a go.mod file that declares it:

```bash
$ mkdir hello # Alternatively, clone it if it already exists in version control.
$ cd hello
$ go mod init github.com/KESNERO/hello
go: creating new go.mod: module github.com/KESNERO/hello
$ cat go.mod
module github.com/KESNERO/hello

go 1.14
```

- The first statement in a Go source file must be package name. Executable commands must always use package main.

- Next, create a file named hello.go inside that directory containing the following Go code:

```go
package main

import "fmt"

func main() {
	fmt.Println("Hello, world.")
}
```

- 现在可以使用go工具来编译和安装这个程序

```bash
$ go install github.com/KESNERO/hello
```

- This command builds the hello command, producing an executable binary. It then installs that binary as $GOPATH/bin/hello 

- 安装目录是被 `GOPATH`和`GOBIN`两个环境变量所控制，如果存在`GOBIN`，那么二进制文件会安装到 `GOBIN`指定的目录下，如果`GOPATH`存在，那么二进制文件会安装到 `$GOPATH/bin`目录下

- 可以使用 `go env` 命令设置这些环境变量的默认值

```bash
$ go env -w GOBIN=/somewhere/else/bin
```

- 可以使用 -u 取消环境变量

```bash
go env -u GOBIN
```

### 从模块导入包

- 我们来写一个 `morestrings` 包，并且在hello程序里面引用合格包
- 首先创建一个创建包路径`$GOPATH/src/github.com/KESNERO/hello/morestrings`，创建一个名为`reverse.go`的文件

```go
// Package morestrings implements additional functions to manipulate UTF-8
// encoded strings, beyond what is provided in the standard "strings" package.
package morestrings

// ReverseRunes returns its argument string reversed rune-wise left to right.
func ReverseRunes(s string) string {
	r := []rune(s)
	for i, j := 0, len(r)-1; i < len(r)/2; i, j = i+1, j-1 {
		r[i], r[j] = r[j], r[i]
	}
	return string(r)
}
```

- 因为 ReverseRunes 函数使用大写字母开头命名，所以会被输出到文件外部，所以可以在其他包中引用`morestrings`包来使用此函数
- 我们使用`go build`命令来编译该包

```go
$ go build github.com/KESNERO/hello/morestrings
```

- 该命令不会输出文件，取而代之会在本地编译缓存中保存编译后的包
- 在确认`morestrings`包编译成功后，我们在`hello`程序中引用它，修改`hello`程序如下

```go
package main

import (
	"fmt"

	"github.com/KESNERO/hello/morestrings"
)

func main() {
	fmt.Println(morestrings.ReverseRunes("Hello, world."))
}
```

- 重新安装 hello 程序

```bash
$ go install github.com/KESNERO/hello
$ hello
.dlrow ,olleH
```

### 从远端模块中安装包

- 因为我们使用了规范的导入路径，所以go工具可以使用这个特性自动地从远端仓库中获取这些包，举个例子，可以在hello程序中直接使用 `github.com/google/go-cmp/cmp`来导入包

```go
package main

import (
	"fmt"

	"github.com/KESNERO/hello/morestrings"
	"github.com/google/go-cmp/cmp"
)

func main() {
	fmt.Println(morestrings.ReverseRunes("!oG ,olleH"))
	fmt.Println(cmp.Diff("Hello World", "Hello Go"))
}
```

- 当你使用`go install, go build, go run`这些命令的时候会自动地从远端模块下载这些包， 包的版本会在go.mod文件中记录

```bash
$ go install
go: finding module for package github.com/google/go-cmp/cmp
go: downloading github.com/google/go-cmp v0.5.0
go: found github.com/google/go-cmp/cmp in github.com/google/go-cmp v0.5.0
$ hello
Hello, Go!
  string(
- 	"Hello World",
+ 	"Hello Go",
  )
$ cat go.mod
module github.com/KESNERO/hello

go 1.14

require github.com/google/go-cmp v0.5.0
$
```

- 模块的依赖会自动地下载到位于`$GOPATH`子目录下的`pkg/mod`当中。下载下来的内容可以被其他所有模块共享，所以下载下来的内容都是设置为只读。可以使用`go clean -modcache`命令清理下载下来的模块

## 测试

- Go 具有轻量级的测试模块包含在`go test`命令和`testing`包内

- 你可以创建一个`_test.go`后缀的文件包含`TestXXX`函数，该函数接收者为`(t *testing.T)`，该测试框架运行每个这样的函数，如果该函数调用了`t.Error`,`t.Fail`之类的函数的话表明该函数的测试失败

- 添加一个对`morestrings`包的测试， 创建 `$GOPATH/github.com/KESNERO/hello/morestrings/reverse_test.go`

```go
package morestrings

import "testing"

func TestReverseRunes(t *testing.T) {
	cases := []struct {
		in, want string
	}{
		{"Hello, world", "dlrow ,olleH"},
		{"Hello, 世界", "界世 ,olleH"},
		{"", ""},
	}
	for _, c := range cases {
		got := ReverseRunes(c.in)
		if got != c.want {
			t.Errorf("ReverseRunes(%q) == %q, want %q", c.in, got, c.want)
		}
	}
}
```

```bash
$ cd $GOPATH/src/github.com/KESNERO/hello/morestrings/
$ go test
PASS
ok  	github.com/KESNERO/hello/morestrings	0.401s
```