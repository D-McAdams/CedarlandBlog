# A Quick Guide To Partial Evaluation 

_I am pleased to share this guest post by Aaron Eline, my teammate and one of the implementors of Cedar partial evaluation. If you'd like to chat further about any of the features described in this post, join us in the [Cedar Slack channel](https://communityinviter.com/apps/cedar-policy/cedar-policy-language). -Darin_

## Enabling partial evaluation

Partial evaluation is released as an “experimental” feature. This means two things

1. We don’t have a formal model or proof of correctness 
2. We expect the API to change somewhat as customers experiment with the feature. (This means we welcome feedback!)

To enable partial evaluation, let’s start a new cargo project with the following `Cargo.toml`

```toml
[package]
name = "pe_example"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
cedar-policy = { features = ["partial-eval"], version = "3.0" }
```

This enables the `partial-eval` experimental feature.

## Using partial evaluation

Partial evaluation exposes one key new function on the `Authorizer` struct: `is_authorized_partial`. ( [source](https://github.com/cedar-policy/cedar/blob/4c8c0c42b1c28b05d63bd9298ee0f7ec935abfec/cedar-policy/src/api.rs#L613)) This takes the same arguments as `is_authorized`, but has a different return type. Where `is_authorized` is guaranteed to always return either `Allow` or `Deny`, `is_authorized_partial` can return `Allow`, `Deny`, or a “residual”: the set of policies it was unable to evaluate. 

## A quick demo

Let’s create a quick demo application that reads a policy set and context off of disk and attempts to authorize:

```rust
use cedar_policy::{PolicySet, Authorizer, Request, Context, Entities};
use std::io::prelude::*;

fn main() {

    let policies_src = std::fs::read_to_string("./policies.cedar").unwrap();
    let policies : PolicySet = policies_src.parse().unwrap();

    let f = std::fs::File::open("./context.json").unwrap();
    let context = Context::from_json_file(f, None).unwrap();

    let auth = Authorizer::new();
    let r = Request::new(Some(r#"User::"Alice""#.parse().unwrap()), 
                         Some(r#"Action::"View""#.parse().unwrap()),
                         Some(r#"Box::"A""#.parse().unwrap()), 
                         context,
                         None).unwrap();
    let answer = auth.is_authorized(&r, &policies, &Entities::empty());
    println!("{:?}", answer);
}
```

Let’s try it with a simple `policies.cedar` :

```
permit(principal,action,resource);
```

and the empty context in `context.json`:

```
{}
```

Running it, we see:

```shell
aeline@88665a58d3ad example % cargo r
   Compiling example v0.1.0 (/Users/aeline/src/example)
    Finished dev [unoptimized + debuginfo] target(s) in 0.54s
     Running `target/debug/example`
Response { decision: Allow, 
           diagnostics: Diagnostics { reason: {PolicyId(PolicyID("policy0"))}, 
           errors: {} } }
```

We get `Allow`, as expected, since the policy should match anything. Let’s try now by changing the line with `is_authorized` to use `is_authorized_partial` :

```
    let answer = auth.is_authorized_partial(&r, &policies, &Entities::empty());
```

Running it:

```shell
aeline@88665a58d3ad example % cargo r
   Compiling example v0.1.0 (/Users/aeline/src/example)
    Finished dev [unoptimized + debuginfo] target(s) in 0.56s
     Running `target/debug/example`
Concrete(Response { decision: Allow, 
                    diagnostics: Diagnostics { reason: {PolicyId(PolicyID("policy0"))},
                    errors: {} } })
```

Now we get a slightly different answer, saying the evaluator was able to produce concrete answer, and that answer was `Allow` . There’s nothing in our policy or context that would prevent a full and concrete answer. 

### Our First Residual

Let’s try a new `policies.cedar`:

```
forbid(principal, action, resource) unless {
    context.secure
};

permit(principal == User::"Alice", action, resource) when {
    context.location == "Baltimore"
};
```

and `context.json` :

```json
{
    "secure" : true,
    "location" : { "__extn" : {
        "fn" : "unknown",
        "arg" : "location"
    }}
}
```

Here, we set the value of the `location` field to a call to the `extension` function `unknown`. `unknown` values are “holes” that the partial evaluator cannot further reduce, and ultimately can lead to evaluation not producing a concrete answer.

Let’s try evaluating our request with these new inputs:

```shell
aeline@88665a58d3ad example % cargo r
    Finished dev [unoptimized + debuginfo] target(s) in 0.15s
     Running `target/debug/example`
Residual(ResidualResponse { 
  residuals: PolicySet { 
    ast: PolicySet { 
      templates: { PolicyID("policy1"): 
      Template { 
        body: TemplateBody { 
          id: PolicyID("policy1"),
          annotations: {}, 
          effect: Permit, 
          principal_constraint: PrincipalConstraint { 
            constraint: Any 
          },          ....
```

The debug printer displays loads of unecessary information in a a hard to read format. Let’s change our code to use the pretty printer, rather than the debug printer:

```rust
match auth.is_authorized_partial(&r, &policies, &Entities::empty()) {
  PartialResponse::Concrete(r) => println!("{:?}", r),
  PartialResponse::Residual(r) => {
    println!("Residuals:");
    for policy in r.residuals().policies() {
      println!("{policy}");
    }
  }
}
```

Re-running with this new printer gives us: 

```
Residuals:
permit(
  principal,
  action,
  resource
) when {
  true && (unknown(location) == "Baltimore")
};
```

We can see that the `forbid` policy has dropped entirely, as `context.secure` was fully known so we know the `forbid` policy doesn’t apply. On the `permit` policy, the head constraints have been simplified away (as `principal` was in fact `User::"alice"`). What remains is the constraint that `context.location` has to equal `"Baltimore"`, plus a residual `true &&` expression which is explained further in [Caveats](#caveats).

If we change our `context.json` to set `context.secure` to be `false`, and re-run, we get:

```shell
dev-dsk-aeline-1d-f2264f25 % cargo r
    Finished dev [unoptimized + debuginfo] target(s) in 0.08s
     Running `target/debug/pe_example`
Response { decision: Deny, 
           diagnostics: Diagnostics { reason: {PolicyId(PolicyID("policy0"))}, 
           errors: [] } }
```

Now we get `Deny`. Even though the value of `context.location` was unknown, it wasn’t needed to compute the final result, as the `forbid` policy trumps the `permit`. 

## A simple application

Let’s use a simple document application as a demo for how one could use Partial Evaluation in a real application. 
In this application there will be two kinds of entities: `User`s and `Document`s. Every document has two attributes: `isPublic` and `owner`. There are two actions: `View` and `Delete`. Requests pass a `context` with two values: the request’s source IP address, `src_ip`, and whether or not the request was authenticated with multiple factors, `mfa_authed`. Here are our policies for this application:

```
// Users can access public documents
permit (
    principal,
    action == DocCloud::Action::"View",
    resource
) when { 
    resource.isPublic
};

// Users can access owned documents if they are mfa-authenticated
permit (
    principal,
    action == DocCloud::Action::"View",
    resource
) when {
    context.mfa_authed && 
    resource.owner == principal 
};

// Users can only delete documents they own, and they both come from the company network and are mfa-authenticated
permit (
    principal,
    action == DocCloud::Action::"Delete",
    resource
) when {
    context.src_ip.isInRange(ip("1.1.1.0/24")) &&
    context.mfa_authed &&
    resource.owner == principal
};
```

### What can Alice access?

Concrete evaluation works for answering the question “Can Alice access Document ABC?”. But it doesn’t work well for answering the question “What documents can Alice access?”. Partial evaluation can help us answer this question by extracting the residual policies around an unknown `resource` that would evaluate to `Allow`, ignoring/folding away constraints that don’t matter or are common to all resources.

We’ll change our simple `main` to the following:

```rust
fn main() {

    let policies_src = std::fs::read_to_string("./policies.cedar").unwrap();
    let policies: PolicySet = policies_src.parse().unwrap();
    

    let f = std::fs::File::open("./context.json").unwrap();
    let context = Context::from_json_file(f, None).unwrap();

    let auth = Authorizer::new();

    let r = RequestBuilder::default()
        .principal(Some(r#"DocCloud::User::"Alice""#.parse().unwrap()))
        .action(Some(r#"DocCloud::Action::"View""#.parse().unwrap()))
        .context(context)
        .build()
        .unwrap();

    match auth.is_authorized_partial(&r, &policies, &Entities::empty()) {
        PartialResponse::Concrete(r) => println!("{:?}", r),
        PartialResponse::Residual(r) => {
            println!("Residuals:");
            for policy in r.residuals().policies() {
                println!("{policy}");
            }
        }
    }
}
```

This is mostly the same as last time, but with one important difference: We’ve dropped `.resource()` from our `RequestBuilder`. This sets `resource` to be an `unknown` value. 

Running this with the following `context.json`:

```json
{
        "mfa_authed" : true,
        "src_ip" : { "__extn" : {
          "fn" : "ip",
          "arg" : "1.1.1.0/24"
        }
}
```

Yields:

```
dev-dsk-aeline-1d-f2264f25 % cargo r
    Finished dev [unoptimized + debuginfo] target(s) in 0.08s
     Running `target/debug/pe_example`
Residuals:

permit(
  principal,
  action,
  resource
) when {
  true && (true && ((unknown(resource)["owner"]) == DocCloud::User::"Alice"))
};


permit(
  principal,
  action,
  resource
) when {
  true && (unknown(resource)["isPublic"])
};
```

Her are some interesting things about this residual set:

1. The policy governing `Delete` is gone. It evaluates to `false` and so is dropped from the residual.
2. The constraint that `action` must equal `View` is gone from the two remaining policies, since it is always `true` for this request. 
3. The constraint on `context.mfa_authed` is also gone since it is also `true`, always.

Let’s try a few more scenarios. First we’ll change our `context.json` to: 

```json
{
        "mfa_authed" : false,
        "src_ip" : { "__extn" : {
          "fn" : "ip",
          "arg" : "1.1.1.0/24"
        }
}
```

Re-running: 

```
dev-dsk-aeline-1d-f2264f25 % cargo r
    Finished dev [unoptimized + debuginfo] target(s) in 0.08s
     Running `target/debug/pe_example`
Residuals:
permit(
  principal,
  action,
  resource
) when {
  true && (unknown(resource)["isPublic"])
};
```

Now we only get the one residual policy. No matter who owns a document, principals without MFA can view only public documents.

Let’s now try changing our `Action` in the request (defined in `main.rs`) to `DocCloud::Action::"Delete"`: 

```shell
dev-dsk-aeline-1d-f2264f25 % cargo r
    Finished dev [unoptimized + debuginfo] target(s) in 0.08s
     Running `target/debug/pe_example`
Response { decision: Deny, diagnostics: Diagnostics { reason: {}, errors: [] } }
```

Now we simply get `Deny`! Even with the `resource` fully unknown, there is no way for any rule involving `Delete` to ever evaluate to true given our `context` .

### What to do with the residuals?

What now? Our original goal was to answer the question: Which resources does Alice have access to? We have shown the simplified policies that would grant access to Alice, but we are not all the way there. The residual policies are essentially constraints that represent Alice’s set of viewable documents. Any document that meets those constraints, i.e., one in which replacing `unknown(resource)` with the document entity ID causes a residual policy to evaluate to true, is included in the set. What you would like to do is use these constraints to efficiently fetch the data that satisfies them. Since Cedar doesn’t have any way of knowing how your specific application stores data, it can’t do this on its own. 

Fortunately, the Cedar language is pretty small, so it’s easy to convert it into another language that can be used to do that. Let’s imagine that for this sample application, our document metadata is stored in a relational database. We could translate our remaining Cedar residual into a SQL query we can submit to the database. If our metadata is in a table called `documents`, with the columns `id`, `is_public`, and `owner` , then we can translate our first residual into the following query:

```sql
SELECT id FROM documents WHERE 
(true AND true AND owner == 'DocCloud::User::"Alice"') 
|| 
(true AND is_public);
```

And our second residual would become:

```sql
SELECT id FROM documents WHERE (true AND is_public);
```

These queries could then be submitted to a database in order to retrieve the set of documents that Alice is authorized to access. Importantly, partial evaluation has already evaluated all of the authorization info that the database is unable to answer, such as whether or not the request was authenticated with MFA.

## Caveats

One missing feature of partial evaluation is being able to make a guarantee to the partial evaluator that an unknown will be filled with a value of a certain type. This why the residuals above contain items like `true && unknown(resource)["isPublic"]` . The partial evaluator can’t guarantee that `resource.isPublic` is always a boolean, so it has to leave the `&&` un-evaluated to preserve the error behavior if `isPublic` is a non-boolean. It would be ideal if `true && unknown` would partially evaluate to `unknown`. Expressions could be more aggressively constant-folded down to values if more type information was known. This remains an area for future work.
