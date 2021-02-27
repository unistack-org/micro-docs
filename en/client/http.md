HTTP micro client allows you make a request not only to microservices, but to other simple http clients. The big plus to use http client in micro that you don't have to move away from micro ecosystem at all. All wrappers, metrics, tracings, logs will be continuing work with micro client as it was before. Flexability remains the same, as always you can pass prepared client to service constructor. Marshaling / Unmarshaling is simple, pass the codec into client and it will be doing work for you. For errors mapping is present too.

In order to make it easier to work with requests and handle errors, the micro proto generator complements the generated code with the necessary options and parameters based on proto method annotations. Also, thanks to the analysis of the path for the http request, the micro client is able to map the elements of the request structure to the url path. At the same time, the api annotation rules from Google are almost completely supported.

In the case of tracing, the endpoint will look like the method name. In the example below, the service name + dot + method name will appear in the metrics and tracking when executing the LookupUser request. Github.LookupUser thereby allowing you to make beautiful metrics and tracer spans without using HTTP routers and built-in functions.

At the moment, the work of the http client micro was checked in the case of simple structures and simple messages. The name of the field in the query string must match the name of the structure field. Nested structures are not supported yet.

Example of a pseudo API Github in the form of a proto description:
```
syntax = "proto3";                                                                                                                                     
                                                                                                                                                       
package github;                                                                                                                                        
option go_package = "github.com/unistack-org/micro-tests/client/http/proto;pb";                                                                       
                                                                                                                                                       
import "api/annotations.proto";                                                                                                                 
import "openapiv2/annotations.proto";                                                                                               
                                                                                                                                                       
service Github {                                                                                                                                       
  rpc LookupUser(LookupUserReq) returns (LookupUserRsp) {                                                                                              
    option (micro.openapiv2.openapiv2_operation) = {                                                                         
      operation_id: "LookupUser";                                                                                                                      
      responses: {                                                                                                                                     
        key: "default";                                                                                                                                
        value: {                                                                                                                                       
          description: "Error response";                                                                                                               
          schema: { json_schema: { ref: ".github.Error"; } }                                                                                           
        }                                                                                                                                              
      }                                                                                                                                                
    };                                                                                                                                                 
    option (micro.api.http) = { get: "/users/{username}"; };                                                                                          
  };                                                                                                                                                   
};                                                                                                                                                     
message LookupUserReq {                                                                                                                                
  string username = 1;                                                                                                                                 
};                                                                                                                                                     
message LookupUserRsp {                                                                                                                                
  string name = 1;                                                                                                                                     
};                                                                                                                                                     
message Error {                                                                                                                                        
  string message = 1;                                                                                                                                  
};                                                          
```
Thus, calling the LookupUser function in the micro with the LookupUserReq{Username: “vtolstov”} parameter, we will form a get request at /users/vtolstov and fill in the response structure. If you pass a non-existent user, the function returns an error of the Error type.
