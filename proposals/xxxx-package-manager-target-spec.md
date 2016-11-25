# Package Manager Custom Layout and Targets


* Proposal: [SE-XXXX](xxxx-package-manager.md)
* Author: [Ankit Aggarwal](https://github.com/aciidb0mb3r)
* Review Manager: TBD
* Status: Discussion

## Introduction

This proposal adds enhancements to `Package.swift` to allow custom layouts and target definitions.

## Motivation

The package manager currently uses a convention system to infer various targets like Swift, C family, tests etc.
This works well for most of the packages but sometimes more flexibility and customization is required,
for example when someone wants to add package manager support to a large project which would be difficult to re-arrange. 
This becomes especially useful when porting an existing C libary to be compatible with package manager, as these libraries have no universal convention. 

## Proposed solution

We propose to extend the `PackageDescription` module (i.e. the manifest file) to allow custom layouts and add additional customization properties to define the targets.

## Detailed Design

### Sources directory:

We propose to introduce a way to define sources directory in the manifest file, using an enum `PackageSourcesDirectory`:

```swift
enum PackageSourcesDirectory {
    /// Automatically infer the sources directory in this order: Sources, Src, src, package root.
    case automatic

    /// The sources directory is package root.
    case packageRoot

    /// The sources are present at this directory, this must be a relative path from the package root.
    case custom(path: String)
}
```

We propose to introduce a property `sourcesDirectory` in `Package` class of type `PackageSourcesDirectory`, with default value `.automatic`.

As an example, consider the following layout and its manifest file:

```
├── MySources
│   ├── Bar
│   ├── Baz
│   └── Foo
└── Tests
    └── FooTests
```

```swift
let package = Package(
    name: "MyPackage",
    sourcesDirectory: .custom(path: "MySources"),
    targets: [
        Target(
            name: "Foo", 
            dependencies: ["Bar", "Baz"]
        ),
    ]
)
```

Using the existing conventions four targets will be created:

* `Bar`, `Baz`: Swift targets.
* `Foo`: Swift target which depends on `Bar` and `Baz`.
* `FooTests`: Swift test target which depends on `Foo`.

### Custom Layouts:

We propose to add the following properties to `PackageDescription.Target` class:

```swift
public final class Target {
    
    /// Path to the modulemap file. Only valid for C family library targets.
    var moduleMap: String?

    /// The preprocessor definitions.
    var defines: [String]

    /// Look for the sources defined by these patterns.
    var sources: [String]

    /// The header search paths. Only valid for C family library targets.
    var headers: [String]
    
    /// The excluded sources patterns. 
    ///
    /// These have higher preference than `sources` property, i.e. if some file is picked by both sources and excludes pattern, it will be excluded.
    var excludes: [String]

    /// Explicity mark the target as executable or a library. Package manager will try to guess the value based on presence of `main.extension` file.
    ///
    /// Note: It is required for Swift executable targets to have a 'main.swift' file containing top-level code.
    var executable: Bool?
}
```

The following pattern will be supported by `headers`, `sources` and `excludes`:

| Pattern  | Description                                   |
| -------- |:--------------------------------------------- | 
| *        | matches all files in the directory            |
| **       | recusively matches all files in the directory | 


For example, consider the following structure:

```
└── somedir
    ├── bar.swift
    ├── baz.swift
    └── foo
        └── baz.swift
```

| Pattern             | Result                                                      |
| ------------------- |:------------------------------------------------------------| 
| `somedir/bar.swift` | somedir/bar.swift                                           |
| `somedir/*`         | somedir/bar.swift, somedir/baz.swift                        |
| `somedir/foo/*`     | somedir/foo/baz.swift                                       |
| `somedir/**`        | somedir/bar.swift, somedir/baz.swift, somedir/foo/baz.swift |

_Note: It might be useful to extend this pattern to match unix glob in future._

### Examples:

These would be manifest of some C packages without changing their file structure or source files:

* LibYAML

```swift
let packages = Package(
    name: "LibYAML",
    targets: [
        Target(
            name: "libyaml",
            sources: [
                "src/*",
            ],
            defines: [
                "YAML_VERSION_MAJOR=0",
                "YAML_VERSION_MINOR=1",
                "YAML_VERSION_PATCH=7",
                "YAML_VERSION_STRING=\"0.1.7\"",
            ]
        )
    ]
)
```

Since there is a include directory and we have not specified any header paths, a module map will be automatically generated.

* Node.js http-parser

```swift
let packages = Package(
    name: "http-parser",
    targets: [
        Target(
            name: "http-parser",
            moduleMap: "module.modulemap",
            sources: [
                "http_parser.c",
            ]
        )
    ]
)
```

```C
module httpparser {
    header "http_parser.h"
    export *
}
```

* libressl

```swift
let packages = Package(
    name: "libressl",
    targets: [
        Target(
            name: "libressl",
            moduleMap: "include/module.modulemap",
            headerPaths: [
                "include",
                "include/compat",
                "crypto",
                "crypto/evp",
                "crypto/asn1",
                "crypto/modes",
            ],
            sources: [
                "ssl/*",
                "tls/*",
                "crypto/**",
            ],
            defines: [
                "OPENSSL_NO_HW_PADLOCK",
                "HAVE_STRCASECMP",
                "HAVE_STRLCPY",
                "HAVE_STRLCAT",
                "HAVE_STRNDUP",
                "HAVE_STRNLEN",
                "HAVE_STRSEP",
                "HAVE_TIMINGSAFE_BCMP",
                "HAVE_MEMMEM",
                "LIBRESSL_INTERNAL",
            ],
            exclude: [
                /* Exclude windows files */
                "crypto/bio/b_win.c",
                "crypto/ui/ui_openssl_win.c",
                "crypto/des/ncbc_enc.c",
                "crypto/poly1305/poly1305-donna.c",

                /* Exclude compat files already present on macOS. */
                "crypto/compat/bsd-asprintf.c",
                "crypto/compat/explicit_bzero_win.c",
                "crypto/compat/inet_pton.c",
                "crypto/compat/posix_win.c",
                "crypto/compat/strlcat.c",
                "crypto/compat/strlcpy.c",
                "crypto/compat/getentropy*",
            ]
        )
    ]
)
```

```C
module libressl {
    exclude header "openssl/cms.h"
    umbrella header "openssl/openssl.h"
    export *
    link "Clibressl"
}
```

* swift-build-tool

```swift
let packages = Package(
    name: "llbuild",
    targets: [
        Target(
            name: "swift-build-tool",
            sources: [
                "lib/Basic/*",
                "lib/llvm/Support/*",
                "lib/Core/*",
                "lib/BuildSystem/*",
                "products/swift-build-tool/swift-build-tool.cpp",
            ],
            executable: true
        )
    ]
)
```

## Impact on existing code

There will no impact on existing code.

## Alternative considered

None at this point.
