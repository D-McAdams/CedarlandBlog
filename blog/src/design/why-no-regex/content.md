## Why doesn't Cedar have regexes or string formatting operators?

Cedar's design methodology is underpinned by a philosophy of safety, as described in ["Why was Cedar created?"](../why-cedar/content.html). Safety considerations are wide-ranging and can include factors such as preventing excessive resource consumption, preserving the ability to analyze polices for correctness, or simply guiding policy authors away from common logical mistakes.

Regular expressions and string formatting operators were intentionally omitted from the language because they work against these safety goals. This blog post describes why they can be dangerous and alternative approaches to writing policies without them.

### These operators are error-prone (and impolite to policy authors)
As an example, consider an authorization policy that wants to make decisions based on a URL and query string received by a web application. A naïve way to do this is to pass the entire URL as a string into the context of the policy evaluator:

```
"context": {
    "url": "https://example.com/path?queryParam1=hello&queryParam2=world"
}
```

If the policy author wanted to make decisions based on subcomponents of the URL, such as the host name or particular query parameters, the policy author might need to apply regexes to match the subcomponents.

```
forbid (principal, action, resource)
unless {
    context.url.matches("^http(s?)://example.com/")
};
```

They would also need to use string formatting operators to normalize the values appropriately. Normalization is important because fields such as `host` are case-insensitive, and a malicious actor could potentially subvert an authorization rule by sending an HTTP request with different capitalization.

```
forbid (principal, action, resource) 
unless {
    //Lowercase the hostname before matching, otherwise a caller
    //could bypass this policy by sending https://EXAMPLE.COM/
    context.url.toLowerCase.matches("^http(s?)://example.com/")
};
```

But, how should policy authors handle the query parameters? Are those case-insensitive or not? Do they have other normalization requirements? And, if so, how would the policy author know? These are application-specific details.

```
forbid (principal, action, resource)
unless {
    // Should this also match "HELLO" or "Hello"?
    context.url.matches("[&\?]queryParam1=hello[&\b]")
};
```

Shifting this burden onto policy authors who may not deeply understand the application details is error-prone. It is also impolite and wasteful, since many different policy authors are bearing the burden of implementing the potentially complex logic. Cedar's philosophy is that application owners should strive to format and normalize data before passing it into the authorization evaluator. This keeps the logic with the owners who have the most domain expertise, and centralizes it in one location so it can benefit all policy authors.

From this perspective, the URL information could be passed into the evaluator in a pre-normalized format such as the following:

```
"context": {
    "url": {
        "transport": "https",
        "host": "example.com",
        "path": "/path",
        "queryParams" {
            "queryParam1": "hello",
            "queryParam2": "world"
        }
    }
}
```

And a policy author could express rules such as the following:

```
forbid (principal, action, resource)
unless {
    context.url.host == "example.com"
};
```

Another reason that regular expressions are error-prone is because of human error. To provide a real life example shared with me by a teammate, a policy author (in a non-Cedar system) wanted to write a rule that allowed anyone in the *"admin"* group to have super-administrator privileges in an application. This was done by examining a user's list of groups and testing against the regex `/admin/`. The mistake was not including the word-boundary conditions in the regex, such that this rule accidentally matched the group *"administrative-assistants"* and theoretically, I guess, the *"badminton-club"*. The administrative assistants discovered this access and found it quite useful in getting their job done.

Regular expressions and string formatting operations are powerful features that are useful in many programming environments. However, in authorization rules that protect access to critical resources, Cedar takes the perspective that simple, safe, auditable rules are preferrable over unbounded flexibility.

### These operators are subject to runtime risks

Regexes and string formatting operations carry runtime risks, as well. Regexes are subject to poor performance characteristics, including worst-case scenarios such as [catastrophic backtracking](https://www.google.com/search?q=regex+catastrophic+backtracking+ddos). String formatting operations can also be abused. To provide an example, consider an operator that concatenates two strings and emits a new string.

```
newString = concat(string1,string2)
```

Presuming the input strings are well-bounded in size, this operator is safe when used once. But, what happens if string concatenations are nested?

````
concat( concat(x,x), concat(x,x))
````

If `x="hello"`, this concatenates the string `"hello"` repeatedly

```
hello
hellohelllo
hellohellohellohello
```

A malicious policy author could push this to an extreme:

```
concat( concat( concat( concat( concat( concat(x,x), concat(x,x)), concat(concat(x,x), concat(x,x))), concat( concat( concat(x,x), concat(x,x)), concat(concat(x,x), concat(x,x)))), concat( concat( concat( concat(x,x), concat(x,x)), concat(concat(x,x), concat(x,x))), concat( concat( concat(x,x), concat(x,x)), concat(concat(x,x), concat(x,x))))), concat( concat( concat( concat( concat(x,x), concat(x,x)), concat(concat(x,x), concat(x,x))), concat( concat( concat(x,x), concat(x,x)), concat(concat(x,x), concat(x,x)))), concat( concat( concat( concat(x,x), concat(x,x)), concat(concat(x,x), concat(x,x))), concat( concat( concat(x,x), concat(x,x)), concat(concat(x,x), concat(x,x))))))
```

Due to the power of exponential growth, it would take only 40 repetitions of concatenation to transform the word "hello" into a 5 terabyte string that exceeds the available memory on nearly all classes of modern, general purpose web servers. To provide bounded runtime latencies, an authorization engine would need to guard against excessive memory allocation and fail at runtime. This requires a degree of guesswork about how much memory is "too much", with a risk of that choice being either too high or too low for someone's legitimate situation.

### These operators are not analyzable
As described in [Why was Cedar created?](../why-cedar/content.html), the language was built using verification-guided development and designed for automated reasoning. These techniques provide two benefits: they give a high-degree of confidence that Cedar is implemented correctly (i.e. bug free), and they also allow analyses in order to give you confidence that the policies you write in Cedar are correct.

Regexes and many string formatting operators are not amenable to these formal reasoning techniques. As a result, these operators simultaneously increase the likelihood of end-user mistakes when writing policies, while at the same time lessoning Cedar's ability to alert users of these mistakes. 

### Alternative Approaches
As mentioned earlier, the most common approach is to pre-format data before passing it into the authorization evaluator, thereby freeing policy authors from the need for regular expressions and string formatting operators.

When this is insufficient, the Cedar team is interested in hearing the requirements. (You can reach the community on the [Cedar Slack channel](https://communityinviter.com/apps/cedar-policy/cedar-policy-language).) Although Cedar is strict in terms of safety, it is also pragmatically motivated to solve real-world scenarios. Quite often, alternative approaches can be found that address requirements while adhering to safety goals. For example, **Cedar includes a `like` operator with the wildcard character `*`**, which solves many situations that would otherwise require regexes. Cedar also supports extended datatypes for fields such as [IP Address](https://docs.cedarpolicy.com/syntax-datatypes.html#ipaddr), with built in validation and helpful operators that can be used policy expressions. Additional extended datatypes can be introduced if they have broad applicability.
