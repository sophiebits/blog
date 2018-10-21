---
layout: post
title: "Type errors with inference need stacks"
---

I've been thinking about Flow errors recently. They're often inscrutable, and this is a problem that comes up in any typechecking engine with cross-function inference.

<!-- more -->

Flow's error format is a little strange and takes newcomers a while to get used to. Once you get used to it it fades into the background.

Once you know the general layout, I repeatedly run into errors that are very confusing and don't point to the real source of the error whenever there is any indirection (which there always is in practice). Flow tells me that two types aren't compatible, but it consistently does a bad job of telling me *why* it thinks two types should be equal.

Let's say I have this program.

{% highlight js %}
/* @flow */

type NumberRecord = {id: number, name: string};
type StringRecord = {id: string, name: string};

// Let's say my actual error is that `records` here was meant to be
// Array<StringRecord> but I accidentally wrote NumberRecord instead.
var recordList: Array<NumberRecord> = JSON.parse('[]');
processRecords(recordList);

function processRecords(records) {
  const id = first(records).id;
  processStringID(id);
}

function processStringID(id: string) {
}

function first<T>(arr: Array<T>): T {
  return arr[0];
}
{% endhighlight %}

This doesn't typecheck because we're passing numberRecord.id (a number) to processStringID. Flow's output is:

```
13:   processStringID(id);
                      ^ Cannot call `processStringID` with `id` bound to `id`
                        because number [1] is incompatible with string [2].
References:
3: type NumberRecord = {id: number, name: string};
                            ^ [1]
16: function processStringID(id: string) {
                                 ^ [2]
```

So I know that Flow thinks `id` should be a number, but I don't know why it thinks that. In this small file, I can quickly trace back to see where it came from. But Flow makes me do all the work here to figure this out. It doesn't tell me that it thinks `first(records)` is a StringRecord. It doesn't tell me that that type comes from records being an Array<StringRecord>.

I wish it told me something like:

```
I believe:
`first(records).id` on foo.js:12 is type string, because
`first(records)` on foo.js:12 is type StringRecord, because
parameter `records` on foo.js:11 is type Array<StringRecord>, because
`recordList` on foo.js:8 is type Array<StringRecord> with an explicit annotation.
```

Then I can look and say, oh, foo.js:8 should use NumberRecord instead. Now, I know that this is a simple example and in reality types can come from multiple sources and many types aren't concisely serializable, so I understand if this exact format isn't possible. But this is the information I want.

I think adding this "stack" is crucial to people understanding errors. We added "component stacks" to React to allow people to see the parent component and it's invaluable to debugging issues more quickly; offering the deepest component/type name (what we did before and essentially what Flow is doing) is only useful if you have no abstraction in your code.

It's tempting to say that we should instead have Flow "guess" at what frame is relevant, or that we should force developers to annotate more so that you can't have long stacks.

Even if you required people to annotate function boundaries within a file (something that IMO is a little onerous), you can still have a complex single function where the logic is hard to trace through. One of the main reasons I like Flow is that I don't need to annotate very much.

It is not helpful to limit the number of locations shown.

Maybe in a magic world Flow could guess that `processRecords(recordList)` is the missing piece and tell me. It's true that that is one of the most relevant "stack frames" here, but there could easily be more than one. What if there is more indirection and there is another function between processRecords and processStringID? (There are always more layers.)

If the stack of Flow's "thought process" for an error is too long, I can add an annotation in the middle to cut it in half. That's a better way for me to support Flow instead of annotating literally all of my code. Flow can say "here's the path of 20 steps I took to get here; if you add an annotation to one of the middle functions it will make this error message simpler to understand".

Think of if JS errors never gave you a call stack, just the location of the code where the error is thrown. Then suppose you have

```
16: function foo(x) {
17:   return x.foo;
18: }
```

and you get "Cannot read property 'foo' from null" on line 17. That error is more useful than if there was no code location at all, but what I really need to know is which *callsite* of `foo` passed in null. It's common to have to go several levels up the call stack to find the function that is actually responsible for the bad data.

Sure, maybe in many cases you don't need the full output, but you often do. It's common for me to waste many minutes trying to figure out why Flow thinks two types should match; it's easy to be stuck even longer if you're not as familiar with Flow. A friend of mine recently added Flow to a 1400-line library, and she wasted many hours on confusing error messages and was tempted to abandon the effort.

In my example, it is easy to trace back because each function is called in only one place. Imagine if it were called in 5 places. Or 20. That's closer to the reality of complex apps. If you really had to, I would accept if Flow offered a `--why` flag that (quickly) gave me this information in a readable format, but I think hiding it by default is strictly worse.

If Flow doesn't show me the full trace, or it shows me one frame which it "guesses" is responsible (often wrongly, because it's impossible to know what my intent was without "understanding" the variable names, etc), then I am left tracing through my entire codebase to figure out how processStringID is getting called with a number, maybe one of hundreds of different paths to that function. But I shouldn't have to do that. Flow already knows what path it is! So why can't it tell me?
