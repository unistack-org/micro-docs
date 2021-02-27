# The first service

## Proto

Micro does not oblige to use protobuf. But for better experience in communication between microservices (client -
server) strongly recommended to use .proto files. It insures developers on a both sides they are using the same
generated .proto structures and services.

Usually .proto files located in the directory of the project. If amount of services which are using .proto file are
growing up, the best way to share .proto files is using any of storage (github, gitlab etc.).

File .proto usually contains from three part:

- header - contains imports and package names
- service - contains service names and methods with annotations
- messages - description for structures, which are using by service

# Example

See code below with .proto file example for better undestanding:

```protobuf
syntax = "proto3"; // syntax of proto file

package github; // service name
option go_package = "domain.tld/some_repo;pb"; // optional, help code generator names packages in a right way

import "api/annotations.proto"; // for using http next to grpc we have to define import
import "openapiv2/annotations.proto"; // for using openapi generation (describe our services) we have to define import 

service Github {// service name - will be using by code generator on the client side and server side support functions (create service client, create service server), in Endpoints and openapi description
  rpc LookupUser(LookupUserReq) returns (LookupUserRsp) {// description for method with structures to receive and respond
    option (micro.openapiv2.openapiv2_operation) = {// openapi annotaiont
      operation_id: "LookupUser"; // operation name in openapi
      responses: {// type of responses
        key: "default"; // using by any of response type except standart one described in the method
        value: {
          description: "Error response"; // openapi description
          schema: {json_schema: {ref: ".github.Error";}}; // link to message type, consists with package name and message name
        }
      }
    };
    option (micro.api.http) = {get: "/users/{username}";}; // describes endpoint which should be used connecting to rpc LookupUser via http with method GET and path /users/username. In order to use POST, PUT, PATCH requests also may contain body. Body is defining the same way as path variable, but instead should be using link to message structure. If body is not pre-defined should be used body:'*' declaration.
  };
};

message LookupUserReq {// request description
  string username = 1; // the username field. Name have to be identical to path variable declaration in option google.api.http GET /users/{username}
};

message LookupUserRsp {// response description
  string name = 1; // here define only one field from api.github.com - name of user
};

message Error {// error description
  string message = 1; // message from api.github.com if user not found
};
```

# Code generation

When a proto file has been created our next step will be code generation. For those purposes we are using
protoc-gen-micro. It generates code based on proto description for server and client sides. The key thing here - imports
have to be available. To make this happen we have to install them. The easiest way to do this - using file tools.go like
example below:

```go
// +build tools

package main

import (
	_ "github.com/unistack-org/micro-proto/openapiv2"
)
```

For using this file only in compilation time, we have to define // +build tools and pass the flag tools (we won't do
that). The command go build / go get see the import and add the dependency.

Next step - script for code generation:

```shell
#!/bin/sh -e

INC=$(go list -f '{{ .Dir }}' -m github.com/unistack-org/micro-proto)
ARGS="-I${INC}"

protoc $ARGS -Iproto --openapiv2_out=disable_default_errors=true,allow_merge=true:./proto/ --go_out=paths=source_relative:./proto/ --micro_out=components="micro|http",debug=true,paths=source_relative:./proto/ proto/*.proto
```

Here we want to generate swagger/openapi specification, go code with structures, micro interfaces and http client,
server. All generated code will be stored in a proto directory.

# Service

## Client

The code below is describing creation client for Github service

```go
package main

import (
	"context"

	mhttp "github.com/unistack-org/micro-client-http/v3"
	jsoncodec "github.com/unistack-org/micro-codec-json/v3"
	pb "github.com/unistack-org/micro-tests/client/http/proto"
	"github.com/unistack-org/micro/v3/client"
	"github.com/unistack-org/micro/v3/logger"
)

func main() {
	hcli := mhttp.NewClient(client.ContentType("application/json"), client.Codec("application/json", jsoncodec.NewCodec()))
	cli := client.NewClientCallOptions(hcli, client.WithAddress("https://api.github.com"))
	gh := pb.NewGithubClient("github", c)

	rsp, err := gh.LookupUser(context.TODO(), &pb.LookupUserReq{Username: "vtolstov"})
	if err != nil {
		logger.Errorf(context.TODO(), err)
	}

	if rsp.Name != "Vasiliy Tolstov" {
		logger.Errorf(context.TODO(), "invalid rsp received: %#+v\n", rsp)
	}
}
```

## Server

The code below is describing creation server for Github service

First step - creating handler

```go
package handler

import (
	"context"
	"errors"
	httpsrv "github.com/unistack-org/micro-server-http/v3"
	pb "github.com/unistack-org/micro-tests/client/http/proto"
	"net/http"
)

type GithubHandler struct{}

func NewGithubHandler() *GithubHandler {
	return &GithubHandler{}
}

func (h *GithubHandler) LookupUser(ctx context.Context, req *pb.LookupUserReq, rsp *pb.LookupUserRsp) error {
	if req.GetUsername() == "" || req.GetUsername() != "vtolstov" {
		httpsrv.SetRspCode(ctx, http.StatusBadRequest)
		return httpsrv.SetError(&pb.Error{Message: "name is not correct"})
	}
	rsp.Name = "Vasiliy Tolstov"
	httpsrv.SetRspCode(ctx, http.StatusOK)
	return nil
}
```

Second step - creating server

```go
package main

import (
	"context"
	mhttp "github.com/unistack-org/micro-client-http/v3"
	jsoncodec "github.com/unistack-org/micro-codec-json/v3"
	httpsrv "github.com/unistack-org/micro-server-http/v3"
	"github.com/unistack-org/micro/v3"
	"github.com/unistack-org/micro/v3/client"
	"github.com/unistack-org/micro/v3/logger"
	"github.com/unistack-org/micro/v3/server"
	"github.com/unistack-org/micro-tests/server/http/handler"
	pb "github.com/unistack-org/micro-tests/client/http/proto"
)

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	/*
	    Options for service, pass the context with cancel
	*/
	options := append([]micro.Option{},
		micro.Server(httpsrv.NewServer(
			server.Name("github-service"),
			server.Version("1.0"),
			server.Address(":8080"),
			server.Context(ctx),
			server.Codec("application/json", jsoncodec.NewCodec()),
		)),
		micro.Client(mhttp.NewClient(
			client.Name("github-client"),
			client.Context(ctx),
			client.Codec("application/json", jsoncodec.NewCodec()),
			client.ContentType("application/json"),
		)),
		micro.Context(ctx),
	)
	// create new service
	srv := micro.NewService(options...)

	// init all stuff
	if err := srv.Init(); err != nil {
		logger.Fatal(ctx, err)
	}
	// create handler
	eh := handler.NewGithubHandler()
	// register handler in server
	if err := pb.RegisterGithubServer(srv.Server(), eh); err != nil {
		logger.Fatal(ctx, err)
	}
	// run service
	if err := srv.Run(); err != nil {
		logger.Fatal(ctx, err)
	}
}
```

Important note:

1. In order to use gRPC instead of http - need to replace http to grpc in --micro_out=components="micro|http"
2. Implementation for gRPC in code has only one difference - need to pass grpcsrv.NewServer()
   instead of httpsrv.NewServer(). Library could be found here:

````
github.com/unistack-org/micro-server-grpc/v3
````
