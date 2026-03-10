## 背景&目标

在使用 `grpc` 时，需要将 `protobuf` 文件构建成对应语言的插桩代码（例如使用 `grpc-go` 需要先将 `protobuf` 构建为 `go` 代码）。但这类插桩代码不属于`源代码`，其实际为 `protobuf` 的构建产物。如果把这类代码放在代码仓库中则会增加维护成本，并且还需要保证插桩代码和 `protobuf` 的一致性。另外，插桩代码的生成依赖本地环境和本地命令，不同的开发者可能会将 `protobuf` 和生成的代码放置在不同的地方，带来更多的维护成本。

综上，使用 bazel 构建 `grpc` 项目可以避免显示生成插桩代码等中间产物，使业务逻辑代码可以**直接**与 `protobuf` 文件进行编译，大大降低开发和构建成本。

> 实际是 bazel 先将 `protobuf` 文件构建为对应语言代码，在使用该产物与业务逻辑代码进行编译。详情请参阅 [`bazel` 官网](https://bazel.build/)。

## 前期准备

在开始前，首先要有一个安装了 `bazel` 的开发环境（作者只在 `linux` 系统上使用过 `bazel`），安装方式可以参考 [`bazel` 官网](https://bazel.build/)。本文以 `grpc-go` 为例，所以还需要准备 `golang` 相关的 `bazel rules`。

* [rules_go](https://github.com/bazel-contrib/rules_go)：提供 `golang` 以及 `grpc-go` 相关 `rules`。
* [gazelle](https://github.com/bazel-contrib/bazel-gazelle)：用于自动生成 `BUILD.bazel` 文件。
* [grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway)：一个开源项目，可以依据 `protobuf` 文件，为 `grpc` 服务生成 `http` 网关代码。

> 随时很多网关组件都原生支持 `grpc` 协议，但 `http` 还是太好用了。

PS：`grpc` 和 `protobuf` 相关的 `bazel rules` 有很多，我这里选择的是 `rules_go` 自带的编译器。

## Hands on

### 搞一个 `proto` 定义

不管是哪个语言的 `grpc` 项目，服务总是由 `proto` 文件定义。所以我们先弄一个最简单的 "hello world" 服务。

```protobuf
syntax = "proto3";

package experimental.grpc_hello_world;

import "google/api/annotations.proto";
import "google/api/field_behavior.proto";
import "google/api/resource.proto";
import "google/api/client.proto";

option go_package = "dominion/experimental/grpc_hello_world;grpc_hello_world";

// ...
service Greeter {
  // ...
  rpc GetHello(HelloRequest) returns (Hello) {
    option (google.api.http) = {
      get: "/v1/{name=world/*}"
    };

    option (google.api.method_signature) = "name";
  }
}

message HelloRequest {
  string name = 1 [
    (google.api.field_behavior) = REQUIRED,
    (google.api.resource_reference) = {
      type: "hello.liukexin.com/Hello"
    }];
}

message Hello {
  option (google.api.resource) = {
    type: "hello.liukexin.com/Hello"
    pattern: "world/{hello}"
  };
  string name = 1; 

  string message = 2;
}
```

怎么这个 "hello world" 看的有点复杂？因为想要通过工具自动化生成 `grpc-gateway` 代码，需要满足 `google apis` 规范，使用**注解**标记 `grpc` 接口与参数，告知如何将其转换成 `http` 接口。例如以下代码：

``` protobuf
option (google.api.http) = {
    get: "/v1/{name=world/*}"
};
```

包含下列含义：

* 将该方法作为 `http GET` 方法。
* `http path` 为 `/v1/world/*`，并将 `path` 中的 `world/*` 通配参数映射到 `grpc` 接口请求参数体中 `name` 字段上。

> 更多关于 `google apis` 的信息，请参考[官网](https://google.aip.dev/general) 与 [github](https://github.com/googleapis/googleapis)。

### 生成 `BUILD.bazel` 

有了 `proto` 文件以后，可以运行 `gazelle` 自动创建 `BUILD.bazel` 文件。有了这个文件，`bazel` 就可以为我们生成对应的 `golang` 代码。`BUILD.bazel` 看上去像下面这样:

```starlark
load("@rules_go//go:def.bzl", "go_library")
load("@rules_go//proto:def.bzl", "go_proto_library")
load("@rules_proto//proto:defs.bzl", "proto_library")

proto_library(
    name = "grpc_hello_world_proto",
    srcs = ["hello_world.proto"],
    visibility = ["//visibility:public"],
    deps = [
        "@googleapis//google/api:annotations_proto",
        "@googleapis//google/api:client_proto",
        "@googleapis//google/api:field_behavior_proto",
        "@googleapis//google/api:resource_proto",
    ],
)

go_proto_library(
    name = "grpc_hello_world_go_proto",
    compilers = [
        "@rules_go//proto:go_grpc_v2",
        "@rules_go//proto:go_proto",
        "@grpc_ecosystem_grpc_gateway//protoc-gen-grpc-gateway:go_gen_grpc_gateway",
    ],
    importpath = "dominion/experimental/grpc_hello_world",
    proto = ":grpc_hello_world_proto",
    visibility = ["//visibility:public"],
    deps = [
        "@googleapis//google/api:annotations_go_proto",
        "@googleapis//google/api:client_go_proto",
        "@googleapis//google/api:field_behavior_go_proto",
        "@googleapis//google/api:resource_go_proto",
    ],
)

go_library(
    name = "grpc_hello_world",
    embed = [":grpc_hello_world_go_proto"],
    importpath = "dominion/experimental/grpc_hello_world",
    visibility = ["//visibility:public"],
)
```

其主要包含三部分：

1. `proto_library` 负责编译 `proto` 文件。
2. `go_proto_library` 根据 `proto` 文件，使用 **`compilers`** 生成 `golang` 代码。
3. `go_library` 将生成的 `golang` 构建成静态库供其他包使用。其中 `importpath` 参数为 `golang package` 的引用路径。

除此以为，如果你跟作者一样，使用的 `gazelle` 存在一些问题的话，需要手动指定 `compilers` 和依赖。

```starlark
# gazelle:go_grpc_compilers @rules_go//proto:go_grpc_v2, @rules_go//proto:go_proto, @grpc_ecosystem_grpc_gateway//protoc-gen-grpc-gateway:go_gen_grpc_gateway

# gazelle:resolve proto google/api/annotations.proto @googleapis//google/api:annotations_proto
# gazelle:resolve proto google/api/http.proto @googleapis//google/api:http_proto
# gazelle:resolve proto google/api/field_behavior.proto @googleapis//google/api:field_behavior_proto
# gazelle:resolve proto google/api/resource.proto @googleapis//google/api:resource_proto
# gazelle:resolve proto google/api/client.proto @googleapis//google/api:client_proto

# gazelle:resolve proto go google/api/annotations.proto @googleapis//google/api:annotations_go_proto
# gazelle:resolve proto go google/api/http.proto @googleapis//google/api:http_go_proto
# gazelle:resolve proto go google/api/field_behavior.proto @googleapis//google/api:field_behavior_go_proto
# gazelle:resolve proto go google/api/resource.proto @googleapis//google/api:resource_go_proto
# gazelle:resolve proto go google/api/client.proto @googleapis//google/api:client_go_proto

gazelle(
    name = "gazelle",
    extra_args = [
        "-index=lazy",
    ],
)
```

* `gazelle:go_grpc_compilers @rules_go//proto:go_grpc_v2, @rules_go//proto:go_proto, @grpc_ecosystem_grpc_gateway//protoc-gen-grpc-gateway:go_gen_grpc_gateway` 指定 `go_proto_library` 使用下面三个编译器，前两个负责 `grpc-go` 构建，后一个负责 `grpc-gateway`。
    * `@rules_go//proto:go_grpc_v2`
    * `@rules_go//proto:go_proto`
    * `@grpc_ecosystem_grpc_gateway//protoc-gen-grpc-gateway:go_gen_grpc_gateway` 

* `gazelle:resolve proto google/api/annotations.proto @googleapis//google/api:annotations_proto` 负责告诉 `gazelle`：`proto` 文件中的 `google/api/annotations.proto` 依赖来自 `@googleapis//google/api:annotations_proto` 这个地方。

* `gazelle:resolve proto go google/api/annotations.proto @googleapis//google/api:annotations_go_proto` 与上一条类似，告诉 `gazelle` 在 `go_proto_library` 中使用的依赖来自 ` @googleapis//google/api:annotations_go_proto`。

### 服务逻辑代码

有了插桩代码，就可编写服务逻辑和入口代码了。例如下面的 `grpc` 服务代码：

``` golang
package main

import (
	"context"
	"flag"
	"log"
	"net"

	"dominion/experimental/grpc_hello_world"

	"google.golang.org/grpc"
	"google.golang.org/grpc/reflection"
)

var port = flag.String("port", "50051", "Port to listen on")

type greeterServer struct {
	grpc_hello_world.UnimplementedGreeterServer
}

func (s *greeterServer) GetHello(ctx context.Context, req *grpc_hello_world.HelloRequest) (*grpc_hello_world.Hello, error) {
	name := req.GetName()
	if name == "" {
		name = "world"
	}

	return &grpc_hello_world.Hello{Name: name, Message: "Hello, " + name + "!"}, nil
}

func main() {
	flag.Parse()

	listener, err := net.Listen("tcp", ":"+*port)
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}

	grpcServer := grpc.NewServer()

	grpc_hello_world.RegisterGreeterServer(grpcServer, &greeterServer{})
	reflection.Register(grpcServer)

	log.Printf("gRPC hello world server listening: %s", *port)
	if err := grpcServer.Serve(listener); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}
```

注意 `imports` 中的 `"dominion/experimental/grpc_hello_world"` 就是我们通过 `bazel` 生成的 `grpc` 插桩代码，这部分代码是不在我们的代码仓库中的，编译时会由 `bazel` 完成依赖引入。代码仓库中只需要一个 `proto` 文件和一个 `main.go` 即可，那些什么 `pb.go`、`_grpc.pb.go` 等文件通通不需要。 同样，`client` 也只需要引用 `dominion/experimental/grpc_hello_world` 即可，也无需再生成额外代码。并且，`bazel` 支持跨仓库源码依赖，即使 `client` 代码不在同一个仓库，也可以依赖本仓库的 `bazel target`（不过作者还没有研究过 `bzlmod` 模式下怎么用，只在 `WORKSPACE` 模式下试过）。

> 插个题外话，如果 `grpc` 的代码不在代码仓库中，代码提示要怎么做呢？`vscode` 可以通过设置 `GOPACKAGESDRIVER` 参数实现使用 `BUILD.bazel` 文件中定义的依赖进行代码提示。更新 `BUILD.bazel` 文件后按 `F1` 执行 `Go: Restart Language Server` 即可。配置可参考...，文档作者找不到了，直接问 gpt 吧。

`grpc-gateway` 入口代码类似，直接将生成的 `grpc-gateway` 代码绑定到端口上即可：

```golang
package main

import (
	"context"
	"flag"
	"log"
	"net/http"

	"dominion/experimental/grpc_hello_world"

	"github.com/grpc-ecosystem/grpc-gateway/v2/runtime"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
)

var port = flag.String("port", ":8080", "Port to listen on")
var grpcPort = flag.String("grpc_port", "localhost:50051", "Port to listen on")

func main() {
	flag.Parse()

	mux := runtime.NewServeMux()
	opts := []grpc.DialOption{grpc.WithTransportCredentials(insecure.NewCredentials())}
	err := grpc_hello_world.RegisterGreeterHandlerFromEndpoint(context.Background(), mux, *grpcPort, opts)
	if err != nil {
		log.Fatalf("failed to serve: %v", err)
	}

	log.Printf("gRPC hello world gateway listening :%s", *port)
	if err := http.ListenAndServe(*port, mux); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}
```

### openapiv2 接口文档生成

在 `BUILD.bazel` 添加如下 `target` 即可生成 `openapiv2` 规范文档：

```starlark
load("@grpc_ecosystem_grpc_gateway//protoc-gen-openapiv2:defs.bzl", "protoc_gen_openapiv2")

protoc_gen_openapiv2(
    name = "grpc_hello_world_openapiv2",
    proto = ":grpc_hello_world_proto",
)
```

使用 `bazel run :examplepb_protoc_gen_openapiv2` 即可生成对应文档。

## 总结

本文介绍了如何使用 `bazel` 编译 `grpc-go` 与 `grpc-gateway`。并且使用相关 `bazel rules` 生成中间代码，进而降低开发和维护成本。