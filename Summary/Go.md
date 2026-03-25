## 如何开始一个微服务开发

### 安装Golang

https://go.dev/

#### 自定义GOPATH、GOPROXY

GOPATH：go相关依赖存放位置，默认在用户文件夹下
GOPROXY：下载源的配置

### protoc安装

下载解压并配置环境变量

https://github.com/protocolbuffers/protobuf/releases

```shell
protoc --version
```

### Kratos

https://go-kratos.dev/docs/getting-started/start/

一个微服务开发的脚手架

#### 安装

```shell
go install github.com/go-kratos/kratos/cmd/kratos/v2@latest
```

#### 代码生成相关依赖

```shell
kratos upgrade
```

#### 创建一个项目

```shell
kratos new helloword
```
