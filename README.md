## 🐛 Bug Report

`openapiv2` generates inline object for the request body if `rpc` has path parameter and uses `body: "*"`, 
instead of creating a definition with the request message name and referencing it in the endpoint body parameter.
This causes clients generators to create classes named like `InlineObject1`, `InlineObject2` for requests. 

## To Reproduce

(Note: https://github.com/smaant/openapiv2-inline-object can be used for easier reproduction.)

Create a `test.proto` file with the following content:

```protobuf
syntax = "proto3";
option go_package = ".;test";
package test;

import "google/api/annotations.proto";

service Service {
  rpc CreateTest(TestRequest) returns (TestResponse) {
    option (google.api.http) = {
      post: "/tests/{test_id}",
      body: "*"
    };
  }
}

message TestRequest {
  string test_id = 1;
  string some_data = 2;
}

message TestResponse {
  string test_data = 1;
}
```

Copy `annotations.proto` and `http.proto` from https://github.com/googleapis/googleapis/tree/master/google/api to the
`third_party/google/api`.

Make sure you have `protoc-gen-openapiv2` in the `PATH`.

Generate OpenAPI definitions:
```shell
protoc -I. -I./third_party --openapiv2_out=v2/ test.proto
```

## Expected behavior

`openapiv2` plugin generates definitions that include endpoint and the type for the request body:

```json5
{
  "paths": {
    "/tests/{testId}": {
      "post": {
        // ...
        "parameters": [
          {
            "name": "testId",
            "in": "path",
            "required": true,
            "type": "string"
          },
          {
            "name": "body",
            "in": "body",
            "required": true,
            "schema": {
              "$ref": "#/definitions/testTestRequest"
            }
          }
        ],
        // ...
      }
    }
  },
  "definitions": {
    "testTestRequest": {
      "type": "object",
      "properties": {
        "some_data": {
          "type": "string"
        }
      }
    },
    // ...
  }
}
```

Notice that `testTestRequest` includes only `some_data` property and doesn't include `testId`
which became a path parameter.

## Actual Behavior

`openapiv2` plugin generates definitions that include endpoint with expected path parameter (`testId`) but the body
parameter includes inline schema instead of referencing a request type definition.

```json5
{
  "paths": {
    "/tests/{testId}": {
      "post": {
        // ...
        "parameters": [
          {
            "name": "testId",
            "in": "path",
            "required": true,
            "type": "string"
          },
          {
            "name": "body",
            "in": "body",
            "required": true,
            "schema": {
              "type": "object",
              "properties": {
                "someData": {
                  "type": "string"
                }
              }
            }
          }
        ],
        // ...
      }
    }
  }
}
```

For the comparison `swagger` plugin with the same input generates a different definition. 
Make sure `protoc-gen-swagger` is in `PATH` and run
```shell
protoc -I. -I./third_party --swagger_out=v1/ test.proto
```

It will produce a following definition:
```json5
{
  "paths": {
    "/tests/{test_id}": {
      "post": {
        // ...
        "parameters": [
          {
            "name": "test_id",
            "in": "path",
            "required": true,
            "type": "string"
          },
          {
            "name": "body",
            "in": "body",
            "required": true,
            "schema": {
              "$ref": "#/definitions/testTestRequest"
            }
          }
        ],
        // ...
      }
    }
  },
  "definitions": {
    "testTestRequest": {
      "type": "object",
      "properties": {
        "test_id": {
          "type": "string"
        },
        "some_data": {
          "type": "string"
        }
      }
    },
  }
}
```

Which is closer to the expected result expect that `testTestResult` includes both `test_id` and `some_data` properties.

## Your Environment
* MacOS 11.6
* `protoc` v3.17.3
* `protoc-gen-openapiv2` v2.7.1
* `protoc-gen-swagger` v1.16.0
