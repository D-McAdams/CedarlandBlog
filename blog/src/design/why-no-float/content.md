## Why doesn't Cedar support floating point numbers?

Sometimes, it's necessary to write an authorization rule against numbers that contain decimal parts. For example, you may want to check if a currency field is greater than `"1.99"`, or compare the output from an ML-based risk scoring system which emits values such as `"0.9234"`. **Cedar supports this using the [Decimal type](https://docs.cedarpolicy.com/policies/syntax-operators.html#function-decimal)** with a fixed precision, where the maximum number of digits after the decimal is four. 

In many other programming environments, there is an alternative representation of numbers known as [floating-point](https://en.wikipedia.org/wiki/Floating-point_arithmetic) which allows for bigger or smaller values with different degrees of precision. For example, a floating point number might approximate the value of pi with a sequence of digits such as `"3.141592653589793…"` up to the maximum size allowed by the data type, often 64 bits or 15 decimal digits.

Cedar does not support floating-point numbers. The reason is because floating-point numbers, by their nature, are  imprecise, and imprecision is not a desirable property of an authorization system. To illustrate this point, consider the following test: `0.1 + 0.2 == 0.3`. This appears as if it should evaluate to true. But, in most programming languages, it doesn't. [This blog post]( https://0.30000000000000004.com/) documents the result of the operation in a variety of different programming languages. The reason this occurs is because fractions such as `1/10` and `1/20` do not have a precise binary representation. Floating-point arithmetic approximates these values with a limited degree of precision.

Floating-point can lead to other quirks as well. For example `262144.0 + 0.01 = 262144.0`. One number plus another number equals the same number. You can see a similar effect in Javascript by taking the number 16128500101100052000 and adding 1; you get the same number back. (Javascript number are always 64-bit floating-point). This behavior occurs when adding large and small numbers. An [excellent blog post]( https://jvns.ca/blog/2023/01/13/examples-of-floating-point-problems/) describes these quirks of floating-point and real world impacts. 

Floating-point is a well-defined and a very useful format in many other programming contexts. We elected not to include it in Cedar not because it’s a “bad” format, but because we think it’s a bad idea to use this kind of arithmetic for authorization decisions.

Because of this, Cedar also does not include mathematical operators such as division that return floating-point values. For example, the operation `1 / 3` would necessitate a floating-point value for the result. Cedar also does not allow multiplication with Decimal types, such as `1 * decimal("0.3")`, as this would lead to the same outcome.

For additional information on the Cedar `Decimal` type and other operators, see the [documentation](https://docs.cedarpolicy.com/policies/syntax-operators.html#function-decimal). `Decimal` supports the following operations:
```
decimal("0.3") == decimal("0.3")                //true
decimal("0.3") != decimal("0.4321")             //true
decimal("0.3").lessThan("922337203685477.5807") //true
decimal("0.3").lessThanOrEqual("0.300")         //true
decimal("0.3").greaterThan("-4.82")             //true
decimal("0.3").greaterThanOrEqual("00.30")      //true
```