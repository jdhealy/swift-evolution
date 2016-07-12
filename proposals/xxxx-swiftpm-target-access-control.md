# SwiftPM Target Access Control

* Proposal: [SE-XXXX](https://github.com/apple/swift-evolution/blob/master/proposals/xxxx-swiftpm-target-access-control.md)
* Author: [Ankit Aggarwal](https://github.com/aciidb0mb3r)
* Status: **In Discussion**
* Review manager: TBD


## Introduction

This proposal aims to address two issues:

1. Control over the targets exposed (and built) when a SwiftPM package is used as a dependency i.e. the targets which are exported and can be used in other packages.

2. Specify external target dependencies of a target.

[swift-evolution thread](https://lists.swift.org/pipermail/swift-build-dev/Week-of-Mon-20160704/000531.html)

## Motivation

#### 1. Control over exposed targets:

SwiftPM allows multiple targets (or modules) inside a package. Most of the time package author will want to provide one (or more) stable public/exported target which should be utilised by other packages. We should actively discourage use of targets which are not meant to be imported by other packages.

Additionally packages usually contain sample usage or example targets which are useful during development or testing of the package but are redundant when the package is used as a dependency. This increases compilation time for the user of the package which can be avoided.

As a concrete example: Vapor has a target called [`Development`](https://github.com/qutheory/vapor/tree/master/Sources/Development).

#### 2. Specify external target dependencies of a target:

Currently all the targets of an external dependency are implicitly built and exposed to the user package. This works well for one target package but becomes unclear which targets are using which target of an external dependency.

Moreover user of a package may only be interested in few targets of a dependency instead of all the exposed targets. Currently there is no way to state this in `Package.swift`.

For e.g.: One would like to use the targets `libc`, `POSIX`, `Basic` of SwiftPM but don't want other targets to be built or exposed in their package.

## Proposed Solution

####1. Control over exposed targets:

I propose that all targets should by default be private/unexported. Authors should explicitly mark the targets they want to expose as exported/public.

To mark a target as exported/public I propose `PackageDescription`'s `Target` gains a `flags` property which would be a `Set` of the following `Flag` enum declared inside `Target` class:

```swift
public enum Flag {
    /// Makes the target public or "exported" for other packages to use.
    case public
}
```

The `Flag` enum will be flexible in case we need to add more attributes in future as opposed to a boolean property to mark the public nature of the target.

`exported` is also a choice instead of `public` which matches the semantics here. However `public` is equally clear in current context.

We can keep some obvious defaults for targets which can be implicitly public for e.g. 

1. Package has only one target.
2. Target with same name as package.

Or have all targets be public (the current behaviour) until some target uses the public flag assuming full control over all the exported target. This has an advantage that only larger projects which cares about this need to maintain it.

However I believe private by default and explicit public declaration is the right way to go here to avoid the misuse of packages/targets which are not intended to act as a dependency and the public targets will become obvious (and documented) in the manifest file.

--

It should be noted that this behaviour cannot be enforced by the compiler right now and there is no way to stop symbols from other modules from leaking out. For e.g. there could be a type used in the public interface which belongs to a private target.

Dependencies of the public targets will also leak and can be imported since they'll become transitive dependency of some target.

Hopefully we can enforce this using compiler feature in future.

Swift compiler might gain support for package-level namespaces and access control in future to solve problems like module name collision i.e. two packages have modules with same name. At that point we will probably need to rethink the manifest file.

####2. Specify external target dependencies of a target:

I propose that enum `Target.Dependency` gains a new case `External(package: String, target: String)` to declare dependency on an external package's target. The enum would look like this after modification:

```swift
/// The description for an individual target or package dependency.
public enum Dependency {
    /// A dependency on a target in the same project.
    case Target(name: String)
    /// A dependency on a target in a external package.
    case External(package: String, target: String)
}
```

Note that the package name is not *really* needed (at least currently) because the target names has to be unique across the dependency graph but it keeps the manifest file cleaner i.e. which external package this external target belongs to.

An external package dependency declaration implicitly becomes dependency of each target in the package. I propose this behaviour should be retained but if a target dependency contains an `External` declaration then all other targets which wants to use that external dependency should explicitly state their dependency on that external package using `External`.

## Detailed Design

####1. Control over exposed targets:

Consider a package with following structure: 

```
├── Package.swift
└── Sources
    ├── FooLibrary
    │   └── Foo.swift
    └── SampleCLI
        └── main.swift
```

The manifest with a public target could look like:

```swift
import PackageDescription

let package = Package(
   name: "FooLibrary",
   targets: [
       Target(name: "FooLibrary", flags: [.public]),
       Target(name: "SampleCLI", dependencies: ["FooLibrary"]),
   ])
```

When this package is used as a dependency only `FooLibrary` is built and is importable.

####2. Specify external target dependencies of a target:

Consider a dependency with following manifest file:

```swift
import PackageDescription

let package = Package(
   name: "FooLibrary",
   targets: [
       Target(name: "Foo"),
       Target(name: "Bar", dependencies: ["Foo"], flags: [.public]),
       Target(name: "Baz", flags: [.public]),
   ])
```

To get only the `Bar` target from the above package, following manifest 
could be written:

```swift
import PackageDescription

let package = Package(
   name: "BarUser",
   targets: [
        Target(name: "BarUser", 
               dependencies: [.External(package: "FooLibrary", target: "Bar")
               ])
   ],
   dependencies: [
       .Package(
           url: "../FooLibrary", 
           majorVersion: 1)
   ])
```

Note: In this case since `Bar` depends on `Foo`, `Foo` will be also be implicitly built but `Baz` need not be compiled at all.

Also Note: If the external dependency is not declared then both `Bar` and `Baz` will be available to `BarUser`.

## Impact on Existing Code

####1. Control over exposed targets:

All targets will become private by default so package authors will need to mark the targets they want to expose as public.

####2. Specify external target dependencies of a target:

None as all the public targets will still be dependencies to the overall package when `External` is not used.

## Alternatives Considered

None at this time.