# Go 编译命令

- [Go 编译命令](#go-编译命令)
  - [编译子命令](#编译子命令)
    - [go run](#go-run)
    - [go build](#go-build)
    - [go install](#go-install)
  - [编译参数](#编译参数)
  - [交叉编译](#交叉编译)
    - [常用参数](#常用参数)
    - [进行交叉编译](#进行交叉编译)
    - [编译缓存](#编译缓存)
    - [缩小编译文件大小](#缩小编译文件大小)
    - [编译信息写入](#编译信息写入)

## 编译子命令

### go run

编译并马上运行 Go 程序，它可接收一个或多个文件参数。但只接收 main 包下的文件作为参数，如果不是 main 包下的文件，则会出错。

在执行 go run 命令后，所编译的二进制文件最终存放在一个临时目录里。可以通过 -n 或 -x 参数进行查看。这两个参数的作用是打印编译过程中的所有执行命令，-n 参数不会继续执行编译后的二进制文件，而 -x 参数会继续执行编译后的二进制文件。

编译器执行了绝大部分编译相关的工作，过程如下：

![](../imgs/go-complie-flow.png)

- 创建编译依赖所需的临时目录。Go 编译器会设置一个临时环境变量 WORK，用于在此工作区编译应用程序，执行编译后的二进制文件，其默认值为系统的临时文件目录路径。可以通过设置 GOTMPDIR 来调整其执行目录。
- 编译和生成编译所需要的依赖。该阶段将会编译和生成标准库中的依赖（如 flag.a、log.a、net/http等）、应用程序中的外部依赖（如 gin-gonic/gin 等），以及应用程序自身的代码，然后生成、链接对应归档文件（.a 文件）和编译配置文件。
- 创建并进入编译二进制文件所需的临时目录。即创建 exe 目录。
- 生成可执行文件。这里主要用到的是 link 工具，该工具读取依赖文件的 Go 归档文件或对象及其依赖项，最终将它们组合为可执行的二进制文件。

![](../imgs/go-complie.png)

- 执行可执行文件。到先前指定的目录 $WORK/b001/exe/main 下执行生成的二进制文件。

在执行 go run 命令后，除非设置了-work 参数，否则会在应用程序结束时自动删除该目录下的相关临时文件。

### go build

编译指定的源文件、软件包及其依赖项，但它不会运行编译后的二进制文件。在该目录下会生成了一个与当前目录名一致的可执行的二进制文件。

从本质上讲，go build 命令和 go run 命令的编译执行过程相差不多，唯一不同的是，go build 命令会生成并执行编译好的二进制文件，将其重命名为当前目录名，并立刻删除编译时生成的临时目录。而在归档文件（.a文件）上，go build 命令和 go run 命令的执行结果是一样的，都是对所需的源码文件进行编译。

### go install

是编译并安装源文件、软件包。“安装”看起来有些迷惑人，但实际上 go install 与 go build、go run 命令在功能上相差不多，最大的区别是 go install 命令会将编译后的相关文件（如可执行的二进制文件、归档文件等）安装到指定的目录中。

go install 命令在编译后，会将生成的二进制文件移到 bin 目录下，其文件名称为 go modules 的项目名，而非目录名。

当设置了环境变量 `$GOBIN` 时，会将生成的二进制文件移到`$GOBIN`下。另外，如果禁用了 go modules，那么将安装到 `$GOPATH/pkg/$GOOS_$GOARCH` 下。

## 编译参数

![](../imgs/go-complie-args.png)

## 交叉编译

交叉编译指通过编译器在某个系统下编译另外一个系统的可执行二进制文件，即目标计算架构的标识与当前运行环境的目标计算架构的标识不同，或者是所构建环境的目标操作系统的标识与当前运行环境的目标操作系统的标识不同。

### 常用参数

![](../imgs/go-complie-cross-args.png)

### 进行交叉编译

```
set GOOS=linux
set CGO_ENABLED=0
go build -a -o cross .
```

### 编译缓存

在 Go 语言中，编译实际上是存在缓存机制的，它一般存储在特定的目录下：

```
go env GOCACHE
```

编译缓存是有一定意义的，并且目前还支持增量编译。这时又引申出一个新的问题，那就是如何界定是否有变更，编译器是否需要重新变更，是否根据文件的时间戳来进行对比呢？

实际上，在 Go 语言早期曾使用过时间戳的方式，但这是有问题的，因为文件的修改时间变更了，并不代表它的文件内容与先前的内容不同，即有可能是改了某个东西，然后又改回去了，因此基于时间戳是不正确的。

### 缩小编译文件大小

在默认情况下，gc 工具链中的链接器创建静态链接的二进制文件。因此，所有 Go 二进制文件都包含 Go  运行时的信息、支持动态类型检查，以及在异常抛出时堆栈跟踪所必需的运行时的类型信息（文件名/行号）。

在 Linux 上，使用 gcc 静态编译并静态链接的一个简单的 C 语言编写的 hello world 程序约为 750KB，而一个等效的 Go 程序使用的 fmt.Printf 的大小为几 MB，但是它包括更强大的运行时支持，以及类型和调试信息，因此两者实际上并不完全等效。

最简单的方法是去掉 DWARF 调试信息和符号表信息，执行命令如下：

```
go build -ldflags="-w -s"
```

![](../imgs/ldflags.png)

upx 工具对可执行文件进行压缩：

```
upx xxx.exe
```

### 编译信息写入

联合使用 go build 命令和 -ldflags 命令，即可在构建时将动态信息设置到二进制文件中。

```go
var version string

func main() {
    fmt.Printf("version: %s\n", version)
}
```

执行以下编译命令：

```
go build -ldflags "-X main.version=1.0.0"
./xxx
```