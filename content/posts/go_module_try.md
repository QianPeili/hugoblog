---
title: "Go Module 小试"
date: 2018-09-01T20:37:42+08:00
---
# go module 小试

go 1.11 前段日子发布了，其中一个新特性就是 `go module` 的支持，go module 可以让我们的项目目录不需要强制在 GOPATH 下。



### 开启 go module

开启 go module 特性主要依赖于一个环境变量： `GO111MODULE`，它有三个值：

- off：不开启 go module 支持，go 命令所有依赖都会从 vendor 和 GOPATH 目录查找。
- on：开启 go module 支持，go 命令不会去 GOPATH 目录查找依赖，这个时候命令处于 `module-aware mode`
- auto(the default) or unset：这种情况下，如果当前目录不在 `${GOPATH}/src` 目录下，并且当前目录或者父级目录包含 go.mod 文件就表示开启 go module.



在 module-aware 模式，GOPATH 不再控制项目的依赖导入，但是依旧会存储下载的依赖源代码（`${GOPATH}/pkg/mod` 目录）以及安装的命令文件（`${GOPATH}/bin`目录）。

###  

### go mod 命令

go mod

```
download    download modules to local cache
edit        edit go.mod from tools or scripts
graph       print module requirement graph
init        initialize new module in current directory
tidy        add missing and remove unused modules
vendor      make vendored copy of dependencies
verify      verify dependencies have expected content
why         explain why packages or modules are needed
```



### go module 基本使用

**初始化一个模块**

`go mod init gomod`

如果是一个新模块，当前目录会生成一个 go.mod 文件，里面的内容是：module gomod。

如果模块以及存在其他依赖管理工具，比如：godep, dep, glide... ，这么这个命令会根据已有配置生产 require 条目。

比如，下面是一份 dep 项目里的 Gopkg.lock 文件内容：

```toml
[[projects]]
  name = "github.com/gin-contrib/sse"
  revision = "22d885f9ecc78bf4ee5d72b937e4bbcdc58e8cae"
  ...

[[projects]]
  name = "github.com/gin-gonic/gin"
  version = "v1.3.0"
  ...

[[projects]]
  name = "github.com/golang/protobuf"
  version = "v1.2.0"
  ...

[[projects]]
  name = "github.com/json-iterator/go"
  version = "v1.1.5"
  ...

[[projects]]
  name = "github.com/mattn/go-isatty"
  version = "v0.0.4"

[[projects]]
  name = "github.com/modern-go/concurrent"
  version = "1.0.3"
  ...

[[projects]]
  name = "github.com/modern-go/reflect2"
  version = "1.0.1"
  ...

[[projects]]
  name = "github.com/ugorji/go"
  version = "v1.1.1"
  ...

[[projects]]
  name = "golang.org/x/sys"
  revision = "fa5fdf94c78965f1aa8423f0cc50b8b8d728b05a"
  ...

[[projects]]
  name = "gopkg.in/go-playground/validator.v8"
  version = "v8.18.2"
  ...

[[projects]]
  name = "gopkg.in/yaml.v2"
  version = "v2.2.1"
  ...

...
```



go mod init 时会提示：

```shell
go: creating new go.mod: module gomod
go: copying requirements from Gopkg.lock
```

然后生成的 go.mod 文件内容：

```mod
module gomod

require (
	github.com/gin-contrib/sse v0.0.0-20170109093832-22d885f9ecc7
	github.com/gin-gonic/gin v1.3.0
	github.com/golang/protobuf v1.2.0
	github.com/json-iterator/go v1.1.5
	github.com/mattn/go-isatty v0.0.4
	github.com/modern-go/concurrent v0.0.0-20180306012644-bacd9c7ef1dd
	github.com/modern-go/reflect2 v0.0.0-20180701023420-4b7aa43c6742
	github.com/ugorji/go v1.1.1
	golang.org/x/sys v0.0.0-20180831094639-fa5fdf94c789
	gopkg.in/go-playground/validator.v8 v8.18.2
	gopkg.in/yaml.v2 v2.2.1
)
```

**添加依赖**

- 在 module-aware 模式，如果使用 go get 下载了新的依赖， go.mod 会自动添加对应的依赖条目。
- 使用 go mod edit -require path@version 往 go.mod 中添加指定依赖，然后通过 go mod verify 或者 go mod download 下载依赖

**删除依赖**

- 通过 go mod edit -droprequire 删除依赖
- 在代码中取消对应的依赖，然后通过 go mod tidy 自动删除没有使用的依赖

**其他**

go mod vendor 命令可以将依赖拷贝到模块的 vendor 目录，方便离线使用。



### 参考

[go - hdr-Module_maintenance](https://golang.org/cmd/go/#hdr-Module_maintenance)

[go - hdr-Module_queries](https://golang.org/cmd/go/#hdr-Module_queries)


