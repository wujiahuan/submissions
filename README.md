# Submissions 📩
[![Swift Version](https://img.shields.io/badge/Swift-4.1-brightgreen.svg)](http://swift.org)
[![Vapor Version](https://img.shields.io/badge/Vapor-3-30B6FC.svg)](http://vapor.codes)
[![CircleCI](https://circleci.com/gh/nodes-vapor/submissions/tree/master.svg?style=svg)](https://circleci.com/gh/nodes-vapor/submissions/tree/master)
[![codebeat badge](https://codebeat.co/badges/b9c894d6-8c6a-4a07-bfd5-29db898c8dfe)](https://codebeat.co/projects/github-com-nodes-vapor-submissions-master)
[![codecov](https://codecov.io/gh/nodes-vapor/submissions/branch/master/graph/badge.svg)](https://codecov.io/gh/nodes-vapor/submissions)
[![Readme Score](http://readme-score-api.herokuapp.com/score.svg?url=https://github.com/nodes-vapor/submissions)](http://clayallsopp.github.io/readme-score?url=https://github.com/nodes-vapor/submissions)
[![GitHub license](https://img.shields.io/badge/license-MIT-blue.svg)](https://raw.githubusercontent.com/nodes-vapor/reset/master/LICENSE)

# Installation

Update your `Package.swift` file.
```swift
.package(url: "https://github.com/nodes-vapor/submissions.git", from: "1.0.0-beta")
```

## Introduction

Submissions was written to minimize the amount of boilerplate needed to write the common tasks of rendering forms and processing and validating data from POST and PATCH requests. Submissions makes it easy to present detailed validation errors for web users as well as API consumers.

## Getting started 🚀

### Making a Submittable model

Let's take a simple Todo class as an example.

```swift
final class Todo: Model, Content, Parameter {
    var title: String

    init(id: Int? = nil, title: String) {
        self.id = id
        self.title = title
    }
}
```

Let's conform our model to `Submittable`. This means we need to associate our model with a type that can be used to validate requests to create or update instances of our model.

```swift 
import Submissions

extension Todo: Submittable {
    struct Submission: SubmissionType {
        let title: String?
    }
}
```

This type is decoded from the request. In order for missing values to result in validation errors (instead of a decoding error) all properties need to be optional. We'll rely on our validation (see below) to catch any missing fields.

Next we'll see how we can associate validators (and labels) with our fields. 

```swift
extension Todo.Submission {
    func fieldEntries() throws -> [FieldEntry] {
        return try [
            makeFieldEntry(keyPath: \.title, label: "Title", validators: [.count(5...)], isOptional: false)
        ]
    }

    ...
```

Using the `makeFieldEntry` helper function we can use type-safe `KeyPath`s to refer to the values in our `Submission` struct. In addition to the list of validators it is possible to supply a `validate` closure that performs async validation. This can be useful when validation requires a call to the database for instance. See the API docs for further information.

The submission type is also used by tags to render labels and any existing values for input fields in a form. Therefore we'll need to provide a way to create a `Submission` value from a todo, or `nil` in case we're creating a new one.

```swift
    // Todo.Submission, continued ...

    init(_ todo: Todo?) {
        title = todo?.title
    }
}
```

After validation the `Submission` value can be used to update our model.

```swift
// Todo: Submittable, continued ...

func update(_ submission: Submission) {
    if let title = submission.title {
        self.title = title
    }
}
```

Creating a new instance of our `Todo` model works slightly differently. We'll need to define another type that can be used to create our models.

```swift
    // extension Todo: Submittable continued

    struct Create: Decodable {
        let title: String
    }

    convenience init(_ create: Create) {
        self.init(id: nil, title: create.title)
    }
```

The way this works is that after decoding and validating the `Submission` value, a value of the `Create` type will be decoded from the same request. It is our duty to make sure that all non-optional properties in the `Create` type have corresponding validators in the `Submission` type. This prevents that errors will be thrown during decoding when fields are missing.

### Validating API requests

Let's create a controller for our Todo related API routes.

```swift
final class APITodoController {
    ...
}
```

in your `routes.swift` add:
```swift
// in func routes
...
let api = router.grouped("api")
let apiTodoController = APITodoController()
// add api routes here
...
```

We'll add the a create route to our `APITodoController:

```swift
func create(req: Request) throws -> Future<Either<Todo, SubmissionValidationError>> {
    return try req.content.decode(Todo.Submission.self)
        .createValid(on: req)
        .save(on: req)
        .promoteErrors()
}
```

and register it as a POST request in `routes.swift`:

```swift
api.post("todos", use: apiTodoController.create)
```

In the route we decode our `Submission` value, we validate it and create a Todo item (using `createValid`) before we save it. With `promoteErrors`, in combination with the return type `Future<Either<Todo, SubmissionValidationError>>`, we can "promote" validation errors to values meaning we can create a proper response out of them. Since both `Todo` and `SubmissionValidationError` conform to `ResponseEncodable`, so does `Either` through the power of conditional conformance. In case of a validation error the response will be:

```json
{
  "error": true,
  "validationErrors": {
    "title": [
      "data is not larger than 5"
    ]
  },
  "reason": "One or more fields failed to pass validation."
}
```

Updating an existing instance follows a similar path. Let's add the following function to our APITodoController.

```swift
func update(req: Request) throws -> Future<Either<Todo, SubmissionValidationError>> {
    return try req.parameters.next(Todo.self)
        .updateValid(on: req)
        .save(on: req)
        .promoteErrors()
}
```

and register it as a PATCH request in `routes.swift`:

```swift
api.patch("todos", Todo.parameter, use: apiTodoController.update)
```

### Validating HTML form requests

> The following examples assume you have your Leaf file (edit.leaf) in a folder called "Todo" under "Resources". 

#### Leaf Tags

When building your HTML form using Leaf you can add inputs for your model's fields like so:

```
#textgroup("title")
```

This will render a form group with an input and any errors stored in the field cache for the "title" field. This produces the following HTML (in case of a new Todo instance without a value, and no errors):

```html
<div class="form-group">
    <label class="control-label" for="title">Title</label>
    <input type="text" name="title" value="">
</div>
```

> Note: Currently only textgroup is supported

Before we can use the tag we have to register it in `configure.swift`. We'll use a helper function from the [`Sugar`](https://github.com/nodes-vapor/sugar/tree/vapor-3) package. 

```swift
// in configure.swift
...
import Sugar
...

// in func configure
...

    services.register { _ -> LeafTagConfig in
        var tags = LeafTagConfig.default()
        tags.use(SubmissionsProvider.tags)
        return tags
    }

...
```

#### Rendering the forms

Now we'll create a controller for the frontend routes.

```swift
final class FrontendTodoController {
    ...
}
```

in your `routes.swift` we'll add:
```swift
// in func routes
...
let frontendTodoController = FrontendTodoController()
// add frontend routes here
...
```

An empty form can be created by populating the fields using the `Submittable` type before rendering the view.

```swift
func renderCreate(req: Request) throws -> Future<View> {
    try req.populateFields(Todo.self)
    return try req.view().render("Todo/edit")
}
```

and in `routes.swift` we'll add:
```swift
router.get ("todos/create", use: frontendTodoController.renderCreate)
```

In order to populate the field with the values of an existing entity we need to first load our entity and put its values in the field cache like so.

```swift
func renderEdit(req: Request) throws -> Future<View> {
    return try req.parameters.next(Todo.self)
        .populateFields(on: req)
        .flatMap { _ in
            try req.view().render("Todo/edit")
        }
}
```

In `routes.swift`:
```swift
router.get ("todos", Todo.parameter, "edit", use: frontendTodoController.renderEdit)
```

#### Validating and storing the data

Creating a new `Todo` is very similar to how we do in the API routes except that now we'll redirect on success and handle the error a bit differently (see below).

```swift
func create(req: Request) throws -> Future<Response> {
    return try req.content.decode(Todo.Submission.self)
        .createValid(on: req)
        .save(on: req)
        .transform(to: req.redirect(to: "/todos"))
        .catchFlatMap(handleCreateOrUpdateError(on: req))
}
```

and in `routes.swift` we'll add:

```swift
router.post("todos/create", use: frontendTodoController.create)
```

Updating should now also look familiar.

```swift
func update(req: Request) throws -> Future<Response> {
    return try req.parameters.next(Todo.self)
        .updateValid(on: req)
        .save(on: req)
        .transform(to: req.redirect(to: "/todos"))
        .catchFlatMap(handleCreateOrUpdateError(on: req))
}
```

In `routes.swift`:
```swift
router.post("todos", Todo.parameter, "edit", use: frontendTodoController.update)
```

One way to deal with errors is to render the edit view again which will now show all validation errors for all fields.

```swift
func handleCreateOrUpdateError(on req: Request) -> (Error) throws -> Future<Response> {
    return { _ in
        try req
            .view().render("Todo/edit")
            .flatMap { view in
                view.encode(for: req)
            }
    }
}
```

## 🏆 Credits

This package is developed and maintained by the Vapor team at [Nodes](https://www.nodesagency.com).
The package owner for this project is [Siemen](https://github.com/siemensikkema).

## 📄 License

This package is open-sourced software licensed under the [MIT license](http://opensource.org/licenses/MIT).
