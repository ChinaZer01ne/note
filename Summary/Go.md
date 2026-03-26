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

# Create a template project
kratos new helloword

cd helloword

# Add a proto template
kratos proto add api/server/server.proto

# Generate the proto code
kratos proto client api/server/server.proto

# Generate the source code of service by proto file
kratos proto server api/server/server.proto -t internal/service

go generate ./...

go build -o ./bin/ ./...

./bin/server -conf ./configs
```

> 注意查看项目中`go.mod`中`go`的版本，安装高版本的`go`会有兼容性问题
#### 加载依赖

```shell
go mod tidy
```


### 接入consul

#### 安装consul

https://developer.hashicorp.com/consul/install

#### 启动consul

```shell
consul agent -dev
```

#### 在consul中增加配置

  - 打开 http://localhost:8500/ui
  - 进入 KV
  - 新建 key：`kratos/helloworld/config.yaml`
  - 值填入你当前 `configs/config.yaml` 的完整内容

通过以下命令查看配置值：
```shell
consul kv get kratos/helloworld/config.yaml
```
#### 增加配置

```go
message Bootstrap {  
  Server server = 1;  
  Data data = 2;  
  Consul consul = 3;  
}

// 增加consul  
message Consul {  
  string scheme = 1;  
  string address = 2;  
  string config_path = 3;  
  string service_name = 4;  
  string service_version = 5;  
}
```

#### 安装依赖

```shell
go get github.com/hashicorp/consul/api
go get github.com/go-kratos/kratos/contrib/config/consul/v2
go get github.com/go-kratos/kratos/contrib/registry/consul/v2
```

安装完依赖需要执行
```shell
go mod tidy
```

#### 接入

`main.go`中修改
```go
// 接入consul  
client, err := consulAPI.NewClient(&consulAPI.Config{  
    Address: "127.0.0.1:8500",  
    Scheme:  "http",  
})  
if err != nil {  
    panic(err)  
}  
cs, err := consulConfig.New(  
    client,  
    consulConfig.WithPath("kratos/helloworld/config.yaml"),  
)  
if err != nil {  
    panic(err)  
}  
c := config.New(  
    config.WithSource(cs),  
)
```