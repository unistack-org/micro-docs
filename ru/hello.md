# Первый сервис

## Proto

Микро не обязывает использовать protobuf для работы. Тем не менее, описание сервиса посредством proto файла позволяет лучше спроектировать сервис. Спроектированный proto файл можно передать разработчикам frontend системы не дожидаясь окончания разработки сервиса. Тем самым используя опианные в proto/openapi файле соглашения можно быть уверенным, что backend и frontend части сервиса будут работать вместе правильно.

Именно поэтому стоит придерживаться описанного выше порядка разработки. Обычно proto файл для сервиса располагается либо в отдельном репозитории (если потребителей сервиса несколько) либо в том же репозитории в директории proto (если потребителем будет только сам сервис).

Файл обычно состоит (в простейшем случае) из трех частей:
- служебная - содержит импорты и название пакета
- сервис - содержит название и методы сервиса с аннотациями
- сообщения - содержит описание всех сообщений, которые использует сервис

# Пример

Ниже приведен пример простейшего прото файла с комментариями:

```protobuf
syntax = "proto3"; // мы рассматриваем только proto3 как наиболее подерживаемый формат

package github; // обычно имя сервиса
option go_package = "domain.tld/some_repo;pb"; // можно не указывать, помогает go генератору использовать верные имена пакетов

import "api/annotations.proto"; // так как в файле для http клиента и сервера присуствуют аннотации, требуется указать импорт
import "openapiv2/annotations.proto"; // так как в файле для http клиента и сервера присуствуют аннотации, требуется указать импорт

service Github { // название сервиса, оно будет использоваться в сгенерированном коде, а также фигурировать в служебных структурах в виде имени Endpoint
  rpc LookupUser(LookupUserReq) returns (LookupUserRsp) { // описание имени метода, принимаемых и отправляемых типов сообщений
    option (micro.openapiv2.openapiv2_operation) = { // openapi аннотация
      operation_id: "LookupUser"; // название операции в openapi
      responses: { // типы ответов
        key: "default"; // используется для всех типов ответов, кроме стандартного, описанного в методе
        value: {
          description: "Error response"; // openapi описание
          schema: { json_schema: { ref: ".github.Error"; } } // ссылка на тип сообщения, состоит из имени пакета, которое мы указали в package и имени сообщения
        }
      }
    };
    option (micro.api.http) = { get: "/users/{username}"; }; // аннотация, сообщаяющая о том, что для вызова метода требутеся сделать GET запрос на путь /users где username берется из структуры запроса и подставляется в путь, например /users/github_user . В случае методов POST/PATCH/PUT может присуствовать еще body:"*"; сообщающая, что все поля структуры запроса следует передать в теле реквеста.
  };
};

message LookupUserReq { // описание сообщения реквеста
  string username = 1; // поле сообщения, оно же используется для составления GET запроса
};

message LookupUserRsp { // описание сообщения респонса
  string name = 1; // поле сообщения ответа, на самом деле github отдает больше полей, приведено исключительно для примера
};

message Error { // описание сообщения об ошибке, для каждого метода могут быть свои ошибки
  string message = 1; // в сообщении github об ошибке присутсвует данное поле
};
```

# Кодогенерация

После того, как proto файл создан необходимо по нему сгенерировать код. Для этих целей используется protoc-gen-micro . Данное приложения на основе proto описания генерирует код клиента и сервера. Основная сложность состоит в том, что proto файлы, указанные в качестве import должны быть доступны на момент генерации. Для этого должны быть установлены пакеты, в которых данные файлы присутствуют. Проще всего этого достичь путем включения в файл tools.go следующего содержимого:

```go
// +build tools

package main

import (
        _ "github.com/unistack-org/micro-proto/openapiv2"
)
```

В первой строке мы указваем, что данный файл участвут в компиляции только, если передан флаг tools (что мы передавать не собираемся). Команда go build / go get видит использование импорта и добавит данную зависимость для скачивания без компиляции.

Следующий шаг это создание скрипта, который будет запускаться для генерации

```shell
#!/bin/sh -e

INC=$(go list -f '{{ .Dir }}' -m github.com/unistack-org/micro-proto)
ARGS="-I${INC}"

protoc $ARGS -Iproto --openapiv2_out=disable_default_errors=true,allow_merge=true:./proto/ --go_out=paths=source_relative:./proto/ --micro_out=components="micro|http",debug=true,paths=source_relative:./proto/ proto/*.proto
```

В данном файле мы указываем, что хотим сгенерировать swagger/openapi спецификацию, go код описания структур а также micro интерфейсы и http клиент и сервер. Все сгенерированные файлы будут находится в директории proto

# Сервис

## Клиент

Ниже представлен пример клиента, использующего сгенерированный нами код:

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

## Сервер

Код, представленный ниже объясняет создание базового сервиса Github

Первый шаг - создание обработчика

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

Следующий шаг - создание сервера

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
	   Опции для сервиса, контекст с функцией cancel()
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
	// Создание нового сервиса и передача в него опций
	srv := micro.NewService(options...)

	// Инициализируем все опции в сервисе
	if err := srv.Init(); err != nil {
		logger.Fatal(ctx, err)
	}
	// Создаем реализацию для сервера
	eh := handler.NewGithubHandler()
	// Регистрируем реализацию в сервере
	if err := pb.RegisterGithubServer(srv.Server(), eh); err != nil {
		logger.Fatal(ctx, err)
	}
	// Запускаем сервис
	if err := srv.Run(); err != nil {
		logger.Fatal(ctx, err)
	}
}
```

Важные замечания:

1. Чтобы использовать grpc вместо http - нужно заменить http на grpc в опции компонентах генератора --micro_out=components="micro|http"
2. Для использования GRPC сервера в коде нужно поменять импорт и использовать grpcsrv.NewServer() вместо httpsrv.NewServer()

````
https://github.com/unistack-org/micro-server-grpc/v3
````
