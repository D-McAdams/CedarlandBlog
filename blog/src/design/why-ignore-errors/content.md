## Why does Cedar ignore policies that error?

When asking Cedar to make an authorization decision, it is possible for an error to occur. Consider the following example:

```
permit (
    principal,
    action == Action::"read",
    resource in Folder::"financial_reports"
) when {
    principal.jobRole == "Finance"
};
```

If the value of `principal.jobRole` is undefined, this will cause an error during evaluation and Cedar will ignore the policy, treating the situation as if the policy never existed. Cedar will then continue evaluating the other policies to make a final authorization decision. Other situations can cause errors, as well, and they can be found by searching for "error" in the [Cedar operator documentation](https://docs.cedarpolicy.com/syntax-operators.html).

This behavior can lead to the questions:
1. Why does Cedar generate an error?
2. And, why does it ignore a policy if it errors?

###  Why does Cedar generate an error?

The decision to generate an error may appear unnecessary at first, especially if there are multiple conditions as in the following example:

```
permit (
    principal,
    action == Action::"read",
    resource in Folder::"financial_reports"
) when {
    principal.jobRole == "Finance" ||
    principal.jobLevel > 10    
};
```

In this example, if `principal.jobRole` was undefined but `principal.jobLevel` was defined, it would be convenient for the policy to not error. The second condition would match and everything would proceed happily.

Implementing this behavior would require treating the undefined attribute as some type of `nil` value. Any operation on this value, such as an equality check, would evaluate to false. This could be done, but it has a side-effect when negations are introduced:

```
// Permit anyone to read the folder UNLESS their job role is "external_contractor".
permit (
    principal,
    action == Action::"read",
    resource in Folder::"financial_reports"
) when {
    !( principal.jobRole == "external_contractor")
};
```

In this example, if a missing value of `principal.jobRole` was treated as a `nil` type, then the expression would surprisingly permit access. This is because `nil == "external_contractor"` would evaluate to false. But, since this is negated, the false becomes true.

As a result, anyone with an undefined value of `jobRole` gets access to the resource. Is this desired behavior? Is it undesired? The answer is indeterminate and therefore Cedar errors. Only the policy author can resolve the ambiguity. To fix the error in this example, the policy author can amend the conditions to test if the attribute exists before using it.

```
// Permit anyone to read the folder UNLESS their job role is "external_contractor".
permit (
    principal,
    action == Action::"read",
    resource in Folder::"financial_reports"
) when {
    principal has robRole &&
    !( principal.jobRole == "external_contractor")
};
```

###  Why does Cedar ignore policies that error?

When a policy errors, Cedar could halt and default the authorization decisiont to `Deny` instead of ignoring the error and proceeding to evaluate other policies. The reason it doesn't halt is for the safety of applications using Cedar. Imagine a system with 100 policies that are running successfully, and then someone adds policy number 101 which contains an error. If Cedar halted on error and emitted a default-deny decision for the entire batch of policies, then 100% of all authorization decisions in the system could begin failing, simply because someone introduced an error in one new policy.

If this behavior is worrisome to you, note that the Cedar evaluator returns diagnostics that indicate if any policies emitted errors. System owners may use this field to monitor for errors or even choose to fail-closed on error, if desired.

###  What about forbid statements? 

It is relatively safe to ignore `permit` statements that error since the impact is to allow less access; something that was intended to be permitted is not permitted. However, the inverse happens when ignoring a `forbid` statement; something that was intended to be forbidden may not be forbidden.

One of the larger debates during Cedar's design was whether errors in `forbid` statements should result in a policy being skipped, same as `permit` statements. The arguments on one side say it feels safer from a security perspective to behave differently and return a Deny decision when a `forbid` statement emits an error. At the same time, this has to be weighed against the blast impact of a mistake in a single policy resulting in 100% of all authorization queries returning a `Deny` decision. This behavior could lead to 100% unavailability for an application, which is also a goal of many types of attacks.

In the end, this was a debate with no winning side. The reality is that `forbid` statements are powerful, and anyone deploying `forbid` statements to production with zero testing beforehand is likely to have a bad day either way. As a result, Cedar behaves consistently - an error in any policy statement, whether `permit` or `forbid`, will result in the policy being skipped during an authorization evaluation. To minimize risk, Cedar policy validation can detect policy statements that may error at runtime so they may be corrected before being used in production. In addition, some authorization systems go further by allowing shadow testing of new policies to audit for unexpected behaviors prior to enforcement. On top of this, Cedar returns diagnostics in the authorization response that indicates if errors occurred. This can be used to monitor for new errors that may arise after a policy is deployed. Or, if desired, system owners may use these diagnostics to observe when an error occurred in a `forbid` statement and can elect to treat that as a fail-closed scenario, if appropriate for their application.
