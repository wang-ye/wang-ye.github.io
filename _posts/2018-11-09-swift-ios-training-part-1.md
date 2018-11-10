## Swift Learning
I participated for our company's Swift/iOS training. Our team has several mobile requests, but none of us knows mobile development. Also, learning some iOS programming is really driving me out of the comfort zone, which I enjoyed very much.

The training took about 6.5 days in total. For the first two days, we learned the language Swift, and for the remaining 4.5 days it is full of iOS app building. I split the experience into two parts. For this one, I would focus more on the Swift language, the the next part is about app building.

Frankly speaking, the Swift programming language did not give me lots of surprise. I feel it is a relatively easy language to learn and use. XCode also provides powerful support for writing Swift code.

Here is my summary of the Swift language features

1. Builtin collections(arrays, dict, set ...) are all implemented by structs, and are passed by values, while the UI framework is in classes, and are passed by references.
2. Optionals: Similar to Java/Scala, it is used to handle null values elegantly, but sometimes make the expressions more complex.
3. Closure: It was extensively used to handle event-driven UI programming, and closures are key for event-driven programming. It has a couple of unique syntax when using closures, but the concept is the same.
4. Guard syntax sugar for error handling - maybe useful, but it is a syntax sugar.
5. Properties: This feature also appears in many other languages including Java. Computed properties is a syntax sugar, but building on top of it, property observer has some interesting use cases.
6. Protocols: Almost the same as interface in other languages
7. Extensions: Seems similar to namespaces in C++. It seems to be widely used in Swift, but abusing it can cause problems.

For the next blog, I will share more about the app building.