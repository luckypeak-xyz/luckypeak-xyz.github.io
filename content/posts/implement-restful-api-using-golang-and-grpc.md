---
author:
  name: ruifeng
  link: https://blog.luckypeak.xyz
  email: me@luckypeak.xyz
  avatar: /safari-pinned-tab.svg
title: "使用 Golang 和 gRPC 提供 RESTful API"
subtitle: ""
date: 2023-09-24
lastmod: 2023-09-24
categories: []
tags: [Go, gRPC]
draft: false
---

使用 Golang 和 GRPC 对外提供一个 API，访问 `http://ip:port?name=yourname` 返回 `{"message": "hello yourname"}`。

<!--more-->

新建 Golang 项目：

```shell
mkdir go-grpc-hello
cd go-grpc-hello
go mod init go-grpc-hello
```

新建 `proto` 文件夹存放 proto 文件，新建 `pb` 文件夹存放生成的与 proto 文件对应的 Go 代码。

新建 `proto/rpc_hello.proto` 文件，在文件中定义 `HelloRequest` 和 `HelloResponse` 消息：

```proto
syntax = "proto3";

package pb;

option go_package = "go-grpc-hello/pb";

message HelloRequest {
  string name = 1;
}

message HelloResponse {
  string message = 1;
}
```

新建 `proto/service_greater.proto` 文件，在文件中定义 `Greater` 服务：

```proto
syntax = "proto3";

package pb;

option go_package = "go-grpc-hello/pb";

import "rpc_hello.proto";

service Greeter {
  rpc SayHello (HelloRequest) returns (HelloResponse) {}
}
```

新建 `Makefile` 文件，定义常用命令：

```Makefile
proto:
	rm -f pb/*.go
	protoc --proto_path=proto \
	--go_out=pb --go_opt=paths=source_relative \
    --go-grpc_out=pb --go-grpc_opt=paths=source_relative \
    proto/*.proto

.PHONY: proto
```

执行 `make proto` 生成 proto 对应的 Go 代码。生成的代码中存在报错，执行 `go mod tidy` 下载所需的包，报错消失。

接下来，实现 `Greater` 服务中定义的 `SayHello` RPC。创建 `gapi` 目录来存放实现 RPC 接口的 Go 代码。

创建 `gapi/server.go`， 定义 `server` 类型：

```go
package gapi

import "go-grpc-hello/pb"

type Server struct {
	pb.UnimplementedGreeterServer
}

func NewServer() *Server {
	return &Server{}
}
```

创建 `gapi/rpc_hello.go`，在 `*Server` 上定义 `SayHello` 方法，以实现 `GreeterServer` 接口：

```go
package gapi

import (
	"context"
	"go-grpc-hello/pb"
	// "log"
)

func (server *Server) SayHello(ctx context.Context, req *pb.HelloRequest) (*pb.HelloResponse, error) {
	// log.Printf("received: %v", req.GetName())
	rsp := &pb.HelloResponse{
		Message: "Hello " + req.Name,
	}
	return rsp, nil
}
```

新建 `server` 文件夹存放 GRPC server 相关代码。创建 `server/mian.go` 实现 GRPC server：

```go mark=23,10
package main

import (
	"go-grpc-hello/gapi"
	"go-grpc-hello/pb"
	"log"
	"net"

	"google.golang.org/grpc"
	"google.golang.org/grpc/reflection" // 使用 evans 作为客户端访问 GRPC 服务
)

func main() {
	println("start grpc server")

	runGrpcServer()
}

func runGrpcServer() {
	server := gapi.NewServer()
	grpcServer := grpc.NewServer()
	pb.RegisterGreeterServer(grpcServer, server)
	reflection.Register(grpcServer) // 使用 evans 作为客户端访问 GRPC 服务

	listener, err := net.Listen("tcp", "0.0.0.0:50051")
	if err != nil {
		log.Fatal("cannot create listener: ", err)
	}
	err = grpcServer.Serve(listener)
	if err != nil {
		log.Fatal("cannot start grpc server: ", err)
	}
}
```

启动 RPC 服务端：

```shell
❯ go run server/main.go
start grpc server
```

接下来通过 `evans` 访问 `SayHello` RPC：

```shell
❯ evans -r --host localhost -p 50051

pb.Greeter@localhost:50051> show service
+---------+----------+--------------+---------------+
| SERVICE |   RPC    | REQUEST TYPE | RESPONSE TYPE |
+---------+----------+--------------+---------------+
| Greeter | SayHello | HelloRequest | HelloResponse |
+---------+----------+--------------+---------------+

pb.Greeter@localhost:50051> service Greeter

pb.Greeter@localhost:50051> call SayHello
name (TYPE_STRING) => world
{
  "message": "Hello world"
}

pb.Greeter@localhost:50051> exit
Good Bye :)
```

现在我们已经有一个可以通过 GRPC 客户端（这里的 evans） 访问的 GRPC 服务，如何将该服务以 RESTful API 的形式开放呢？
可以借助 [grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway) 来实现。

我们这里使用 `protoc`，所以根据 README 文件所述，对于 grpc-gateway 我们要想 generate stubs 需要手动添加依赖：

```
google/api/annotations.proto
google/api/field_behavior.proto
google/api/http.proto
google/api/httpbody.proto
```

对于 protoc-gen-openapiv2 则需要手动添加 `protoc-gen-openapiv2/options` 目录。

在 proto 目录下添加上述文件，修改 `service_greater.proto` 以提供 RESTful API：

```proto
syntax = "proto3";

package pb;

option go_package = "go-grpc-hello/pb";

import "rpc_hello.proto";
import "google/api/annotations.proto";
import "protoc-gen-openapiv2/options/annotations.proto";

option (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_swagger) = {
  info: {
    title: "Go grpc hello";
    version: "1.2";
    contact: {
      name: "luckypeak";
      url: "https://github.com/yrf105";
    };
  };
};


service Greeter {
  rpc SayHello (HelloRequest) returns (HelloResponse) {
    option (google.api.http) = {
      get: "/v1/greeter"
    };
    option (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_operation) = {
      description: "Use this api to say hello";
      summary: "Say hello";
    };
  };
}
```

修改 Makefile 中的 proto 命令：

```Makefile
proto:
	rm -f pb/*.go
	protoc --proto_path=proto \
	  --go_out=pb --go_opt=paths=source_relative \
    --go-grpc_out=pb --go-grpc_opt=paths=source_relative \
		--grpc-gateway_out=pb --grpc-gateway_opt=paths=source_relative \
		--openapiv2_out=doc/swagger --openapiv2_opt=allow_merge=true,merge_file_name=hello \
    proto/*.proto
```

创建 `doc/swagger` 存放生成的 `hello.swagger.json` 文件（可以将文件粘贴在 [swagger](https://swagger.io/) 查看）。执行 `go mod tidy` 安装缺失的包。

在 `server/main.go` 中实现 `runGatewayServer`：

```go
func main() {
	go runGatewayServer()
	runGrpcServer()
}

func runGrpcServer() {
	// ...
}

func runGatewayServer() {
	server := gapi.NewServer()

	jsonOpt := runtime.WithMarshalerOption(runtime.MIMEWildcard, &runtime.JSONPb{
		MarshalOptions: protojson.MarshalOptions{
			UseProtoNames: true,
		},
		UnmarshalOptions: protojson.UnmarshalOptions{
			DiscardUnknown: true,
		},
	})
	grpcMux := runtime.NewServeMux(jsonOpt)
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()
	err := pb.RegisterGreeterHandlerServer(ctx, grpcMux, server)
	if err != nil {
		log.Fatal("cannot start grpc gateway server: ", err)
	}

	mux := http.NewServeMux()
	mux.Handle("/", grpcMux)

	listener, err := net.Listen("tcp", "0.0.0.0:50052")
	if err != nil {
		log.Fatal("cannot create listener: ", err)
	}
	fmt.Println("start http server")
	err = http.Serve(listener, mux)
	if err != nil {
		log.Fatal("cannot start http server: ", err)
	}
}
```

在浏览器中输入 localhost:50052/v1/greeter?name=world 返回 `{"message":"Hello world"}`。

除了在 [swagger](https://swagger.io/) 查看中查看生成的 `hello.swagger.json` 文件。我们还可以自己服务。

将 https://github.com/swagger-api/swagger-ui/tree/master/dist 下的所有代码拷贝到 `doc/swagger` 中。创建 `doc/embed.go`（这也是使用 [go:embed 的最佳实践](https://github.com/golang/go/issues/41191#issuecomment-686621090)）：

```go
package doc

import "embed"

//go:embed swagger
var Swagger embed.FS
```

首先修改 `doc/swagger/swagger-initializer.js` 中 `url` 为 `url: "hello.swagger.json"`，然后修改 `runGatewayServer` 以服务静态文件：

```go focus=22:23
import (
	// ...
	"go-grpc-hello/doc"
    // ...
)

func main() {
	go runGatewayServer()
	runGrpcServer()
}

func runGrpcServer() {
	// ...
}

func runGatewayServer() {
	// ...

	mux := http.NewServeMux()
	mux.Handle("/", grpcMux)

	fs := http.FileServer(http.FS(doc.Swagger))
	mux.Handle("/swagger/", http.StripPrefix("/", fs))

	listener, err := net.Listen("tcp", "0.0.0.0:50052")
	if err != nil {
		log.Fatal("cannot create listener: ", err)
	}
	// ...
}
```

输入 `localhost/swagger` 访问 swagger ui。
