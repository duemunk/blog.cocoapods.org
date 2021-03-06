---
layout: post
title:  "Pod Authors Guide to CocoaPods Frameworks"
author: marius
categories: cocoapods releases
date:       2014-12-25
---

TL;DR: _CocoaPods 0.36_ will bring the long-awaited support for Frameworks and Swift.
It isn't released and considered stable yet, but a beta is now available for everyone via `[sudo] gem install cocoapods --pre`.
Pod authors will especially want to try this version to make sure their pods will work with the upcoming release. This is because if a single dependency in a user's project requires being a framework, then your Pod will also become a framework.

<!-- more -->

## What is special about Frameworks integrated by CocoaPods?

With CocoaPods, Frameworks are mostly set up in way similar to how it is done via Xcode.
This is to make the entire integration inspectable, understandable and allows us to unleash the power of the whole existing toolchain.

Many of the tools only play nicely together in an Xcode environment where certain build variables are present.
Cocoa Touch Frameworks use Clang Modules, which are also required to import and link them to your Swift app.
Therefore the module map is included in the built framework bundle.

{% breaking_image /assets/blog_img/Frameworks-Developers/frameworks_vs_libraries.png %}

## Dynamic Frameworks vs. Static Libraries

So what's the difference between those two product types?

Dynamic Frameworks are bundles, which basically means that they are directories with the file suffix `.framework` and Finder treats them mostly like regular files. If you tap into a framework, you will see a common directory structure:

{% breaking_image /assets/blog_img/Frameworks-Developers/bananakit_framework.png %}

They bundle extra data besides a binary, which is in that case dynamically linkable and holds different slices for each architecture. Up to this point it is the same as a static library. However, a framework holds the following additional data:

* **The Public Headers** - These are stripped for application targets, as they are only important to distribute the framework as code for compilation. The public headers also include the generated headers for public Swift symbols, e.g. `Alamofire-Swift.h`.
* **A Code Signature For The Whole Contents** - This has to be (re-)calculated when embedding a framework into an application target, as the headers are stripped before.
* **Its Resources** - The resources used e.g. Images for UI components.
* **Hosted Dynamic Frameworks and Libraries** - This can be the case for so called Umbrella Frameworks provided by Apple. There are no use-cases where this happens with CocoaPods.
* **The Clang Module Map** and **the Swift modules** - These are mostly internal toolchain artifacts, which carry declarations about API / header visibility and module link-ability.
* **An Info.plist** - This specifies author, version and copyright information.


One caveat about bundling resources is, that until now we had to embed all resources into the application bundle. These resources were referenced programmatically by `[NSBundle mainBundle]`.

Pod authors were able to use `mainBundle` referencing to include resources the Pod brought into the app bundle.
But with frameworks, you have to make sure that you reference them more specifically by getting a reference to your framework's bundle e.g.

```objective-c
# in Objective-C
[NSBundle bundleForClass:<#ClassFromPodspec#>]

# or in swift
NSBundle(forClass: <#ClassFromPodspec#>)
```

This will then work for both frameworks and static libraries. There are very few cases where you want to reference the main bundle directly or indirectly, e.g. by using `[UIImage imageNamed:]`.

The advantage to this improved resource handling is that resources won't conflict when they have the same names because they are namespaced by the framework bundle.
Furthermore we don't have to apply the build rules to the resources ourselves as e.g. asset catalogs and storyboards need to be compiled. This should decrease build times for project using Pods that include many resources.

### Module Names

Names of Clang Modules are limited to be C99ext-identifiers. This means that they can only contain alphanumeric characters and underscores, and cannot begin with a number. Looking through the official spec repo, we discovered some popular pods, which don't match these requirements.

Before, as a Pod author, you could use `header_dir` to customize the name prefixing your headers from the user target. E.g. if your pod is named `123BánànâKit`, you could set it to `BananaKit`, it is available by `import <BananaKit/BananaKit.h>` instead of `#import <123BánànâKit/BananaKit.h>`.

We are still supporting this usage, but also introducing a new attribute `module_name`, which you declare in your Podspecs. This new attribute has the advantage that it will be properly linted and verified, otherwise we will work from the the `header_dir` option. If either attribute is not present then we will derive with the spec's name to match the Clang Module name requirements.

In a nutshell, look at the following Swift snippet, which concisely expresses the way in which we determine the module name.

```swift
//let c99ext_identifier: String -> String?
func module_name(spec: Specification) -> String {
  return spec.module_name
    ?? c99ext_identifier(spec.header_dir)
    ?? c99ext_identifier(spec.name)!
}
```

### Module Maps

A Module Map is a declaration of the headers, which form the public (or private) interface of a Clang Module.
Luckily, those have been designed so that they can stay in the background and the developer can leverage known and existing structures, without having to learn the DSL.
The default modulemap looks basically always the same:

```
framework module BananaKit {
  umbrella header "BananaKit.h"

  export *
  module * { export * }
}
```

This references only one file explicitly: the umbrella header.

You can export the public API for your framework inside a umbrella header and *all transitive imported* headers.
Clang will take care of making module exports that can be imported by Objective-C and Swift.

### What Does *Transitive Imported* Mean In This Context?

Transitive relations are a [mathematical concept](http://en.wikipedia.org/wiki/Transitive_relation):

>Whenever an element *a* is related to an element *b*, and *b* is in turn related to an element *c*, then *a* is also related to *c*.

We have here the binary relation of a header file, which imports another header file.
The transitive closure means that all headers, which are imported by header files, which you have imported from a certain file are indirectly imported to that file, too. That's also the same for all header files, which are imported by the collection of those headers.
Surely, you know this property from your app target, whenever you import a header, which imports other headers, the classes and symbols, which are defined there, are also available in your app code.
The same has to apply for import statements in Umbrella Headers and effect the module visibility with Clang modules.


### What are Umbrella Headers?

For our example it could look like this:

```objc
#import <Foundation/Foundation.h>

@import Monkey;

#import "BKBananaFruit.h"
#import "BKBananaPalmTree.h"
#import "BKBananaPalmTreeLeaf.h"

FOUNDATION_EXPORT double BananaKitVersionNumber;
FOUNDATION_EXPORT const unsigned char BananaKitVersionString[];
```

The original purpose is to index all public headers of a directory to have a shorthand for imports/includes to access the full API of a library. Over time, they began to cover more and more purposes:

* With **(Cocoa Touch) Frameworks**: they allow quick access to versioning values defined in their Info.plist by on-the-fly generated C code. Therefore they have to define an interface to make them accessible. These are the constant declarations found in the Xcode template prefixed by `FOUNDATION_EXPORT`.
* With **Clang modules**: they are used to define the public interface of a module.
* With **Swift**: they are the bridging header for the framework module, which essentially means that all Objective-C code you're interfacing from Swift within your framework has to be part of its public API.

### Current Situation with Existing Podspecs

There has never been a declarable Umbrella Header in Xcode. So Pod authors have never had to specify one.

Though it has always been a known pattern to have one public header, which imports *all* other public headers *transitively*. This isn't always the case.

For this reason, CocoaPods takes responsibility and generates a custom Umbrella Header (e.g. `Pods-iOS Example-AFNetworking-umbrella.h`). This is injected by a custom module map, so that we don't run into name ambiguities. Otherwise, the default module map would assume it has the same name as the framework, which could already been taken.

Our generated header imports all declared public headers. This also defines the `FOUNDATION_EXPORT`s for the versioning constants, with the name which is used by CocoaPods for the framework to integrate. Furthermore this avoids problems in some special cases: e.g. AFNetworking has a subspec, which provides categories for UIKit that has its own mass-import header `AFNetworking+UIKit.h`, which isn't imported by the `AFNetworking.h` header for OSX compatibility.

To use this subspec in Swift without a generated umbrella header, you would need to create a bridging header and use an import like `#import <AFNetworking/AFNetworking+UIKit.h`. With the generated umbrella header, you just need to `import AFNetworking` if you have the subspec included in your Podfile. If your pod doesn't work out of the box, you can use `pod lib lint --use-frameworks <YourPod.podspec>`, to check what is wrong. We tried that with different popular pods and sometimes ran into issues caused by misconfigured public headers.


#### About Public Headers

Public Headers in Podspecs are declared by `s.public_header_files = ["Core/*.h", "Tree/**.h"]`.

If you don't include this specification, then all your headers would be public.
This isn't recommeded for most cases.

Generally, you should make sure that you have self-contained headers and that those only expose the parts of your implementation which is consumed by your Pods users. This has several advantages:

* It allows you to refactor the private implementation part without necessarily releasing a major update, which makes the version migration easier and allows you to focus on further improving your pod instead of explain your users how the API has changed.
* It impedes misusage, because you would need to modify header access to use or manipulate classes or properties, which are not intended to be used externally.


#### Common Header Pitfalls

If you have an header like this:

```objectivec
/// BKBananaFruit.h

#import "BKBananaTree.h"
#import "monkey.h"

@interface BKBananaFruit
@property (nonatomic, weak) BKBananaTree *tree;
- (void)peel:(Monkey *)monkey;
@end
```

And if you get an error like the one below, don't let it fool you.

{% breaking_image /assets/blog_img/Frameworks-Developers/bananakit-error-non-modular-headers.png %}

You can include headers inside frameworks, **but not quoted headers**, which are not in scope of the framework's public headers. So you have two choices in this case: Either make the header `BKBananaFruit.h` private by excluding it from the public header declaration or use a system import to import the monkey.

```diff
-#import "monkey.h"
+#import <monkey/monkey.h>
````

#### Xcode Oddities

We have seen this error a few times during development.

```
<unknown>:0: error: could not build Objective-C module 'BananaKit'
```

An error like above can appear, if you develop a framework in Xcode, and you alter header visibility to fix build problems like described previously and try to ensure a clean build state by executing the Clean action (**⌘+
⇧+K** in Xcode). In this situation, it can be helpful to nuke the products build directory (alas `DerivedData`) manually from the file system.

## Updating

To install the latest Beta of CocoaPods you can run:

```bash
$ [sudo] gem install cocoapods --prerelease
```

Until version 1.0 we strongly encourage you to keep CocoaPods up-to-date. For more user facing changes, you can take a look at our [upcoming official release blog post](https://github.com/CocoaPods/blog.cocoapods.org/pull/59).

For all the details, take a look into the
[PR](https://github.com/CocoaPods/CocoaPods/pull/2835)!
