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

Currently all targets of each external dependency are built and treated as dependencies of all targets of the dependent package. This works well for one target package but becomes unclear which targets are using which target of an external dependency.

Moreover user of a package may only be interested in few targets of a dependency instead of all the exposed targets. Currently there is no way to state this in `Package.swift`.

For e.g.: One would like to use the targets `libc`, `POSIX`, `Basic` of SwiftPM but don't want other targets to be built or exposed in their package.

## Proposed Solution

####1. Control over exposed targets:

We propose that all targets should by default be private/unexported with some sensible defaults. For all other cases authors should explicitly mark the targets they want to expose as exported/public.

To mark a target as exported/public we propose `PackageDescription`'s `Package` gains a `export` string array property which contains a list of exported targets.

We propose the following default behaviour:

1. Export the target with same name as the package name.
2. Using export will override the above behvaior and author controls which targets are exported.

When there is no target matching the package name and package does not contain any executable provide a warning which can be supressed by `export: []`.

The same behavior is also valid for executables to provide support for a package to build and use executables at runtime.

--

It should be noted that this behaviour cannot be enforced by the compiler right now and there is no way to stop symbols from other modules from leaking out. For e.g. there could be a type used in the public interface which belongs to a private target.

Dependencies of the public targets will also leak and can be imported since they'll become transitive dependency of some target.

Hopefully we can enforce this using compiler feature in future.

Swift compiler might gain support for package-level namespaces and access control in future to solve problems like module name collision i.e. two packages have modules with same name. At that point we will probably need to rethink the manifest file.

####2. Specify external target dependencies of a target:

We propose that enum `Target.Dependency` gains a new case `External(package: String, target: String)` to declare dependency on an external package's target. The enum would look like this after modification:

```swift
/// The description for an individual target or package dependency.
public enum Dependency {
    /// A dependency on a target in the same project.
    case Target(name: String)
    /// A dependency on a target in a external package.
    case External(package: String, target: String)
}
```
package here refers to the name of the package defined in the manifest file. This might get awkward if the package URL basename and package name are different but that should be okay as this can be considered as an advanced feature.

Note that the package name is not *really* needed (at least currently) because the target names has to be unique across the dependency graph but it keeps the manifest file cleaner i.e. which external package this external target belongs to.

An external package dependency declaration implicitly becomes dependency of each target in the package. We propose this behaviour should be retained but if a target dependency contains an `External` declaration then all other targets which wants to use that external dependency should explicitly state their dependency on that external package using `External`.

## Detailed Design

####1. Control over exposed targets:

Consider a package with following structure: 

```
├── Package.swift
└── Sources
    ├── FooLibrary
    │   └── Foo.swift
    ├── BarLibrary
    │   └── Bar.swift
    └── SampleCLI
        └── main.swift
```

The manifest with a public target could look like:

```swift
import PackageDescription

let package = Package(
   name: "FooLibrary",
   targets: [
       Target(name: "FooLibrary"),
       Target(name: "SampleCLI", dependencies: ["FooLibrary"]),
   ])
```

When this package is used as a dependency only `FooLibrary` is built and is importable because `FooLibrary` target matches the package name.

In case author wants to export `BarLibrary` too, `export: []` can be used:

```swift
import PackageDescription

let package = Package(
   name: "FooLibrary",
   targets: [
       Target(name: "FooLibrary"),
       Target(name: "BarLibrary"),
       Target(name: "SampleCLI", dependencies: ["FooLibrary"]),
   ],
   export: ["FooLibrary", "BarLibrary"]
)
```

In this case `FooLibrary` and `BarLibrary` is built and exposed to the user package.

####2. Specify external target dependencies of a target:

Consider a dependency with following manifest file:

```swift
import PackageDescription

let package = Package(
   name: "FooLibrary",
   targets: [
       Target(name: "Foo"),
       Target(name: "Bar", dependencies: ["Foo"]),
       Target(name: "Baz"),
   ])
```

To get only the `Bar` target from the above package, following manifest 
could be written:

```swift
import PackageDescription

let package = Package(
   name: "BarUser",
   targets: [
        Target(
               name: "BarUser", 
               dependencies: [.External(package: "FooLibrary", target: "Bar")])
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

Package authors with targets not matching the package name will need to mention the targets they want to expose in the `export: []`.

####2. Specify external target dependencies of a target:

None as all the public targets will still be dependencies to the overall package when `External` is not used.

## Alternatives Considered

####1. Control over exposed targets:

To mark a target as exported/public we considered adding a `flags` property to `PackageDescription`'s `Target` which would be a `Set` of the following `Flag` enum declared inside `Target` class:

```swift
public enum Flag {
    /// Makes the target public or "exported" for other packages to use.
    case public
}
```

The `Flag` enum will be flexible in case we need to add more attributes in future as opposed to a boolean property to mark the public nature of the target.

But manifest might become over complicated for most simple or single module packages so it makes sense to provide the above mentioned defaults. We think this option is less clearer for the proposed default behavior.

`public` was considered instead of `export` but `export` seems better for following reasons:

* This feature is really about what targets are exposed (available to be built and depended on) to consuming packages, as the proposal notes it isn't changing the visibility at all.
* This becomes more obvious when we consider that we should be able to "export" (make available to be built and depended on) executable targets

## Acknowledgements

Daniel Dunbar
