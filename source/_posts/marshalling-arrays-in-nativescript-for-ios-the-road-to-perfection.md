title: Marshalling Arrays in NativeScript for iOS - The Road to Perfection
date: 2015-09-21
---

We at [NativeScript](https://www.nativescript.org/) take performance very seriously, so we try to improve it in each release. In this blog post I want to show you how marshalling between [JavaScript arrays](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array) and [Objective-C arrays](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/Foundation/Classes/NSArray_Class/) has evolved between four different versions of the [NativeScript Runtime for iOS](https://github.com/NativeScript/ios-runtime).

<!-- more -->

> If you are not familiar with the idea of *marshalling* in general, please take a look at the docs: [Marshalling overview](http://docs.nativescript.org/runtimes/ios/marshalling/Marshalling-Overview).

## v0.1 - Creating a Deep Copy in JavaScript
In the very first release of NativeScript for iOS, if some method required `NSArray` as an argument, the user had to manually create a `NSArray` copy of the JavaScript array before calling the method:
```javascript
function JSArrayToNSArray(jsArray) {
    let nsArray = new NSMutableArray();
    for (let currentJSObject of jsArray) {
        // Objective-C Arrays can't contain nil, so use NSNull object instead.
        // Marshalling of each element of the array is still done in native.
        nsArray.addObject(currentJSObject != null ? currentJSObject : NSNull.null());
    }
    return nsArray;
}

var textChecker = new UITextChecker(); // Shorthand for alloc/init
textChecker.setIgnoredWords(JSArrayToNSArray(["NativeScript"]));
```
This was the easiest way for us to implement it and in fact required no special native code. Needless to say, the code didn't look as nice as it could and the performance of the testing apps suffered, but it was a **great start to get something working**.

## v0.3 - Creating a Deep Copy in Native
After a complete rewrite and some more stabilizing to the code architecture, we were able to give some attention to details. In this version of NativeScript for iOS, the user could pass a plain JavaScript array to the above method. The bridge verified that it was a JavaScript array and implicitly handled the copying to `NSArray` in the Objective-C++ world:
```objectivec
NSArray* NativeScript::toObject(JSC::ExecState* execState, JSC::JSArray* jsArray) {
    NSMutableArray* nsArray = [NSMutableArray array];

    for (unsigned i = 0; i < jsArray->length(); ++i) {
        JSC::JSValue currentJSObject = jsArray->get(execState, i);
        id currentObject = NativeScript::toObject(execState, currentJSObject);

        // Objective-C Arrays can't contain nil, so use NSNull object instead.
        [nsArray addObject:currentObject ?: [NSNull null]];
    }

    return nsArray;
}
```
```javascript
var textChecker = new UITextChecker();
textChecker.setIgnoredWords(["NativeScript"]);
```
The first approach was still working, but this was the preferred way now. This way **the code looked cleaner *and* there was a decent performance gain**.

## v1.0 - Creating a Shallow Copy
But we didn't stop here. For the official release there was no longer a deep copy of JavaScript arrays, but a shallow one ([#64](https://github.com/NativeScript/ios-runtime/pull/64) by [@fealebenpae](https://github.com/fealebenpae)). Instead of copying the contents of the entire array, the native object now stored a strong reference to the JavaScript object. Marshalling of each element was done on demand.

> For this release not only JavaScript arrays could be implicitly marshalled to `NSArray`, but also any array-like object.

According to the [Apple docs](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/Foundation/Classes/NSArray_Class/) a good way to change how `NSArray` stores its elements is by subclassing it. There are two required methods to be implemented which look something like the following:
```objectivec
@implementation TNSJavaScriptArrayAdapter : NSArray {
    JSC::Strong<JSC::JSArray> _jsArray;
    JSC::ExecState* _execState;
}

- (instancetype)initWithJSArray:(JSC::JSArray*)jsArray execState:(JSC::ExecState*)execState {
    if (self = [self init]) {
        _jsArray = jsArray;
        _execState = execState;
    }
    return self;
}

- (NSUInteger)count {
    return _jsArray->length()
}

- (id)objectAtIndex:(NSUInteger)index {
    JSC::JSValue currentJSObject = jsArray->get(_execState, index);
    id currentObject = NativeScript::toObject(execState, currentJSObject);
    return currentObject;
}
@end
```
There wasn't so big of a performance gain this time, but this implementation **reduced the memory footprint** of our apps by skipping a copy of each array.

## v1.2 - Creating a Shallow Copy and Implementing `NSFastEnumeration`
The last change (so far) for marshalling JavaScript arrays to Objective-C was the implementation of the [`NSFastEnumeration`](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/NSFastEnumeration_protocol/) protocol ([#222](https://github.com/NativeScript/ios-runtime/pull/222) by [@fealebenpae](https://github.com/fealebenpae)). You can learn more about `NSFastEnumeration` on [NSHipster](http://nshipster.com/enumerators/), but the essence is a single method named `countByEnumeratingWithState:objects:count:`, which the aforementioned `TNSJavaScriptArrayAdapter` implements:
```objectivec
- (NSUInteger)countByEnumeratingWithState:(NSFastEnumerationState *)state objects:(id [])stackbuf count:(NSUInteger)len {
    // Uninitialized
    if (state->state == 0) {
        state->state = 1;
        state->mutationsPtr = reinterpret_cast<unsigned long*>(self);
        state->extra[0] = 0; // Current index
    }

    NSUInteger count = 0;
    NSUInteger currentIndex = state->extra[0];
    for (; count < len && currentIndex < _jsArray->length(); currentIndex++, count++) {
        JSC::JSValue currentJSObject = jsArray->get(_execState, index);
        id currentObject = NativeScript::toObject(execState, currentJSObject);
        *stackbuf++ = currentObject;
    }
    state->extra[0] = currentIndex;

    state->itemsPtr = stackbuf;
    return count;
}
```
The runtime is given a preallocated buffer on the stack, and fills it up accordingly. This way there are fewer method calls and objects can be loaded concurrently. In certain tests the same JavaScript code was **up to 30x faster** than in the previous release.

> Another thing that was made possible in this release of NativeScript for iOS was creating a [custom subclass of an Objective-C class in JavaScript](http://docs.nativescript.org/runtimes/ios/how-to/ObjC-Subclassing) and implementing the [JavaScript iteration protocol](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols). The runtime will look it up and will create a stub implementation of the `NSFastEnumeration` protocol which consumes the provided JavaScript one.

## Conclusion
I've shown you only one aspect of the NativeScript Runtime, but you can see how it is constantly pushing the edges to be as fast as it can, while taking less memory and making the code easier to read.

If you have any further suggestions or questions about NativeScript, don't hesitate to [open an issue on GitHub](https://github.com/NativeScript/ios-runtime/issues/new).
