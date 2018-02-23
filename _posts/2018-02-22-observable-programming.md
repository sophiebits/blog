---
layout: post
title: "Observable programming"
---

Excel is unusually good at allowing you to build complex programs while allowing you to see the values of every intermediate computation.

<!-- more -->

As a result, you often find people who claim to struggle with the abstract thinking that most other programming requires but who construct elaborate Excel workbooks with ease. However, Excel struggles to make the formulas themselves easily observable. (Ever tried to audit the correctness of a complex Excel sheet?) Excel formulas also fail at supporting abstraction well.

In contrast, traditional programming environments make the code you write much clearer. This property is what allows code review to be useful. But they’re usually terrible at allowing you to observe the behavior of your program when subjected to concrete values.

Our best tools are usually manual log statements and debuggers where you can step through code in one direction but which don’t facilitate the same type of observability that Excel produces without trying.

How can we build tools that give us the best of both worlds? Will they be inspired by rr (which lets you step backwards in time) and Prepack (which evaluates many branches at once) or will they be entirely different?

At work we have an interesting framework for handling data privacy. An privacy policy, which is conceptually a single function that returns a boolean value (allow or deny), is written not in conventional code but in a DSL that is an array of `Allow*()` or `Deny*()` rules.

```
// (not our actual privacy rules)
canLoadPostRules = [
  new AllowIfViewerIsAuthor(),
  new DenyIfAuthorIsNotVisible(),
  new DenyIfUnpublished(),
  new AlwaysAllow(),
]
```

This seemed clumsy to me at first, but two interesting tools have come out of it. First, we are able to use production profiling data to intelligently and automatically reorder the checks for better efficiency without changing the result (ex: if the example's second Deny check is faster, it can be checked first). Second, we have an internal “privacy debugger” tool that lets you run a privacy policy against a particular input and see exactly how it comes to a decision, including how each subpolicy was processed. The output is rather similar to what Excel shows for every spreadsheet. The tool is specific to this rule DSL, but maybe it doesn’t have to be.

What would it look like if you could run it on any function in a codebase or use it to instrument an entire web request, seeing a complete execution trace of all the executed code?
