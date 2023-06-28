## Why doesn't Cedar allow wildcards in Entity Ids?

**TLDR: This can be a risky anti-pattern. This post explains the risks and how to accomplish it when absolutely necessary.**

Cedar support wildcards for string matching via the [like](https://docs.cedarpolicy.com/syntax-operators.html#like-string-matching-with-wildcard) operator. However, wildcards are not allowed for Entity Id matching. Before explaining why, the question itself deserves explanation as not everyone will necessary understand *why* someone would want wildcard matching in Entity Ids.

For those new to Cedar, [Entity Ids]( https://docs.cedarpolicy.com/syntax-entity.html) are the means to refer to a specific principal or resource within a policy statement. Examples may look like this:

```
permit (
    principal == User::"5fb883fb-229c-48bc-b186-e7ed9074b536",
    action,
    resource == File::"8d60e1b7-ed63-419b-b198-13ac9e803ee7"
);
```

The fields labeled "User" and "File" are the Entity Ids. They consist of a type like `User`, plus an identifier like `5fb883fb-229c-48bc-b186-e7ed9074b536`.

In this example, we have used UUIDs as the identifiers. Synthetic identifiers are recommended by the [Cedar Security guidance](https://docs.cedarpolicy.com/security.html). When using UUIDs, the benefits of having wildcards in the identifier may not be readily apparent. After all, trying to match against a pattern such as `User::"5fb883*4b536"`is unlikely to be useful in practice.

Therefore, questions about wildcard support most often arise in situations where the Entity Id is not synthetic, but rather a string that encodes information about the system. For example, the identifier might be a path that describes a hierarchical nesting of resources:

```
File::"/Org:923902/Department:4992/Folder:MonthlyReports/January2023.pdf"
```

If wildcards were allowed by Cedar, then access to all folders for a specific department could be represented by using a wildcard in the identifier, as follows:

```
permit (
    principal == User::"5fb883fb-229c-48bc-b186-e7ed9074b536",
    action,
    resource == File::"/Org:923902/Department:4992/*"
);
```
Another situation where identifiers can encode information is when they include metadata, such as the owner of the resource:

```
File::"user:32432423/image123.jpg"
File::"user:32432423/image456.jpg"
```

In this example, each file name is prepended with the userId of the owner, and someone can be granted access to the files they own via a policy such as:

```
permit (
    principal == User::"5fb883fb-229c-48bc-b186-e7ed9074b536",
    action,
    resource == File::"user:32432423/*"
);
```
### Security Considerations
As alluded to earlier in this post, embedding system information like this into identifiers is against the [Cedar Security guidance]( https://docs.cedarpolicy.com/security.html#security-best-practices-for-applications-using-cedar). Here's an extract from the guidance:

> Use unique, immutable, and non-recyclable IDs for entity identifiers….

There are a class of authorization vulnerabilities that arise from not following this guidance. One of the impacts occurs when an identifier is recycled. For example, if access is granted to a customer named `User::"alice"`, and then Alice departs and a new customer acquires the same username `User::"alice"`, then the new user gets access to everything granted by policies that still reference `User::"alice"`.

Another subtle vulnerability occurs when identifiers require normalization. For example, consider the identifier we observed previously:

```
File::"/Org:923902/Department:4992/Folder:MonthlyReports/January2023.pdf"
```

What would be the impact if the filesystem was case-insensitive?

```
// All these identifiers resolve to the same resource
File::"/Org:923902/Department:4992/Folder:monthlyreports/january2023.pdf"
File::"/Org:923902/Department:4992/Folder:MonthlyReports/January2023.pdf"
File::"/Org:923902/Department:4992/Folder:MONTHLYREPORTS/JANUARY2023.pdf"
```

Imagine if the file was retrievable via an HTTP request to a web application:
```
GET https://example.com/Org_923902/Department_4992/Folder_MONTHLYREPORTS/JANUARY2023.pdf
```

If the application forgot to normalize the filename it received in the HTTP request prior to authorization, then the following rule could be bypassed by the caller simply by using different capitalization:

```
// UH-OH: This forbid rule won't work when is_authorized()
// is invoked with an uppercase version of the filename.
forbid ()
    principal == User::"5fb883fb-229c-48bc-b186-e7ed9074b536",
    action,
    resource == File::"/Org:923902/Department:4992/Folder:MonthlyReports/January2023.pdf"
);
```

Other side-effects of poorly-formed identifiers include the following nuisances:

| Id Style | Impact |
|---------|----------------|
| `User::"alice"` | Alice can't change her username without re-creating all policies |
| `File::"user:12354323:image.jpg"` | File can't be assigned to a new owner without re-creating all policies |
| `File::"folder:VacationPhotos/image123.jpg"` | File can't be moved to a new Folder without re-creating all policies |

In summary, prefer to use synthetic identifiers (such as UUID) whenever feasible. If this security guidance is followed, then wildcards in the Entity Id provide little value. Cedar offers alternative, more robust mechanisms to deliver the same capabilities that the wildcards were providing. For example, Cedar has built-in support for [hierarchical relationships](https://docs.cedarpolicy.com/terminology.html#groups-and-hierarchies), so these don't need to be embedded in the identifier. Cedar also has built-in support for [attribute-based access controls](https://docs.cedarpolicy.com/terminology.html), so concepts such as "owner" don't need to be embedded in the identifiers.

### But, what if I still need wildcards? 

Despite the security guidance, there may be situations when non-synthetic identifiers are unavoidable. This could occur, for example, if identifiers are constructed by an external system outside the control of the application owner. If security risks are acknowledged and mitigated, and the situation is unavoidable, then there can be a legitimate need to perform wildcard matching on the identifier.

Cedar allows wildcard matching in the policy conditions: 

```
// Valid Syntax
permit (
    principal == User::"5fb883fb-229c-48bc-b186-e7ed9074b536",
    action,
    resource
) when {
    resource.id like "*part1*part2*" // <-- Wildcards are allowed in policy conditions
};
```

But, Cedar blocks this in the policy head:

```
// ERROR. Invalid Syntax
permit (
    principal == User::"5fb883fb-229c-48bc-b186-e7ed9074b536",
    action,
    resource == File::*part1*part2*" // <-- Wildcards prohibited in policy head
);
```

The real question is therefore: why does Cedar prohibit wildcards in the policy head, but allow them in the conditions?  The reasoning rests on understanding why the policy head is special, and the tradeoffs of using wildcards.

The policy head defines where a policy is "attached", meaning whether it applies to a single resource, a predefined group of resources, or perhaps any resource where certain conditions are met. The more narrowly scoped the policy head can be, the more efficient a system can perform in terms of policy storage, retrieval, and evaluation. This is an important consideration as the number of policies in a system increases. 

When policies are "attached" to a specific resource or resource-group, this also enables different types of authorization queries such as "list all the resources a principal can access".

Using wildcards in an entity identifier is equivalent to saying "I don't know exactly which resource this will apply to, but it should match any resource where the identifier value satisfies a set of conditions". This is attribute-based access control (ABAC), which must be represented in the Cedar conditions. The reasoning for this is [described by my teammate Emina Torlak in her talk](https://www.youtube.com/watch?v=k6pPcnLuOXY&t=2325s).

### Performance and Scaling Considerations
Something to be aware of when using lots and lots of ABAC policies modeled in this way, where a resource is unspecified in the policy head, is that you may see performance linearly slow as the number of policies increases. This is because there is no shortcut to ignore the policy. Each one must be evaluated, because each policy could potentially match any resource; we don't know for sure until the conditions are evaluated.

Another consideration is that hosted storage solutions will often place limits on the total amount of policies that can be stored for an unspecified value of resource. For example, at the time of this writing, Amazon Verified Permissions will allow a maximum of [200,000 bytes]( https://docs.aws.amazon.com/verifiedpermissions/latest/userguide/quotas.html). 

Lastly, by not specifying a specific resource identifier in the policy, it is no longer feasible to answer the question "What resources can a principal access?" by examining the policies, because the information isn't in the policies. The best that Cedar could answer is the conditions under which a resource is accessible.

All of these tradeoffs are inherent to any ABAC approach. ABAC is a powerful and useful tool, though it has impacts worth understanding. 

Therefore, the reason Cedar only allows wildcards in the conditions is to make it clear there is no magic. Wildcards result in an ABAC-style policy with all the same tradeoffs and considerations as any other ABAC policy. 

In summary, blocking wildcards in the policy head is Cedar's way of nudging end-users toward best practices that result in the best security, the best scaling, and the most features, while at the same time allowing an escape-valve via ABAC conditions for the less common situations that require it.

### Related Issues

The solution described above requires application owners to manually add an attribute on the principal/resource named "id" (or whatever you want to call it) and then fill in the value with the identifier. To make this easier and more syntactically pleasing, there is currently an [open GitHub issue]( https://github.com/cedar-policy/cedar/issues/119) proposing a syntax that "just works" without extra work by the application owner:

```
permit (
    principal == User::"5fb883fb-229c-48bc-b186-e7ed9074b536",
    action,
    resource
) when {
    resource like File::"image*.jpg"
};
```
At the time of this writing, this issue is still under discussion. 
