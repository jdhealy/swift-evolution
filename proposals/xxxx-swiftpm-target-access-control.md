# SwiftPM Target Access Control

* Proposal: [SE-XXXX](https://github.com/apple/swift-evolution/blob/master/proposals/xxxx-swiftpm-target-access-control.md)
* Author: [Ankit Aggarwal](https://github.com/aciidb0mb3r)
* Status: **In Discussion**
* Review manager: TBD


## Introduction

This proposal aims to address two issues:

1. Control over the targets exposed (and built) when a SwiftPM package is used as a dependency.

2. Import (and build) selected targets of a dependency.

[swift-evolution thread]()

## Motivation

#### 1. Control over exposed targets:

SwiftPM allows multiple targets (or modules) inside a package. Packages usually contain sample usage or example targets which are useful during development or testing of the package but are redundant when the package is used as a dependency. This increases compile time for the user of the package.

As a concrete example: Vapor has a target called [`Development`](https://github.com/qutheory/vapor/tree/master/Sources/Development).

#### 2. Import selected targets:

Sometimes user of a package is only interested in few targets of a dependency instead of all the targets. Currently there is no way to state this in `Package.swift` and all the targets are implicitly built and exposed to the user package.

For e.g.: I would like to use the targets `libc`, `POSIX`, `Basic` of SwiftPM but don't want other targets to be built or exposed in my package.

## Proposed Solution

####1. Control over exposed targets:

I propose that package authors be able mark the targets they don't want to be exposed as `private` i.e. the `private` targets will be built when that package is root package but not when the package is used as a dependency.

To mark a target as `private` I propose `PackageDescription`'s `Target` gains a `isPrivate` boolean property which defaults to `false`.

####2. Import selected targets:

I propose that package user be able to specify the targets they want to import into their package.

To specify the targets to be import I propose to add an optional string array property `targets` in `PackageDescription`'s `Package.Dependency` which defaults to `nil` i.e. all targets.

Instead of an optional string array property an enum can also be used:

```swift
enum ImportedTargets {
    case allTargets // Import all the targets, default value.
    case targets([String]) // Import only these targets.
}
```

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

The manifest with `private` target could look like:

```swift
import PackageDescription

let package = Package(
   name: "FooLibrary",
   targets: [
       Target(name: "FooLibrary"),
       Target(name: "SampleCLI", isPrivate: true),
   ])
```

When this package is used as a dependency only `FooLibrary` is built and is importable.

Targets can have other targets as dependency inside a package. A `private` target should only be a dependency to other private targets. For e.g. A manifest like this should result in a build failure.

```swift
import PackageDescription

let package = Package(
   name: "FooLibrary",
   targets: [
       Target(name: "FooCore", isPrivate: true),
       Target(name: "FooLibrary", dependencies: ["FooCore"]), // Error FooCore is private.
       Target(name: "SampleCLI", dependencies: ["FooCore"], isPrivate: true), // Not an error because SampleCLI is private.
   ])
```
Error: `FooCore` is a private target, it cannot be a dependency to the public target `FooLibrary`.

####2. Import selected targets:

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
   name: "FooUser",
   dependencies: [
       .Package(
           url: "../FooLibrary", 
           majorVersion: 1, 
           targets: ["Bar"])
   ])
```
Note: In this case since Bar depends on Foo, Foo will be also be implicitly built and be available.

Any target mentioned in `targets` and not present in the package should result in build failure.

## Impact on Existing Code

There will be no impact on existing code as these features are additive.

## Alternatives Considered

None at this time.