---
category: "openapi"
title: "Documents"
description: "Build a complete Feather OpenAPI document from namespaces, schemas, operations, and paths."
order: 2
---

# Documents
---

An OpenAPI document is assembled from small Swift types. The schema types define the public HTTP representation: which fields are exposed, which fields are required, and how clients interact with the resource.

This guide follows the current System Module `Variable` API generator in the [Feather CMS example application](https://github.com/feathercms/feather). It builds a document for a protected admin API and shows how the following pieces fit together:

```text
namespace and schema types
    ↓
schema types and request/response bodies
    ↓
operations and parameters
    ↓
path items and path collection
    ↓
OpenAPI document
    ↓
YAML or JSON specification and generated API types
```

## Installation

The document model comes from `FeatherOpenAPI`:

```swift
.package(
    url: "https://github.com/feather-framework/feather-openapi",
    exact: "1.0.0-rc.2"
),
```

```swift
.product(
    name: "FeatherOpenAPI",
    package: "feather-openapi"
),
```

Feather applications can also use the reusable generator helpers from the `FeatherOpenAPIGenerator` product in `feather-core`. Those helpers provide patterns such as bearer-protected operations, standard HTTP error responses, delete operations, and search request/response schemas.

`FeatherOpenAPIGenerator` comes from the `feather-core` package used by Feather CMS, not from `FeatherOpenAPI`. The YAML-writing generator target also declares [Yams](https://github.com/jpsim/Yams) as a direct dependency.

## Create a namespace for the document types

Use a namespace enum to group the feature's OpenAPI types:

```swift
public enum Variable {}
```

The nested types below use `Variable` as their namespace and describe the public HTTP representation of the resource.

## Define reusable field schemas

The System Module keeps reusable field definitions under `Variable.Schemas`. Each field can define an OpenAPI component identifier, an example, a type, and additional metadata:

```swift
import FeatherOpenAPI

extension Variable {
    enum Schemas {}
    enum Parameters {}
}

extension Variable.Schemas {
    struct ID: StringSchemaRepresentable {
        var openAPIIdentifier: String { "SystemVariableIDField" }
        var example: String? { "V1StGXR8_Z5jdHi6B-myT" }
    }

    struct Key: StringSchemaRepresentable {
        var openAPIIdentifier: String { "SystemVariableKeyField" }
        var example: String? { "sys_variable_timezone" }
    }

    struct Value: StringSchemaRepresentable {
        var openAPIIdentifier: String { "SystemVariableValueField" }
        var example: String? { "UTC" }
    }

    struct Name: StringSchemaRepresentable {
        var openAPIIdentifier: String { "SystemVariableNameField" }
        var example: String? { "default_timezone" }
    }

    struct Notes: StringSchemaRepresentable {
        var openAPIIdentifier: String { "SystemVariableNotesField" }
        var example: String? { "Application default timezone." }
    }

    struct Search: StringSchemaRepresentable {
        var openAPIIdentifier: String { "SystemVariableSearch" }
        var example: String? { "mail" }
    }

    struct Limit: IntSchemaRepresentable {
        var openAPIIdentifier: String { "SystemVariableListLimit" }
        var description: String? {
            "Maximum number of results. Defaults to 20; values are clamped to 1-100."
        }
        var defaultValue: Int? { 20 }
        var example: Int? { 20 }
    }

    struct UpdateMask: StringSchemaRepresentable {
        var openAPIIdentifier: String { "SystemVariableUpdateMask" }
        var example: String? { "key,value,name,notes" }
    }
}

extension Variable.Query {
    enum Schemas {}
}

extension Variable.Query.Schemas {
    struct Filter: ArraySchemaRepresentable {
        var openAPIIdentifier: String { "SystemVariableIDsFilter" }
        var items: (any SchemaRepresentable)? {
            Variable.Schemas.ID()
        }
        var required: Bool { false }
    }
}
```

Referencing these types from several schemas deduplicates the generated components and keeps examples and field names consistent across endpoints.

## Define resource schemas

The detail response exposes the resource returned by `GET`, `POST`, `PUT`, and `PATCH`. The create schema intentionally omits the generated ID:

```swift
extension Variable.Get {
    struct Schema: ObjectSchemaRepresentable {
        var openAPIIdentifier: String { "SystemVariableDetailSchema" }

        var propertyMap: SchemaMap {
            [
                "id": Variable.Schemas.ID().reference(),
                "key": Variable.Schemas.Key(),
                "value": Variable.Schemas.Value(),
                "name": Variable.Schemas.Name().reference(required: false),
                "notes": Variable.Schemas.Notes().reference(required: false),
            ]
        }
    }
}

extension Variable.Post {
    struct Schema: ObjectSchemaRepresentable {
        var openAPIIdentifier: String { "SystemVariableCreateSchema" }

        var propertyMap: SchemaMap {
            [
                "key": Variable.Schemas.Key(),
                "value": Variable.Schemas.Value(),
                "name": Variable.Schemas.Name().reference(required: false),
                "notes": Variable.Schemas.Notes().reference(required: false),
            ]
        }
    }
}
```

The `reference(required: false)` call makes an optional property while keeping the field's shared component. A nullable optional update field is different: the client can omit it or explicitly send `null` to clear it.

## Define request bodies and responses

Request bodies declare the media type and schema. Responses reuse the detail schema instead of copying its property map:

```swift
extension Variable.Post {
    struct RequestBody: RequestBodyRepresentable {
        var openAPIIdentifier: String { "SystemVariableRequestBody" }

        var contentMap: ContentMap {
            [.json: Content(Schema().reference())]
        }
    }
}

extension Variable.Get {
    struct Response: JSONResponseRepresentable {
        var openAPIIdentifier: String { "SystemVariableDetailResponse" }
        var description: String = "SystemVariable response"
        var schema = Schema().reference()
    }
}
```

The update endpoints can reuse the create schema for `PUT`, while `PATCH` uses a separate schema with optional and nullable fields plus an update mask:

```swift
extension Variable.Patch {
    struct RequestBody: RequestBodyRepresentable {
        var openAPIIdentifier: String { "SystemVariablePatchRequestBody" }
        var contentMap: ContentMap {
            [.json: Content(Schema().reference())]
        }
    }

    struct Schema: ObjectSchemaRepresentable {
        var openAPIIdentifier: String { "SystemVariablePatchSchema" }

        var propertyMap: SchemaMap {
            [
                "key": Variable.Schemas.Key().reference(required: false),
                "value": Variable.Schemas.Value().reference(required: false),
                "name": Variable.Schemas.Name().nullable()
                    .reference(required: false),
                "notes": Variable.Schemas.Notes().nullable()
                    .reference(required: false),
            ]
        }
    }
}
```

## Define parameters

Parameters are also reusable types. The System Module uses a path parameter for identifiable operations and query parameters for list filtering:

```swift
extension Variable.Parameters {
    struct ID: PathParameterRepresentable {
        var openAPIIdentifier: String { "SystemVariableIDParameter" }
        var name: String { "systemVariableId" }
        var description: String? { "SystemVariable ID" }
        var schema: any OpenAPISchemaRepresentable {
            Variable.Schemas.ID().reference()
        }
    }

    struct Search: QueryParameterRepresentable {
        var name: String { "search" }
        var required: Bool { false }
        var description: String? {
            "Search across variable keys, values, names, and notes"
        }
        var schema: any OpenAPISchemaRepresentable {
            Variable.Schemas.Search().reference()
        }
    }

    struct IDs: QueryParameterRepresentable {
        var name: String { "ids" }
        var required: Bool { false }
        var description: String? { "Filter by one or more variable IDs" }
        var schema: any OpenAPISchemaRepresentable {
            Variable.Query.Schemas.Filter().reference()
        }
    }

    struct Limit: QueryParameterRepresentable {
        var name: String { "limit" }
        var required: Bool { false }
        var description: String? {
            "Defaults to 20; values are clamped to 1-100"
        }
        var schema: any OpenAPISchemaRepresentable {
            Variable.Schemas.Limit().reference()
        }
    }

    struct UpdateMask: QueryParameterRepresentable {
        var name: String { "updateMask" }
        var required: Bool { true }
        var description: String? {
            "Comma-separated list of fields to update"
        }
        var schema: any OpenAPISchemaRepresentable {
            Variable.Schemas.UpdateMask().reference()
        }
    }
}
```

The generated list endpoint therefore has `search`, `ids`, and `limit` query parameters, while `GET`, `PUT`, and `PATCH` under the ID path share the same `systemVariableId` path parameter.

The shared tag is defined once and inherited by every Variable operation:

```swift
extension Variable {
    struct Tag: TagRepresentable {
        var name: String = "SystemVariables"
        var description: String? = "Manage system variables."
    }
}
```

## Define operations

An operation combines parameters, request bodies, responses, tags, and an operation ID. In Feather applications, `BearerProtectedOperation` adds the standard bearer authentication requirement:

```swift
import FeatherOpenAPI
import FeatherOpenAPIGenerator

extension Variable {
    enum List {}
    enum Get {}
    enum Post {}
    enum Delete {}
    enum Query {}
    enum ID {}
    enum Put {}
    enum Patch {}

    public protocol Operation: OperationRepresentable {}
    public protocol IdentifiableOperation: Operation {}
}

extension Variable.Operation {
    public var tags: [any TagRepresentable] { [Variable.Tag()] }
}

extension Variable.IdentifiableOperation {
    public var parameters: [any ParameterRepresentable] {
        [Variable.Parameters.ID().reference()]
    }
}

extension Variable.Get {
    struct Operation: Variable.IdentifiableOperation,
        BearerProtectedOperation
    {
        var operationId: String? { "systemVariableGet" }

        var responseMap: ResponseMap {
            [
                200: Response().reference(),
                404: DefaultHTTPErrorContentResponse(
                    description: "SystemVariable not found"
                ),
            ]
        }
    }
}
```

The corresponding create operation adds a request body and reports `201` plus the resource-specific errors:

```swift
extension Variable.Post {
    struct Operation: Variable.Operation, BearerProtectedOperation {
        var operationId: String? { "systemVariablePost" }

        var requestBody: (any RequestBodyRepresentable)? {
            RequestBody().reference()
        }

        var responseMap: ResponseMap {
            [
                201: Variable.Get.Response().reference(),
                400: DefaultHTTPErrorContentResponse(
                    description: "Bad Request"
                ),
                409: DefaultHTTPErrorContentResponse(
                    description: "Variable key already exists"
                ),
            ]
        }
    }
}
```

The generated System Module document applies the same pattern to list, delete, put, patch, and query operations. Each operation gets a stable operation ID, the `SystemVariables` tag, bearer security, and the responses appropriate to that action.

## Assemble path items

A path item groups operations for one URL. The collection endpoint supports list, create, and delete:

```swift
extension Variable {
    struct PathItem: PathItemRepresentable {
        var get: (any OperationRepresentable)? { List.Operation() }
        var post: (any OperationRepresentable)? { Post.Operation() }
        var delete: (any OperationRepresentable)? { Delete.Operation() }
    }
}
```

The identifiable path groups the operations that require a variable ID:

```swift
extension Variable.ID {
    struct PathItem: PathItemRepresentable {
        var get: (any OperationRepresentable)? { Variable.Get.Operation() }
        var put: (any OperationRepresentable)? { Variable.Put.Operation() }
        var patch: (any OperationRepresentable)? {
            Variable.Patch.Operation()
        }
    }
}
```

## Assemble the resource paths

The path collection is the resource's public URL map. The `Variable` generator adds a collection path, a query path, and an identifiable resource path:

```swift
extension Variable {
    struct PathCollection: PathCollectionRepresentable {
        var pathMap: PathMap {
            [
                "api/v1/admin/system/variables": PathItem(),
                "api/v1/admin/system/variables/query": Query.PathItem(),
                "api/v1/admin/system/variables/{systemVariableId}":
                    ID.PathItem(),
            ]
        }
    }
}
```

The resulting endpoints are:

- `GET /api/v1/admin/system/variables` — list variables
- `POST /api/v1/admin/system/variables` — create a variable
- `DELETE /api/v1/admin/system/variables` — delete variables
- `POST /api/v1/admin/system/variables/query` — search variables
- `GET /api/v1/admin/system/variables/{systemVariableId}` — get one variable
- `PUT /api/v1/admin/system/variables/{systemVariableId}` — replace a variable
- `PATCH /api/v1/admin/system/variables/{systemVariableId}` — partially update a variable

## Compose the document

The application-level document merges path collections from the System Module and other modules. Components are collected from those same path collections:

```swift
import FeatherOpenAPI
import FeatherOpenAPIGenerator

struct Info: InfoRepresentable {
    var title: String { "System Admin API" }
    var version: String { "0.1.0" }
}

struct Server: ServerRepresentable {
    var url: any LocationRepresentable {
        Location("http://127.0.0.1:8080/")
    }
}

struct PathCollection: PathCollectionRepresentable {
    var pathMap: PathMap {
        Variable.PathCollection().pathMap
    }
}

struct Document: DocumentRepresentable {
    var info: any OpenAPIInfoRepresentable = Info()
    var servers: [any OpenAPIServerRepresentable] = [Server()]

    private var collection: PathCollection {
        PathCollection()
    }

    var paths: PathMap {
        collection.pathMap
    }

    var components: OpenAPIComponentsRepresentable {
        collection.components
    }
}

let document = Document()
let openAPIDocument = document.openAPIDocument()
```

In the full Feather CMS generator, `PathCollection` merges `Permission`, `Variable`, and `Job` collections. A module can therefore contribute paths without owning the complete application document:

```swift
struct PathCollection: PathCollectionRepresentable {
    var pathMap: PathMap {
        var paths: PathMap = [:]
        paths.merge(Permission.PathCollection().pathMap) {
            current, _ in current
        }
        paths.merge(Variable.PathCollection().pathMap) {
            current, _ in current
        }
        paths.merge(Job.PathCollection().pathMap) { current, _ in current }
        return paths
    }
}
```

## Write the generated specification

The CMS generator serializes the document to YAML and writes multiple OpenAPI versions for downstream code generation:

```swift
import Foundation
import Yams

let document = Document()
let openAPIDocument = document.openAPIDocument()
let encoder = YAMLEncoder()
let output = URL(fileURLWithPath: "openapi", isDirectory: true)

try FileManager.default.createDirectory(
    at: output,
    withIntermediateDirectories: true
)

for version in ["", "@v3_1_0", "@v3_2_0"] {
    let value: String

    switch version {
    case "":
        value = try encoder.encode(openAPIDocument)
    case "@v3_1_0":
        value = try encoder.encode(
            openAPIDocument.convert(to: .v3_1_0)
        )
    default:
        value = try encoder.encode(
            openAPIDocument.convert(to: .v3_2_0)
        )
    }

    try value.write(
        to: output.appending(path: "system-admin\(version).yaml"),
        atomically: true,
        encoding: .utf8
    )
}
```

The generated YAML is then used to generate the strongly typed API surface in `Sources/APIs/Admin`. The generated `Operations.SystemVariableGet.Input`, `Operations.SystemVariableGet.Output`, and related types are the compile-time contract consumed by the server implementation.

## Generate the Swift API package

The generated YAML or JSON is the input for [Swift OpenAPI Generator](https://github.com/apple/swift-openapi-generator). It can generate the schemas, operation input and output types, a typed client, and server protocols from the same document. The Feather CMS modules keep this configuration in `openapi-generator-config.yml`:

```yaml
generate:
  - types
  - client
  - server
accessModifier: public
namingStrategy: idiomatic
```

Run the generator against the document written by the previous step:

```sh
swift-openapi-generator generate \
    --config scripts/openapi-generator-config.yml \
    --output-directory Sources/APIs/Admin \
    openapi/system-admin.yaml
```

The generated package is transport-independent. On the server, connect the generated server protocols to a transport such as [OpenAPIHummingbird](https://github.com/hummingbird-project/swift-openapi-hummingbird) or [OpenAPIVapor](https://github.com/vapor/swift-openapi-vapor). On the client, use [OpenAPIAsyncHTTPClient](https://github.com/swift-server/swift-openapi-async-http-client) for server-side Swift, or [OpenAPIURLSession](https://github.com/apple/swift-openapi-urlsession) for iOS, macOS, and other Foundation-based applications.

This separates the API contract from the HTTP transport: the generated types, client methods, and server operations stay the same while the application chooses the transport required by its runtime.

## The complete boundary

The System Module example keeps each OpenAPI concern in the right place:

- schema types describe the public JSON shape;
- request bodies and responses describe HTTP payloads;
- operations describe authentication, parameters, and status codes;
- path collections describe URLs and HTTP methods;
- the document composes all module paths and components;
- the generator writes the OpenAPI document;
- OpenAPI code generation produces the typed server/client contract.

This keeps the generated API contract synchronized with the implementation without making transport details responsible for document composition.
