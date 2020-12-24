# 简介

## protobuf

Protocol Buffer (简称Protobuf) 是Google出品的性能优异、跨语言、跨平台的序列化库

## gRPC

gRPC是谷歌推出的高性能RPC框架，默认支持protobuf

# ProtoBuf

编写 `.proto` 文件， 然后通过编译器根据不同语音生成不同的文件，比如go语音就是生成  `.pb.go` 文件，C语音生成 `.h` 和 `.cc` 文件，生成的文件里面除了数据结构还有些工具方法，比如字段的`getter`

分为v2和v3版本，一般通过开头指明版本

```
syntax = "proto3";
```

## 语法

### message

在 proto 中，所有结构化的数据都被称为 message。

```protobuf
syntax = "proto3";
package proto;
option go_package = ".;proto";
message User {
    string name=1;
    int32 age=2;
}
message Id {
    int32 uid=1;
}
//要生成server rpc代码
service ServiceSearch{
    rpc SaveUser(User) returns (Id){}
    rpc UserInfo(Id) returns (User){}
}
```

- message名称采用驼峰命名法，首字母大写
- message中每个字段都有一个唯一的编号
  - 可以不连续
  - 编号1-15来放常用的字段，可以适当保留一些方便后续增加常用字段
  - 编号19000-19999为protobuf协议保留字段，不能使用
- 可使用关键字`reserved`来定义为保留字段，让另外的版本的`.proto`文件不能使用这些编号
- 可用`repeated`来表示重复多次的数字，常用来表示数组

### package

在`.proto`文件中使用`package`声明包名，避免命名冲突。

```protobuf
syntax = "proto3";package foo.bar;message Open {...}
```

在其他的消息格式定义中可以使用**包名+消息名**的方式来使用类型，如：

```protobuf
message Foo {    ...    foo.bar.Open open = 1;    ...}
```

```protobuf
syntax = "proto3";
package hello;

message HelloRequest {
  string greeting = 1;
}

message HelloResponse {
  string reply = 1;
  repeated int32 number=4;
}

service HelloService {
  rpc SayHello(HelloRequest) returns (HelloResponse){}
}
```

### service

如果想要将消息类型用在RPC(远程方法调用)系统中，可以在`.proto`文件中定义一个RPC服务接口，protocol编译器会根据所选择的不同语言生成服务接口代码。例如，想要定义一个RPC服务并具有一个方法，该方法接收`SearchRequest`并返回一个`SearchResponse`，此时可以在`.proto`文件中进行如下定义：

```protobuf
service SearchService {    
    rpc Search (SearchRequest) returns (SearchResponse) {}
}
```

生成的接口代码作为客户端与服务端的约定，服务端必须实现定义的所有接口方法，客户端直接调用同名方法向服务端发起请求。



# gRPC

采用 ProtoBuf 作为 IDL，则需要定义 service 类型。生成客户端和服务端代码。用户自行实现服务端代码中的调用接口，并且利用客户端代码来发起请求到服务端

## 服务端

服务端相关代码如下，主要定义了 HelloServiceServer 接口，用户可以自行编写实现代码。

```
type HelloServiceServer interface {
        SayHello(context.Context, *HelloRequest) (*HelloResponse, error)
}

func RegisterHelloServiceServer(s *grpc.Server, srv HelloServiceServer) {
        s.RegisterService(&_HelloService_serviceDesc, srv)
}
```

用户需要自行实现服务端接口，代码如下。

比较重要的，创建并启动一个 gRPC 服务的过程：

- 创建监听套接字：lis, err := net.Listen("tcp", port)；
- 创建服务端：grpc.NewServer()；
- 注册服务：pb.RegisterHelloServiceServer()；
- 启动服务端：s.Serve(lis)

```
type server struct{}

// 这里实现服务端接口中的方法。
func (s *server) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
    return &pb.HelloReply{Message: "Hello " + in.Name}, nil
}

// 创建并启动一个 gRPC 服务的过程：创建监听套接字、创建服务端、注册服务、启动服务端。
func main() {
    lis, err := net.Listen("tcp", port)
    if err != nil {
        log.Fatalf("failed to listen: %v", err)
    }
    s := grpc.NewServer()
    pb.RegisterHelloServiceServer(s, &server{})
    s.Serve(lis)
}
```

编译并启动服务端。

## 客户端

主要和实现了 HelloServiceClient 接口。用户可以通过 gRPC 来直接调用这个接口

```
type HelloServiceClient interface {
        SayHello(ctx context.Context, in *HelloRequest, opts ...grpc.CallOption) (*HelloResponse, error)
}

type helloServiceClient struct {
        cc *grpc.ClientConn
}

func NewHelloServiceClient(cc *grpc.ClientConn) HelloServiceClient {
        return &helloServiceClient{cc}
}

func (c *helloServiceClient) SayHello(ctx context.Context, in *HelloRequest, opts ...grpc.CallOption) (*HelloResponse, error) {
        out := new(HelloResponse)
        err := grpc.Invoke(ctx, "/hello.HelloService/SayHello", in, out, c.cc, opts...)
        if err != nil {
                return nil, err
        }
        return out, nil
}
```

用户直接调用接口方法：创建连接、创建客户端、调用接口。

```
func main() {
    // Set up a connection to the server.
    conn, err := grpc.Dial(address, grpc.WithInsecure())
    if err != nil {
        log.Fatalf("did not connect: %v", err)
    }
    defer conn.Close()
    c := pb.NewHelloServiceClient(conn)

    // Contact the server and print out its response.
    name := defaultName
    if len(os.Args) > 1 {
        name = os.Args[1]
    }
    r, err := c.SayHello(context.Background(), &pb.HelloRequest{Name: name})
    if err != nil {
        log.Fatalf("could not greet: %v", err)
    }
    log.Printf("Greeting: %s", r.Message)
}
```
