## Why do Cedar policies end with a semi-colon?

At one time or another, we all forget to include the semi-colon at the end of a policy statement. This can lead to the question: *why is it there?*

```
// Sample Policy
permit (principal, action, resource)
when {
    principal == resource.owner
};
```

The semi-colon is not needed for parsing. In fact, early prototypes of Cedar didn't include it. The reason it's there is for safety.  Consider what would happen if the example above was accidentally truncated after the first line and a semi-colon wasn't required.

```
permit( principal, action, resource)
```

In such a scenario, this would be a valid statement. It allows *"any principal"* to perform *"any action"* on *"any resource"*. In other words, the system has failed-open because of a truncation mistake. The semi-colon protects against truncation errors.

Now, you might say this feature needn't be baked into the Cedar grammar. After all, it could be the responsibility of everyone who is storing or transmitting policies to sign them or include a checksum, and to enforce that upon receipt to detect accidental truncation. For anyone concerned about data corruption, those remain helpful practices for any data. But, we wanted to ensure Cedar was safe by default for all audiences. And, it's not hard to imagine situations where a checksum wouldn't help at all, such as a bad copy-and-paste where the end-user forgot to highlight the entire policy and only copied a subset into a UI. The little semi-colon is the safety net to catch such mistakes.
