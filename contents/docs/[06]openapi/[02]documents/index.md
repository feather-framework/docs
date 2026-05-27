---
category: "openapi"
title: "Documents"
description: "Learn how to describe an OpenAPI document using Feather OpenAPI."
order: 2
---

# Documents
---

Feather OpenAPI lets you describe an OpenAPI document as a set of Swift values.

This guide uses a small todo API because it is easy to understand. The same structure can be used for larger production APIs.

## Installation

Add the package:

```swift
.package(
    url: "https://github.com/feather-framework/feather-openapi",
    exact: "1.0.0-beta.7"
),
```

Then add the product:

```swift
.product(
    name: "FeatherOpenAPI",
    package: "feather-openapi"
),
```

## Define document info

```swift
import FeatherOpenAPI

struct TodoAPIInfo: InfoRepresentable {
    var title: String { "Todo API" }
    var version: String { "1.0.0" }
}
```

The info section gives the API a name and version.

## Define a server

```swift
struct TodoServer: ServerRepresentable {
    var url: any LocationRepresentable {
        "https://api.example.com"
    }
}
```

Feather OpenAPI can use simple values for locations. A string can represent a URL location.

## Define schemas

Schemas describe JSON objects and fields.

```swift
struct TodoIDField: IntSchemaRepresentable {
    var example: Int? { 1 }
}

struct TodoTitleField: StringSchemaRepresentable {
    var example: String? { "Buy milk" }
}

struct TodoObject: ObjectSchemaRepresentable {
    var propertyMap: SchemaMap {
        [
            "id": TodoIDField().reference(),
            "title": TodoTitleField(),
        ]
    }
}
```

Using small schema types makes the API document easier to reuse and maintain.

## Define request and response types

```swift
struct TodoCreateRequestBody: RequestBodyRepresentable {
    var contentMap: ContentMap {
        [
            .json: Content(TodoObject().reference())
        ]
    }
}

struct TodoResponse: JSONResponseRepresentable {
    var description: String = "Todo response"
    var schema = TodoObject().reference()
}
```

The request body says that the endpoint accepts JSON. The response says that it returns JSON in the shape of `TodoObject`.

## Define an operation

```swift
struct TodoCreateOperation: OperationRepresentable {
    var requestBody: RequestBodyRepresentable? {
        TodoCreateRequestBody().reference()
    }

    var responseMap: ResponseMap {
        [
            200: TodoResponse().reference()
        ]
    }
}
```

An operation represents one HTTP action, such as `GET /todos` or `POST /todos`.

## Define paths

```swift
struct TodoPathItems: PathItemRepresentable {
    var post: OperationRepresentable? {
        TodoCreateOperation()
    }
}

struct TodoPaths: PathCollectionRepresentable {
    var pathMap: PathMap {
        [
            "todos": TodoPathItems()
        ]
    }
}
```

This adds a `POST /todos` operation to the document.

## Build the document

```swift
struct TodoDocument: DocumentRepresentable {
    var info: OpenAPIInfoRepresentable = TodoAPIInfo()
    var servers: [any OpenAPIServerRepresentable] = [
        TodoServer()
    ]
    var paths: PathMap = TodoPaths().pathMap
    var components: OpenAPIComponentsRepresentable = TodoPaths().components
}

let document = TodoDocument()
let openAPIDocument = document.openAPIDocument()
```

From here, the document can be encoded as JSON or YAML using the tools in your OpenAPI workflow.

## Tags and security

Feather OpenAPI also supports tags and security requirements.

```swift
struct TodoTag: TagRepresentable {
    var name: String = "Todos"
    var description: String? = "Manage todo items."
}

struct BearerTokenScheme: SecuritySchemeRepresentable {
    var type: OpenAPI.SecurityScheme.SecurityType {
        .http(
            scheme: "bearer",
            bearerFormat: "token"
        )
    }
}
```

The library deduplicates repeated tags and security requirements when building the document. That keeps larger specifications cleaner when multiple operations reuse the same definitions.
