title: Generating TypeScript Declarations in NativeScript for JavaScript/CoffeeScript Code Completion
date: 2015-10-17
tags:
---

One thing that I like about [NativeScript](https://github.com/NativeScript/NativeScript) is the ease with which I am able to call native APIs. But sometimes the process of getting the language projections from [Objective-C](http://docs.nativescript.org/runtimes/ios/marshalling/Marshalling-Overview)/[Java](http://docs.nativescript.org/runtimes/android/metadata/accessing-packages) to JavaScript right is somewhat tedious and error-prone. In this post I want to show you how the recent [1.4 release](https://github.com/NativeScript/ios-runtime/releases/tag/v1.4.0) of NativeScript for iOS has made this experience a lot more pleasant.

<!-- more -->

## Enter TypeScript Declarations
What are those things are you asking? Here is an explanation straight from the [TypeScript docs](http://www.typescriptlang.org/Handbook#modules-working-with-other-javascript-libraries):
> To describe the shape of libraries not written in TypeScript, we need to declare the API that the library exposes. Because most JavaScript libraries expose only a few top-level objects, modules are a good way to represent them. We call declarations that don't define an implementation "ambient". Typically these are defined in `.d.ts` files. If you're familiar with C/C++, you can think of these as `.h` files or `extern`.

## Generating TypeScript Declarations from Native APIs
Starting with version 1.4 of NativeScript for iOS, you are able to generate TypeScript declarations on the fly from Objective-C APIs. Executing the following command from the root of your NativeScript app will produce a `app/typings` folder with a `.d.ts` file for each iOS framework.
```bash
$ TNS_TYPESCRIPT_DECLARATIONS_PATH="$(pwd)/app/typings" tns build ios
```
> **How this works:** Here we are declaring the `TNS_TYPESCRIPT_DECLARATIONS_PATH` [environment variable](https://wiki.archlinux.org/index.php/Environment_variables) which flows down to [`xcodebuild`](https://developer.apple.com/library/mac/documentation/Darwin/Reference/ManPages/man1/xcodebuild.1.html) which subsequently triggers the NativeScript metadata generator build step. Because of this you are able to effortlessly generate TypeScript declarations for any newly added third party framework or CocoaPod. Android support is [under active development](https://github.com/NativeScript/android-dts-generator), but for now you can use the pre-generated [android.d.ts](https://github.com/NativeScript/NativeScript/blob/master/android17.d.ts) file.

## TypeScript Declaration Example
 Here is a simple example. Somewhere in the iOS SDK there is a `<UIKit/UIApplication.h>` header declaring the [`UIApplicationMain`](https://developer.apple.com/library/ios/documentation/UIKit/Reference/UIKitFunctionReference/#//apple_ref/c/func/UIApplicationMain) function:
```objectivec
int UIApplicationMain(int argc, char *argv[], NSString* principalClassName, NSString* delegateClassName);
```
After building your app in the aforementioned way, there should be a `objc!UIKit.d.ts` file with the same function declared, but this time with a TypeScript signature:
```typescript
declare function UIApplicationMain(argc: number, argv: interop.Reference<string>, principalClassName: string, delegateClassName: string): number;
```

> **Note:** Apple recently introduced Objective-C generics in Xcode 7. With a recent pull request in the iOS metadata generator ([#341](https://github.com/NativeScript/ios-runtime/pull/341)),  those are expressed in the generated `.d.ts` files as well. For example the `+ (NSArray<NSBundle*>*)allBundles` Objective-C method is projected as `static allBundles(): NSArray<NSBundle>` in TypeScript.

## Using TypeScript Declarations Outside TypeScript
**This is all great, you say, but I *hate* TypeScript.** I totally understand you and agree with you, but fear not - there is a way to combine the best of both worlds. You can use most of the benefits of native code completion not only in TypeScript, but also in JavaScript (or CoffeeScript).

One way of doing so is by using [WebStorm](https://www.jetbrains.com/webstorm/). It is a world-renowned IDE for web development and a personal favorite of mine. It works on Windows, Mac OS or Linux and has an unlimited trial with 30 minute session limitation or a completely free [EAP (Early Access Program) version](http://confluence.jetbrains.com/display/WI/WebStorm+EAP). It has support for JavaScript, CoffeeScript and TypeScript. But even more - it has cross-language code analysis, which is why it is best for our scenario. It can pick up the metadata from the TypeScript declarations and provide intelligent code completion in all other languages:

Here is how it looks with CoffeeScript:
{% asset_img code-completion.png "CoffeeScript code completion from TypeScript declarations" %}
We are getting some nice fuzzy code completion on the [`CFDateFormatterStyle`](https://developer.apple.com/library/prerelease/ios//documentation/CoreFoundation/Reference/CFDateFormatterRef/index.html#//apple_ref/doc/constant_group/Date_Formatter_Styles) enum.

## Moving Forward - JSDoc Documentation for Native Frameworks
In the next release we would like to make the experience even smoother. We are working on optionally including documentation from [Xcode docsets](https://kapeli.com/dash_guide) (or [HeaderDoc](https://developer.apple.com/library/mac/documentation/DeveloperTools/Conceptual/HeaderDoc/tags/tags.html) tags for third-party frameworks) in the generated TypeScript declarations.

Again, this is not limited to WebStorm only, but here is how the documentation for the [`UIApplicationMain`](https://developer.apple.com/library/ios/documentation/UIKit/Reference/UIKitFunctionReference/#//apple_ref/c/func/UIApplicationMain) function looks like with the WebStorm [JSDoc](http://usejsdoc.org/) visualizer in CoffeeScript:
{% asset_img docs.png "CoffeeScript documentation for UIApplicationMain" %}

Tell us what you think about this feature in the related issue - [#353](https://github.com/NativeScript/ios-runtime/issues/353).

There are some [known issues](https://github.com/NativeScript/ios-runtime/issues/282) with generating TypeScript declarations for now, but if you find any new bugs in the generated TypeScript declarations we will be glad to fix them if you [report them on GitHub](https://github.com/NativeScript/ios-runtime/issues/new).

**UPDATE 1:** You can also use the [Babel](https://www.npmjs.com/package/nativescript-dev-babel), [CoffeeScript](https://www.npmjs.com/package/nativescript-dev-coffeescript) and [TypeScript](https://www.npmjs.com/package/nativescript-dev-typescript) NativeScript CLI plugins to ease the configuration of the the transpilers in your NativeScript project.

**UPDATE 2:** JSDoc comments for native APIs are now implemented - [#37](https://github.com/NativeScript/ios-metadata-generator/pull/37).
