title: Under the Curtains of <HTMLView /> for NativeScript for iOS
date: 2015-08-03
---

We just released [version 1.2 of NativeScript](https://www.nativescript.org/blog/nativescript-1.2-release---live-sync-push-notifications-native-plugins-and-more) which among many new features included a new [`<HTMLView />`](https://docs.nativescript.org/ApiReference/ui/html-view/HOW-TO) element. I'll show you what stands behind it with nothing but plain JavaScript running on the [iOS runtime](https://github.com/NativeScript/ios-runtime).

<!-- more -->

We are going to start with the iOS ["Hello, World!" template](https://docs.nativescript.org/runtimes/ios/getting-started/HelloWorld) and we will replace the contents of the label from `NSString` with `NSAttributedString`. On iOS the `<HTMLView />` element uses `NSAttributedString` to render itself. You can read more about it in the [Apple docs](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/Foundation/Classes/NSAttributedString_Class/), but in short "it manages character strings and associated sets of attributes (for example, font and kerning) that apply to individual characters or ranges of characters in the string".

[{% asset_img html-view-demo.png "Hello, NativeScript!" %}](https://gist.github.com/jasssonpet/1637318e302096148885)

> You can see the whole code example in [this gist](https://gist.github.com/jasssonpet/1637318e302096148885).

Here is how you create a `NSAttributedString` using Swift 2 (not released yet as of writing):
```swift
func attributedStringFromHTMLString(str: NSString) throws -> NSAttributedString {
    let encodedData = str.dataUsingEncoding(NSUTF8StringEncoding)!
    let attributeOptions: [String: AnyObject] = [
        NSDocumentTypeDocumentAttribute: NSHTMLTextDocumentType,
        NSCharacterEncodingDocumentAttribute: NSUTF8StringEncoding
    ]

    return try NSAttributedString(data: encodedData, options: attributeOptions, documentAttributes: nil)
}
```

And here is how you do it with NativeScript 1.2:
```javascript
function attributedStringFromHTMLString(str) {
    var encodedData = NSString.stringWithString(str).dataUsingEncoding(NSUTF8StringEncoding)
    var attributeOptions = {
        [NSDocumentTypeDocumentAttribute]: NSHTMLTextDocumentType,
        [NSCharacterEncodingDocumentAttribute]: NSUTF8StringEncoding
    }

    return NSAttributedString.alloc().initWithDataOptionsDocumentAttributesError(encodedData, attributeOptions, null)
}
```

These snippets look quite similar, don't you think? The JavaScript code may look very simple, but there is a bit more going on. Let's see the details one by one:

## Error Handling
We are calling [the following method](https://developer.apple.com/library/prerelease/ios/documentation/UIKit/Reference/NSAttributedString_UIKit_Additions/index.html#//apple_ref/occ/instm/NSAttributedString/initWithData:options:documentAttributes:error:):
```objectivec
 - (instancetype)initWithData:(NSData *)data
                      options:(NSDictionary *)options
           documentAttributes:(NSDictionary **)dict
                        error:(NSError **)error;
```

In Objective-C, you pass a reference to a `NSError` variable and the method will set it to a `NSError` object if there has been some kind of problem.

As you might have noticed, this method has 4 parameters, but we are calling it with 3 arguments. This is because in Swift 2 Apple introduced [tighter integration with Objective-C error handling](https://developer.apple.com/library/prerelease/ios/documentation/Swift/Conceptual/BuildingCocoaApps/AdoptingCocoaDesignPatterns.html#//apple_ref/doc/uid/TP40014216-CH7-ID10).  In Swift 2 you can skip the last parameter and use the `try`/`catch` mechanism to check if an error occurred.

With the 1.2 release of the iOS runtime we made the same thing possible ([#186](https://github.com/NativeScript/ios-runtime/issues/186), [docs](https://docs.nativescript.org/runtimes/ios/marshalling/Marshalling-Overview#nserror--marshalling)) and you can enjoy this neat feature before even Swift 2 is released. If you skip the last argument, this method will now throw a JavaScript object which wrapps the `NSError` object if the out parameter is set. You can use the JavaScript `try`/`catch` construct to check if an error occurred. (If you override such methods, any thrown JavaScript error will be respectively wrapped in a `NSError` object.)

## ES6 Computed Property Names
One of the major changes in the 1.2 release of the iOS runtime is that we updated our JavaScriptCore engine. Our last update was more than 10 months ago, but we now use CMake to build our runtime and the JavaScriptCore engine and this allowed us to seemlessly upgrade it two times in the last release. This means that you will be using the latest JavaScript features such as [computed property names](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Object_initializer#Computed_property_names), [arrow functions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions), [template strings](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/template_strings) on _all_ iOS versions as soon as they become available (we bundle the JavaScriptCore engine with each app).

Lets take a look at this object:
```javascript
var attributeOptions = {
    [NSDocumentTypeDocumentAttribute]: NSHTMLTextDocumentType,
    [NSCharacterEncodingDocumentAttribute]: NSUTF8StringEncoding
};
```

Starting with ECMAScript 6, the object initializer syntax also supports computed property names. That allows you to put an expression in brackets `[]`, that will be computed as the property name. In this snippet `NSDocumentTypeDocumentAttribute` and `NSCharacterEncodingDocumentAttribute` are Objective-C `NSString` global variables exposed in the global JavaScript scope.


## Marshalling from JSON Object to `NSDictionary`
Notice that the `initWithData:options:documentAttributes:error:` method accepts a `NSDictionary` object as second parameter, but we are passing a plain JavaScript object. This is possible because the iOS runtime implicitly wraps it in a `NSDictionary` object ([#64](https://github.com/NativeScript/ios-runtime/pull/64)). It doesn't create a copy of the object, so this operation should be pretty fast.

This is only some of the magic that happens in 5 lines of code in the iOS runtime. If you have any suggestions you would like to see implemented, we will be glad to [discuss them on GitHub](https://github.com/NativeScript/ios-runtime/issues/new).

**P.S.** More of a joke, but the last feature that allows the Swift and JavaScript code to look so alike is [JavaScript ASI](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Lexical_grammar#Automatic_semicolon_insertion).
