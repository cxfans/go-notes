# CGO 编程

最初 CGO 是为了达到方便从 Go 语言函数调用 C 语言函数（用 C 语言实现 Go 语言声明的函数）以复用 C 语言资源这一目的而出现的（因为 C 语言还会涉及回调函数，自然也会涉及到从 C 语言函数调用 Go 语言函数（用 Go 语言实现 C 语言声明的函数））。现在，它已经演变为 C 语言和 Go 语言双向通讯的桥梁。

- [类型转换](type/README.md)