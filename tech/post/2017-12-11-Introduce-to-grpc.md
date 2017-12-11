# gRPC Introduction(1)


---

## Introduction

gRPC is a modern open source high performance RPC framework that can run in any environment. It can efficiently connect services in and across data centers with pluggable support for load balancing, tracing, health checking and authentication. It is also applicable in last mile of distributed computing to connect devices, mobile applications and browsers to backend services.

## Hello World

If you open the gRPC's [website](https://grpc.io/), you will find there are about 10 languages that support, in this blog we 
will use python to build the demo.

### Environment

Please ensure python2.7+ (or python3.4) is installed in your environment, we need to install these two libraries:
1. grpcio: basic grpc framework library
```
pip install grpio
```
2. grpcio-tools: Protobuf code generator
```
pip install grpcio-tools
```

### Generate proto file

gRPC uses ProtoBuf to serialize and deserialize objects, it's faster than JSON, XML and easy to use, before you can create your own server and client, you need to create your own proto file and generate the `stub` files.
```
syntax = "proto3";

package helloworld;

//The greeter service definition
service Greeter{
	//Send a greeting
	rpc SayHello(HelloRequest) returns (HelloReply){}	
}

//The request message containing the user's name
message HelloRequest{
	string name = 1;
}

//The reply message containing the greetings
message HelloReply{
	string message = 1;
}

```
The code above defines one interface which is named `SayHello` and the corresponding `Messages`. 
**NOTE:** Every object in ProtoBuf is a message and there are four different interface types (or methods) in gRPC:

1. Unary RPCs where the client sends a single request to the server and gets a single response back, just like a normal function call.
```
rpc SayHello(HelloRequest) returns (HelloResponse){
}
```
2. Server streaming RPCs where the client sends a request to the server and gets a stream to read a sequence of messages back. The client reads from the returned stream until there are no more messages.

```
rpc LotsOfReplies(HelloRequest) returns (stream HelloResponse){
}
```
3. Client streaming RPCs where the client writes a sequence of messages and sends them to the server, again using a provided stream. Once the client has finished writing the messages, it waits for the server to read them and return its response.
```
rpc LotsOfGreetings(stream HelloRequest) returns (HelloResponse) {
}
```
4. Bidirectional streaming RPCs where both sides send a sequence of messages using a read-write stream. The two streams operate independently, so clients and servers can read and write in whatever order they like: for example, the server could wait to receive all the client messages before writing its responses, or it could alternately read a message then write a message, or some other combination of reads and writes. The order of messages in each stream is preserved.
```
rpc BidiHello(stream HelloRequest) returns (stream HelloResponse){
}
```
It's obvious we use the basic method `Unary RPCs` for demonstration. Now we can use gRPC tools to generate the python stub files:
```
python -m grpc_tools.protoc -I. --python_out=. --grpc_python_out=. helloworld.proto
```
Parameters:
1. `-I{PATH}`: Specify in which folder the ProtoBuf generator will try to search
2. `--python_out`: Specify the folder where to put the generated interface files.
3. `--grpc_python_out`: Specify the folder where to put the generate messages objects.
4. `{protofile}`: Specify which proto file will be used to generate.

After the command, the folder will contain these three files as below:
```
---hello_world
    |
    ---helloworld.protp
    |
    ---helloworld_pb2.py
    |
    ---helloworld_pb2_grpc.py
```

### Write Server/Client code

Then we can use the generated files to create our own server and client:
```
#greeter_server.py
from concurrent import  futures
import time

import grpc

import helloworld_pb2
import helloworld_pb2_grpc

_ONE_DAY_IN_SECONDS = 60 * 60 * 24


class Greeter(helloworld_pb2_grpc.GreeterServicer):

    def SayHello(self, request, context):
        return helloworld_pb2.HelloReply(message="Hello, %s" % request.name)


def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    helloworld_pb2_grpc.add_GreeterServicer_to_server(Greeter(), server)
    server.add_insecure_port('[::]:50051')
    server.start()
    try:
        while True:
            time.sleep(_ONE_DAY_IN_SECONDS)
    except KeyboardInterrupt:
        server.stop()

if __name__ == '__main__':
    serve()
```
The server code is quite simple, once it runs, the server will listen to port `50051` and send back the message with a prefix 'Hello'. And the client codes are below:
```
#greeter_client.py
from __future__ import print_function
import grpc

import helloworld_pb2
import helloworld_pb2_grpc

def run():
    channel = grpc.insecure_channel("localhost:50051")
    stub = helloworld_pb2_grpc.GreeterStub(channel)
    reqonse = stub.SayHello(helloworld_pb2.HelloRequest(name='tommylike'))
    print("Greeter client received:" + reqonse.message)


if __name__ == '__main__':
    run()
```

## Run Hello world

Two individual terminals are required to run this program
```
python greeter_server.py
```
and
```
python greeter_client.py
```
We can tell the client received the message from server via console output:
```
Greeter client received: tommylike
```
### Beyond Hello World
Some of the technological details regarding ProtoBuf and the gRPC's advanced features will be introduced in the next article.
