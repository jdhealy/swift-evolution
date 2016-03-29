# SwiftPM: Adding development package as a dependency

* Proposal: [SE-NNNN](https://github.com/apple/swift-evolution/blob/master/proposals/NNNN-name.md)
* Author(s): [Ankit Aggarwal](https://github.com/aciidb0mb3r)
* Status: **Awaiting review**
* Review manager: TBD

## Introduction

I propose to enable SwiftPM to use a package that is still under development as a dependency for another package during testing and development.

Swift-evolution thread: [link](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20160328/013710.html)

## Motivation

During development of libraries, developers commonly want to try out their module package
as a dependency. This emulates the typical library use-case and is currently not possible
in SwiftPM without first checking in and tagging that library. These extra steps, while
reasonable for an already-built package, are an unnecessary burden for a package that remains
in development. 

Forcing the user to modify the library package inside `Packages` to continue development or
continuously reclone the package after recommiting and retagging strain the process of
building the library in the first place.

## Detail Design

Under this proposal, the *root* package will be allowed to specify a `DevPackage` dependency.
This dependency will *not* clone the package inside `Packages/` or require the dependency to be 
under version control. This will free the developer to continue iterative testing, expansion,
and enhancements without being tied to the current dependency system.

This approach limits `DevPackage` dependencies to local file systems. Remote repositories
cannot be used with this keyword. 

The following example demonstrate how to include a `DevPackage` while building a swift package.
In this example, two `DevPackage`s are specified (MyLib and MyLib2)

```
$ swift build --dev-pkg=../MyLib --dev-pkg=../MyLib2
```

Under this design:

* `DevPackage` is limited strictly to the root package.
* A `DevPackage` is not copied inside `Packages/` and does not require version control.
* SwiftPM uses the `DevPackage`'s source directory for building, permitting in-place development on the local file system.
* SwiftPM disallows non-local `DevPackage` sources. To use a remote package, the developer must first clone a package and then specify the local path.
* There are no version specifications for `DevPackage`.
* In case of a collision due to other dependencies, the `DevPackage` will be preferred.

## Impact on existing code

This proposal does not impact existing code.

## Alternatives considered

I propose two possible alternatives to this problem:

1. Create a executable target within the library package for development testing.
2. Use XCTest to test the library.

Both alternate approaches permit testing a library module but they will not simulate a full SwiftPM package.

Another alternative that was considered was adding the entry in Manifest file instead of commandline override:

The following example demonstrates what a manifest file would look like. In this example,
the `DevPackage` is specified using a local path and the `majorVersion` is used as is for 
this `DevPackage`.

```swift
import PackageDescription

let package = Package(
    name: "MyLibraryTester",
    dependencies: [
        .Package(url: "https://github.com/apple/example-package-fisheryates.git", majorVersion: 1),
        .DevPackage(localPath: "../MyAwesomeLibrary", majorVersion: 1),
    ]
)
```
However it has following issues:

1) You have to remember to modify the manifest file back at some point, and if you are iterating frequently this is tedious and error-prone

2) We don’t want any chance that `DevPackage` gets into the package graph and thus the ecosystem.

## Acknowledgements

Thanks to [Erica Sadun](https://github.com/erica) for inputs.
