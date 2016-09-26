title: NativeScript iOS Runtime Support for Xcode 8/iOS 10 SDK
date: 2016-09-14
---

[Xcode 8](https://developer.apple.com/xcode/) is just around the block and will soon come as a recommended Apple update to all of you. Here we'll see what implications will this have on your NativeScript development workflow.

<!-- more -->

This Xcode release comes with [many new features](https://developer.apple.com/library/content/releasenotes/DeveloperTools/RN-Xcode/Introduction.html) such as runtime issues, thread sanitizer, visual native memory debugging, simplified code signing and many more that can be also quite useful for your NativeScript applications, but in this post we are going to focus on a more specific topic - the latest iOS SDK. Xcode 8 comes with the iOS 10 SDK and this is what you'll be using to build and [generate metadata](https://github.com/NativeScript/ios-runtime-docs/blob/master/Overview.md#metadata) for your applications.

> Don't worry, applications built with the iOS 10 SDK can be still deployed on devices with lower OS versions down to the deployment target of the application which is iOS 8 by default as of now.

## Class Properties
There are many [API differences](https://developer.apple.com/library/content/releasenotes/General/iOS10APIDiffs/) between the iOS 9 and the iOS 10 SDKs, but there is one specific change that we want to inform you about and that will likely affect many of you who use iOS APIs in your JavaScript modules. And that is many Objective-C static methods have been upgraded to the newely introduced [Objective-C class properties syntax](https://developer.apple.com/videos/play/wwdc2016/405/?time=305).

Here is an example:
```objectivec
// In Xcode 7/iOS 9 SDK currentDevice is declared as a static method.
@interface UIDevice : NSObject
+ (UIDevice *)currentDevice;
@end

// In Xcode 8/iOS 10 SDK this is changed to a static property.
@interface UIDevice : NSObject
@property (class, readonly) UIDevice *currentDevice;
@end
```

And here are the JavaScript projections that you can call from NativeScript:
```typescript
// When building with Xcode 7/iOS 9 SDK
class UIDevice extends NSObject {
    static currentDevice(): UIDevice;
}

// When building with Xcode 8/iOS 10 SDK
class UIDevice extends NSObject {
    static readonly currentDevice: UIDevice;
}
```

What this means is that you have to change all `UIDevice.currentDevice()` calls to `UIDevice.currentDevice` in your JavaScript code when building with Xcode 8.

To help you make this transition easier, we have also updated our [TypeScript generator tool](https://jasssonpet.github.io/generating-typescript-declarations-in-nativescript-for-javascript-coffeescript-code-completion/) to support the iOS 10 SDK. Please make use of it even if you are not fond of TypeScript - these declarations can be used from most IDEs that support JavaScript completion and will help you catch such changes at compile time.

## Recommended Migration Path
Here are our suggestions:
* Update to Xcode 8 and make it your default coding environment. Make sure it is set as the active command line tool.
* Update your apps with the 2.3 release of the iOS runtime, NativeScript Core modules and NativeScript CLI. They have the required changes to support Xcode 8.
* If you are a plugin developer:
    * Build and test your plugins to be compatible with the iOS 10 SDK.
    * Make any needed changes to the native iOS APIs calls in your JavaScript files.
    * Republish your plugins and update the system requirements to Xcode 8.
* If you are an application developer:
    * Build and test your application to be compatible with the iOS 10 SDK.
    * Make any needed changes to the native iOS APIs calls in your JavaScript files.
    * If you are using plugins that are not yet updated for Xcode 8, contact the respective plugin maintainers or make a PR for that.

SDK updates can be a bit frustrating, but we are hoping that Xcode 8 will be the new default soon.