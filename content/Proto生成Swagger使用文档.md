---
title: Proto生成Swagger使用文档
categories: protobuffer
comments: true
---

# Proto生成Swagger使用文档



此文档描述了一些在proto定义过程中，常见的一些swagger描述，可以在使用的过程中进行一些参考

<!--more-->

### **grpc-gateway 生成http服务**[grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway)

1. 不修改protoc文件，根据默认规则生成相应的http路由
2. 修改protoc文件，根据添加的注释生成自定义http路由
3. 不修改protoc文件，定义一个配置文件，根据配置文件生成自定义http路由

### 修改protoc文件，添加google/api相关内容生成自定义http路由

1. API相关规范 [AIP-127: HTTP and gRPC Transcoding](https://google.aip.dev/127)
2. [google/api相关参数](https://github.com/googleapis/googleapis/tree/master/google/api)
   1. 

### 添加validate相关内容生成对应的校验规则

1. [validate相关库](https://github.com/bufbuild/protoc-gen-validate/blob/main/validate/validate.proto)

- [2.](https://github.com/bufbuild/protoc-gen-validate/blob/main/README.md) [使用示例](https://github.com/bufbuild/protoc-gen-validate/blob/main/README.md)

### **swag文档生成**

1.  [protoc-gen-openapiv2插件](https://github.com/grpc-ecosystem/grpc-gateway/tree/main/protoc-gen-openapiv2)
2. [openapiv2/options相关参数](https://github.com/grpc-ecosystem/grpc-gateway/tree/main/protoc-gen-openapiv2/options)
3. [customizing_openapi](https://grpc-ecosystem.github.io/grpc-gateway/docs/mapping/customizing_openapi_output/)
4. [使用示例](https://github.com/grpc-ecosystem/grpc-gateway/blob/main/examples/internal/proto/examplepb/a_bit_of_everything.proto)
5. [option相关用法](https://developers.google.com/protocol-buffers/docs/proto3#options)
6. **protoc-gen-openapiv2** 会根据 google.api 和  grpc.gateway.protoc_gen_openapiv2.options 生成相应的swag文件

#### Swagger Option

##### swagger基础信息

```ProtoBuf
option (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_swagger) = {
    info: {
        title: "a swagger test demo", // 应用标题
        version: "0.1", //提供的API版本
        contact: {// 公开API联系信息
            name: "Ethan Leo",
            url: "",
            email: "ethanleo@yeah.net"
        }
    },
    base_path: "/v1",
    host: "localhost",// api host
    schemes: [HTTP, HTTPS] // API 协议 HTTP HTTPS  WS WSS;
};
```

##### **需要的依赖**

```ProtoBuf
import "protoc-gen-openapiv2/options/annotations.proto";
```

##### 接口定义

```ProtoBuf
rpc CreateUser (CreateUserRequest) returns (CreateUserResponse) {
    option (google.api.http) = {
        post: "/v1/users"
        body: "*"
    };
    option (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_operation) = {
        summary: "创建用户" // swagger 接口名称
        description: "创建用户" // swagger 接口描述
        tags: "用户" // swagger 标签
        schemes: [HTTP, HTTPS], // 支持的协议
    };
};
```

在每个rpc的定义中添加两个类型的options

- google.api.http 中定义了接口的路径以及请求方法
- grpc.gateway.protoc_gen_openapiv2.options.openapiv2_operation 定义了接口的名称以及接口的描述等信息

##### **需要的依赖**

```ProtoBuf
import "google/api/annotations.proto";
import "protoc-gen-openapiv2/options/annotations.proto";
```

##### 参数定义

```ProtoBuf
message User {
    string id = 1;
    string name = 2 [(grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) = {
        description: "姓名", // 参数的名称
        default: "Ethan", // 参数默认值
        min_length: 1, // 参数最小值 备注
        max_length: 10, // 参数最大值 备注
    },
        (google.api.field_behavior) = REQUIRED, // 是否必须
        (validate.rules).string = {
            min_len: 1, // 最小长度 校验规则
            max_len: 10, // 最大长度 校验规则
        }

    ];
    int32 age = 3 [(grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) =
        {
            description: "年龄",
            default: "0",
            type: INTEGER,
        },
        (validate.rules).int32 = {
            gte: 0
        }
    ];
    
    Hobby hobby = 4 [(grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) = {
        description: "爱好",
        default: "HOBBY_DANCE",
        enum: [
            "1",
            "2"
        ],
        type: INTEGER,
    }, (google.api.field_behavior) = REQUIRED,
        (validate.rules).enum = {
            in: [1, 2]
        }
    ];
}
```

- grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field 中定义了一些参数字段的备注信息
- google.api.field_behavior 中指定了参数是否必须（默认生成的swagger文档中是非必须）REQUIRED 指定字段必须
-  validate.rules 指定了字段的校验规则

##### **需要的依赖**

```ProtoBuf
import "google/api/field_behavior.proto";
import "validate/validate.proto";
import "protoc-gen-openapiv2/options/annotations.proto";
```

##### 注意事项

在枚举进行注释的时候需要在枚举的定义上面加入` .` 才会让注释生效 https://github.com/grpc-ecosystem/grpc-gateway/issues/2670#issuecomment-1257679009

```ProtoBuf
// Hobby is Hobby.       
enum Hobby {
    // 默认
    HOBBY_UNSPECIFIED = 0;
    // 跳舞
    HOBBY_DANCE = 1;
    // 唱歌
    HOBBY_SING = 2;
}
```

### 完整示例

### proto文件

```ProtoBuf
syntax = "proto3";

package tutorial;

import "google/api/annotations.proto";
import "google/api/field_behavior.proto";
import "protoc-gen-openapiv2/options/annotations.proto";
import "validate/validate.proto";


option go_package = "grpc-gateway-demo/gen/proto/v1;pb";

option (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_swagger) = {
    info: {
        title: "a swagger test demo", // 应用标题
        version: "0.1", //提供的API版本
        contact: {// 公开API联系信息
            name: "Ethan Leo",
            url: "",
            email: "ethanleo@yeah.net"
        }
    },
    base_path: "/v1",
    host: "localhost",// api host
    schemes: [HTTP, HTTPS] // API 协议 HTTP HTTPS  WS WSS;
};

service UserService {
    rpc CreateUser (CreateUserRequest) returns (CreateUserResponse) {
        option (google.api.http) = {
            post: "/v1/users"
            body: "*"
        };
        option (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_operation) = {
            summary: "创建用户" // swagger 接口名称
            description: "创建用户" // swagger 接口描述
            tags: "用户" // swagger 标签
            schemes: [HTTP, HTTPS],
        };
    };

    rpc GetUser (GetUserRequest) returns (User) {
        option (google.api.http) = {
            get: "/v1/users/{id}",
        };
        option (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_operation) = {
            summary: "获取用户",
            tags: "用户",
            description: "根据用户ID获取用户",
        };


    };

}

message User {
    string id = 1;
    string name = 2 [(grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) = {
        description: "姓名",
        default: "Ethan",
        min_length: 1, // 会在swagger的备注中生成，对实际代码没有影响
        max_length: 10,
    },
        (google.api.field_behavior) = REQUIRED,
        (validate.rules).string = {
            min_len: 1, // 会生成validate函数,调用时进行参数校验
            max_len: 10, 
        }

    ];
    int32 age = 3 [(grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) =
        {
            description: "年龄",
            default: "0",
            type: INTEGER,
        },
        (validate.rules).int32 = {
            gte: 0
        }
    ];
    Hobby hobby = 4 [(grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) = {
        description: "爱好",
    }, (google.api.field_behavior) = REQUIRED,
        (validate.rules).enum = {
            in: [1, 2]
        }
    ];
}
// Hobby is Hobby.
enum Hobby {
    // 默认
    HOBBY_UNSPECIFIED = 0;
    // 跳舞
    HOBBY_DANCE = 1;
    // 唱歌
    HOBBY_SING = 2;
}


message GetUserRequest {
    string id = 1 [(grpc.gateway.protoc_gen_openapiv2.options.openapiv2_field) = {
        field_configuration: {
            path_param_name: "user_id"
        }
    }];
}

message CreateUserRequest {
    User user = 1;
}

message CreateUserResponse {
    string message = 1;
}
```

### 生成的swagger json

```JSON
{
  "swagger": "2.0",
  "info": {
    "title": "a swagger test demo",
    "version": "0.1",
    "contact": {
      "name": "Ethan Leo",
      "email": "ethanleo@yeah.net"
    }
  },
  "tags": [
    {
      "name": "UserService"
    }
  ],
  "host": "localhost",
  "basePath": "/v1",
  "schemes": [
    "http",
    "https"
  ],
  "consumes": [
    "application/json"
  ],
  "produces": [
    "application/json"
  ],
  "paths": {
    "/v1/users": {
      "post": {
        "summary": "创建用户",
        "description": "创建用户",
        "operationId": "UserService_CreateUser",
        "responses": {
          "200": {
            "description": "A successful response.",
            "schema": {
              "$ref": "#/definitions/tutorialCreateUserResponse"
            }
          },
          "default": {
            "description": "An unexpected error response.",
            "schema": {
              "$ref": "#/definitions/rpcStatus"
            }
          }
        },
        "parameters": [
          {
            "name": "body",
            "in": "body",
            "required": true,
            "schema": {
              "$ref": "#/definitions/tutorialCreateUserRequest"
            }
          }
        ],
        "tags": [
          "用户"
        ]
      }
    },
    "/v1/users/{id}": {
      "get": {
        "summary": "获取用户",
        "description": "根据用户ID获取用户",
        "operationId": "UserService_GetUser",
        "responses": {
          "200": {
            "description": "A successful response.",
            "schema": {
              "$ref": "#/definitions/tutorialUser"
            }
          },
          "default": {
            "description": "An unexpected error response.",
            "schema": {
              "$ref": "#/definitions/rpcStatus"
            }
          }
        },
        "parameters": [
          {
            "name": "id",
            "in": "path",
            "required": true,
            "type": "string"
          }
        ],
        "tags": [
          "用户"
        ]
      }
    }
  },
  "definitions": {
    "protobufAny": {
      "type": "object",
      "properties": {
        "@type": {
          "type": "string"
        }
      },
      "additionalProperties": {}
    },
    "rpcStatus": {
      "type": "object",
      "properties": {
        "code": {
          "type": "integer",
          "format": "int32"
        },
        "message": {
          "type": "string"
        },
        "details": {
          "type": "array",
          "items": {
            "$ref": "#/definitions/protobufAny"
          }
        }
      }
    },
    "tutorialCreateUserRequest": {
      "type": "object",
      "properties": {
        "user": {
          "$ref": "#/definitions/tutorialUser"
        }
      }
    },
    "tutorialCreateUserResponse": {
      "type": "object",
      "properties": {
        "message": {
          "type": "string"
        }
      }
    },
    "tutorialHobby": {
      "type": "string",
      "enum": [
        "HOBBY_UNSPECIFIED",
        "HOBBY_DANCE",
        "HOBBY_SING"
      ],
      "default": "HOBBY_UNSPECIFIED",
      "description": "Hobby is Hobby.\n\n - HOBBY_UNSPECIFIED: 默认\n - HOBBY_DANCE: 跳舞\n - HOBBY_SING: 唱歌"
    },
    "tutorialUser": {
      "type": "object",
      "properties": {
        "id": {
          "type": "string"
        },
        "name": {
          "type": "string",
          "default": "Ethan",
          "description": "姓名",
          "maxLength": 10,
          "minLength": 1,
          "required": [
            "name"
          ]
        },
        "age": {
          "type": "integer",
          "format": "int32",
          "default": "0",
          "description": "年龄"
        },
        "hobby": {
          "$ref": "#/definitions/tutorialHobby",
          "description": "爱好"
        }
      },
      "required": [
        "name"
      ]
    }
  }
}
```

### 导入Yapi

![img](https://wowjoycom.feishu.cn/space/api/box/stream/download/asynccode/?code=NjJhYTE1ZWZmMThhMjQ1YTE3NTc2YjVhNmY5YjAwY2NfbUFOR1NENzdaekRvZlNMdEtYNDhWZ0NpOHhEVzNXcDNfVG9rZW46Ym94Y255YWx5RVRGaGZSOWdOelFKRmZvQ3FmXzE2Nzg2NzMxMDk6MTY3ODY3NjcwOV9WNA)

![img](https://wowjoycom.feishu.cn/space/api/box/stream/download/asynccode/?code=MTU1YjIyOWE1ZmE2OGMzZjA4MzA4NzBmYTVlNmQ1MTJfT2NRaTJmQUZNRjZYQ0hnVmZrSXczQTNUWFloZVc4NWZfVG9rZW46Ym94Y240ZTlVWTc0M3RZMFlCS203NnZMQVNlXzE2Nzg2NzMxMDk6MTY3ODY3NjcwOV9WNA)