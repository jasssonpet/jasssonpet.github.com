title: Deep Dive in the NativeScript Runtime for iOS
date: 2015-12-31
---

With this blog post I would want to shed some light on how the [NativeScript runtime for iOS](https://github.com/NativeScript/ios-runtime) works. As a developer you may be wondering how all of this is working under the hood. Before you get started, you may want to watch the official [introductionary video](https://www.youtube.com/watch?v=I3_ZnWTj-NA) to gain a more broad perspective on how NativeScript works. This post covers the iOS part of the things.

<!-- more -->

Truth is, that the runtime for iOS is a dwarf standing on the shoulders of the following giants:

## JavaScriptCore
[JavaScriptCore](https://github.com/WebKit/webkit/tree/master/Source/JavaScriptCore) is the JavaScript engine used to parse and execute JavaScript source files. By itself, a JavaScript engine isn't tied to any particular environment. For example, browsers extend it with [web stuff](https://developer.mozilla.org/en-US/docs/Web/API) (`window`, `document`, `navigator`, ...) and Node.js does with it with [its own APIs](https://nodejs.org/api/globals.html) (`global`, `require`, `process`, ...). Тhe case with [NativeScript for iOS](https://github.com/NativeScript/ios-runtime) is that it hooks within the JavaScript engine and makes interacting with the Objective-C runtime from JavaScript possible.

Built-in JavaScriptCore engine, but

**Note:** JavaScript source files and the NativeScript runtime are bundled in the final app package.

## Clang

[Clang](https://github.com/llvm-mirror/clang) is used by the [metadata generator tool](https://github.com/NativeScript/ios-metadata-generator) to parse any [Objective-C header](https://en.wikipedia.org/wiki/Include_directive) from the platform SDK or third-party framework and read the described metadata. This includes classes, interfaces, function/method signatures, availability information and so on. Before going with Clang we have [tried](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/ObjCRuntimeRef/index.html) [several](https://developer.apple.com/library/mac/documentation/Darwin/Reference/ManPages/man1/gen_bridge_metadata.1.html%29) [different](https://github.com/atsushieno/nclang) [methods](https://github.com/llvm-mirror/clang/tree/master/bindings/python), but after all - only the compiler could give you the whole picture. It was harder to bootstrap it initially, but after that it paid off.

This step is very similar by its nature to what the [Swift Clang importer](https://swift.org/compiler-stdlib/#compiler-architecture) is doing.

After processing all supplied information from the platform SDKs, the metadata generation tool compresses it in a highly optimized binary format that is also shipped in the application bundle. It can also optionally generate [TypeScript projections of the Objective-C declarations](/generating-typescript-declarations-in-nativescript-for-javascript-coffeescript-code-completion) to allow users easier time figuring APIs out.

The process of generating metadata is done on each built on your machine and uses your local Xcode SDK. This way you can get any new APIs by simply updating your Xcode installation.

## libffi
[libffi](https://github.com/atgreen/libffi) is the real glue to make it all work. It ties between the JavaScriptCore engine and the underlying native APIs. libffi allows an interpreted language such as JavaScript to dynamically invoke existing C functions (and Objective-C methods, which in fact are nothing but C functions) or create new ones at runtime based on the build-time generated [type signatures](https://en.wikipedia.org/wiki/Type_signature).

## Objective-C Runtime
[Objective-C Runtime](https://github.com/opensource-apple/objc4) (compared to C++) is quite a dynamic language and combined with its runtime library it allows to create new classes at runtime, implement interfaces, add/replace methods, reflect method signatures and more. This flexibility is very powerful for interacting from a language like JavaScript.

If you are not familiar with Objective-C - tutorial

## Conclusion
All of this wouldn't be possible without the open source and extensible nature of all those projects. By reading the source code we were able to figure out how some edge cases work, making workarounds and even providing upstream bug fixes. Working in the open is definitely the way to go and NativeScript is following in the right way.
